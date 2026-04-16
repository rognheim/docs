---
title: UniFi OS Server
last_reviewed: 2026-03-07
owner: Rognheim
---

# :simple-ubiquiti:


## **UniFi OS Server**

---

### **Introduction**

The UniFi OS Server is the new standard for self-hosting UniFi, replacing the legacy UniFi Network Server. While the Network Server provided basic hosting functionality, it lacked support for key UniFi OS features like Organizations, IdP Integration, or Site Magic SD-WAN. With a fully unified operating system, UniFi OS Server now delivers the same management experience as UniFi-native–including CloudKeys, Cloud Gateways, and Official UniFi Hosting–and is fully compatible with Site Manager for centralized, multi-site control.

---

## **Install UniFi OS Server on Raspberry Pi**

!!! Info "This guide assumes you are installing UniFi OS Server on a Raspberry Pi"

This guide explains how to install **UniFi OS Server** on a Raspberry Pi and apply basic security hardening for production use.

!!! warning
    If you are migrating from **UniFi Network Server** to **UniFi OS Server**, stop and disable the Network Server before installing UniFi OS Server to avoid port conflicts and data corruption.

---

### **Prerequisites**

- Raspberry Pi 4 or newer (4GB+ RAM recommended)
- Raspberry Pi OS (64-bit) up to date
- Internet connectivity
- Non-root user with `sudo` privileges

---

#### 1. Update the System

Update package lists and upgrade installed packages:

```bash
sudo apt update && sudo apt upgrade -y
```

Reboot if the kernel or critical libraries were upgraded:

```bash
sudo reboot
```

---

#### 2. Install Required Dependencies

Install Podman and networking dependencies:

```bash
sudo apt install -y podman slirp4netns curl wget
```

Verify installation:

```bash
podman --version
```

---

#### 3. Download UniFi OS Server Installer

1. Navigate to the official UniFi OS Server **Releases** or **Download** page.
2. Right-click the download button.
3. Select **Copy link address**.

The URL will look similar to:

```
https://fw-download.ubnt.com/data/unifi-os-server/<version>-linux-arm64-<build-id>.sh
```

!!! tip
    Ensure you download the **ARM64** version for Raspberry Pi OS 64-bit.

Download using either `curl` or `wget`:

##### Using curl

```bash
curl -O <uos_server_download_link>
```

##### Using wget

```bash
wget <uos_server_download_link>
```

---

#### 4. Make the Installer Executable

```bash
chmod +x <downloaded_file_name>
```

Example:

```bash
chmod +x unifi-os-server-4.2.23-linux-arm64.sh
```

---

#### 5. Run the Installer

```bash
sudo ./<downloaded_file_name>
```

Example:

```bash
sudo ./unifi-os-server-4.2.23-linux-arm64.sh
```

The installer will:

- Configure Podman container
- Create required services
- Register `uosserver` systemd service

---

#### 6. Manage the UniFi OS Server Service

##### Start Service

```bash
sudo systemctl start uosserver
```

##### Stop Service

```bash
sudo systemctl stop uosserver
```

##### Enable Auto-Start at Boot

```bash
sudo systemctl enable uosserver
```

##### Disable Auto-Start at Boot

```bash
sudo systemctl disable uosserver
```

##### Check Service Status

```bash
sudo systemctl status uosserver
```

---

#### 7. Initial Setup

1. Open a browser and navigate to:

```
https://<raspberry-pi-ip>:11443
```

2. Create or sign in with a **UI Account**.
3. Complete the setup wizard.
4. Adopt UniFi devices (APs, switches, gateways).

For detailed adoption steps, refer to UniFi’s official device adoption documentation.

---

### **Basic Security Hardening**

Running UniFi OS Server on a Raspberry Pi in production requires baseline hardening.

---

#### 1. Change Default Password

If still using default credentials for the `pi` user:

```bash
passwd
```

Or disable the default user entirely:

```bash
sudo deluser pi
```

---

#### 2. Create a Dedicated Admin User

```bash
sudo adduser adminuser
sudo usermod -aG sudo adminuser
```

Disable SSH login for other users if not required.

---

#### 3. Secure SSH Access

Edit SSH configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Recommended settings:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

!!! warning
    Ensure SSH key authentication works before disabling password authentication.

---

#### 4. Configure a Firewall (UFW)

Install UFW:

```bash
sudo apt install -y ufw
```

Allow required services:

```bash
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 443/tcp       # HTTPS
sudo ufw allow 80/tcp        # HTTP (optional)
```

Enable firewall:

```bash
sudo ufw enable
```

Check status:

```bash
sudo ufw status
```

---

#### 5. Enable Automatic Security Updates

Install unattended upgrades:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

---

#### 6. Set Static IP Address

For production deployments, configure a static IP via:

- Router DHCP reservation (recommended), or
- `/etc/dhcpcd.conf`

This ensures stable device adoption and controller availability.

---

#### 7. Regular Backups

Inside UniFi OS:

- Enable **automatic backups**
- Download periodic manual backups
- Store backups off-device---

---

<!-- ### **Configuration files**

- Not documented yet or not applicable for this topic.

---

### **Installation**

- To be documented.

---

### **Post-install tasks**

- To be documented.

--- -->

### **Known issues**

- No known issues documented yet.

