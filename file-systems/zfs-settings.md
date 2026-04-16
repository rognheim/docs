---
title: ZFS Settings
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-openzfs:

## **ZFS Settings**

---

### **Introduction**

This page records the ZFS properties that are worth setting deliberately in this platform. The goal is not to tune every knob. The goal is to use a small, predictable set of dataset properties that match the workload and are easy to explain later.

---

### **Scope**

This page owns:

- baseline ZFS property choices
- workload-specific dataset examples
- validation commands after property changes

This page does not own:

- pool topology decisions. See [ZFS Overview](zfs-overview.md)
- backup job design. See [Proxmox Backup Server](../operating-systems/proxmox-backup-server.md)

---

### **Prerequisites**

- A working ZFS pool already created.
- A clear dataset boundary for the workload you want to tune.
- A rollback point before changing properties on important data.

---

### **Dependencies / related pages**

- [ZFS Overview](zfs-overview.md)
- [Proxmox](../hypervisors/proxmox.md)

---

### **Configuration files**

- Dataset properties stored in ZFS itself
- `/etc/modprobe.d/zfs.conf` for optional ARC tuning on the host

---

### **Implementation**

#### **1. Start with a simple baseline**

These are the defaults worth setting on many ordinary datasets:

```bash
zfs set compression=lz4 tank/appdata
zfs set atime=off tank/appdata
zfs set xattr=sa tank/appdata
```

Why these matter:

- `compression=lz4`: good default balance of savings and speed
- `atime=off`: avoids extra writes for read-heavy workloads
- `xattr=sa`: a sensible default for Linux metadata-heavy workloads

#### **2. Use workload-specific datasets**

Do not force one dataset to fit every access pattern. Create separate datasets when record size, retention, or sharing semantics differ.

Examples:

```bash
zfs create tank/appdata
zfs create tank/media
zfs create tank/backups
zfs create tank/timemachine
```

#### **3. Apply settings by workload**

##### **General application data**

```bash
zfs set compression=lz4 tank/appdata
zfs set atime=off tank/appdata
zfs set recordsize=16K tank/appdata
```

Use this for databases or small-file-heavy application data only when testing shows the smaller `recordsize` helps. If not, keep the default.

##### **Large media files**

```bash
zfs set compression=lz4 tank/media
zfs set atime=off tank/media
zfs set recordsize=1M tank/media
```

This keeps large sequential files efficient and avoids tuning the media dataset like a database.

##### **Backup datasets**

```bash
zfs set compression=lz4 tank/backups
zfs set atime=off tank/backups
zfs set sync=standard tank/backups
zfs set quota=4T tank/backups
```

The important part is predictability. Backups should not consume the entire pool because one retention job drifted.

##### **macOS Time Machine style datasets**

```bash
zfs create -o casesensitivity=insensitive tank/timemachine
zfs set xattr=sa tank/timemachine
zfs set quota=1T tank/timemachine
```

`casesensitivity=insensitive` must be chosen at dataset creation time.

#### **4. Keep dangerous settings off unless you can justify them**

Avoid turning these on casually:

- `dedup=on`
- `sync=disabled`
- host ARC tuning without a measured memory-pressure reason

Examples of defaults worth preserving:

```bash
zfs set dedup=off tank/appdata
zfs set sync=standard tank/appdata
```

#### **5. Tune host ARC only when the host is actually under pressure**

Temporary inspection:

```bash
cat /proc/spl/kstat/zfs/arcstats | head
```

Persistent ARC cap example:

```bash
echo "options zfs zfs_arc_max=4294967296" | sudo tee /etc/modprobe.d/zfs.conf
sudo update-initramfs -u
```

Only do this when the host is starved for RAM by more important workloads. Do not cargo-cult ARC caps onto every system.

---

### **Validation / troubleshooting**

#### **Basic validation**

Check the effective properties:

```bash
zfs get compression,atime,xattr,recordsize,quota,sync tank/appdata
zfs get compression,atime,recordsize tank/media
```

Check space behavior:

```bash
zfs list -o space tank
```

Check pool health after property changes on busy systems:

```bash
zpool status -v
zpool iostat -v 1
```

#### **Troubleshooting checklist**

- If a dataset performs poorly:
  - confirm the workload matches the chosen `recordsize`
  - check whether the pool is simply saturated
- If a quota behaves unexpectedly:
  - inspect snapshots and refreservations before assuming live data is the problem
- If host memory pressure increases:
  - inspect ARC use before setting an arbitrary cap

---

### **Known issues**

- `casesensitivity` is a creation-time choice. If it is wrong, the clean fix is usually a new dataset plus data migration.
- Over-tuning ZFS is a common self-inflicted problem. Use a small number of intentional settings and document why each one exists.
