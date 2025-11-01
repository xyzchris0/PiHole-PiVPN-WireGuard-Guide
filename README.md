#  Pi-hole + PiVPN (WireGuard) Setup Guide for Raspberry Pi 5

A comprehensive guide to installing and configuring **Pi-hole** and **PiVPN (WireGuard)** on a Raspberry Pi 5 running Raspberry Pi OS Lite.

---

##  Table of Contents

1. [Overview](#overview)
2. [System Requirements](#system-requirements)
3. [Network Preparation](#network-preparation)
4. [Installation Steps](#installation-steps)
   - [1. Update System](#1-update-system)
   - [2. Set Static IP](#2-set-static-ip)
   - [3. Install Pi-hole](#3-install-pi-hole)
   - [4. Configure DNS and Admin UI](#4-configure-dns-and-admin-ui)
   - [5. Update VLAN DNS Settings](#5-update-vlan-dns-settings)
   - [6. Install PiVPN (WireGuard)](#6-install-pivpn-wireguard)
5. [Verification](#verification)
6. [Maintenance Commands](#maintenance-commands)
7. [Troubleshooting](#troubleshooting)
8. [Extra Notes](#extra-notes)
9. [License](#license)

---

## Overview

This guide assumes that you:
- You are using **Raspberry Pi OS Lite (64-bit)** in **headless** mode.
- You plan to deploy Pi-hole and PiVPN on a **dedicated VLAN** for isolation and security.

### Goals

- **Pi-hole:** A network-wide DNS ad blocker that filters ads, telemetry, and trackers.  
  Learn more about the Pi-Hole here: [https://pi-hole.net](https://pi-hole.net)
- **PiVPN (WireGuard):** A secure, self-hosted VPN server for encrypted remote access.  
  Learn more about the Pi-VPN here: [https://www.pivpn.io](https://www.pivpn.io)

---

##  System Requirements

| Component | Minimum | Recommended |
|------------|----------|-------------|
| Device | Raspberry Pi 4 | Raspberry Pi 5 |
| OS | Raspberry Pi OS Lite (64-bit) | Debian 12 (Bookworm) |
| Storage | 4 GB | 8 GB+ |
| Network | Ethernet | VLAN-aware setup (optional) |


---

##  Network Preparation

Pre for installation:

1. Confirm that your Pi has **internet connectivity**.  
2. Confirm your **VLAN/subnet configuration**.  
3. SSH into the Pi:
   ```bash
   ssh pi@<your_pi_ip>
   ```

To view your current interface and IP address:
```bash
ip a
```
Look for your active interface.

1. eth0 = ethernet
2. wlan0 = wireless internet

---

## Instal Steps

### 1. Update System

Update your system :
```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

---

### 2. Set up a Static IP

THis will help any issues of your IP changing on Reboot

Edit your DHCP configuration:
```bash
sudo nano /etc/dhcpcd.conf
```

Update the bottom at the bottom:
```bash
interface eth0
static ip_address=192.120.52.186/24
static routers=192.120.52.1
static domain_name_servers=192.120.52.1 1.1.1.1 8.8.8.8
```

Now you can apply changes:
```bash
sudo systemctl restart dhcpcd
ip a
```

>  **Note:** If you are using VLANs, your interface may appear as `eth0.52` instead of `eth0`.

Reboot:
```bash
sudo reboot
```

---

### 3. Install Pi-hole

Use the official install script:
```bash
curl -sSL https://install.pi-hole.net | bash
```

During setup:
1. Choose the correct network interface (`eth0` or `wlan0`).
2. Select your **Upstream DNS Provider** (e.g., Cloudflare or Quad9).
3. Accept or skip **third-party blocklists**.
4. Enable or disable **query logging**.
5. Choose a **privacy level**.
6. Set your admin password:
   ```bash
   pihole -a -p
   ```

---

### 4. Configure DNS and Admin UI

Access Pi-hole’s dashboard:
```
http://192.120.52.186/admin
```

Navigate to **Settings → DNS**:
- Enable “Listen on all interfaces” (or *Permit all origins* for VLAN use).
- Optional: Enable **Conditional Forwarding** if you want hostname resolution across subnets.

---

### 5. Update VLAN DNS Settings

On your router or VLAN configuration:
- Set **Primary DNS:** `192.120.52.186` (your Pi-hole)
- Set **Secondary DNS:** A backup (example: `1.1.1.1`)

Example VLAN layout:

| VLAN | Purpose | DNS |
|------|----------|-----|
| `.52` | Pi-hole / DNS | 192.120.52.186 |
| `.54` | IoT Network | 192.130.54.23 |
| `.52` | VPN Server | 192.120.52.186 |

---

### 6. Install PiVPN (WireGuard)

Run the official PiVPN script:
```bash
curl -L https://install.pivpn.io | bash
```

During setup:
1. Choose **WireGuard**.
2. Select your user account (`pi`).
3. Choose the interface (`eth0` or VLAN `eth0.52`).
4. Allow unattended upgrades.
5. Reboot:
   ```bash
   sudo reboot
   ```

To add VPN clients:
```bash
pivpn add
```
Client configuration files are stored in `/home/pi/configs/`. Import these into your WireGuard app.

---

##  Verification

Check Pi-hole status:
```bash
pihole status
```

Check WireGuard service:
```bash
sudo systemctl status wg-quick@wg0
```

Test DNS resolution:
```bash
dig google.com @192.120.52.186
```

Check your VPN connection by visiting:
```
https://whatismyipaddress.com
```

If the IP matches your home network, VPN routing is working correctly.

---

## Maintenance Commands

| Task | Command |
|------|----------|
| Update Pi-hole | `pihole -up` |
| Update PiVPN | `pivpn update` |
| Restart Pi-hole | `sudo systemctl restart pihole-FTL` |
| Restart WireGuard | `sudo systemctl restart wg-quick@wg0` |
| Show VPN peers | `sudo wg show` |
| Change Pi-hole password | `pihole -a -p` |
| Reconfigure PiVPN | `pivpn -r` |

---

## Troubleshooting

| Problem | Possible Fix |
|----------|---------------|
| DNS not resolving | Ensure clients use Pi-hole as DNS; restart `pihole-FTL` |
| Web UI not loading | Restart `lighttpd`: `sudo systemctl restart lighttpd` |
| VPN not connecting | Check that port `51820/udp` is open and forwarded on your router |
| Static IP lost | Recheck `/etc/dhcpcd.conf` syntax |
| Slow browsing | Disable conditional forwarding or change upstream DNS |

---

## Additional Notes

### Secure SSH Access
Harden SSH to prevent password logins:
```bash
sudo nano /etc/ssh/sshd_config
```
Set:
```
PermitRootLogin no
PasswordAuthentication no
```
Then restart SSH:
```bash
sudo systemctl restart ssh
```

### Backup Configurations
Backup Pi-hole and WireGuard configs:
```bash
sudo tar -czvf pihole_pivpn_backup.tar.gz /etc/pihole /etc/wireguard
```

---

##  License

MIT License © 2025 