---
title: Windows
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :fontawesome-brands-windows:

## **Windows**

---

### **Introduction**

Windows is mainly used here as either a desktop-style VM on Proxmox or as a host for WSL-based tooling. It is not a core server platform in this documentation set, so this page focuses on the few workflows that are actually useful in the homelab: building a stable VM, adding VirtIO drivers, and enabling WSL cleanly.

---

### **Scope**

This page owns:

- Windows 11 VM configuration on Proxmox
- driver and guest-agent handoff
- WSL installation for local tooling

This page does not own:

- unofficial licensing or activation shortcuts
- application deployment guidance that belongs on Linux-first pages

---

### **Prerequisites**

- A Windows installation ISO.
- The VirtIO driver ISO uploaded to Proxmox.
- A valid Microsoft license or evaluation path handled separately.

---

### **Dependencies / related pages**

- [Proxmox](../hypervisors/proxmox.md)
- [Debian](debian.md)

---

### **Configuration files**

- Proxmox VM config, for example `/etc/pve/qemu-server/240.conf`
- Optional `%UserProfile%\\.wslconfig` if WSL memory or CPU limits need tuning

---

### **Implementation**

#### **1. Build the VM in Proxmox**

Recommended baseline for a modern Windows 11 VM:

- OS: Microsoft Windows
- Machine: `q35`
- BIOS: `OVMF (UEFI)`
- TPM: enabled, version `2.0`
- SCSI controller: `VirtIO SCSI single`
- Disk: `128 GB`, `SSD emulation`, `Discard`, and `IO thread`
- CPU type: `host`
- CPU cores: `4` minimum for general desktop use
- Memory: `8 GB` minimum
- Network model: `VirtIO`
- Guest agent: enabled

Before first boot, attach the VirtIO ISO as a second CD/DVD drive.

#### **2. Install Windows and load VirtIO storage and network drivers**

During setup:

1. Choose language and keyboard layout.
2. If a key is not being entered during setup, choose the standard "I don't have a product key" path and handle licensing later through normal Microsoft workflows.
3. Choose `Custom: Install Windows only`.
4. When no disk appears, select `Load driver`.
5. Browse the VirtIO ISO and load the storage driver for Windows 11.
6. Repeat for the network driver if needed after setup.

Once the disk appears, continue the installation normally.

#### **3. Install guest integrations after first login**

After the desktop comes up:

- install the VirtIO guest tools from the mounted ISO
- confirm network, ballooning, and storage drivers are healthy in Device Manager
- install the QEMU guest agent service if your VirtIO bundle includes it

This gives Proxmox cleaner shutdowns, IP reporting, and better lifecycle handling.

#### **4. Enable WSL when the VM is used as a development workstation**

From an elevated PowerShell session:

```powershell
wsl --install
```

Reboot when prompted, then list available distributions:

```powershell
wsl.exe --list --online
```

Install Debian:

```powershell
wsl.exe --install Debian
```

After first launch inside the Debian WSL instance:

```bash
sudo apt-get update
sudo apt-get install -y bash-completion ca-certificates git wget
```

Keep WSL for local tooling, shell work, and lightweight automation. If the workload is a long-running server role, move it back to the normal Linux infrastructure instead of leaving it inside a desktop VM.

#### **5. Optional: cap WSL resource usage**

If the Windows VM is memory-constrained, add a `%UserProfile%\\.wslconfig` file such as:

```ini
[wsl2]
memory=4GB
processors=4
```

Then restart WSL:

```powershell
wsl --shutdown
```

---

### **Validation / troubleshooting**

#### **Basic validation**

- Proxmox shows the guest IP and clean shutdown behavior.
- Device Manager has no missing storage or network drivers.
- `wsl --status` reports a healthy WSL installation.
- The Debian WSL instance can reach the network and install packages.

#### **Troubleshooting checklist**

- If the installer cannot see the disk:
  - verify the VirtIO ISO is mounted
  - reload the correct Windows 11 storage driver
- If the VM has no network after install:
  - install the VirtIO network driver from the ISO
- If WSL fails to start:
  - confirm virtualization features remain enabled in the VM
  - run `wsl --status` and `wsl --shutdown` before reinstalling a distribution

---

### **Known issues**

- Windows VMs often look “done” before the VirtIO and guest-agent pieces are actually complete. Validate those before calling the build finished.
- WSL is useful for tooling, but it should not become a quiet replacement for the Linux server estate. If a service matters operationally, document and run it on the normal Linux path.
