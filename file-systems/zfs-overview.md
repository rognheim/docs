---
title: ZFS Overview
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-openzfs:

## **ZFS Overview**

---

### **Introduction**

ZFS is the storage baseline for the platform. It is used because it combines pooled storage, checksummed data integrity, snapshots, and flexible dataset boundaries in one model that works well for hypervisor storage, backup targets, and service data with different retention needs.

---

### **Scope**

This page owns:

- the storage model used across the active docs
- pool and dataset design guidance
- essential operator commands for pool health, snapshots, and recovery work

This page does not own:

- per-workload property tuning. See [ZFS Settings](zfs-settings.md)
- backup orchestration and restore workflows in Proxmox Backup Server. See [Proxmox Backup Server](../operating-systems/proxmox-backup-server.md)

---

### **Prerequisites**

- A system with ZFS support, typically Proxmox or Debian.
- A clear decision on pool topology before creation.
- A naming scheme for pools and datasets that reflects ownership and lifecycle.

---

### **Dependencies / related pages**

- [ZFS Settings](zfs-settings.md)
- [Proxmox](../hypervisors/proxmox.md)
- [Proxmox Backup Server](../operating-systems/proxmox-backup-server.md)

---

### **Configuration files**

- On-disk pool metadata managed by ZFS itself
- `/etc/modprobe.d/zfs.conf` for optional ARC tuning
- `/etc/zfs/zpool.cache`
- `/etc/default/zfs` when host-level service behavior needs adjustment

---

### **Implementation**

#### **1. Choose the pool shape deliberately**

General guidance for this platform:

- mirrors when future growth flexibility matters most
- RAIDZ2 when the pool is made of larger disks and predictable fault tolerance matters more
- separate pools only when there is a real operational boundary, not just a naming preference

Do the vdev design work before data lands on the system. Pool reshaping options are limited compared with ordinary filesystems.

#### **2. Split data into datasets by lifecycle, not by aesthetic preference**

A practical layout looks like this:

```text
tank
├── vmdata
├── backups
├── appdata
├── media
└── scratch
```

Why this matters:

- `vmdata` has different performance and snapshot needs than `media`
- `backups` should have clear retention and restore-readiness rules
- `scratch` can be noisy without polluting the policy for long-lived data

#### **3. Keep the essential command set small and memorable**

Pool health:

```bash
zpool status -v
zpool list
zpool iostat -v 1
```

Dataset inventory:

```bash
zfs list
zfs list -o space
zfs get compression,atime,recordsize,mountpoint tank/appdata
```

Create and tune a dataset:

```bash
zfs create tank/appdata
zfs set compression=lz4 tank/appdata
zfs set atime=off tank/appdata
```

Snapshots:

```bash
zfs snapshot tank/appdata@before-upgrade
zfs rollback tank/appdata@before-upgrade
```

Replication:

```bash
zfs send tank/appdata@before-upgrade | zfs receive backup/appdata
```

#### **4. Treat snapshots as short-horizon recovery, not as your only backup**

Snapshots are excellent for:

- pre-change checkpoints
- accidental deletion recovery
- short-term rollback safety

They do not replace an independent backup target. If the pool dies, snapshots on that same pool die with it.

#### **5. Keep pool hygiene routine and boring**

Healthy ZFS operations are repetitive:

- scrub regularly
- watch capacity headroom
- investigate checksum or device errors early
- replace weak disks before they turn into pool drama

Routine scrub:

```bash
zpool scrub tank
zpool status tank
```

#### **6. Document replacement work as you perform it**

When a disk degrades:

```bash
zpool status -v
ls -l /dev/disk/by-id
zpool replace tank <old-disk-id> <new-disk-id>
```

Always use stable disk identifiers when replacing hardware. Device names like `/dev/sdX` are too easy to get wrong under pressure.

---

### **Validation / troubleshooting**

#### **Basic validation**

- `zpool status -v` reports no degraded vdevs and no checksum errors
- `zfs list -o space` makes sense relative to expected dataset growth
- snapshots exist where they are expected to exist
- backup or replication targets can actually receive test data

#### **Troubleshooting checklist**

- If the pool is healthy but performance feels bad:
  - check pool saturation with `zpool iostat -v 1`
  - inspect dataset-level property mismatches
  - confirm the workload belongs on ZFS and is not just overloading slow disks
- If capacity is nearing exhaustion:
  - delete old snapshots only after understanding what is holding space
  - move bulk or short-lived data into a separate dataset if retention differs
- If a device reports errors:
  - inspect SMART data
  - map the physical disk before replacing anything
  - do not wait for a second failure just because the pool is still online

---

### **Known issues**

- ZFS rewards deliberate planning and punishes casual pool design changes later.
- Capacity creep is one of the easiest ways to make a healthy pool unstable. Leave usable headroom instead of planning around 100 percent occupancy.
- Deduplication is not a default tuning knob here. It is expensive, and most homelab workloads do not benefit enough to justify it.
