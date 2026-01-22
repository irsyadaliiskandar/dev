#include <a_samp>
#include <sscanf2>
#include <zcmd>
#include <foreach>
#include <YSI\y_ini>
#include <YSI\y_timers>

#define MAX_FRACTION_GARAGES 20
#define MAX_VEHICLES_PER_GARAGE 15
#define GARAGE_INTERIOR 0
#define GARAGE_VW_OFFSET 1000

#define COLOR_INFO 0x00C8FFFF
#define COLOR_SUCCESS 0x00FF00FF
#define COLOR_ERROR 0xFF0000FF
#define COLOR_FACTION 0x8B4513FF

enum E_FRACTION_GARAGE {
    garID,
    garFraction,
    Float:garX,
    Float:garY,
    Float:garZ,
    garInterior,
    garVW,
    garMaxSlots,
    garCurrent,
    garLocked,
    garPin[7],
    garName[32],
    garPickup,
    garLabel,
    bool:garExists
};

enum E_FRACTION_VEHICLE {
    fvID,
    fvGarageID,
    fvModel,
    fvPlate[11],
    fvColor1,
    fvColor2,
    Float:fvX,
    Float:fvY,
    Float:fvZ,
    Float:fvA,
    fvFuel,
    fvMileage,
    fvStorage,
    fvCurrentStorage,
    bool:fvParked,
    fvObject,
    fvTextLabel
};

new FractionGarages[MAX_FRACTION_GARAGES][E_FRACTION_GARAGE];
new FractionVehicles[MAX_VEHICLES_PER_GARAGE * MAX_FRACTION_GARAGES][E_FRACTION_VEHICLE];
new Iterator:GarageVehicles<MAX_VEHICLES_PER_GARAGE * MAX_FRACTION_GARAGES>;

// Player variables
new PlayerEditingGarage[MAX_PLAYERS] = {-1, ...};
new PlayerGarageMenu[MAX_PLAYERS] = {-1, ...};

// Forward declarations
forward LoadFractionGarages();
forward OnGarageLoaded(index);
forward SaveGarage(garageid);
forward OnVehicleParked(vehicleid, garageid);

public OnGameModeInit() {
    print("=======================================");
    print(" Fraction Garage System v1.0");
    print(" Loading garages for factions 1-5...");
    print("=======================================");
    
    // Initialize iterator
    Iter_Init(GarageVehicles);
    
    // Create default garages for factions 1-5
    CreateDefaultFractionGarages();
    
    // Load from file
    LoadFractionGarages();
    
    return 1;
}

public OnGameModeExit() {
    // Save all garages
    for(new i = 0; i < MAX_FRACTION_GARAGES; i++) {
        if(FractionGarages[i][garExists]) {
            SaveGarage(i);
        }
    }
    return 1;
}

CreateDefaultFractionGarages() {
    // Faction 1 - LSPD (Los Santos Police Department)
    CreateFractionGarage(1, 1568.6229, -1695.0386, 5.8906, 0, "LSPD Main Garage", 10);
    CreateFractionGarage(1, 1537.7761, -1675.6884, 5.8906, 0, "LSPD Motor Pool", 8);
    
    // Faction 2 - FBI (Federal Bureau of Investigation)
    CreateFractionGarage(2, 291.3651, 180.2344, 1007.1719, 0, "FBI Headquarters Garage", 8);
    CreateFractionGarage(2, 302.4915, 196.6953, 1007.1719, 0, "FBI Secure Garage", 6);
    
    // Faction 3 - Army (San Andreas Army)
    CreateFractionGarage(3, 274.5420, 1982.2356, 17.6406, 0, "Army Base Garage", 12);
    CreateFractionGarage(3, 299.8569, 1977.8165, 17.6406, 0, "Army Heavy Vehicles", 8);
    
    // Faction 4 - Medic (San Andreas Medic)
    CreateFractionGarage(4, 1172.4922, -1323.2427, 15.4030, 0, "All Saints Garage", 8);
    CreateFractionGarage(4, 2034.9521, -1402.2230, 17.2947, 0, "County General Garage", 6);
    
    // Faction 5 - News (San Andreas News)
    CreateFractionGarage(5, 643.8264, -1359.3162, 13.5391, 0, "SAN Studio Garage", 6);
    CreateFractionGarage(5, 780.4919, -1336.6854, 13.5391, 0, "News Van Pool", 4);
}

CreateFractionGarage(fractionid, Float:x, Float:y, Float:z, interior, name[], maxslots) {
    for(new i = 0; i < MAX_FRACTION_GARAGES; i++) {
        if(!FractionGarages[i][garExists]) {
            FractionGarages[i][garID] = i;
            FractionGarages[i][garFraction] = fractionid;
            FractionGarages[i][garX] = x;
            FractionGarages[i][garY] = y;
            FractionGarages[i][garZ] = z;
            FractionGarages[i][garInterior] = interior;
            FractionGarages[i][garVW] = GARAGE_VW_OFFSET + i;
            FractionGarages[i][garMaxSlots] = maxslots;
            FractionGarages[i][garCurrent] = 0;
            FractionGarages[i][garLocked] = 0;
            format(FractionGarages[i][garPin], 7, "123456");
            format(FractionGarages[i][garName], 32, name);
            FractionGarages[i][garExists] = true;
            
            // Create pickup and label
            FractionGarages[i][garPickup] = CreatePickup(1239, 1, x, y, z, 0);
            new label[128];
            format(label, sizeof(label), "[GARAGE %s]\n{F5DEB3}%s\n{FFFFFF}Slot: 0/%d\n{FFA500}Gunakan /garage", 
                GetFractionName(fractionid), name, maxslots);
            FractionGarages[i][garLabel] = Create3DTextLabel(label, COLOR_FACTION, x, y, z, 15.0, 0, 1);
            
            printf("Created garage for faction %d: %s (ID: %d)", fractionid, name, i);
            return i;
        }
    }
    return -1;
}

public LoadFractionGarages() {
    new File:file = fopen("fraction_garages.ini", io_read);
    if(file) {
        new line[256], idx = 0;
        while(fread(file, line)) {
            if(idx >= MAX_FRACTION_GARAGES) break;
            
            new garID, fraction, Float:x, Float:y, Float:z, interior, vw, maxslots, current, locked, pin[7], name[32];
            if(sscanf(line, "p<|>ddfddddddds[7]s[32]", 
                garID, fraction, x, y, z, interior, vw, maxslots, current, locked, pin, name)) {
                printf("Error loading garage line: %s", line);
                continue;
            }
            
            FractionGarages[idx][garID] = garID;
            FractionGarages[idx][garFraction] = fraction;
            FractionGarages[idx][garX] = x;
            FractionGarages[idx][garY] = y;
            FractionGarages[idx][garZ] = z;
            FractionGarages[idx][garInterior] = interior;
            FractionGarages[idx][garVW] = vw;
            FractionGarages[idx][garMaxSlots] = maxslots;
            FractionGarages[idx][garCurrent] = current;
            FractionGarages[idx][garLocked] = locked;
            format(FractionGarages[idx][garPin], 7, pin);
            format(FractionGarages[idx][garName], 32, name);
            FractionGarages[idx][garExists] = true;
            
            // Create pickup and label
            FractionGarages[idx][garPickup] = CreatePickup(1239, 1, x, y, z, 0);
            new label[256];
            format(label, sizeof(label), "[GARAGE %s]\n{F5DEB3}%s\n{FFFFFF}Slot: %d/%d\n{FFA500}Gunakan /garage", 
                GetFractionName(fraction), name, current, maxslots);
            FractionGarages[idx][garLabel] = Create3DTextLabel(label, COLOR_FACTION, x, y, z, 15.0, 0, 1);
            
            idx++;
        }
        fclose(file);
        printf("Loaded %d fraction garages from file.", idx);
    } else {
        printf("No fraction_garages.ini found, using default garages.");
    }
    return 1;
}

public SaveGarage(garageid) {
    new file[128];
    format(file, sizeof(file), "fraction_garages.ini");
    
    new line[256];
    format(line, sizeof(line), "%d|%d|%f|%f|%f|%d|%d|%d|%d|%d|%s|%s\n",
        FractionGarages[garageid][garID],
        FractionGarages[garageid][garFraction],
        FractionGarages[garageid][garX],
        FractionGarages[garageid][garY],
        FractionGarages[garageid][garZ],
        FractionGarages[garageid][garInterior],
        FractionGarages[garageid][garVW],
        FractionGarages[garageid][garMaxSlots],
        FractionGarages[garageid][garCurrent],
        FractionGarages[garageid][garLocked],
        FractionGarages[garageid][garPin],
        FractionGarages[garageid][garName]);
    
    new File:f = fopen(file, io_append);
    if(f) {
        fwrite(f, line);
        fclose(f);
    }
    return 1;
}

stock GetFractionName(fractionid) {
    new name[32];
    switch(fractionid) {
        case 1: name = "LSPD";
        case 2: name = "FBI";
        case 3: name = "ARMY";
        case 4: name = "MEDIC";
        case 5: name = "NEWS";
        default: name = "CIVILIAN";
    }
    return name;
}

stock IsPlayerInFraction(playerid, fractionid) {
    // Ganti dengan sistem fraksi Anda
    // Contoh: if(PlayerInfo[playerid][pFaction] == fractionid) return 1;
    return 1; // Temporary
}

stock GetPlayerFraction(playerid) {
    // Ganti dengan sistem fraksi Anda
    // return PlayerInfo[playerid][pFaction];
    return 1; // Temporary - untuk testing
}

stock GetPlayerRank(playerid) {
    // Ganti dengan sistem rank fraksi Anda
    // return PlayerInfo[playerid][pRank];
    return 6; // Temporary - rank tinggi untuk testing
}
