# Pauls & Friends TTT - Server Documentation (TTT2)

This document provides comprehensive technical instructions for deploying, configuring, and maintaining the "Pauls & Friends TTT" Garry's Mod server running Trouble in Terrorist Town 2 (TTT2). It is designed for Ubuntu 24.04 LTS using LinuxGSM.

---

## 1. Initial Server Setup

**System Requirements (for 300+ mods):**

- **CPU:** High single-core performance (3.5GHz+ recommended). Garry's Mod is predominantly single-threaded.
- **RAM:** Minimum 8GB (16GB recommended due to the heavy mod load and engine memory leaks).
- **Storage:** 50GB+ NVMe/SSD for OS, server files, FastDL, and Workshop content.
- **Network:** 1Gbps uplink for FastDL serving and smooth 66-tick performance.

**Dependency Installation (Ubuntu 24.04 LTS):**
Ubuntu 24.04 (Noble Numbat) requires enabling the 32-bit architecture to run Source engine binaries and SteamCMD.

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install curl wget file tar bzip2 gzip unzip bsdmainutils python3 util-linux ca-certificates binutils bc jq tmux netcat-openbsd lib32gcc-s1 lib32stdc++6 libsdl2-2.0-0:i386 steamcmd nginx mariadb-server fail2ban -y
```

**User Account Creation:**
For security reasons, the server must never run as `root`.

```bash
sudo adduser gmodserver
# Follow the prompts to set a strong password.
# Switch to the new user:
su - gmodserver
```

**Firewall Configuration (UFW):**
GMod and LinuxGSM require specific ports. Ensure UFW is active.

```bash
sudo ufw enable
sudo ufw allow 27015/tcp # RCON/Query
sudo ufw allow 27015/udp # Game Traffic
sudo ufw allow 27005/udp # Client Port
sudo ufw allow 27020/udp # SourceTV
sudo ufw allow 80/tcp    # FastDL (Nginx)
sudo ufw allow 22/tcp    # SSH
sudo ufw reload
```

**LinuxGSM Installation:**
As the `gmodserver` user, download and install LinuxGSM:

```bash
wget -O linuxgsm.sh https://linuxgsm.sh && chmod +x linuxgsm.sh && bash linuxgsm.sh gmodserver
./gmodserver install
```

_Verification Step:_ Run `./gmodserver details` to confirm the installation directories and ensure all dependencies are met.

---

## 2. LinuxGSM Configuration

**Server Instance Configuration:**
LinuxGSM configuration files override default server parameters. Edit the main LinuxGSM config file:

```bash
nano lgsm/config-lgsm/gmodserver/gmodserver.cfg
```

Add/modify the following parameters:

```ini
gamemode="terrortown"
defaultmap="ttt_waterworld"
maxplayers="32"
tickrate="66"
port="27015"
clientport="27005"
wsapikey="YOUR_STEAM_WEB_API_KEY"
wscollectionid="YOUR_WORKSHOP_COLLECTION_ID"

# Disable Lua Auto-Refresh to prevent massive lag spikes with 300+ mods
fn_parms(){
    parms="-game garrysmod -strictportbind -ip ${ip} -port ${port} +clientport ${clientport} +tv_port ${tvport} +map ${defaultmap} +servercfgfile ${servercfg} -maxplayers ${maxplayers} -tickrate ${tickrate} +gamemode ${gamemode} -disableluarefresh +host_workshop_collection ${wscollectionid} -authkey ${wsapikey}"
}
```

**FastDL (Fast Download) Setup via Nginx:**
Serving 300+ mods via Steam Workshop can lead to timeouts. A FastDL mirror is essential.
As `root` or a user with `sudo` privileges:

```bash
sudo nano /etc/nginx/sites-available/fastdl
```

```nginx
server {
    listen 80;
    server_name fastdl.pauls-ttt.com; # Replace with your domain/IP
    root /home/gmodserver/serverfiles/garrysmod;

    location / {
        autoindex on;
    }

    # Security: Block access to configuration and databases
    location ~ \.(cfg|db|sqlite|txt|log)$ {
        deny all;
    }
}
```

Enable the site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/fastdl /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

**Cronjobs (Monitoring, Backup, Updates):**
As the `gmodserver` user, automate server tasks:

```bash
crontab -e
```

Add the following lines:

```bash
*/5 * * * * /home/gmodserver/gmodserver monitor > /dev/null 2>&1
*/30 * * * * /home/gmodserver/gmodserver update > /dev/null 2>&1
0 0 * * 0 /home/gmodserver/gmodserver update-lgsm > /dev/null 2>&1
0 5 * * * /home/gmodserver/gmodserver backup > /dev/null 2>&1
0 4 * * * /home/gmodserver/gmodserver restart > /dev/null 2>&1 # Daily restart for stability
```

_Verification Step:_ Run `crontab -l` to ensure cronjobs are saved. Access `http://fastdl.pauls-ttt.com/maps/` in a web browser to test Nginx FastDL (ensure no `.cfg` files are accessible).

---

## 3. Garry's Mod Server Configuration

**`server.cfg` Customization:**
File: `/home/gmodserver/serverfiles/garrysmod/cfg/server.cfg`

```ini
hostname "Pauls & Friends TTT2 | Custom Roles | FastDL"
sv_password ""       // Set if you want a private server
rcon_password ""     // Disabled for security; use server console/LinuxGSM instead

// Network & FastDL
sv_downloadurl "http://fastdl.pauls-ttt.com/"
sv_allowdownload 0   // Forces clients to use FastDL or Workshop
sv_allowupload 0     // Security: prevent malicious uploads
net_maxfilesize 64

// Tickrate & Rates (Optimized for 66 Tick)
sv_minrate 1048576
sv_maxrate 0
sv_minupdaterate 66
sv_maxupdaterate 66
sv_mincmdrate 66
sv_maxcmdrate 66
gmod_physiterations 2 // Lowers physics CPU usage

// TTT2 Specific Variables
ttt_preptime_seconds 30
ttt_firstpreptime 60
ttt_posttime_seconds 15
ttt_haste_mode 1
ttt_minimum_players 2
```

**`autoexec.cfg`:**
File: `/home/gmodserver/serverfiles/garrysmod/cfg/autoexec.cfg`
Used for commands that must execute before the first map loads.

```ini
log on
sv_logbans 1
sv_logecho 1
sv_logfile 1
sv_log_onefile 0
```

**Map Rotation:**
Edit `serverfiles/garrysmod/cfg/mapcycle.txt`:

```text
ttt_waterworld
ttt_minecraft_b5
ttt_rooftops_a2_f1
ttt_clue_se
```

_Verification Step:_ Start the server (`./gmodserver start`). Check the console (`./gmodserver console`) to verify the 66 tickrate and map loading without errors. Press `CTRL+B` then `D` to detach from the tmux console.

---

## 4. Steam Workshop Integration

**Workshop Collection Setup:**

1. Create a Steam Workshop Collection and set it to "Unlisted" or "Public".
2. Add all 300+ mods to this collection.
3. Retrieve the Collection ID from the URL.
4. Retrieve a Steam Web API Key from `https://steamcommunity.com/dev/apikey`.
5. Enter both in `lgsm/config-lgsm/gmodserver/gmodserver.cfg` as shown in Section 2.

**`workshop.lua` Configuration:**
To force clients to download the addons, create `/home/gmodserver/serverfiles/garrysmod/lua/autorun/server/workshop.lua`.

```lua
-- Automatically instruct clients to download the collection
resource.AddWorkshop("YOUR_WORKSHOP_COLLECTION_ID")

-- For critical individual items that fail to mount properly via collection:
-- resource.AddWorkshop("123456789")
```

**Handling 300+ Mods (Dependency Resolution & Load Order):**

- **Avoid 300 individual Workshop items:** Loading 300+ individual Workshop items causes a "Connection Timed Out" error for joining players due to the time it takes the client to query Steam.
- **Optimization:** Pack custom materials, models, and sounds into a single `.gma` using GMAD, or extract them directly to the `serverfiles/garrysmod/` directories to serve via FastDL.
- **Load Order:** Garry's Mod loads addons alphabetically. If a TTT2 role depends on a base mod, ensure the base mod's folder name precedes the role mod's folder name, or use `hook.Add("Initialize", ...)` in Lua to ensure dependencies are loaded.

_Verification Step:_ Join the server with an empty `garrysmod/addons/` folder locally and verify that Workshop content downloads automatically.

---

## 5. Performance and Stability Optimization

**Server Performance Tuning:**

- **Lua Auto-Refresh:** Disabled in Section 2 via `-disableluarefresh`. This is critical; scanning 300+ mod directories on every file change causes massive lag spikes.
- **CPU Optimization:** Do not use `cpulimit`. Ensure the host has a high-clock-speed CPU. Lower `gmod_physiterations` in `server.cfg`.
- **Database Storage:** Use `mysqloo` or `tmysql4` to offload player data (ULX, TTT2 stats, Pointshop) to a MySQL/MariaDB database instead of local SQLite. This prevents file I/O bottlenecks.
  ```bash
  # Assuming mariadb is installed (Section 1)
  sudo mysql -u root -p
  CREATE DATABASE ttt2_data
  CREATE USER 'gmoduser'@'localhost' IDENTIFIED BY 'strongpassword'
  GRANT ALL PRIVILEGES ON ttt2_data.* TO 'gmoduser'@'localhost'
  FLUSH PRIVILEGES
  EXIT
  ```
- **Automated Restarts:** Garry's Mod leaks memory over time, especially with many mods. The cronjob defined in Section 2 (daily at 4 AM) is mandatory to clear RAM.

_Verification Step:_ Monitor server performance via `htop` or the `./gmodserver monitor` command. Ensure memory usage drops after the 4 AM restart. Check MySQL connection via ingame logs.

---

## 6. Security Implementation

**SSH Hardening:**
Edit `/etc/ssh/sshd_config` as `root`:

```ini
PermitRootLogin no
PasswordAuthentication no # Ensure SSH keys are set up first!
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

**Fail2Ban Configuration:**
Protect SSH and Nginx from brute-force attacks:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

**File Permissions:**
Ensure the `gmodserver` user only has access to its own files:

```bash
sudo chown -R gmodserver:gmodserver /home/gmodserver
sudo chmod -R 750 /home/gmodserver
```

**DDoS Protection:**

- Disable RCON by leaving `rcon_password ""` in `server.cfg` (prevents RCON brute-force and amplification attacks). Use `./gmodserver console` instead.
- Use a host with built-in Game DDoS mitigation (e.g., OVH Game Anti-DDoS) as `iptables` alone cannot drop volumetric attacks fast enough.

**Admin Privileges:**
Use ULX or ServerGuard. Configure `users.txt` or ULX via the ingame GUI (XGUI). Do not grant `superadmin` to temporary staff.

_Verification Step:_ Attempt to connect via SSH using a password (should be rejected). Attempt to use an RCON tool (should fail). Check FastDL URLs for `.cfg` files (should return 403 Forbidden).

---

## 7. Troubleshooting Guide

**Common LinuxGSM Errors:**

- _Tmux not found / Server won't attach:_ Run `tmux kill-server`, then `./gmodserver start`.
- _GLIBC / lib32gcc-s1 errors:_ Run `./gmodserver details`. If dependencies are missing, rerun the `apt install` command from Section 1.

**Garry's Mod Server Crashes:**

- Use `./gmodserver debug` to start the server in the foreground and watch for Segmentation Faults.
- Check logs in `lgsm/log/server/`.
- _Crash on Startup:_ Usually a missing map, missing Workshop collection, or syntax error in `server.cfg`.
- _Crash during gameplay:_ Often caused by poorly coded Lua in a Workshop addon or an entity exceeding the engine limits. Remove recent addons.

**Workshop Addon Conflicts:**
With 300+ mods, Lua conflicts are guaranteed.

- Check `garrysmod/data/errors.txt` (if using an error logging script).
- Disable half the addons, test, and binary-search to find the culprit.

**Network Connectivity Problems:**

- _"Connection failed after 4 retries"_: Check UFW ports (`sudo ufw status`) and ensure port forwarding is active on the router/VPS panel.

_Verification Step:_ Intentionally stop the server process using `kill -9 <pid>`. Wait 5 minutes for the LinuxGSM monitor cronjob to detect the crash and restart the server automatically. Check `lgsm/log/script/` to confirm the restart.

---

## 8. Additional Components

**Backup and Recovery:**
LinuxGSM includes a backup tool.

- Command: `./gmodserver backup`
- Archives are stored in `/home/gmodserver/lgsm/backup/`.
- **Offsite backup:** Setup an `rsync` cronjob to push these archives to an external S3 bucket or storage server.

**Server Migration Guide:**
To migrate to a new machine:

1. Setup Ubuntu 24.04 and install LinuxGSM on the new machine.
2. Archive the `serverfiles/garrysmod/data`, `serverfiles/garrysmod/cfg`, and the `lgsm/config-lgsm` directories.
3. Transfer via `scp`:
   ```bash
   scp backup.tar.gz gmodserver@new-server-ip:/home/gmodserver/
   ```
4. Extract and start the new server.

**Player Data Preservation:**
Always backup the `garrysmod/data/` folder and `garrysmod/sv.db` (SQLite database) before wiping or reinstalling. If using MySQL, schedule regular `mysqldump` backups:

```bash
mysqldump -u gmoduser -p ttt2_data > /home/gmodserver/ttt2_data_backup.sql
```

_Verification Step:_ Run `./gmodserver backup` and verify the `.tar.gz` file is created in `lgsm/backup/`.
