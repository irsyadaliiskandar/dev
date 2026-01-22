# **Sistem Taxi dengan Fare $0.1 per Meter**

## **1. Taxi_System_Advanced.pwn**
```pawn
#include <a_samp>
#include <YSI\y_timers>
#include <YSI\y_iterate>
#include <YSI\y_ini>
#include <streamer>

#define TAXI_CALL_NUMBER 933
#define FARE_PER_METER 0.1 // $0.1 per meter
#define BASE_FARE 5 // Base fare $5
#define MAX_TAXI_ORDERS 50

#define COLOR_TAXI 0xFFD700AA
#define COLOR_ORDER 0x00FF00AA
#define COLOR_ERROR 0xFF0000AA
#define COLOR_INFO 0x00C8FFAA
#define COLOR_SUCCESS 0x00FF00AA

enum E_TAXI_ORDER {
    toID,
    toCustomerID,
    toDriverID,
    Float:toPickupX,
    Float:toPickupY,
    Float:toPickupZ,
    Float:toDropX,
    Float:toDropY,
    Float:toDropZ,
    toPickupInterior,
    toPickupVW,
    Float:toDistance,
    toFare,
    toStatus, // 0=Waiting, 1=Accepted, 2=PickedUp, 3=Completed, 4=Cancelled
    toTimer,
    toPickupObject,
    Text3D:toPickupLabel,
    toPickupCP,
    toDropCP,
    toMapIcon,
    bool:toExists
};

enum E_TAXI_DRIVER {
    tdPlayerID,
    tdOnDuty,
    tdCurrentOrder,
    tdVehicleID,
    tdTotalEarnings,
    tdRating,
    tdRatingCount,
    tdFareTimer,
    tdTotalFare,
    Float:tdLastX,
    Float:tdLastY,
    Float:tdLastZ
};

new TaxiOrders[MAX_TAXI_ORDERS][E_TAXI_ORDER];
new TaxiDriver[MAX_PLAYERS][E_TAXI_DRIVER];
new Iterator:ActiveOrders<MAX_TAXI_ORDERS>;
new Iterator:AvailableDrivers<MAX_PLAYERS>;

// TextDraws
new PlayerText:FareText[MAX_PLAYERS];
new PlayerText:DistanceText[MAX_PLAYERS];

// Variables
new PlayerSelectingDest[MAX_PLAYERS];
new Float:PlayerDestPos[MAX_PLAYERS][3];

public OnGameModeInit() {
    print("=======================================");
    print(" Advanced Taxi System v3.0 Loaded");
    print(" Fare: $0.1 per meter + $5 base");
    print("=======================================");
    
    Iter_Init(ActiveOrders);
    Iter_Init(AvailableDrivers);
    
    return 1;
}

public OnPlayerConnect(playerid) {
    // Reset taxi data
    TaxiDriver[playerid][tdPlayerID] = playerid;
    TaxiDriver[playerid][tdOnDuty] = 0;
    TaxiDriver[playerid][tdCurrentOrder] = -1;
    TaxiDriver[playerid][tdVehicleID] = INVALID_VEHICLE_ID;
    TaxiDriver[playerid][tdTotalEarnings] = 0;
    TaxiDriver[playerid][tdRating] = 0;
    TaxiDriver[playerid][tdRatingCount] = 0;
    TaxiDriver[playerid][tdFareTimer] = -1;
    TaxiDriver[playerid][tdTotalFare] = BASE_FARE;
    
    // Create textdraws
    FareText[playerid] = CreatePlayerTextDraw(playerid, 320.0, 350.0, "Fare: $5.00");
    PlayerTextDrawAlignment(playerid, FareText[playerid], 2);
    PlayerTextDrawFont(playerid, FareText[playerid], 2);
    PlayerTextDrawLetterSize(playerid, FareText[playerid], 0.4, 1.6);
    PlayerTextDrawColor(playerid, FareText[playerid], 0x00FF00FF);
    PlayerTextDrawSetOutline(playerid, FareText[playerid], 1);
    PlayerTextDrawSetShadow(playerid, FareText[playerid], 1);
    
    DistanceText[playerid] = CreatePlayerTextDraw(playerid, 320.0, 370.0, "Distance: 0.0 km");
    PlayerTextDrawAlignment(playerid, DistanceText[playerid], 2);
    PlayerTextDrawFont(playerid, DistanceText[playerid], 2);
    PlayerTextDrawLetterSize(playerid, DistanceText[playerid], 0.3, 1.3);
    PlayerTextDrawColor(playerid, DistanceText[playerid], 0x00C8FFFF);
    PlayerTextDrawSetOutline(playerid, DistanceText[playerid], 1);
    PlayerTextDrawSetShadow(playerid, DistanceText[playerid], 1);
    
    PlayerSelectingDest[playerid] = 0;
    
    return 1;
}

public OnPlayerDisconnect(playerid, reason) {
    // Jika driver on duty, auto off duty
    if(TaxiDriver[playerid][tdOnDuty]) {
        ForceDriverOffDuty(playerid, "Disconnected");
    }
    
    // Jika customer punya order aktif, cancel
    CancelPlayerOrder(playerid);
    
    // Hapus textdraws
    PlayerTextDrawDestroy(playerid, FareText[playerid]);
    PlayerTextDrawDestroy(playerid, DistanceText[playerid]);
    
    return 1;
}

// ==================== PLAYER COMMANDS ====================

CMD:call(playerid, params[]) {
    new number;
    if(sscanf(params, "d", number)) {
        SendClientMessage(playerid, COLOR_INFO, "Gunakan: /call [nomor]");
        SendClientMessage(playerid, COLOR_INFO, "Nomor taxi: {FFD700}933");
        return 1;
    }
    
    if(number != TAXI_CALL_NUMBER) {
        return Error(playerid, "Nomor tidak valid!");
    }
    
    if(Iter_Count(AvailableDrivers) == 0) {
        return Error(playerid, "Tidak ada taxi driver yang tersedia saat ini!");
    }
    
    // Cek apakah sudah punya order aktif
    if(GetPlayerOrderID(playerid) != -1) {
        return Error(playerid, "Anda sudah memiliki order taxi aktif!");
    }
    
    // Mulai proses order
    StartTaxiOrderProcess(playerid);
    return 1;
}

CMD:taxiduty(playerid, params[]) {
    if(pData[playerid][pJob] != 1 && pData[playerid][pJob2] != 1) {
        return Error(playerid, "Anda bukan pekerja taxi!");
    }
    
    if(!IsPlayerInAnyVehicle(playerid)) {
        return Error(playerid, "Anda harus berada di dalam taxi!");
    }
    
    new vehicleid = GetPlayerVehicleID(playerid);
    new modelid = GetVehicleModel(vehicleid);
    
    // Cek apakah kendaraan taxi
    if(modelid != 438 && modelid != 420 && modelid != 560) {
        return Error(playerid, "Anda harus berada di kendaraan taxi!");
    }
    
    if(GetPlayerState(playerid) != PLAYER_STATE_DRIVER) {
        return Error(playerid, "Anda harus menjadi driver!");
    }
    
    if(TaxiDriver[playerid][tdOnDuty] == 0) {
        // ON DUTY
        TaxiDriver[playerid][tdOnDuty] = 1;
        TaxiDriver[playerid][tdVehicleID] = vehicleid;
        TaxiDriver[playerid][tdCurrentOrder] = -1;
        
        SetPlayerColor(playerid, COLOR_TAXI);
        SetPlayerMarkerForPlayer(playerid, playerid, COLOR_TAXI);
        
        GetPlayerPos(playerid, TaxiDriver[playerid][tdLastX], TaxiDriver[playerid][tdLastY], TaxiDriver[playerid][tdLastZ]);
        
        // Tambah ke driver available
        Iter_Add(AvailableDrivers, playerid);
        
        SendClientMessageToAllEx(COLOR_TAXI, 
            "[TAXI] %s sedang ON DUTY! {FFFFFF}Gunakan {FFD700}/call %d", 
            ReturnName(playerid), TAXI_CALL_NUMBER);
            
        Info(playerid, "Anda sekarang ON DUTY sebagai taxi driver!");
        Info(playerid, "Keluar dari taxi akan membuat Anda OFF DUTY otomatis!");
        
        // Start timer untuk update position
        SetTimerEx("UpdateDriverPosition", 5000, true, "i", playerid);
        
    } else {
        // OFF DUTY
        ForceDriverOffDuty(playerid, "Manual off duty");
    }
    
    return 1;
}

CMD:myorder(playerid, params[]) {
    new orderid = GetPlayerOrderID(playerid);
    if(orderid == -1) {
        return Error(playerid, "Anda tidak memiliki order taxi aktif!");
    }
    
    ShowOrderInfo(playerid, orderid);
    return 1;
}

CMD:cancelorder(playerid, params[]) {
    new orderid = GetPlayerOrderID(playerid);
    if(orderid == -1) {
        // Cek jika driver
        if(TaxiDriver[playerid][tdOnDuty] && TaxiDriver[playerid][tdCurrentOrder] != -1) {
            orderid = TaxiDriver[playerid][tdCurrentOrder];
        } else {
            return Error(playerid, "Anda tidak memiliki order aktif!");
        }
    }
    
    new reason[64];
    if(sscanf(params, "s[64]", reason)) reason = "Dibatalkan oleh pemesan";
    
    CancelTaxiOrder(orderid, reason, playerid);
    return 1;
}

CMD:acceptorder(playerid, params[]) {
    if(!TaxiDriver[playerid][tdOnDuty]) {
        return Error(playerid, "Anda harus ON DUTY untuk menerima order!");
    }
    
    if(TaxiDriver[playerid][tdCurrentOrder] != -1) {
        return Error(playerid, "Anda sudah memiliki order aktif!");
    }
    
    new orderid;
    if(sscanf(params, "d", orderid)) {
        // Tampilkan list order yang available
        ShowAvailableOrders(playerid);
        return 1;
    }
    
    if(orderid < 0 || orderid >= MAX_TAXI_ORDERS || !TaxiOrders[orderid][toExists]) {
        return Error(playerid, "Order ID tidak valid!");
    }
    
    if(TaxiOrders[orderid][toStatus] != 0) {
        return Error(playerid, "Order ini sudah diambil driver lain!");
    }
    
    // Terima order
    AcceptTaxiOrder(orderid, playerid);
    return 1;
}

CMD:pickup(playerid, params[]) {
    if(!TaxiDriver[playerid][tdOnDuty]) {
        return Error(playerid, "Anda harus ON DUTY!");
    }
    
    if(TaxiDriver[playerid][tdCurrentOrder] == -1) {
        return Error(playerid, "Anda tidak memiliki order aktif!");
    }
    
    new orderid = TaxiDriver[playerid][tdCurrentOrder];
    
    if(TaxiOrders[orderid][toStatus] != 1) {
        return Error(playerid, "Order belum ready untuk pickup!");
    }
    
    // Cek apakah sudah dekat pickup point
    new Float:dist = GetDistanceBetweenPoints(
        TaxiDriver[playerid][tdLastX], TaxiDriver[playerid][tdLastY], TaxiDriver[playerid][tdLastZ],
        TaxiOrders[orderid][toPickupX], TaxiOrders[orderid][toPickupY], TaxiOrders[orderid][toPickupZ]);
    
    if(dist > 10.0) {
        return Error(playerid, "Anda belum sampai di lokasi pickup!");
    }
    
    // Customer masuk ke taxi
    if(!IsPlayerInVehicle(TaxiOrders[orderid][toCustomerID], TaxiDriver[playerid][tdVehicleID])) {
        return Error(playerid, "Customer belum masuk ke taxi Anda!");
    }
    
    // Mulai trip
    StartTaxiTrip(orderid, playerid);
    return 1;
}

CMD:complete(playerid, params[]) {
    if(!TaxiDriver[playerid][tdOnDuty]) {
        return Error(playerid, "Anda harus ON DUTY!");
    }
    
    if(TaxiDriver[playerid][tdCurrentOrder] == -1) {
        return Error(playerid, "Anda tidak memiliki order aktif!");
    }
    
    new orderid = TaxiDriver[playerid][tdCurrentOrder];
    
    if(TaxiOrders[orderid][toStatus] != 2) {
        return Error(playerid, "Order belum dalam perjalanan!");
    }
    
    // Cek apakah sudah sampai tujuan
    new Float:dist = GetDistanceBetweenPoints(
        TaxiDriver[playerid][tdLastX], TaxiDriver[playerid][tdLastY], TaxiDriver[playerid][tdLastZ],
        TaxiOrders[orderid][toDropX], TaxiOrders[orderid][toDropY], TaxiOrders[orderid][toDropZ]);
    
    if(dist > 10.0) {
        return Error(playerid, "Anda belum sampai di lokasi tujuan!");
    }
    
    // Selesaikan trip
    CompleteTaxiTrip(orderid, playerid);
    return 1;
}

// ==================== TAXI ORDER PROCESS ====================

StartTaxiOrderProcess(playerid) {
    SendClientMessage(playerid, COLOR_INFO, 
        "[TAXI] Silakan pilih lokasi tujuan Anda di peta!");
    SendClientMessage(playerid, COLOR_INFO, 
        "Klik lokasi di peta, lalu ketik {FFD700}yes {FFFFFF}untuk konfirmasi.");
    
    // Tampilkan peta fullscreen
    ShowPlayerDialog(playerid, DIALOG_TAXI_MAP, DIALOG_STYLE_MSGBOX,
        "{FFD700}[TAXI] Pilih Lokasi Tujuan",
        "{FFFFFF}Silakan buka peta (TAB) dan pilih lokasi tujuan.\n\
        Setelah memilih, ketik {FFD700}yes {FFFFFF}di chat untuk konfirmasi.",
        "OK", "");
    
    PlayerSelectingDest[playerid] = 1;
    
    // Set timer untuk timeout
    SetTimerEx("TaxiOrderTimeout", 120000, false, "i", playerid);
}

public OnPlayerClickMap(playerid, Float:fX, Float:fY, Float:fZ) {
    if(PlayerSelectingDest[playerid]) {
        PlayerDestPos[playerid][0] = fX;
        PlayerDestPos[playerid][1] = fY;
        PlayerDestPos[playerid][2] = fZ;
        
        // Tampilkan marker di map
        SetPlayerMapIcon(playerid, 50, fX, fY, fZ, 0, 0xFF0000FF, MAPICON_LOCAL);
        
        SendClientMessageEx(playerid, COLOR_SUCCESS,
            "[TAXI] Lokasi tujuan ditandai di peta! ({%.1f}, {%.1f})", fX, fY);
        SendClientMessage(playerid, COLOR_INFO,
            "Ketik {FFD700}yes {FFFFFF}untuk konfirmasi order, atau {FFD700}no {FFFFFF}untuk batal.");
    }
    return 1;
}

public OnPlayerText(playerid, text[]) {
    if(PlayerSelectingDest[playerid]) {
        if(strcmp(text, "yes", true) == 0 || strcmp(text, "y", true) == 0) {
            if(PlayerDestPos[playerid][0] == 0.0 && PlayerDestPos[playerid][1] == 0.0) {
                SendClientMessage(playerid, COLOR_ERROR, "Silakan pilih lokasi di peta terlebih dahulu!");
                return 0;
            }
            
            // Konfirmasi order
            CreateTaxiOrder(playerid);
            PlayerSelectingDest[playerid] = 0;
            RemovePlayerMapIcon(playerid, 50);
            return 0;
            
        } else if(strcmp(text, "no", true) == 0 || strcmp(text, "n", true) == 0) {
            SendClientMessage(playerid, COLOR_ERROR, "Order taxi dibatalkan.");
            PlayerSelectingDest[playerid] = 0;
            RemovePlayerMapIcon(playerid, 50);
            return 0;
        }
    }
    return 1;
}

CreateTaxiOrder(playerid) {
    // Cari slot kosong
    new orderid = Iter_Free(ActiveOrders);
    
    if(orderid == -1) {
        Error(playerid, "Maaf, sistem order penuh. Silakan coba lagi nanti.");
        PlayerSelectingDest[playerid] = 0;
        return -1;
    }
    
    // Dapatkan posisi player sekarang
    new Float:pickupX, Float:pickupY, Float:pickupZ;
    GetPlayerPos(playerid, pickupX, pickupY, pickupZ);
    
    // Hitung jarak dan fare
    new Float:distance = GetDistanceBetweenPoints(pickupX, pickupY, pickupZ, 
        PlayerDestPos[playerid][0], PlayerDestPos[playerid][1], PlayerDestPos[playerid][2]);
    
    new fare = BASE_FARE + floatround(distance * FARE_PER_METER);
    
    // Setup order data
    TaxiOrders[orderid][toID] = orderid;
    TaxiOrders[orderid][toCustomerID] = playerid;
    TaxiOrders[orderid][toDriverID] = INVALID_PLAYER_ID;
    TaxiOrders[orderid][toPickupX] = pickupX;
    TaxiOrders[orderid][toPickupY] = pickupY;
    TaxiOrders[orderid][toPickupZ] = pickupZ;
    TaxiOrders[orderid][toDropX] = PlayerDestPos[playerid][0];
    TaxiOrders[orderid][toDropY] = PlayerDestPos[playerid][1];
    TaxiOrders[orderid][toDropZ] = PlayerDestPos[playerid][2];
    TaxiOrders[orderid][toPickupInterior] = GetPlayerInterior(playerid);
    TaxiOrders[orderid][toPickupVW] = GetPlayerVirtualWorld(playerid);
    TaxiOrders[orderid][toDistance] = distance;
    TaxiOrders[orderid][toFare] = fare;
    TaxiOrders[orderid][toStatus] = 0;
    TaxiOrders[orderid][toExists] = true;
    
    // Buat pickup marker
    CreateOrderMarkers(orderid);
    
    // Tambah ke iterator
    Iter_Add(ActiveOrders, orderid);
    
    // Kirim notifikasi ke semua driver available
    foreach(new i : AvailableDrivers) {
        SendClientMessageEx(i, COLOR_ORDER,
            "[ORDER] #{FFFFFF}%d {00FF00}Baru! {FFFFFF}Jarak: {FFD700}%.1f km {FFFFFF}Fare: {FFD700}$%s",
            orderid, distance/1000, FormatMoney(fare));
        SendClientMessageEx(i, COLOR_INFO,
            "Gunakan {FFD700}/acceptorder %d", orderid);
    }
    
    // Info ke customer
    SendClientMessageEx(playerid, COLOR_SUCCESS,
        "[TAXI] Order berhasil dibuat! ID: {FFD700}#%d", orderid);
    SendClientMessageEx(playerid, COLOR_INFO,
        "Jarak: {FFD700}%.1f km {FFFFFF}| Estimasi Fare: {FFD700}$%s",
        distance/1000, FormatMoney(fare));
    SendClientMessage(playerid, COLOR_INFO,
        "Tunggu driver menerima order Anda...");
    
    // Set timer untuk auto-cancel (10 menit)
    TaxiOrders[orderid][toTimer] = SetTimerEx("AutoCancelOrder", 600000, false, "i", orderid);
    
    // Reset player destination
    PlayerSelectingDest[playerid] = 0;
    
    return orderid;
}

CreateOrderMarkers(orderid) {
    // Create pickup object and label
    TaxiOrders[orderid][toPickupObject] = CreateDynamicObject(19130, 
        TaxiOrders[orderid][toPickupX], 
        TaxiOrders[orderid][toPickupY], 
        TaxiOrders[orderid][toPickupZ] - 0.9,
        0.0, 0.0, 0.0,
        TaxiOrders[orderid][toPickupVW], TaxiOrders[orderid][toPickupInterior]);
    
    new pickupstr[128];
    format(pickupstr, sizeof(pickupstr),
        "[TAXI PICKUP #%d]\n{FFFFFF}Customer: %s\n{FFD700}Fare: $%s\n{FFFFFF}Gunakan /cancelorder",
        orderid, ReturnName(TaxiOrders[orderid][toCustomerID]), 
        FormatMoney(TaxiOrders[orderid][toFare]));
    
    TaxiOrders[orderid][toPickupLabel] = CreateDynamic3DTextLabel(pickupstr, COLOR_ORDER,
        TaxiOrders[orderid][toPickupX], 
        TaxiOrders[orderid][toPickupY], 
        TaxiOrders[orderid][toPickupZ] + 0.5,
        15.0, INVALID_PLAYER_ID, INVALID_VEHICLE_ID, 1,
        TaxiOrders[orderid][toPickupVW], TaxiOrders[orderid][toPickupInterior]);
    
    // Create pickup checkpoint untuk driver nanti
    TaxiOrders[orderid][toPickupCP] = CreateDynamicCP(
        TaxiOrders[orderid][toPickupX], 
        TaxiOrders[orderid][toPickupY], 
        TaxiOrders[orderid][toPickupZ],
        3.0, TaxiOrders[orderid][toPickupVW], 
        TaxiOrders[orderid][toPickupInterior]);
    
    // Create drop checkpoint
    TaxiOrders[orderid][toDropCP] = CreateDynamicCP(
        TaxiOrders[orderid][toDropX], 
        TaxiOrders[orderid][toDropY], 
        TaxiOrders[orderid][toDropZ],
        3.0, -1, -1);
    
    // Create map icon untuk driver
    TaxiOrders[orderid][toMapIcon] = CreateDynamicMapIcon(
        TaxiOrders[orderid][toPickupX], 
        TaxiOrders[orderid][toPickupY], 
        TaxiOrders[orderid][toPickupZ],
        55, 0xFF0000FF, -1, -1,
        TaxiOrders[orderid][toPickupVW], 
        TaxiOrders[orderid][toPickupInterior]);
}

AcceptTaxiOrder(orderid, driverid) {
    if(!TaxiOrders[orderid][toExists] || TaxiOrders[orderid][toStatus] != 0) {
        return 0;
    }
    
    // Update order status
    TaxiOrders[orderid][toStatus] = 1; // Accepted
    TaxiOrders[orderid][toDriverID] = driverid;
    
    // Update driver data
    TaxiDriver[driverid][tdCurrentOrder] = orderid;
    
    // Hapus dari available drivers
    Iter_Remove(AvailableDrivers, driverid);
    
    // Hentikan auto-cancel timer
    KillTimer(TaxiOrders[orderid][toTimer]);
    
    // Update pickup label
    UpdatePickupLabel(orderid);
    
    // Notify customer
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        SendClientMessageEx(TaxiOrders[orderid][toCustomerID], COLOR_SUCCESS,
            "[TAXI] Driver %s menerima order Anda!", ReturnName(driverid));
        SendClientMessageEx(TaxiOrders[orderid][toCustomerID], COLOR_INFO,
            "Plate taxi: {FFD700}%s", 
            GetVehicleNumberPlate(TaxiDriver[driverid][tdVehicleID]));
    }
    
    // Notify driver
    SendClientMessageEx(driverid, COLOR_SUCCESS,
        "[TAXI] Anda menerima order #{FFD700}%d {FFFFFF}dari {FFD700}%s",
        orderid, ReturnName(TaxiOrders[orderid][toCustomerID]));
    SendClientMessageEx(driverid, COLOR_INFO,
        "Jarak: {FFD700}%.1f km {FFFFFF}| Fare: {FFD700}$%s",
        TaxiOrders[orderid][toDistance]/1000, 
        FormatMoney(TaxiOrders[orderid][toFare]));
    
    // Set checkpoint untuk driver
    SetPlayerCheckpoint(driverid, 
        TaxiOrders[orderid][toPickupX], 
        TaxiOrders[orderid][toPickupY], 
        TaxiOrders[orderid][toPickupZ], 3.0);
    
    return 1;
}

StartTaxiTrip(orderid, driverid) {
    if(!TaxiOrders[orderid][toExists] || TaxiOrders[orderid][toStatus] != 1) {
        return 0;
    }
    
    // Update status
    TaxiOrders[orderid][toStatus] = 2; // On trip
    
    // Hapus pickup checkpoint dan object
    DestroyDynamicCP(TaxiOrders[orderid][toPickupCP]);
    DestroyDynamicObject(TaxiOrders[orderid][toPickupObject]);
    DestroyDynamic3DTextLabel(TaxiOrders[orderid][toPickupLabel]);
    DestroyDynamicMapIcon(TaxiOrders[orderid][toMapIcon]);
    
    // Reset fare untuk perjalanan
    TaxiDriver[driverid][tdTotalFare] = BASE_FARE;
    
    // Start fare timer (update setiap detik)
    TaxiDriver[driverid][tdFareTimer] = SetTimerEx("UpdateFareMeter", 1000, true, "ii", driverid, orderid);
    
    // Tampilkan fare meter
    PlayerTextDrawSetString(driverid, FareText[driverid], "Fare: $5.00");
    PlayerTextDrawShow(driverid, FareText[driverid]);
    
    // Juga untuk customer
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        PlayerTextDrawSetString(TaxiOrders[orderid][toCustomerID], 
            FareText[TaxiOrders[orderid][toCustomerID]], "Fare: $5.00");
        PlayerTextDrawShow(TaxiOrders[orderid][toCustomerID], 
            FareText[TaxiOrders[orderid][toCustomerID]]);
        
        // Tampilkan distance text
        new diststr[64];
        format(diststr, sizeof(diststr), "Distance: %.1f km", 
            TaxiOrders[orderid][toDistance]/1000);
        PlayerTextDrawSetString(TaxiOrders[orderid][toCustomerID], 
            DistanceText[TaxiOrders[orderid][toCustomerID]], diststr);
        PlayerTextDrawShow(TaxiOrders[orderid][toCustomerID], 
            DistanceText[TaxiOrders[orderid][toCustomerID]]);
    }
    
    // Set checkpoint tujuan
    SetPlayerCheckpoint(driverid, 
        TaxiOrders[orderid][toDropX], 
        TaxiOrders[orderid][toDropY], 
        TaxiOrders[orderid][toDropZ], 3.0);
    
    // Juga untuk customer
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        SetPlayerCheckpoint(TaxiOrders[orderid][toCustomerID], 
            TaxiOrders[orderid][toDropX], 
            TaxiOrders[orderid][toDropY], 
            TaxiOrders[orderid][toDropZ], 3.0);
    }
    
    // Notify both
    SendClientMessage(driverid, COLOR_INFO, 
        "[TAXI] Perjalanan dimulai! Fare meter aktif.");
    SendClientMessage(TaxiOrders[orderid][toCustomerID], COLOR_INFO, 
        "[TAXI] Perjalanan dimulai! Fare meter aktif.");
    
    return 1;
}

CompleteTaxiTrip(orderid, driverid) {
    if(!TaxiOrders[orderid][toExists] || TaxiOrders[orderid][toStatus] != 2) {
        return 0;
    }
    
    // Hentikan fare timer
    KillTimer(TaxiDriver[driverid][tdFareTimer]);
    TaxiDriver[driverid][tdFareTimer] = -1;
    
    // Dapatkan final fare
    new finalfare = TaxiDriver[driverid][tdTotalFare];
    
    // Cek apakah customer punya cukup uang
    if(GetPlayerMoney(TaxiOrders[orderid][toCustomerID]) < finalfare) {
        // Coba potong dari bank
        if(pData[TaxiOrders[orderid][toCustomerID]][pBank] >= finalfare) {
            pData[TaxiOrders[orderid][toCustomerID]][pBank] -= finalfare;
            SendClientMessageEx(TaxiOrders[orderid][toCustomerID], COLOR_INFO,
                "[BANK] $%s dipotong untuk biaya taxi.", FormatMoney(finalfare));
        } else {
            // Tidak cukup uang
            SendClientMessage(driverid, COLOR_ERROR, 
                "Customer tidak memiliki cukup uang!");
            SendClientMessage(TaxiOrders[orderid][toCustomerID], COLOR_ERROR, 
                "Anda tidak memiliki cukup uang untuk membayar taxi!");
            
            // Cancel order
            CancelTaxiOrder(orderid, "Insufficient funds", driverid);
            return 0;
        }
    } else {
        // Potong dari uang cash
        GivePlayerMoney(TaxiOrders[orderid][toCustomerID], -finalfare);
    }
    
    // Berikan uang ke driver
    GivePlayerMoney(driverid, finalfare);
    TaxiDriver[driverid][tdTotalEarnings] += finalfare;
    
    // Update order status
    TaxiOrders[orderid][toStatus] = 3; // Completed
    
    // Hapus checkpoint
    DisablePlayerCheckpoint(driverid);
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        DisablePlayerCheckpoint(TaxiOrders[orderid][toCustomerID]);
    }
    
    // Hapus drop checkpoint
    DestroyDynamicCP(TaxiOrders[orderid][toDropCP]);
    
    // Hide fare meter
    PlayerTextDrawHide(driverid, FareText[driverid]);
    PlayerTextDrawHide(driverid, DistanceText[driverid]);
    
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        PlayerTextDrawHide(TaxiOrders[orderid][toCustomerID], 
            FareText[TaxiOrders[orderid][toCustomerID]]);
        PlayerTextDrawHide(TaxiOrders[orderid][toCustomerID], 
            DistanceText[TaxiOrders[orderid][toCustomerID]]);
    }
    
    // Notify both
    SendClientMessageEx(driverid, COLOR_SUCCESS,
        "[TAXI] Perjalanan selesai! Anda mendapatkan {FFD700}$%s",
        FormatMoney(finalfare));
    
    SendClientMessageEx(TaxiOrders[orderid][toCustomerID], COLOR_SUCCESS,
        "[TAXI] Terima kasih! Biaya: {FFD700}$%s",
        FormatMoney(finalfare));
    
    // Reset driver order
    TaxiDriver[driverid][tdCurrentOrder] = -1;
    TaxiDriver[driverid][tdTotalFare] = BASE_FARE;
    
    // Tambah driver kembali ke available
    if(TaxiDriver[driverid][tdOnDuty]) {
        Iter_Add(AvailableDrivers, driverid);
    }
    
    // Tampilkan rating dialog
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        ShowRatingDialog(TaxiOrders[orderid][toCustomerID], driverid);
    }
    
    // Hapus order setelah delay
    SetTimerEx("RemoveCompletedOrder", 10000, false, "i", orderid);
    
    return 1;
}

CancelTaxiOrder(orderid, reason[], byplayerid = INVALID_PLAYER_ID) {
    if(!TaxiOrders[orderid][toExists]) {
        return 0;
    }
    
    // Hentikan timer jika ada
    if(TaxiOrders[orderid][toTimer] != -1) {
        KillTimer(TaxiOrders[orderid][toTimer]);
    }
    
    // Jika sedang dalam perjalanan, hentikan fare timer
    if(TaxiOrders[orderid][toStatus] == 2 && TaxiOrders[orderid][toDriverID] != INVALID_PLAYER_ID) {
        KillTimer(TaxiDriver[TaxiOrders[orderid][toDriverID]][tdFareTimer]);
        TaxiDriver[TaxiOrders[orderid][toDriverID]][tdFareTimer] = -1;
    }
    
    // Notify customer
    if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
        SendClientMessageEx(TaxiOrders[orderid][toCustomerID], COLOR_ERROR,
            "[TAXI] Order #{FFD700}%d {FF0000}dibatalkan. {FFFFFF}Alasan: %s",
            orderid, reason);
        
        // Hapus checkpoint dan textdraw
        DisablePlayerCheckpoint(TaxiOrders[orderid][toCustomerID]);
        PlayerTextDrawHide(TaxiOrders[orderid][toCustomerID], 
            FareText[TaxiOrders[orderid][toCustomerID]]);
        PlayerTextDrawHide(TaxiOrders[orderid][toCustomerID], 
            DistanceText[TaxiOrders[orderid][toCustomerID]]);
    }
    
    // Notify driver jika ada
    if(IsPlayerConnected(TaxiOrders[orderid][toDriverID]) && 
       TaxiDriver[TaxiOrders[orderid][toDriverID]][tdCurrentOrder] == orderid) {
        SendClientMessageEx(TaxiOrders[orderid][toDriverID], COLOR_ERROR,
            "[TAXI] Order #{FFD700}%d {FF0000}dibatalkan. {FFFFFF}Alasan: %s",
            orderid, reason);
        
        // Reset driver data
        TaxiDriver[TaxiOrders[orderid][toDriverID]][tdCurrentOrder] = -1;
        TaxiDriver[TaxiOrders[orderid][toDriverID]][tdTotalFare] = BASE_FARE;
        
        // Hapus checkpoint dan textdraw
        DisablePlayerCheckpoint(TaxiOrders[orderid][toDriverID]);
        PlayerTextDrawHide(TaxiOrders[orderid][toDriverID], 
            FareText[TaxiOrders[orderid][toDriverID]]);
        PlayerTextDrawHide(TaxiOrders[orderid][toDriverID], 
            DistanceText[TaxiOrders[orderid][toDriverID]]);
        
        // Hentikan fare timer
        if(TaxiDriver[TaxiOrders[orderid][toDriverID]][tdFareTimer] != -1) {
            KillTimer(TaxiDriver[TaxiOrders[orderid][toDriverID]][tdFareTimer]);
            TaxiDriver[TaxiOrders[orderid][toDriverID]][tdFareTimer] = -1;
        }
        
        // Tambah kembali ke available drivers jika masih on duty
        if(TaxiDriver[TaxiOrders[orderid][toDriverID]][tdOnDuty]) {
            Iter_Add(AvailableDrivers, TaxiOrders[orderid][toDriverID]);
        }
    }
    
    // Hapus semua dynamic objects
    if(IsValidDynamicObject(TaxiOrders[orderid][toPickupObject])) {
        DestroyDynamicObject(TaxiOrders[orderid][toPickupObject]);
    }
    
    if(IsValidDynamic3DTextLabel(TaxiOrders[orderid][toPickupLabel])) {
        DestroyDynamic3DTextLabel(TaxiOrders[orderid][toPickupLabel]);
    }
    
    if(IsValidDynamicCP(TaxiOrders[orderid][toPickupCP])) {
        DestroyDynamicCP(TaxiOrders[orderid][toPickupCP]);
    }
    
    if(IsValidDynamicCP(TaxiOrders[orderid][toDropCP])) {
        DestroyDynamicCP(TaxiOrders[orderid][toDropCP]);
    }
    
    if(IsValidDynamicMapIcon(TaxiOrders[orderid][toMapIcon])) {
        DestroyDynamicMapIcon(TaxiOrders[orderid][toMapIcon]);
    }
    
    // Hapus dari iterator
    Iter_Remove(ActiveOrders, orderid);
    
    // Reset data
    TaxiOrders[orderid][toExists] = false;
    
    return 1;
}

// ==================== TIMER FUNCTIONS ====================

public UpdateFareMeter(driverid, orderid) {
    if(!TaxiDriver[driverid][tdOnDuty] || TaxiOrders[orderid][toStatus] != 2) {
        KillTimer(TaxiDriver[driverid][tdFareTimer]);
        TaxiDriver[driverid][tdFareTimer] = -1;
        return 0;
    }
    
    // Simpan posisi sebelumnya
    static Float:lastPos[MAX_PLAYERS][3];
    
    // Dapatkan posisi sekarang
    new Float:x, Float:y, Float:z;
    GetVehiclePos(TaxiDriver[driverid][tdVehicleID], x, y, z);
    
    // Hitung jarak dari posisi terakhir
    if(lastPos[driverid][0] != 0.0) {
        new Float:distance = GetDistanceBetweenPoints(
            lastPos[driverid][0], lastPos[driverid][1], lastPos[driverid][2],
            x, y, z);
        
        // Tambah fare berdasarkan jarak ($0.1 per meter)
        if(distance > 1.0) { // Minimal 1 meter
            new fareadd = floatround(distance * FARE_PER_METER);
            if(fareadd > 0) {
                TaxiDriver[driverid][tdTotalFare] += fareadd;
                
                // Update fare text
                new farestr[64];
                format(farestr, sizeof(farestr), "Fare: $%s", 
                    FormatMoney(TaxiDriver[driverid][tdTotalFare]));
                PlayerTextDrawSetString(driverid, FareText[driverid], farestr);
                
                // Juga untuk customer
                if(IsPlayerConnected(TaxiOrders[orderid][toCustomerID])) {
                    PlayerTextDrawSetString(TaxiOrders[orderid][toCustomerID], 
                        FareText[TaxiOrders[orderid][toCustomerID]], farestr);
                }
            }
        }
    }
    
    // Update posisi terakhir
    lastPos[driverid][0] = x;
    lastPos[driverid][1] = y;
    lastPos[driverid][2] = z;
    
    return 1;
}

public UpdateDriverPosition(playerid) {
    if(!TaxiDriver[playerid][tdOnDuty]) {
        return 0;
    }
    
    // Update posisi terakhir driver
    GetPlayerPos(playerid, 
        TaxiDriver[playerid][tdLastX], 
        TaxiDriver[playerid][tdLastY], 
        TaxiDriver[playerid][tdLastZ]);
    
    return 1;
}

public AutoCancelOrder(orderid) {
    if(TaxiOrders[orderid][toExists] && TaxiOrders[orderid][toStatus] == 0) {
        CancelTaxiOrder(orderid, "Timeout - no driver accepted");
    }
    return 1;
}

public TaxiOrderTimeout(playerid) {
    if(PlayerSelectingDest[playerid]) {
        SendClientMessage(playerid, COLOR_ERROR, 
            "[TAXI] Order timeout! Silakan coba lagi.");
        PlayerSelectingDest[playerid] = 0;
        RemovePlayerMapIcon(playerid, 50);
    }
    return 1;
}

public RemoveCompletedOrder(orderid) {
    if(TaxiOrders[orderid][toExists] && TaxiOrders[orderid][toStatus] == 3) {
        Iter_Remove(ActiveOrders, orderid);
        TaxiOrders[orderid][toExists] = false;
    }
    return 1;
}

// ==================== EVENT HANDLERS ====================

public OnPlayerStateChange(playerid, newstate, oldstate) {
    // Jika driver keluar dari kendaraan
    if(oldstate == PLAYER_STATE_DRIVER && TaxiDriver[playerid][tdOnDuty]) {
        ForceDriverOffDuty(playerid, "Keluar dari kendaraan");
    }
    
    // Jika customer keluar dari taxi saat dalam perjalanan
    if(oldstate == PLAYER_STATE_PASSENGER) {
        new orderid = GetPlayerOrderID(playerid);
        if(orderid != -1 && TaxiOrders[orderid][toStatus] == 2) {
            // Customer keluar sebelum sampai tujuan
            CancelTaxiOrder(orderid, "Customer left the vehicle", playerid);
            
            // Potong biaya dari bank
            new penalty = floatround(TaxiOrders[orderid][toFare] * 0.5); // 50% penalty
            if(pData[playerid][pBank] >= penalty) {
                pData[playerid][pBank] -= penalty;
                SendClientMessageEx(playerid, COLOR_ERROR,
                    "[BANK] $%s dipotong karena membatalkan perjalanan taxi.", 
                    FormatMoney(penalty));
            }
        }
    }
    
    return 1;
}

public OnPlayerEnterCheckpoint(playerid) {
    // Driver sampai di pickup point
    if(TaxiDriver[playerid][tdOnDuty] && TaxiDriver[playerid][tdCurrentOrder] != -1) {
        new orderid = TaxiDriver[playerid][tdCurrentOrder];
        
        if(TaxiOrders[orderid][toStatus] == 1) {
            // Cek apakah di pickup checkpoint
            new Float:dist = GetDistanceBetweenPoints(
                TaxiDriver[playerid][tdLastX], TaxiDriver[playerid][tdLastY], TaxiDriver[playerid][tdLastZ],
                TaxiOrders[orderid][toPickupX], TaxiOrders[orderid][toPickupY], TaxiOrders[orderid][toPickupZ]);
            
            if(dist < 5.0) {
                SendClientMessage(playerid, COLOR_INFO,
                    "[TAXI] Anda sampai di lokasi pickup! Tunggu customer masuk.");
                SendClientMessage(playerid, COLOR_INFO,
                    "Gunakan {FFD700}/pickup {FFFFFF}saat customer sudah masuk.");
            }
        }
        else if(TaxiOrders[orderid][toStatus] == 2) {
            // Cek apakah di drop checkpoint
            new Float:dist = GetDistanceBetweenPoints(
                TaxiDriver[playerid][tdLastX], TaxiDriver[playerid][tdLastY], TaxiDriver[playerid][tdLastZ],
                TaxiOrders[orderid][toDropX], TaxiOrders[orderid][toDropY], TaxiOrders[orderid][toDropZ]);
            
            if(dist < 5.0) {
                SendClientMessage(playerid, COLOR_INFO,
                    "[TAXI] Anda sampai di lokasi tujuan! Gunakan {FFD700}/complete");
            }
        }
    }
    
    return 1;
}

// ==================== HELPER FUNCTIONS ====================

ForceDriverOffDuty(playerid, reason[]) {
    if(!TaxiDriver[playerid][tdOnDuty]) return 0;
    
    TaxiDriver[playerid][tdOnDuty] = 0;
    
    // Cancel current order jika ada
    if(TaxiDriver[playerid][tdCurrentOrder] != -1) {
        CancelTaxiOrder(TaxiDriver[playerid][tdCurrentOrder], 
            "Driver off duty: " + reason, playerid);
        TaxiDriver[playerid][tdCurrentOrder] = -1;
    }
    
    // Hentikan semua timer
    if(TaxiDriver[playerid][tdFareTimer] != -1) {
        KillTimer(TaxiDriver[playerid][tdFareTimer]);
        TaxiDriver[playerid][tdFareTimer] = -1;
    }
    
    // Hapus dari available drivers
    Iter_Remove(AvailableDrivers, playerid);
    
    // Reset warna
    SetPlayerColor(playerid, COLOR_WHITE);
    
    // Hide textdraws
    PlayerTextDrawHide(playerid, FareText[playerid]);
    PlayerTextDrawHide(playerid, DistanceText[playerid]);
    
    SendClientMessageEx(playerid, COLOR_ERROR, 
        "[TAXI] Anda OFF DUTY. Alasan: %s", reason);
    
    return 1;
}

GetPlayerOrderID(playerid) {
    foreach(new i : ActiveOrders) {
        if(TaxiOrders[i][toCustomerID] == playerid && 
           (TaxiOrders[i][toStatus] == 0 || TaxiOrders[i][toStatus] == 1 || TaxiOrders[i][toStatus] == 2)) {
            return i;
        }
    }
    return -1;
}

CancelPlayerOrder(playerid) {
    foreach(new i : ActiveOrders) {
        if(TaxiOrders[i][toCustomerID] == playerid) {
            CancelTaxiOrder(i, "Player disconnected", playerid);
            return 1;
        }
    }
    return 0;
}

ShowAvailableOrders(playerid) {
    new str[2048];
    format(str, sizeof(str), "ID\tCustomer\tDistance\tFare\tStatus\n");
    
    new count = 0;
    foreach(new i : ActiveOrders) {
        if(TaxiOrders[i][toStatus] == 0) {
            new line[256];
            format(line, sizeof(line), "%d\t%s\t%.1f km\t$%s\tWaiting\n",
                i,
                ReturnName(TaxiOrders[i][toCustomerID]),
                TaxiOrders[i][toDistance]/1000,
                FormatMoney(TaxiOrders[i][toFare]));
            
            strcat(str, line);
            count++;
        }
    }
    
    if(count == 0) {
        SendClientMessage(playerid, COLOR_ERROR, "Tidak ada order yang tersedia.");
    } else {
        ShowPlayerDialog(playerid, DIALOG_TAXI_ORDERS, DIALOG_STYLE_TABLIST_HEADERS,
            "{FFD700}Available Taxi Orders", str, "Accept", "Cancel");
    }
    
    return count;
}

UpdatePickupLabel(orderid) {
    if(!IsValidDynamic3DTextLabel(TaxiOrders[orderid][toPickupLabel])) return;
    
    new labelstr[128];
    if(TaxiOrders[orderid][toStatus] == 0) {
        format(labelstr, sizeof(labelstr),
            "[TAXI PICKUP #%d]\n{FFFFFF}Customer: %s\n{FFD700}Fare: $%s\n{FFFFFF}Status: Waiting",
            orderid, ReturnName(TaxiOrders[orderid][toCustomerID]), 
            FormatMoney(TaxiOrders[orderid][toFare]));
    } else if(TaxiOrders[orderid][toStatus] == 1) {
        format(labelstr, sizeof(labelstr),
            "[TAXI PICKUP #%d]\n{FFFFFF}Driver: %s\n{FFD700}Fare: $%s\n{FFFFFF}Status: Accepted",
            orderid, ReturnName(TaxiOrders[orderid][toDriverID]), 
            FormatMoney(TaxiOrders[orderid][toFare]));
    }
    
    UpdateDynamic3DTextLabelText(TaxiOrders[orderid][toPickupLabel], COLOR_ORDER, labelstr);
}

ShowRatingDialog(customerid, driverid) {
    new str[256];
    format(str, sizeof(str),
        "Berikan rating untuk driver %s:\n\n\
        "WHITE_E"★★★★★ - Sangat Puas\n\
        "WHITE_E"★★★★☆ - Puas\n\
        "WHITE_E"★★★☆☆ - Cukup\n\
        "WHITE_E"★★☆☆☆ - Kurang Puas\n\
        "WHITE_E"★☆☆☆☆ - Sangat Buruk",
        ReturnName(driverid));
    
    ShowPlayerDialog(customerid, DIALOG_TAXI_RATE, DIALOG_STYLE_LIST,
        "{FFD700}Rate Your Driver", str, "Rate", "Skip");
}

ShowOrderInfo(playerid, orderid) {
    new str[512];
    
    if(TaxiOrders[orderid][toCustomerID] == playerid) {
        format(str, sizeof(str),
            "{FFD700}[YOUR TAXI ORDER #%d]\n\n\
            {FFFFFF}Status: %s\n\
            Distance: {FFD700}%.1f km\n\
            Estimated Fare: {FFD700}$%s",
            orderid,
            GetOrderStatusText(TaxiOrders[orderid][toStatus]),
            TaxiOrders[orderid][toDistance]/1000,
            FormatMoney(TaxiOrders[orderid][toFare]));
            
        if(TaxiOrders[orderid][toStatus] == 1) {
            format(str, sizeof(str), "%s\n\nDriver: {FFD700}%s", 
                str, ReturnName(TaxiOrders[orderid][toDriverID]));
        }
    }
    
    ShowPlayerDialog(playerid, DIALOG_UNUSED, DIALOG_STYLE_MSGBOX,
        "{FFD700}Order Information", str, "OK", "");
}

stock GetOrderStatusText(status) {
    new text[20];
    switch(status) {
        case 0: text = "{FFFF00}Waiting for driver";
        case 1: text = "{00FF00}Driver accepted";
        case 2: text = "{FFA500}On trip";
        case 3: text = "{00FF00}Completed";
        case 4: text = "{FF0000}Cancelled";
        default: text = "Unknown";
    }
    return text;
}

// ==================== DIALOG RESPONSES ====================

public OnDialogResponse(playerid, dialogid, response, listitem, inputtext[]) {
    if(dialogid == DIALOG_TAXI_ORDERS && response) {
        new orderid = strval(inputtext);
        AcceptTaxiOrder(orderid, playerid);
        return 1;
    }
    
    if(dialogid == DIALOG_TAXI_RATE && response) {
        new rating = 5 - listitem;
        
        // Simpan rating driver
        // Anda perlu menyimpan last driver ID
        new driverid = pData[playerid][pLastTaxiDriver];
        
        if(IsPlayerConnected(driverid)) {
            // Update rating
            TaxiDriver[driverid][tdRating] = 
                ((TaxiDriver[driverid][tdRating] * TaxiDriver[driverid][tdRatingCount]) + rating) / 
                (TaxiDriver[driverid][tdRatingCount] + 1);
            TaxiDriver[driverid][tdRatingCount]++;
            
            SendClientMessageEx(playerid, COLOR_SUCCESS,
                "Terima kasih! Anda memberikan %d bintang untuk %s",
                rating, ReturnName(driverid));
                
            SendClientMessageEx(driverid, COLOR_SUCCESS,
                "%s memberikan rating %d bintang untuk Anda!",
                ReturnName(playerid), rating);
        }
        return 1;
    }
    
    return 0;
}
```

## **2. Defines yang Diperlukan**
```pawn
// Dialog IDs
#define DIALOG_TAXI_MAP 6000
#define DIALOG_TAXI_ORDERS 6001
#define DIALOG_TAXI_RATE 6002

// Vehicle models untuk taxi
#define TAXI_MODEL_1 438  // Cabbie
#define TAXI_MODEL_2 420  // Taxi
#define TAXI_MODEL_3 560  // Sultan (bisa jadi taxi mewah)
```

## **3. Update pData (player data)**
```pawn
// Tambahkan di enum player data:
enum E_PLAYER_DATA {
    // ... data lainnya
    
    // Taxi system
    pLastTaxiDriver,
    pSelectingTaxiDest,
    Float:pTaxiDestX,
    Float:pTaxiDestY,
    Float:pTaxiDestZ,
    
    // ... data lainnya
}
```

## **Fitur Utama Sistem:**

### **1. Order System:**
- Customer: `/call 933` → Pilih lokasi di peta → Ketik `yes`
- Fare otomatis dihitung: **$5 base + $0.1 per meter**
- Lokasi tujuan ditandai di peta dengan marker merah

### **2. Driver System:**
- `/taxiduty` untuk mulai kerja
- Auto off duty jika keluar dari taxi
- `/acceptorder` untuk ambil orderan
- Fare meter real-time saat perjalanan

### **3. Auto Systems:**
- **Driver keluar taxi** → Auto off duty
- **Customer keluar taxi** → Auto cancel + potong 50% dari bank
- **Order timeout** (10 menit) → Auto cancel
- **Fare update otomatis** setiap detik berdasarkan jarak

### **4. Payment System:**
- Otomatis potong dari uang cash
- Jika tidak cukup, potong dari bank
- Jika bank tidak cukup, order dibatalkan

### **5. Map System:**
- Marker pickup untuk driver
- Checkpoint otomatis untuk driver dan customer
- Map icon untuk order yang waiting

Sistem ini sangat otomatis dan realistis dengan fare berdasarkan jarak sebenarnya!
