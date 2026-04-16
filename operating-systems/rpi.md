---
title: Raspberry Pi OS
last_reviewed: 2026-03-09
owner: Rognheim
---

# :fontawesome-brands-raspberry-pi:


## **Raspberry Pi OS**

---

### **Introduction**

This page defines the Raspberry Pi OS baseline for infra appliance roles, primarily hosting UniFi OS Server.

This is not the standard workload platform for application stacks; Debian 13 VMs remain the primary runtime baseline.

---

### **Prerequisites**

- Raspberry Pi 4 or newer (4 GB+ recommended)
- Raspberry Pi OS 64-bit installed and updated
- Static DHCP reservation or static IP
- SSH key-based admin access

---

### **Configuration files**

- `/etc/hostname`
- `/etc/hosts`
- `/etc/ssh/sshd_config`
- `/etc/ufw/*`
- `/etc/apt/apt.conf.d/20auto-upgrades`
- `/etc/apt/apt.conf.d/50unattended-upgrades`

UniFi application runbook:

- [UniFi OS Server](../networking/unifi-os-server.md)

---

### **Implementation workflow**

#### 1. Patch and baseline system

```shell
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

#### 2. Create dedicated admin user and secure SSH

```shell
sudo adduser admin
sudo usermod -aG sudo admin
```

Apply SSH baseline:

- `PermitRootLogin no`
- `PasswordAuthentication no`
- key-based access only

Hardening reference: [Linux Hardening](../security/linux-hardening.md)

#### 3. Apply host firewall baseline

Allow only required services (example):

- `22/tcp` from management VLAN
- UniFi service ports required for your deployment

#### 4. Configure reliable network identity

- Use DHCP reservation (recommended)
- Keep DNS pointed to Technitium resolver
- Confirm management VLAN placement

#### 5. Install and operate UniFi OS Server

Follow full runbook:

- [UniFi OS Server](../networking/unifi-os-server.md)

---

### **Validation checklist**

```shell
hostnamectl
ip -brief addr
sudo sshd -t
sudo ufw status verbose
systemctl status uosserver
```

Expected:

- Host identity and IP are stable.
- SSH and firewall controls are active.
- UniFi service is healthy and reachable on intended port.

---

### **Troubleshooting**

1. Controller unavailable after reboot

- Confirm static lease/IP did not change.
- Check `uosserver` service state and logs.
- Verify firewall and VLAN path to management clients.

2. Device adoption unreliable

- Verify same L2/L3 reachability expectations for adoption flow.
- Check controller DNS name resolution and TLS trust path.

---

### **Operational guardrails**

- Keep Raspberry Pi role narrow (network/controller appliance).
- Avoid co-locating unrelated public workloads on this host.
- Keep periodic backups of controller configuration off-device.
- Re-test SSH and firewall controls after major updates.
