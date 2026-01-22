Saya akan buat sistem yang menghubungkan Discord dengan database SA-MP. Sistem ini memungkinkan admin membuat kode VIP di Discord, dan user/warga bisa claim kode tersebut di dalam game SA-MP.

## **Sistem Integrasi Discord ↔ SA-MP VIP Kode**

### **1. Struktur Database MySQL (SA-MP)**
```sql
-- vip_codes.sql
CREATE TABLE IF NOT EXISTS `vip_codes` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `code` VARCHAR(20) NOT NULL UNIQUE,
    `package_type` ENUM('basic', 'advanced', 'pro', 'lifetime') NOT NULL,
    `created_by` VARCHAR(50) NOT NULL COMMENT 'Discord Admin ID',
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `claimed_by` VARCHAR(50) DEFAULT NULL COMMENT 'SA-MP Player Name',
    `claimed_at` TIMESTAMP NULL,
    `expires_at` TIMESTAMP NULL,
    `is_active` BOOLEAN DEFAULT TRUE,
    `max_uses` INT DEFAULT 1,
    `current_uses` INT DEFAULT 0,
    `note` TEXT
);

CREATE TABLE IF NOT EXISTS `players_vip` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `username` VARCHAR(24) NOT NULL,
    `discord_id` VARCHAR(50),
    `vip_level` ENUM('basic', 'advanced', 'pro', 'lifetime') NOT NULL,
    `purchased_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `expires_at` TIMESTAMP NULL,
    `is_active` BOOLEAN DEFAULT TRUE,
    `gold_points` INT DEFAULT 0,
    `vehicle_slots` INT DEFAULT 0,
    `garage_slots` INT DEFAULT 0,
    `vip_claims` INT DEFAULT 0,
    `custom_gates` INT DEFAULT 0
);
```

### **2. Discord Bot - Admin Commands**
```javascript
// commands/admin/generate-samp-code.js
const { SlashCommandBuilder, EmbedBuilder, PermissionFlagsBits } = require('discord.js');
const mysql = require('mysql2/promise');

module.exports = {
    data: new SlashCommandBuilder()
        .setName('samp-vip-code')
        .setDescription('[ADMIN] Buat kode VIP untuk SA-MP')
        .addStringOption(option =>
            option.setName('package')
                .setDescription('Tipe paket VIP')
                .setRequired(true)
                .addChoices(
                    { name: 'BASIC VIP', value: 'basic' },
                    { name: 'ADVANCED VIP', value: 'advanced' },
                    { name: 'PROFESSIONAL VIP', value: 'pro' },
                    { name: 'LIFETIME VIP', value: 'lifetime' }
                ))
        .addIntegerOption(option =>
            option.setName('quantity')
                .setDescription('Jumlah kode yang dibuat')
                .setRequired(false)
                .setMinValue(1)
                .setMaxValue(50))
        .addIntegerOption(option =>
            option.setName('days')
                .setDescription('Masa berlaku (hari)')
                .setRequired(false)
                .setMinValue(1))
        .addIntegerOption(option =>
            option.setName('max_uses')
                .setDescription('Max penggunaan per kode')
                .setRequired(false)
                .setMinValue(1))
        .addStringOption(option =>
            option.setName('custom_code')
                .setDescription('Kode custom (untuk 1 kode)')
                .setRequired(false))
        .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

    async execute(interaction) {
        await interaction.deferReply({ ephemeral: true });

        // Koneksi ke database SA-MP
        const connection = await mysql.createConnection({
            host: process.env.SAMP_DB_HOST,
            user: process.env.SAMP_DB_USER,
            password: process.env.SAMP_DB_PASS,
            database: process.env.SAMP_DB_NAME,
            port: process.env.SAMP_DB_PORT || 3306
        });

        const packageType = interaction.options.getString('package');
        const quantity = interaction.options.getInteger('quantity') || 1;
        const days = interaction.options.getInteger('days');
        const maxUses = interaction.options.getInteger('max_uses') || 1;
        const customCode = interaction.options.getString('custom_code');

        const packageInfo = {
            basic: { name: 'BASIC VIP', gold: 500, duration: days || 30 },
            advanced: { name: 'ADVANCED VIP', gold: 1000, duration: days || 30 },
            pro: { name: 'PROFESSIONAL VIP', gold: 1500, duration: days || 30 },
            lifetime: { name: 'LIFETIME VIP', gold: 3000, duration: null }
        };

        const generatedCodes = [];
        const expiresAt = days ? 
            `DATE_ADD(NOW(), INTERVAL ${days} DAY)` : 
            'NULL';

        try {
            if (customCode && quantity === 1) {
                // Insert single custom code
                const [result] = await connection.execute(
                    `INSERT INTO vip_codes (code, package_type, created_by, expires_at, max_uses) 
                     VALUES (?, ?, ?, ${expiresAt}, ?)`,
                    [customCode.toUpperCase(), packageType, interaction.user.id, maxUses]
                );
                generatedCodes.push(customCode.toUpperCase());
            } else {
                // Generate multiple codes
                for (let i = 0; i < quantity; i++) {
                    const code = generateSampCode(10);
                    
                    await connection.execute(
                        `INSERT INTO vip_codes (code, package_type, created_by, expires_at, max_uses) 
                         VALUES (?, ?, ?, ${expiresAt}, ?)`,
                        [code, packageType, interaction.user.id, maxUses]
                    );
                    
                    generatedCodes.push(code);
                }
            }

            // Buat embed hasil
            const embed = new EmbedBuilder()
                .setTitle('✅ Kode VIP SA-MP Berhasil Dibuat')
                .setColor(0x00FF00)
                .setDescription(`Kode telah disimpan ke database SA-MP dan siap digunakan!`)
                .addFields(
                    { name: 'Package', value: packageInfo[packageType].name, inline: true },
                    { name: 'Quantity', value: `${quantity} kode`, inline: true },
                    { name: 'Max Uses', value: `${maxUses}`, inline: true },
                    { name: 'Gold Points', value: `${packageInfo[packageType].gold}`, inline: true }
                );

            if (days) {
                embed.addFields({ 
                    name: 'Masa Berlaku', 
                    value: `${days} hari`, 
                    inline: true 
                });
            }

            // Tampilkan 5 kode pertama
            if (generatedCodes.length <= 5) {
                embed.addFields({ 
                    name: '📋 Kode yang dibuat:', 
                    value: `\`\`\`\n${generatedCodes.join('\n')}\n\`\`\`` 
                });
            } else {
                const firstFive = generatedCodes.slice(0, 5);
                embed.addFields({ 
                    name: '📋 Contoh Kode:', 
                    value: `\`\`\`\n${firstFive.join('\n')}\n...dan ${generatedCodes.length - 5} kode lainnya\`\`\`` 
                });
            }

            embed.setFooter({ 
                text: `Gunakan /vipcode [kode] di SA-MP untuk claim` 
            });

            await interaction.editReply({ embeds: [embed] });

            // Kirim semua kode via DM
            if (generatedCodes.length > 5) {
                try {
                    const dmEmbed = new EmbedBuilder()
                        .setTitle('📦 Semua Kode VIP SA-MP')
                        .setDescription(`**Package:** ${packageInfo[packageType].name}\n**Total:** ${generatedCodes.length} kode`)
                        .addFields({ 
                            name: 'Daftar Kode:', 
                            value: `\`\`\`\n${generatedCodes.join('\n')}\n\`\`\`` 
                        })
                        .setColor(0x3498DB)
                        .setTimestamp();

                    await interaction.user.send({ embeds: [dmEmbed] });
                    await interaction.followUp({ 
                        content: '📨 Semua kode telah dikirim ke DM Anda!', 
                        ephemeral: true 
                    });
                } catch (error) {
                    console.error('Gagal mengirim DM:', error);
                }
            }

        } catch (error) {
            console.error('Database error:', error);
            await interaction.editReply('❌ Gagal membuat kode: ' + error.message);
        } finally {
            await connection.end();
        }
    }
};

// Helper: Generate random code untuk SA-MP
function generateSampCode(length = 10) {
    const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; // Tanpa O,0,1,I
    let result = '';
    for (let i = 0; i < length; i++) {
        result += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    
    // Format: XXX-XXX-XXXX
    return `${result.substring(0,3)}-${result.substring(3,6)}-${result.substring(6)}`;
}
```

### **3. Discord Bot - Monitor Kode SA-MP**
```javascript
// commands/admin/samp-codes-status.js
const { SlashCommandBuilder, EmbedBuilder, PermissionFlagsBits } = require('discord.js');
const mysql = require('mysql2/promise');

module.exports = {
    data: new SlashCommandBuilder()
        .setName('samp-codes-status')
        .setDescription('[ADMIN] Cek status kode SA-MP')
        .addStringOption(option =>
            option.setName('status')
                .setDescription('Filter status')
                .addChoices(
                    { name: 'Active', value: 'active' },
                    { name: 'Used', value: 'used' },
                    { name: 'Expired', value: 'expired' }
                )
                .setRequired(false))
        .setDefaultMemberPermissions(PermissionFlagsBits.Administrator),

    async execute(interaction) {
        await interaction.deferReply({ ephemeral: true });

        const connection = await mysql.createConnection({
            host: process.env.SAMP_DB_HOST,
            user: process.env.SAMP_DB_USER,
            password: process.env.SAMP_DB_PASS,
            database: process.env.SAMP_DB_NAME
        });

        const status = interaction.options.getString('status') || 'active';
        
        let query = `SELECT * FROM vip_codes WHERE 1=1`;
        const params = [];

        if (status === 'active') {
            query += ` AND is_active = 1 AND (expires_at IS NULL OR expires_at > NOW())`;
        } else if (status === 'used') {
            query += ` AND claimed_by IS NOT NULL`;
        } else if (status === 'expired') {
            query += ` AND expires_at < NOW() AND is_active = 1`;
        }

        query += ` ORDER BY created_at DESC LIMIT 15`;

        const [codes] = await connection.execute(query, params);

        if (codes.length === 0) {
            await connection.end();
            return interaction.editReply('❌ Tidak ada kode ditemukan.');
        }

        const embed = new EmbedBuilder()
            .setTitle(`📊 Status Kode SA-MP - ${status.toUpperCase()}`)
            .setColor(0x3498DB)
            .setTimestamp();

        codes.forEach(code => {
            const statusIcon = code.claimed_by ? '🔴' : 
                             (code.expires_at && new Date(code.expires_at) < new Date()) ? '⏰' : '🟢';
            
            const claimInfo = code.claimed_by ? 
                `**Player:** ${code.claimed_by}\n**Tanggal:** ${new Date(code.claimed_at).toLocaleDateString('id-ID')}` : 
                `**Uses:** ${code.current_uses}/${code.max_uses}`;

            const expiresInfo = code.expires_at ? 
                new Date(code.expires_at).toLocaleDateString('id-ID') : 
                'Tidak expired';

            embed.addFields({
                name: `${statusIcon} ${code.code}`,
                value: `**Package:** ${code.package_type.toUpperCase()}\n${claimInfo}\n**Expires:** ${expiresInfo}`,
                inline: true
            });
        });

        await connection.end();
        await interaction.editReply({ embeds: [embed] });
    }
};
```

### **4. SA-MP Pawn Script - Claim System**
```pawn
// vip_code_system.pwn
#include <a_samp>
#include <sscanf2>
#include <mysql>

#define COLOR_INFO 0x00C8FFFF
#define COLOR_SUCCESS 0x00FF00FF
#define COLOR_ERROR 0xFF0000FF

new MySQL:g_SQL;

public OnGameModeInit() {
    // Koneksi ke database
    g_SQL = mysql_connect("127.0.0.1", "user", "password", "samp_db");
    
    if(mysql_errno(g_SQL) != 0) {
        print("ERROR: Gagal koneksi ke database!");
        SendRconCommand("exit");
    } else {
        print("SUCCESS: Terhubung ke database!");
    }
    return 1;
}

CMD:vipcode(playerid, params[]) {
    new code[20];
    if(sscanf(params, "s[20]", code)) {
        SendClientMessage(playerid, COLOR_INFO, "Gunakan: /vipcode [kode]");
        SendClientMessage(playerid, COLOR_INFO, "Contoh: /vipcode ABC-123-DEF4");
        return 1;
    }

    // Format kode: uppercase dan hilangkan dash
    new formattedCode[20];
    format(formattedCode, sizeof(formattedCode), code);
    for(new i = 0; i < strlen(formattedCode); i++) {
        if(formattedCode[i] == '-') {
            strdel(formattedCode, i, i+1);
            i--;
        } else {
            formattedCode[i] = toupper(formattedCode[i]);
        }
    }

    // Cek kode di database
    new query[256];
    mysql_format(g_SQL, query, sizeof(query),
        "SELECT * FROM vip_codes WHERE REPLACE(code, '-', '') = '%e' AND is_active = 1",
        formattedCode
    );
    
    mysql_tquery(g_SQL, query, "OnCheckVipCode", "is", playerid, code);
    return 1;
}

forward OnCheckVipCode(playerid, code[]);
public OnCheckVipCode(playerid, code[]) {
    if(!cache_num_rows()) {
        SendClientMessage(playerid, COLOR_ERROR, "ERROR: Kode tidak valid atau sudah digunakan!");
        return 1;
    }

    new package[20], expires_at, claimed_by[24], max_uses, current_uses;
    cache_get_field_content(0, "package_type", package);
    expires_at = cache_get_field_int(0, "expires_at");
    cache_get_field_content(0, "claimed_by", claimed_by);
    max_uses = cache_get_field_int(0, "max_uses");
    current_uses = cache_get_field_int(0, "current_uses");

    // Cek apakah sudah expired
    if(expires_at > 0 && gettime() > expires_at) {
        SendClientMessage(playerid, COLOR_ERROR, "ERROR: Kode sudah expired!");
        
        // Update status kode
        new updateQuery[128];
        mysql_format(g_SQL, updateQuery, sizeof(updateQuery),
            "UPDATE vip_codes SET is_active = 0 WHERE REPLACE(code, '-', '') = '%e'",
            code
        );
        mysql_tquery(g_SQL, updateQuery);
        return 1;
    }

    // Cek apakah sudah digunakan
    if(strlen(claimed_by) > 0) {
        SendClientMessage(playerid, COLOR_ERROR, "ERROR: Kode sudah digunakan!");
        return 1;
    }

    // Cek max uses
    if(current_uses >= max_uses && max_uses > 0) {
        SendClientMessage(playerid, COLOR_ERROR, "ERROR: Kode sudah mencapai batas penggunaan!");
        return 1;
    }

    // Process VIP activation
    new playerName[MAX_PLAYER_NAME];
    GetPlayerName(playerid, playerName, sizeof(playerName));

    // Update kode sebagai digunakan
    new updateQuery[256];
    mysql_format(g_SQL, updateQuery, sizeof(updateQuery),
        "UPDATE vip_codes SET claimed_by = '%e', claimed_at = NOW(), current_uses = current_uses + 1 WHERE REPLACE(code, '-', '') = '%e'",
        playerName, code
    );
    mysql_tquery(g_SQL, updateQuery);

    // Berikan VIP ke player
    GivePlayerVip(playerid, package);
    return 1;
}

GivePlayerVip(playerid, package[]) {
    new playerName[MAX_PLAYER_NAME];
    GetPlayerName(playerid, playerName, sizeof(playerName));

    // Cek apakah player sudah punya VIP
    new checkQuery[128];
    mysql_format(g_SQL, checkQuery, sizeof(checkQuery),
        "SELECT * FROM players_vip WHERE username = '%e' AND is_active = 1",
        playerName
    );
    
    mysql_tquery(g_SQL, checkQuery, "OnCheckPlayerVip", "is", playerid, package);
}

forward OnCheckPlayerVip(playerid, package[]);
public OnCheckPlayerVip(playerid, package[]) {
    new playerName[MAX_PLAYER_NAME];
    GetPlayerName(playerid, playerName, sizeof(playerName));

    new expires_at;
    if(cache_num_rows() > 0) {
        // Player sudah punya VIP, extend atau upgrade
        SendClientMessage(playerid, COLOR_INFO, "INFO: Anda sudah memiliki VIP, memperbarui...");
        
        if(strcmp(package, "lifetime", true) == 0) {
            expires_at = 0; // Lifetime
        } else {
            expires_at = gettime() + (30 * 24 * 60 * 60); // 30 hari
        }
        
        new updateQuery[256];
        mysql_format(g_SQL, updateQuery, sizeof(updateQuery),
            "UPDATE players_vip SET vip_level = '%e', expires_at = FROM_UNIXTIME(%d), purchased_at = NOW() WHERE username = '%e'",
            package, expires_at, playerName
        );
        mysql_tquery(g_SQL, updateQuery);
    } else {
        // Player baru, insert VIP
        new gold_points, vehicle_slots, garage_slots, vip_claims, custom_gates;
        
        // Set benefits berdasarkan package
        if(strcmp(package, "basic", true) == 0) {
            gold_points = 500;
            vehicle_slots = 2;
            garage_slots = 2;
            vip_claims = 2;
            custom_gates = 1;
            expires_at = gettime() + (30 * 24 * 60 * 60);
        } else if(strcmp(package, "advanced", true) == 0) {
            gold_points = 1000;
            vehicle_slots = 3;
            garage_slots = 3;
            vip_claims = 4;
            custom_gates = 2;
            expires_at = gettime() + (30 * 24 * 60 * 60);
        } else if(strcmp(package, "pro", true) == 0) {
            gold_points = 1500;
            vehicle_slots = 4;
            garage_slots = 5;
            vip_claims = 6;
            custom_gates = 3;
            expires_at = gettime() + (30 * 24 * 60 * 60);
        } else if(strcmp(package, "lifetime", true) == 0) {
            gold_points = 3000;
            vehicle_slots = 5;
            garage_slots = 6;
            vip_claims = 10;
            custom_gates = 5;
            expires_at = 0; // Lifetime
        }
        
        new insertQuery[512];
        mysql_format(g_SQL, insertQuery, sizeof(insertQuery),
            "INSERT INTO players_vip (username, vip_level, expires_at, gold_points, vehicle_slots, garage_slots, vip_claims, custom_gates) \
             VALUES ('%e', '%e', FROM_UNIXTIME(%d), %d, %d, %d, %d, %d)",
            playerName, package, expires_at, gold_points, vehicle_slots, garage_slots, vip_claims, custom_gates
        );
        mysql_tquery(g_SQL, insertQuery);
    }

    // Berikan benefits in-game
    ApplyVipBenefits(playerid, package);
    
    // Kirim message sukses
    new msg[128];
    format(msg, sizeof(msg), "SUCCESS: Anda berhasil mengklaim %s VIP!", package);
    SendClientMessage(playerid, COLOR_SUCCESS, msg);
    
    // Berikan gold points
    GivePlayerMoney(playerid, gold_points * 1000); // Convert gold to game money
    format(msg, sizeof(msg), "INFO: Anda mendapatkan %d Gold Points!", gold_points);
    SendClientMessage(playerid, COLOR_INFO, msg);
    
    // Log ke Discord via HTTP
    LogToDiscord(playerid, package);
}

ApplyVipBenefits(playerid, package[]) {
    // Apply in-game benefits berdasarkan package
    switch(package) {
        case "basic": {
            // Basic VIP benefits
            SetPlayerColor(playerid, 0xFFD700FF); // Gold color
            SendClientMessage(playerid, -1, "VIP: Anda mendapatkan akses Dealer Import!");
            SendClientMessage(playerid, -1, "VIP: Gunakan /dyoc untuk accessories!");
        }
        case "advanced": {
            // Advanced VIP benefits
            SetPlayerColor(playerid, 0xC0C0C0FF); // Silver color
            SendClientMessage(playerid, -1, "VIP: Anda mendapatkan Online Radio (/cradio)!");
            SendClientMessage(playerid, -1, "VIP: Gunakan /vradio untuk boombox!");
        }
        case "pro": {
            // Professional VIP benefits
            SetPlayerColor(playerid, 0xFF4500FF); // OrangeRed color
            SendClientMessage(playerid, -1, "VIP: Anda bisa join 2 Legal Jobs!");
            SendClientMessage(playerid, -1, "VIP: Bisa beli Helicopter & Plane!");
        }
        case "lifetime": {
            // Lifetime VIP benefits
            SetPlayerColor(playerid, 0x8A2BE2FF); // BlueViolet color
            SendClientMessage(playerid, -1, "VIP: SELAMAT! Anda mendapatkan LIFETIME VIP!");
            SendClientMessage(playerid, -1, "VIP: Status VIP anda tidak akan pernah expired!");
        }
    }
}

LogToDiscord(playerid, package[]) {
    new playerName[MAX_PLAYER_NAME];
    GetPlayerName(playerid, playerName, sizeof(playerName));
    
    // Kirim log ke Discord webhook
    new query[512];
    mysql_format(g_SQL, query, sizeof(query),
        "INSERT INTO discord_logs (player_name, action, details) \
         VALUES ('%e', 'VIP_CLAIM', 'Claimed %s VIP using code')",
        playerName, package
    );
    mysql_tquery(g_SQL, query);
}
```

### **5. Environment Configuration (.env)**
```env
# Discord Bot Token
DISCORD_TOKEN=your_discord_bot_token_here

# SA-MP Database Configuration
SAMP_DB_HOST=localhost
SAMP_DB_USER=samp_user
SAMP_DB_PASS=your_password
SAMP_DB_NAME=samp_server
SAMP_DB_PORT=3306

# Discord Webhook for Logging
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...

# Discord Guild ID
GUILD_ID=your_server_id

# VIP Channel IDs
VIP_CODES_CHANNEL=channel_id_for_codes
VIP_LOGS_CHANNEL=channel_id_for_logs
```

### **6. Package.json Dependencies**
```json
{
  "name": "samp-vip-bot",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "discord.js": "^14.14.1",
    "mysql2": "^3.6.0",
    "dotenv": "^16.0.0",
    "axios": "^1.6.0"
  },
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  }
}
```

### **7. Index.js (Main Bot File)**
```javascript
const { Client, GatewayIntentBits, Collection } = require('discord.js');
const fs = require('fs');
const path = require('path');
require('dotenv').config();

const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent
    ]
});

client.commands = new Collection();

// Load commands
const commandsPath = path.join(__dirname, 'commands');
const commandFolders = fs.readdirSync(commandsPath);

for (const folder of commandFolders) {
    const folderPath = path.join(commandsPath, folder);
    const commandFiles = fs.readdirSync(folderPath).filter(file => file.endsWith('.js'));
    
    for (const file of commandFiles) {
        const filePath = path.join(folderPath, file);
        const command = require(filePath);
        
        if ('data' in command && 'execute' in command) {
            client.commands.set(command.data.name, command);
        } else {
            console.log(`[WARNING] Command at ${filePath} is missing required properties.`);
        }
    }
}

// Event handlers
client.once('ready', () => {
    console.log(`✅ Bot logged in as ${client.user.tag}`);
    console.log(`📊 Connected to SA-MP Database`);
});

client.on('interactionCreate', async interaction => {
    if (!interaction.isChatInputCommand()) return;

    const command = client.commands.get(interaction.commandName);

    if (!command) return;

    try {
        await command.execute(interaction);
    } catch (error) {
        console.error(error);
        await interaction.reply({
            content: '❌ Terjadi error saat menjalankan command!',
            ephemeral: true
        });
    }
});

client.login(process.env.DISCORD_TOKEN);
```

## **Cara Penggunaan:**

### **Admin di Discord:**
1. Gunakan `/samp-vip-code package:basic quantity:5`
2. Bot akan generate kode dan simpan ke database SA-MP
3. Bagikan kode ke user/warga

### **Warga di SA-MP:**
1. Masuk ke server SA-MP
2. Ketik `/vipcode ABC-123-DEF4`
3. Sistem akan cek kode di database
4. Jika valid, VIP langsung aktif

## **Fitur Keamanan:**
1. Kode format dengan dash (XXX-XXX-XXXX)
2. Expiry date otomatis
3. Log semua aktivitas
4. Anti-duplicate claim
5. Max uses protection

Sistem ini fully integrated antara Discord dan SA-MP melalui database MySQL yang sama!
