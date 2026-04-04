# Paul & Friends TTT - Server Documentation (TTT2)

This document provides comprehensive technical instructions for deploying, configuring, and maintaining the "Paul & Friends TTT" Garry's Mod server running Trouble in Terrorist Town 2 (TTT2). It is designed for Ubuntu 24.04 LTS using LinuxGSM.

---

## 1. Initial Server Setup

**System Requirements (for 300+ mods):**

- **CPU:** High single-core performance (3.5GHz+ recommended). Garry's Mod is predominantly single-threaded.
- **RAM:** Minimum 8GB (16GB recommended due to the heavy mod load and engine memory leaks).
- **Storage:** 50GB+ NVMe/SSD for OS, server files, and Workshop content.
- **Network:** 1Gbps uplink for smooth 66-tick performance.

**Dependency Installation (Ubuntu 24.04 LTS):**
Ubuntu 24.04 (Noble Numbat) requires enabling the 32-bit architecture to run Source engine binaries and SteamCMD.

```bash
sudo timedatectl set-timezone Europe/Berlin
sudo dpkg --add-architecture i386
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget file tar bzip2 gzip unzip bsdmainutils python3 util-linux ca-certificates binutils bc jq tmux netcat-openbsd lib32gcc-s1 lib32stdc++6 libsdl2-2.0-0:i386 steamcmd mariadb-server fail2ban pigz -y
```

**User Account Creation:**
For security reasons, the server must never run as `root`.

```bash
sudo adduser gmodserver
# Follow the prompts to set a strong password.
# Switch to the new user:
su - gmodserver
```

**Optional: Sudo Privileges for gmodserver Service**
Grant tightly scoped sudo so `gmodserver` can only manage its own systemd service.

```bash
# Create a systemd unit (if not already present); runs as the gmodserver user
sudo tee /etc/systemd/system/gmodserver.service > /dev/null << 'EOF'
[Unit]
Description=LinuxGSM Garry's Mod Server (gmodserver)
After=network.target

[Service]
User=gmodserver
WorkingDirectory=/home/gmodserver
ExecStart=/home/gmodserver/gmodserver start
ExecStop=/home/gmodserver/gmodserver stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable gmodserver
```

Configure restricted sudoers entry:

```bash
sudo visudo -f /etc/sudoers.d/gmodserver
```

Paste:

```
Defaults:gmodserver env_reset
Defaults:gmodserver secure_path="/usr/sbin:/usr/bin:/sbin:/bin"
gmodserver ALL=(root) /usr/bin/systemctl start gmodserver, /usr/bin/systemctl stop gmodserver, /usr/bin/systemctl restart gmodserver, /usr/bin/systemctl status gmodserver
```

_Press `Ctrl+O` to save, then `Ctrl+X` to exit._

Security best practices:

- Do not use `NOPASSWD` unless strictly necessary; requiring a password is safer.
- Limit to the exact commands and unit name; never grant broad `systemctl` access.
- Keep `/etc/sudoers.d/gmodserver` owned by root and mode `440`.

Verification:

- List privileges: `sudo -l` (as gmodserver).
- Start service: `sudo systemctl start gmodserver` then `sudo systemctl status gmodserver`.
- Attempt an unapproved command (e.g. `sudo systemctl reboot`) should be denied.

Troubleshooting:

- If `visudo` reports errors, fix syntax before saving.
- If `systemctl` path differs (`which systemctl`), update `secure_path` and command entries.
- Ensure the unit exists: `systemctl list-unit-files | grep gmodserver`.

**Firewall Configuration (UFW):**
GMod and LinuxGSM require specific ports. Ensure UFW is active.

```bash
sudo ufw enable
sudo ufw allow 27015/udp # Game Traffic
sudo ufw allow 27005/udp # Client Port
sudo ufw allow 27020/udp # SourceTV
sudo ufw allow 22/tcp    # SSH
sudo ufw reload
```

**Optional: SSH Key Authentication**
Set up key-based SSH for `gmodserver` and disable password login.

Generate key on your admin machine:

```bash
ssh-keygen -t ed25519 -C "PaulsTTT admin" -f ~/.ssh/pauls-ttt
```

Deploy the public key to the server:

```bash
# Replace <server-ip> accordingly
ssh gmodserver@ < server-ip > 'mkdir -p ~/.ssh && chmod 700 ~/.ssh'
scp ~/.ssh/pauls-ttt.pub gmodserver@ < server-ip > :/home/gmodserver/.ssh/
ssh gmodserver@ < server-ip > 'cat ~/.ssh/pauls-ttt.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chown -R gmodserver:gmodserver ~/.ssh'
```

Harden SSH daemon:

```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

Verification:

- Connect with key: `ssh -i ~/.ssh/pauls-ttt gmodserver@<server-ip>` then `whoami` (should print `gmodserver`).
- Test sudo prompt: `sudo -v` then `sudo -l`.
- Ensure UFW shows SSH open: `sudo ufw status`.

Troubleshooting:

- `Permission denied (publickey)`: check file modes (`~/.ssh` 700, `authorized_keys` 600), ownership (`gmodserver:gmodserver`).
- Use `ssh -vv gmodserver@<server-ip>` to debug auth.
- If locked out, use console access to temporarily set `PasswordAuthentication yes`, restart SSH, then fix keys and revert.

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
nano /home/gmodserver/lgsm/config-lgsm/gmodserver/gmodserver.cfg
```

Add/modify the following parameters:

```ini
gslt=""
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

_Verification Step:_ Run `crontab -l` to ensure cronjobs are saved.

---

## 3. Garry's Mod Server Configuration

**`server.cfg` Customization:**
File: `/home/gmodserver/serverfiles/garrysmod/cfg/gmodserver.cfg`

**`autoexec.cfg`:**
File: `/home/gmodserver/serverfiles/garrysmod/cfg/autoexec.cfg`
Used for commands that must execute before the first map loads.

```ini
sv_minrate 1048576
sv_maxrate 0
sv_minupdaterate 66
sv_maxupdaterate 66
sv_mincmdrate 66
sv_maxcmdrate 66
```

**Mounting Counter-Strike: Source (CS:S) Content:**
Many Garry's Mod maps and addons (especially in TTT) require Counter-Strike: Source assets. To mount CS:S content to your `gmodserver` instance, follow these steps:

1. **Obtain CS:S Content and copy to GMod Server:**
   Install content to be mounted using LinuxGSM to download the required assets. Refer to the [official LinuxGSM CS:S server guide](https://linuxgsm.com/servers/cssserver/) for complete installation instructions.

   ```bash
   adduser cssserver
   su - cssserver
   curl -Lo linuxgsm.sh https://linuxgsm.sh && chmod +x linuxgsm.sh && bash linuxgsm.sh cssserver
   ./cssserver auto-install
   cp -R /home/cssserver/serverfiles/cstrike /home/gmodserver/serverfiles/cstrike
   chown -R gmodserver:gmodserver /home/gmodserver/serverfiles/cstrike
   ```

2. **Mount Content in `mount.cfg`:**
   Edit the mount configuration file:

   ```bash
   nano /home/gmodserver/serverfiles/garrysmod/cfg/mount.cfg
   ```

   Update the file to include the absolute path to the directories for each game:

   ```ini
   "mountcfg"
   {
     "cstrike"	"/home/gmodserver/css/cstrike"
   }
   ```

_Verification Step:_ Start the server (`./gmodserver start`). Check the console (`./gmodserver console`) to verify the 66 tickrate and map loading without errors. To verify the CS:S mount was successful, change the level to a mounted CS:S map (e.g., `changelevel cs_italy`) via the console. Press `CTRL+B` then `D` to detach from the tmux console.

---

## 4. Performance and Stability Optimization

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

## 5. Security Implementation

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
Protect SSH from brute-force attacks:

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

_Verification Step:_ Attempt to connect via SSH using a password (should be rejected). Attempt to use an RCON tool (should fail).

---

## 6. Troubleshooting Guide

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

## 7. Additional Components

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
