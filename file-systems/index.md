---
title: File systems
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :fontawesome-solid-hard-drive:


## **File systems**

---

### **Purpose**

This section defines the storage baseline for the platform, with ZFS as the primary model for VM workloads, dataset layout, and backup planning.

---

### **Primary path**

- ZFS-backed Proxmox storage for hypervisor and VM workloads
- Workload-aware datasets with explicit lifecycle boundaries
- Backup and restore planning that aligns with Proxmox Backup Server

---

### **What belongs here**

- Storage topology and dataset strategy
- ZFS property guidance by workload profile
- Snapshot, replication, and restore-readiness expectations
- Capacity and pool-health operational notes

---

### **Canonical pages**

- [ZFS Overview](zfs-overview.md)
- [ZFS Settings](zfs-settings.md)
- [Proxmox Backup Server](../operating-systems/proxmox-backup-server.md)

---

### **Secondary references**

- [Proxmox](../hypervisors/proxmox.md)
- [Debian](../operating-systems/debian.md)
- [TrueNAS](../operating-systems/truenas.md)

---

### **Drafts / planned**

- No active draft storage pages are promoted in the main navigation during this pass.

---

### **Guardrails**

- Separate high-churn and long-retention data into different datasets.
- Keep dataset naming, ownership, and retention intent documented.
- Validate restore paths regularly, not only backup job success.
- Monitor scrub health, capacity headroom, and device error trends.
