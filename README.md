# Satisfactory Dedicated Server (Linux + Tailscale)  
Private Server Setup Guide

This document explains how to install, configure, secure, and manage a **Satisfactory Dedicated Server** running on **Ubuntu Server (headless)**, with **Tailscale** for secure remote access.  
All steps reflect the exact process used to set up the server on my System76 Darter Pro laptop.

---

## Quick Reference Commands

### Server Control
```bash
sudo systemctl start satisfactory      # Start server
sudo systemctl stop satisfactory       # Stop server
sudo systemctl restart satisfactory    # Restart server
sudo systemctl status satisfactory     # Check status
```

### Logs & Monitoring
```bash
journalctl -u satisfactory -f          # Watch live logs
journalctl -u satisfactory -n 100      # View last 100 lines
```

### Server Info
```bash
tailscale ip                           # Get Tailscale IP address
systemctl is-active satisfactory       # Quick status check
```

### Updates
```bash
sudo systemctl stop satisfactory
sudo -u satisfactory /opt/steamcmd/steamcmd.sh +force_install_dir /opt/satisfactory +login anonymous +app_update 1690800 validate +quit
sudo systemctl start satisfactory
```

### File Uploads (from Windows PowerShell)
```powershell
scp C:\path\to\file user@100.x.x.x:/opt/satisfactory/
scp -r C:\path\to\folder user@100.x.x.x:/opt/satisfactory/
```

---

## Table of Contents
- [Overview](#overview)
- [System Requirements](#system-requirements)
- [1. Base System Setup](#1-base-system-setup)
- [2. Create Satisfactory User](#2-create-satisfactory-user)
- [3. Install SteamCMD](#3-install-steamcmd)
- [4. Install Satisfactory Server](#4-install-satisfactory-server)
- [5. Systemd Service Setup](#5-systemd-service-setup)
- [6. Firewall Rules](#6-firewall-rules)
- [7. Tailscale Setup](#7-tailscale-setup)
- [8. Connecting From Windows or Remote Machines](#8-connecting-from-windows-or-remote-machines)
- [9. Managing the Server](#9-managing-the-server)
- [10. Updating the Server](#10-updating-the-server)
- [11. Restarting / Stopping / Logs](#11-restarting--stopping--logs)
- [12. Uploading Files to the Server](#12-uploading-files-to-the-server)
- [13. Troubleshooting](#13-troubleshooting)

---

## Overview

This setup provides:

- Headless Ubuntu Server  
- SteamCMD-based Satisfactory server installation  
- Managed via `systemd`  
- Secure remote multiplayer via Tailscale  
- No port forwarding necessary  
- Clean folder structure under `/opt/`  
- Easy updates and automatic restarts

---

## System Requirements

- Ubuntu Server 22.04+ or 24.04+  
- 8 GB RAM minimum
- Fast CPU (single-core performance matters)  
- SSD recommended  
- Internet connectivity  
- Tailscale installed on all players’ machines

---

## 1. Base System Setup

Update the OS:

```bash
sudo apt update
sudo apt upgrade -y
```

Install basic dependencies:

```bash
sudo apt install curl wget lib32gcc-s1 lib32stdc++6 -y
```

---

## 2. Create Satisfactory User

```bash
sudo useradd -m satisfactory
sudo passwd satisfactory
sudo usermod -aG sudo satisfactory
```

---

## 3. Install SteamCMD

Create install directory:

```bash
sudo mkdir -p /opt/steamcmd
sudo chown satisfactory:satisfactory /opt/steamcmd
cd /opt/steamcmd
```

Download and extract:

```bash
sudo -u satisfactory wget https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz
sudo -u satisfactory tar -xvzf steamcmd_linux.tar.gz
```

---

## 4. Install Satisfactory Server

Create server directory:

```bash
sudo mkdir -p /opt/satisfactory
sudo chown satisfactory:satisfactory /opt/satisfactory
```

Install via SteamCMD:

```bash
sudo -u satisfactory /opt/steamcmd/steamcmd.sh
```

At the `Steam>` prompt:

```plaintext
login anonymous
force_install_dir /opt/satisfactory
app_update 1690800 validate
quit
```

Successful install ends with:

```plaintext
Success! App '1690800' fully installed.
```

---

## 5. Systemd Service Setup

Create the service file:

```bash
sudo nano /etc/systemd/system/satisfactory.service
```

Paste:

```ini
[Unit]
Description=Satisfactory Dedicated Server
After=network.target

[Service]
Type=simple
User=satisfactory
WorkingDirectory=/opt/satisfactory
ExecStart=/opt/satisfactory/FactoryServer.sh -log -unattended -multihome=0.0.0.0
Restart=on-failure
RestartSec=10

# Optional resource limits:
# CPUQuota=200%
# MemoryHigh=10G
# MemoryMax=12G

[Install]
WantedBy=multi-user.target
```

Enable and start service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable satisfactory
sudo systemctl start satisfactory
```

Check status:

```bash
sudo systemctl status satisfactory
```

Watch logs:

```bash
journalctl -u satisfactory -f
```

---

## 6. Firewall Rules (I didn't do this)

If `ufw` is enabled:

```bash
sudo ufw allow 15777/udp
sudo ufw allow 15000/udp
sudo ufw allow 7777/udp
sudo ufw reload
```

On a LAN-only or Tailscale-only setup, port forwarding is not needed.

---

## 7. Tailscale Setup

Install:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Authenticate via the URL provided.

Check Tailscale IP:

```bash
tailscale ip
```

Output will look like:

```plaintext
100.x.x.x
fd7a:115c:a1e0::xxxx
```

That `100.x.x.x` address is your Satisfactory server address.

---

## 8. Connecting From Windows or Remote Machines

On each client machine:

1. Install Tailscale: https://tailscale.com/download  
2. Log in  
3. (If needed) owner approves device at https://login.tailscale.com/admin/machines  
4. Launch Satisfactory → Server Manager → Add Server  

Enter:

```plaintext
100.x.x.x   (the server Tailscale IP)
```

Set admin password and create/load a save.

---

## 9. Managing the Server

Start the server:

```bash
sudo systemctl start satisfactory
```

Stop the server:

```bash
sudo systemctl stop satisfactory
```

Restart the server:

```bash
sudo systemctl restart satisfactory
```

Watch live logs:

```bash
journalctl -u satisfactory -f
```

---

## 10. Updating the Server

### Manual Update

Stop the service:

```bash
sudo systemctl stop satisfactory
```

Update using SteamCMD:

```bash
sudo -u satisfactory /opt/steamcmd/steamcmd.sh +force_install_dir /opt/satisfactory +login anonymous +app_update 1690800 validate +quit
```

Restart:

```bash
sudo systemctl start satisfactory
```

### Automated Daily Updates

The server updates automatically at **6 AM PST/PDT** via cron job.

**View current cron schedule:**

```bash
sudo crontab -l
```

**Edit cron schedule:**

```bash
sudo crontab -e
```

**Current automated update:**

```cron
0 6 * * * TZ=America/Los_Angeles /home/satisfactory/update_satisfactory.sh
```

**View update logs:**

```bash
sudo tail -f /var/log/satisfactory_updates.log
```

**View last 50 log entries:**

```bash
sudo tail -n 50 /var/log/satisfactory_updates.log
```

**Disable automatic updates:**

```bash
sudo crontab -e
# Comment out the update line by adding # at the start:
# 0 6 * * * TZ=America/Los_Angeles /home/satisfactory/update_satisfactory.sh
```

**Note:** The cron job runs with root privileges and automatically adjusts for daylight saving time (PST/PDT). Updates include stopping the server, downloading latest files via SteamCMD, and restarting the service.

---

## 11. Restarting / Stopping / Logs

Common commands:

```bash
sudo systemctl status satisfactory
sudo systemctl restart satisfactory
sudo systemctl stop satisfactory
journalctl -u satisfactory -f
```

---

## 12. Uploading Files to the Server

Upload from Windows (PowerShell):

Upload a file:

```powershell
scp C:\path\to\file user@100.x.x.x:/opt/satisfactory/
```

Upload an entire folder:

```powershell
scp -r C:\path\to\folder user@100.x.x.x:/opt/satisfactory/
```

If using Tailscale, replace `100.x.x.x` with the server’s Tailscale IP.

If filenames contain spaces, wrap them in quotes:

```powershell
scp "C:\path\My Blueprint.sbp" user@100.x.x.x:/opt/satisfactory/
```

View uploaded files:

```bash
ls /opt/satisfactory
```

---

## 13. Troubleshooting

### Server not appearing in Satisfactory
- Ensure Tailscale is running on all devices  
- Check server status:

```bash
sudo systemctl status satisfactory
```

- Confirm Tailscale IP:

```bash
tailscale ip
```

### Log shows no activity
Check logs:

```bash
journalctl -u satisfactory -f
```

### Server shut down when laptop lid closed
Disable all sleep/suspend modes:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Allow lid closure without suspending:

```bash
sudo nano /etc/systemd/logind.conf
```

Find and modify these lines (remove `#` if commented):

```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Restart the service:

```bash
sudo systemctl restart systemd-logind
```

### Initial SteamCMD install loops
Run SteamCMD interactively the first time:

```bash
/opt/steamcmd/steamcmd.sh
```

---

This README fully documents the installation, networking setup, and long-term maintenance of the Satisfactory Dedicated Server.