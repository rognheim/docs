---
title: Proxmox Backup Server
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-proxmox:


## **Proxmox Backup Server**

---

### **Introduction**

This page documents the backup-appliance role for the platform. The preferred pattern is a dedicated Proxmox Backup Server VM on Proxmox, with backup storage kept on the host and presented to the guest through VirtioFS.

---

### **Scope**

This page owns:

- the PBS VM baseline and host/guest boundary for backup infrastructure
- repository and update expectations inside the PBS guest
- the host-backed datastore pattern used in this environment
- validation checks for datastore availability and Proxmox integration

This page does not own hypervisor pool design beyond the backup dataset handoff, guest workload backup schedules, or application-specific restore workflows.

---

### **Dependencies / related pages**

- [Proxmox](../hypervisors/proxmox.md)
- [ZFS Overview](../file-systems/zfs-overview.md)
- [ZFS Settings](../file-systems/zfs-settings.md)
- [Debian](debian.md)
- [Secrets Management](../security/secrets-management.md)

---

### **Implementation**

#### 1. Create the PBS VM baseline

This guide assumes Proxmox Backup Server runs as a VM in Proxmox.

Recommended defaults:

- enable `Start at boot`
- machine type `q35`
- enable `Qemu Agent`
- use `VirtIO SCSI single`
- set CPU type to `host`
- allocate at least 4 vCPUs and 4 to 8 GiB RAM
- place the OS disk on local SSD-backed storage, not on the backup dataset itself

Keep the backup datastore separate from the guest OS disk.

#### 2. Apply the repository and update baseline

Add the non-subscription repositories to `/etc/apt/sources.list`:

```shell
deb http://deb.debian.org/debian bookworm main contrib
deb http://deb.debian.org/debian bookworm-updates main contrib

deb http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription

deb http://security.debian.org/debian-security bookworm-security main contrib
```

Comment out the enterprise repository in `/etc/apt/sources.list.d/pve-enterprise.list` if you are not using a subscription:

```shell
# deb https://enterprise.proxmox.com/debian/pbs bookworm pbs-enterprise
```

Update the guest:

```shell
apt-get update
apt dist-upgrade
reboot
```

#### 3. Prepare the host-backed ZFS datastore

Example dataset creation on the Proxmox host:

```shell
zfs create pool-z1/backups
zfs create pool-z1/backups/proxmox-backup-server
```

Optional quota:

```shell
zfs set quota=1T pool-z1/backups/proxmox-backup-server
```

Allow PBS write access to the datastore path. Inside PBS, the service user is typically `backups` with uid `34`.

```shell
chown 34:34 /pool-z1/backups/proxmox-backup-server
chmod 770 /pool-z1/backups/proxmox-backup-server
```

#### 4. Attach the datastore with VirtioFS

On the Proxmox host:

1. Open `Datacenter -> Directory Mappings -> Add`
2. Create a mapping for `/pool-z1/backups/proxmox-backup-server`
3. Add a `VirtioFS` device to the PBS VM using that mapping

Inside the PBS guest, create a mount point and mount it:

```shell
mkdir -p /mnt/proxmox-backup-server
mount -t virtiofs proxmox-backup-server /mnt/proxmox-backup-server
```

To mount automatically at boot:

```shell
proxmox-backup-server /mnt/proxmox-backup-server virtiofs rw,defaults,x-systemd.automount,_netdev 0 0
```

Add that line to `/etc/fstab`.

#### 5. Register the datastore and connect Proxmox nodes

In the PBS web UI:

- go to `Datastore -> Add`
- choose `/mnt/proxmox-backup-server`
- assign a clear datastore name

In Proxmox VE:

- go to `Datacenter -> Storage -> Add -> Proxmox Backup Server`
- point the node or cluster to the PBS instance
- configure credentials or API token access

---

### **Operations / validation**

Recommended checks after setup:

```shell
findmnt /mnt/proxmox-backup-server
proxmox-backup-manager datastore list
```

Validate from the Proxmox side:

- confirm the PBS storage entry is healthy in `Datacenter -> Storage`
- run a real backup job, not only a connectivity test
- verify garbage collection and prune jobs on the datastore
- perform at least one restore test for a representative VM or workload

Operational hygiene:

- monitor `zpool status`, SMART health, and available capacity on the Proxmox host
- schedule regular scrub operations on the backing pool
- keep PBS configuration backup and datastore backup concerns documented separately

---

### **Known issues**

- The older SMB-mount approach is retained only as historical context; VirtioFS is the preferred path for this environment.
- This page assumes the datastore lives on Proxmox-hosted ZFS storage. If backup storage is external, use the storage protocol and ownership model that matches that system instead.
