---
title: Security
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-security:


## **Security**

---

### **Purpose**

This section defines the control map for host, runtime, identity, and data-protection decisions across the platform.

---

### **Primary path**

- Debian service hosts with a hardened operational baseline
- File-based secret handling for compose-managed services
- Public exposure routed through Traefik, Authelia, and CrowdSec
- Backup and recovery posture treated as part of security, not separate from it

---

### **What belongs here**

- Security standards for hosts, credentials, encryption, hashing, and malware response
- Control verification expectations before public exposure
- Cross-section references for networking, runtime, and recovery controls
- Drafts that are actively consolidating the platform security baseline

---

### **Canonical pages**

- [Security overview](index.md)
- [Firewall](../networking/firewall.md)
- [Authelia](../networking/authelia.md)
- [CrowdSec](../networking/crowdsec.md)
- [Traefik](../networking/traefik.md)

---

### **Secondary references**

- [Debian](../operating-systems/debian.md)
- [Docker](../infrastructure-tools/docker.md)
- [Proxmox Backup Server](../operating-systems/proxmox-backup-server.md)

---

### **Drafts / planned**

- [Linux Hardening](linux-hardening.md)
- [Secrets Management](secrets-management.md)
- [Encryption](encryption.md)
- [Hashing](hashing.md)
- [Malware](malware.md)

---

### **Guardrails**

- Keep plaintext secrets out of Git, compose files, and `.env` values.
- Keep internet-facing exposure behind documented edge controls.
- Keep exceptions documented with owner, review date, and expiry where possible.
- Keep restore drills and incident-response steps tied back to real operational workflows.
