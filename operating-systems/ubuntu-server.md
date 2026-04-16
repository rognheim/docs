---
title: Ubuntu Server
status: working
last_reviewed: 2026-03-07
owner: Rognheim
---

# :fontawesome-brands-ubuntu:


## **Ubuntu Server**

---

!!! Warning 

    I no longer use Ubuntu Server for any of my VM's. Consider this page outdated.

---

### **Introduction**

Ubuntu Server is a popular, open-source operating system designed for professional and self-hosted server environments. Offering stability, performance, and security, it supports a wide range of applications, including web, email, file, and database servers. With its extensive software repository, regular updates, and strong community support, Ubuntu Server has become a preferred choice for individuals seeking a versatile and reliable platform for their self-hosted services.

---

### **VM Configuration**

!!! Note "This guide assumes you are installing Ubuntu-Server as a VM in Proxmox"

- Right-click Node and Create VM.

- General: Enable advanced settings and enable **start at boot**.

- OS: Use ISO image and set Guest OS to Linux.

- System: Enable **Qemu Agent**. We will install this later. Set the SCSI Controller to **VirtIO SCSI single**.

- Disks: Give the installation 64 GB of space. Enable **SSD emulation**. Enable **Discard** in order to enable thin provisioning. Also make sure that **IO thread** is enabled.

- CPU: Change the CPU type to **host** and give it a minimum of 4 Cores. There is no need to enable any CPU flags when using CPU type as host. The Guest VM will get complete access to the host CPU.

- Memory: Enable **Balooning Device** and set Memory to 4 GB (4096 MiB).

- Network: Leave as default.

---

### **Installation**

- Go through the Ubuntu Server Installer

---

### **Post-install tasks**

```shell
sudo apt-get update
```

```shell
sudo apt-get upgrade
```

```shell
sudo apt-get install qemu-guest-agent
```

- Enable QEMU Guest Agent in Proxmox. VM -> Options -> QEMU Guest Agent -> enable. Then restart the VM.

#### Configure the timezone

```shell
sudo timedatectl set-timezone "your_time_zone" eg. Europe/City
```

#### Configure static IP

```shell
sudo cp 50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
```

```shell
sudo rm 50-cloud-init.yaml
```

```shell
sudo touch 99-config.yaml && sudo nano 99-config.yaml
```

```shell
sudo chmod 600 99-config.yaml
```

```shell
network:
  version: 2
  renderer: networkd
  ethernets:
    enp6s18:
      addresses:
        - 192.168.10.XX/24
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
          search: [domain.example]
          addresses: [9.9.9.9, 8.8.8.8]
```

```shell
sudo netplan apply
```

---

### **Install Docker**

1. [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
2. [Linux post-installation steps for Docker Engine](https://docs.docker.com/engine/install/linux-postinstall/)

---

### **(Optional) Execute docker commands without sudo**

```shell
sudo usermod -aG docker ${USER}
```

restart the VM for the changes to take effect

---

### **Docker post-install tasks**

```bash
sudo mkdir docker-appdata /opt
```

```bash
sudo git clone [private docker repository]
```

```bash
sudo chown -R "username":"username" "/opt/docker-appdata/compose-files"
```

create a user-defined bridge network in docker called "local"

```shell
docker network create local
```

inspect the network to locate the default gateway. this is important when setting up traefik

```shell
docker network inspect local
```

---

### **Mounting a TrueNAS SMB share in Ubuntu**

install cifs-utils by running the following command:

```shell
sudo apt-get install cifs-utils
```

create a file to store the SMB credentials

```shell
sudo nano /etc/.smb_creds
```

add the following info to the file

```shell
username="smb_username"
password="smb_password"
```

modify the file permission to 600.

```shell
sudo chmod 600 /etc/.smb_creds
```

on the ubuntu host, create a folder where you want to mount the contents of the TrueNAS SMB share.

```shell
sudo mkdir /mnt/"folder"/
```

modify `/etc/fstab` with the following.

```shell
sudo nano /etc/fstab
```

```shell
//ip-of-nas-server/enter-remote-samba-share/location /enter-local-mount/location/here/ cifs credentials=/etc/.truenas_creds,iocharset=utf8,uid=enter_your_uid_here,gid=enter_your_gid_here,noperm 0 0
```

example:

```shell
//192.168.10.143/media /mnt/media/ cifs credentials=/etc/.smb_creds,iocharset=utf8,uid=1000,gid=1000,noperm 0 0
```

once the path is added to fstab run

```shell
sudo systemctl daemon-reload
```

```shell
sudo mount -a
```

---

### **Known issues**

#### 1. TRIM not automatically running

```bash
sudo fstrim -av
```

adding to daily crontab right before daily backups in Proxmox to limit the size of the backups

```bash
sudo crontab -e
```

add 'fstrim -av' to crontab

```shell
### [MANUAL TRIM SCHEDULE]

# run TRIM manually every day at 03:30
# 30 3 * * * fstrim -av
```

CTRL + X to save---

### **Configuration files**

- Not documented yet or not applicable for this topic.

