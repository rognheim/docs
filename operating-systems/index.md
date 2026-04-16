---
title: Operating systems
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-application-cog-outline:


## **Operating systems**

---

### **Purpose**

This section defines OS roles for the platform and how each role maps to provisioning, runtime, and security standards.

---

### **Primary path**

- Debian 13 is the default service-host operating system.
- Proxmox Backup Server is the dedicated backup appliance role.
- Raspberry Pi OS remains the default small-footprint appliance option.

---

### **What belongs here**

- OS role boundaries and placement in the platform
- Baseline expectations for service hosts and appliance systems
- Operating-system references that support the main server workflow
- Links out to hypervisor, security, and runtime standards when ownership lives elsewhere

---

### **Canonical pages**

- [Debian](debian.md)
- [Proxmox Backup Server](proxmox-backup-server.md)
- [Raspberry Pi OS](rpi.md)
- [Docker](../infrastructure-tools/docker.md)
- [Linux Hardening](../security/linux-hardening.md)

---

### **Secondary references**

- [TrueNAS](truenas.md)
- [Home Assistant OS](home-assistant.md)
- [Ubuntu Server](ubuntu-server.md)
- [Windows](windows.md)
- [macOS](macos.md)
- [Bazzite](bazzite.md)

---

### **Drafts / planned**

- No additional OS role pages are being promoted during this pass.

---

### **Guardrails**

- Keep new service workloads on Debian unless a role-specific exception is documented.
- Keep appliance hosts narrow in scope and isolated from unrelated public workloads.
- Keep OS drift controlled through automation and runbooks.
- Keep network and security policy owned by the relevant platform sections.
