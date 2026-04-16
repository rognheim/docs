---
title: Plex Media Server
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-plex:


## **Plex Media Server**

---

### **Introduction**

This page documents the current Plex Media Server pattern for the platform. The working baseline is an unprivileged Debian LXC on Proxmox with optional Intel iGPU passthrough for hardware transcoding and carefully controlled media mounts.

---

### **Scope**

This page owns:

- the LXC-specific deployment assumptions for Plex
- hardware-transcoding access from Proxmox into the guest
- post-install media-mount considerations for this environment

This page does not own reverse-proxy policy, request workflows, or the wider media-automation stack.

---

### **Dependencies / related pages**

- [Proxmox](../hypervisors/proxmox.md)
- [Docker](../infrastructure-tools/docker.md)
- [qBittorrent](qbittorrent.md)
- [Sonarr](sonarr.md)
- [Radarr](radarr.md)
- [Tautulli](tautulli.md)

---

### **Implementation**

!!! Info "Debian 12 `Unprivileged` LXC with Proxmox"

---

#### Install Plex Media Server

[Official Plex Linux Repository Documentation](https://support.plex.tv/articles/235974187-enable-repository-updating-for-supported-linux-server-distributions/)

**1.** Update the container

```shell
apt update && apt upgrade -y
```

**2.** Install `curl` to enable fetching of URLs and `gnupg` for secure authentication

```shell
apt install curl gnupg -y
```

**3.** Enable `Plex Media Server` repository

```shell
echo deb https://downloads.plex.tv/repo/deb public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
```

```shell
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
```

**4.** Install Plex Media Server

```shell
apt update && apt install plexmediaserver -y
```

**5.** Install `Intel(R) Graphics Compute Runtime`

```shell
apt install intel-opencl-icd
```

**6.** Find the IP adress of the container

```shell
ip a | grep eth0
```

**7.** Open a browser and navigate to http://IP:32400/manage to complete the installation

---

#### Enable Intel iGPU Quicksync for Hardware Transcoding

!!! Info "Ensure that the iGPU is enabled in BIOS"

**1.** On the Proxmox VE host

Run the following command

```shell
ls -l /dev/dri
```

You will get something like this:

```shell
root@pve:~# ls -l /dev/dri/
total 0
drwxr-xr-x 2 root root        100 Apr 10 08:53 by-path
crw-rw---- 1 root video  226,   0 Apr 10 08:53 card0
crw-rw---- 1 root video  226,   1 Apr 10 08:53 card1
crw-rw---- 1 root render 226, 128 Apr 10 08:53 renderD128
```

!!! Warning "If you don’t see the above, or see “No such file or directory”, then your Intel iGPU is not enabled properly. Please check your BIOS settings."

**2.** Find the group owner of renderD128 on Proxmox

```shell
grep render /etc/group
```
```shell
render:x:103:
```

**3.** Find the group owner of renderD128 on Plex LXC

The renderD128 device is all that is needed by the Plex LXC. It is owned by group render on Proxmox, with group ID 103. Now we need to link the render group from Proxmox to the Plex LXC. However, in the Plex LXC the render group may have a different ID. So we do the same command in the Plex LXC to find the group owner.

```shell
grep render /etc/group
```
```shell
render:x:106:
```

**4.** Add plex to the render group on the Plex LXC

```shell
groupmod -a -U plex render
```

**5.** Verify that plex is added to video and render

```shell
cat /etc/group | grep plex
```
```shell
video:x:44:plex
render:x:106:plex
plex:x:996:
```

**6.** On Proxmox, add the following lines to the bottom of the container configuration file located at `/etc/pve/lxc/<CONTAINER ID>.conf`

```shell
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0 0
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 106
lxc.idmap: g 106 103 1
lxc.idmap: g 107 100107 65429
```
```shell
# Syntax: lxc.idmap
# Column 1: u/g define map for user or group ids
# Column 2: Range start for container
# Column 3: Range start for host
# Column 4: Length of range
# i.e., g 0 100000 106 = Map gids 100000-100106 on host to 0-106 in container
```


**7.** Add the following line to `/etc/subgid`

```shell
root:106:1
```

**8.** Reboot the container

```shell
reboot now
```

---

### **Operations / validation**

#### Mounting a SMB share in an unprivileged LXC

Normally SMB shares are unwritable and unreadable unless the LXC is ran as a privileged container. Runnin a privileged container comes with many security risks so here I will show you a clever way to mount a SMB share in an unprivileged LXC.

**1.** On the Proxmox host, install cifs-utils by running the following command: (Should already be installed)

```shell
apt install cifs-utils
```

**2.** Create a file to store the SMB credentials

```shell
nano /etc/.smb_creds
```

**3.** Add the following info to the file

```shell
username="smb_username"
password="smb_password"
```

**4.** Modify the file permission to 600

```shell
chmod 600 /etc/.smb_creds
```

!!! Info "The first digit `6` represents the permissions for the file's owner. `6` stands for "read and write", where `4` stands for "read" and `2` stands for "write". So, the owner can read and write the file but cannot execute it as a program."

!!! Info "The second digit `0` represents the permissions for the group that owns the file. `0` stands for no permissions, meaning members of the group cannot read, write, or execute the file."

!!! Info "The third digit, also `0`, defines the permissions for all other users. `0` here again means no permissions for others."

!!! Success "So effectively the command chmod 600 /etc/.smb_creds is setting the permissions on the file /etc/.smb_creds so that only the owner can read and write it, while the group owner and others have no permissions. This is commonly used for sensitive files, such as files that store passwords or other credentials, in this case Samba (SMB) credentials."

**5.** Modify `/etc/fstab` with the following

```shell
//ip-of-nas-server/enter-remote-samba-share/location /enter-local-mount/location/here/ cifs credentials=/etc/.truenas_creds,iocharset=utf8,uid=enter_your_uid_here,gid=enter_your_gid_here 0 0
```

Example: (With path to inside the LXC subvolume)

```shell
//192.168.10.10/media2 /rpool/data/subvol-220-disk-0/mnt/media cifs credentials=/etc/.smb_creds,iocharset=utf8,uid=100000,gid=100000 0 0
```

Once the path is added to fstab run

```shell
systemctl daemon-reload
```

Then finally run

```shell
mount -a
```

---

### **Known issues**

#### 1. Proxmox startup delay

- [X] Remember to put a delay on startup so that TrueNAS starts before the Plex LXC

#### 2. Resource usage

- Requires more resources when Plex must transcode container, codec, or subtitle formats for limited client devices.

#### 3. Mount dependencies

- If media is mounted from another storage appliance, startup ordering matters. Do not let Plex boot before the remote storage path is actually available.

---

### **Configuration files**

- Proxmox LXC config: `/etc/pve/lxc/<container-id>.conf`
- Host mount definition: `/etc/fstab`
- SMB credentials file when using CIFS mounts: `/etc/.smb_creds`
- Plex defaults and service data inside the LXC:
  - `/etc/default/plexmediaserver`
  - `/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/`
