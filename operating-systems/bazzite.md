---
title: Bazzite
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-gamepad-variant-outline:

## **Bazzite**

---

### **Role summary**

Bazzite is a Fedora Atomic–style gaming and desktop operating system. In this repository it is a workstation or test guest, not a core server OS. The useful documentation target is therefore a concise VM reference rather than a long production runbook.

---

### **Use case in this platform**

Use Bazzite when you want:

- a desktop-oriented Linux guest for testing GPU-adjacent workflows
- a gaming-focused VM for experimentation
- a modern Linux desktop reference without turning it into a permanent infrastructure dependency

Avoid using it as the default place for long-running homelab services. Those belong on the Debian and containerized paths instead.

---

### **Minimal examples**

#### **Proxmox VM baseline**

- Guest OS type: Linux
- Machine: `q35`
- BIOS: `OVMF (UEFI)`
- Disk: `64-128 GB`, `SSD emulation`, `Discard`, `IO thread`
- CPU type: `host`
- CPU cores: `4`
- Memory: `8 GB` minimum
- Network model: `VirtIO`

Unlike the Windows VM workflow, Bazzite should be treated as a Linux guest. Do not label the guest OS as Microsoft Windows just because the VM shape is similar.

#### **Installation flow**

1. Upload the Bazzite ISO to Proxmox.
2. Create the VM with the Linux-oriented settings above.
3. Boot the installer and complete the normal desktop installation.
4. After first boot, confirm network and display behavior before adding optional passthrough devices.

#### **Post-install checklist**

- apply system updates from the desktop workflow Bazzite expects
- confirm the root filesystem and desktop session are healthy after the first reboot
- document any passthrough or controller-specific changes in the VM notes

---

### **Limits / caveats**

- Bazzite is intentionally opinionated and desktop-focused, which makes it a poor default for server tasks in this documentation set.
- If the VM exists only to test a game, controller, or desktop workflow, keep the page short and avoid over-documenting one-off tweaks.
- GPU passthrough and gaming latency work belong in a dedicated guide if this becomes a recurring platform pattern.

---

### **Related pages**

- [Proxmox](../hypervisors/proxmox.md)
- [Debian](debian.md)
