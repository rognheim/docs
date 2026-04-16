---
title: DNS
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-dns:


## **DNS**

---

### **Introduction**

This page defines the DNS baseline for the platform using Technitium DNS as the local resolver and local-zone authority.

Design goals:

- Reliable internal service discovery
- Predictable certificate automation and reverse proxy routing
- Split-horizon behavior between internal and external name resolution
- Centralized, auditable DNS changes

---

### **Prerequisites**

- VLAN model implemented as defined in [VLAN](vlan.md)
- DNS host deployed on management or internal services VLAN
- Static IP for DNS host (example: `10.0.1.11`)
- Cloudflare used for public authoritative DNS
- Traefik + Authelia + CrowdSec stack available for public apps

---

### **Configuration files and paths**

Technitium stack paths (recommended):

- `/opt/docker-appdata/compose-files/technitium-dns/docker-compose.yml`
- `/opt/docker-appdata/compose-files/technitium-dns/.env`
- `/opt/docker-appdata/compose-files/technitium-dns/config/`
- `/opt/docker-appdata/compose-files/technitium-dns/data/`

Client and host resolver paths:

- Debian host resolver: `/etc/resolv.conf` (or systemd-resolved pathing)
- Proxmox VM clone DNS setting via Cloud-Init (`qm set --nameserver`)

---

### **Implementation workflow**

#### 1. Deploy Technitium DNS with Docker Compose

Example `docker-compose.yml`:

```yaml
services:
  technitium-dns:
    image: technitium/dns-server:latest
    container_name: technitium-dns
    restart: unless-stopped
    secrets:
      - admin_password
    environment:
      - TZ=Europe/Oslo
      - DNS_SERVER_ADMIN_PASSWORD_FILE=/run/secrets/admin_password
      - DNS_SERVER_DOMAIN=internal.rognheim.no
      - DNS_SERVER_FORWARDERS=1.1.1.1,9.9.9.9
      - DNS_SERVER_FORWARDER_PROTOCOL=Https
    volumes:
      - config:/etc/dns
    networks:
      - dns
    ports:
      - 10.0.20.21:53:53/udp
      - 10.0.20.21:53:53/tcp
      - 10.0.20.21:5380:5380/tcp
    sysctls:
      - net.ipv4.ip_local_port_range=1024 65535

secrets:
  admin_password:
    file: ./secrets/admin_password

volumes:
  config:

networks:
  dns:
    external: true
```

Use secret handling policy from [Secrets Management](../security/secrets-management.md).

#### 2. Configure resolver and forwarders

In Technitium admin UI:

- Enable recursive resolution for internal clients.
- Set upstream forwarders (for example: Quad9, Cloudflare, or ISP resolver mix).
- Enable DNSSEC validation when compatible with your upstream strategy.

#### 3. Create local authoritative zones

Recommended local zones:

- `internal.rognheim.no` for internal services
- Optional delegated sub-zones like `srv.internal.rognheim.no`, `dmz.internal.rognheim.no`

Add A/AAAA/CNAME records for service entrypoints used by Traefik and internal apps.

#### 4. Implement split-horizon behavior

Pattern:

- Internal clients resolve internal services from Technitium local zones.
- Public clients resolve internet-facing records from Cloudflare.
- Public hostnames may point to public edge while internal-only names stay local only.

For public-exposed apps, keep certificate and routing behavior aligned with [Traefik](traefik.md).

#### 5. Standardize DNS assignment by VLAN

Use firewall policy to force client DNS paths where required:

- IoT/Lab VLAN -> allow DNS only to Technitium
- Optional hardening: block direct outbound DNS from untrusted VLANs

Reference ACL model in [Firewall](firewall.md).

#### 6. Configure VM template and clones

From Proxmox clone workflow:

```shell
qm set 901 --nameserver 10.0.1.11 --searchdomain internal.rognheim.no
qm cloudinit update 901
```

Template/clone process is defined in [Cloud-Init](../infrastructure-tools/cloud-init.md).

---

### **Validation checklist**

```shell
dig @10.0.1.11 prometheus.internal.rognheim.no +short
dig @10.0.1.11 grafana.internal.rognheim.no +short
dig @10.0.1.11 rognheim.no +short
nslookup loki.internal.rognheim.no 10.0.1.11
```

Expected:

- Internal names resolve to private IPs.
- Public names resolve correctly through upstream/public records.
- DNS response is consistent across management and service hosts.

---

### **Troubleshooting**

1. Internal hostnames do not resolve

- Confirm client is using Technitium as primary resolver.
- Confirm zone and record exist in Technitium.
- Confirm firewall allows DNS (TCP/UDP 53) from source VLAN.

2. Public record resolves internally to wrong endpoint

- Review split-horizon entries and CNAME chain.
- Confirm Cloudflare record intent (proxied vs DNS-only) matches design.

3. Intermittent resolution failures

- Verify upstream forwarder health and timeout settings.
- Check for conflicting DHCP-provided DNS servers on clients.

---

### **Operational guardrails**

- Keep Technitium as the single source of truth for local DNS.
- Keep public authority in Cloudflare; avoid duplicating public-zone management locally.
- Track DNS record ownership and change rationale for critical services.
- Avoid hardcoded IP references in app configs when DNS name is stable and managed.
