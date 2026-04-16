---
title: Networking
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-network:


## **Networking**

---

### **Purpose**

This section defines network trust boundaries, DNS behavior, traffic policy, and secure public exposure patterns for the platform.

---

### **Primary path**

- VLAN-segmented internal networking with explicit trust boundaries
- Technitium for local DNS and Cloudflare for public authoritative DNS
- Traefik, Authelia, and CrowdSec for public ingress and policy enforcement
- WireGuard for private remote access

---

### **What belongs here**

- VLAN and subnet design
- Firewall and DNS policy
- Public ingress routing and reverse-proxy behavior
- Network-side dependencies for backup, monitoring, and service exposure

---

### **Canonical pages**

- [VLAN](vlan.md)
- [Firewall](firewall.md)
- [DNS](dns.md)
- [Traefik](traefik.md)
- [Authelia](authelia.md)
- [CrowdSec](crowdsec.md)
- [WireGuard](wireguard.md)
- [Cloudflare](cloudflare.md)

---

### **Secondary references**

- [Tailscale](tailscale.md)
- [UniFi OS Server](unifi-os-server.md)

---

### **Drafts / planned**

- [Network UPS Tools](network-ups-tools.md) remains visible as a draft operational reference.

---

### **Guardrails**

- Keep inter-VLAN policy default-deny with explicit allowlists only.
- Keep public services behind Traefik instead of direct host-port sprawl.
- Keep local DNS centralized and documented.
- Keep permanent firewall exceptions documented with owner and expiry.
