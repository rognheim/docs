---
title: WireGuard
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-wireguard:


## **WireGuard**

---

### **Introduction**

This page defines the WireGuard remote-access baseline using `wg-easy` as the canonical deployment on Docker Compose.

Primary outcomes:

- Private remote management path into homelab services
- Controlled route advertisement to selected VLANs
- DNS resolution of internal names through Technitium

---

### **Prerequisites**

- Debian 13 VM host with Docker and Compose installed
- Public DNS record and firewall/NAT path for WireGuard UDP port
- DNS baseline implemented in [DNS](dns.md)
- VLAN and ACL policies implemented in [VLAN](vlan.md) and [Firewall](firewall.md)

---

### **Configuration files and paths**

- `/opt/docker-appdata/compose-files/wireguard/docker-compose.yml`
- `/opt/docker-appdata/compose-files/wireguard/.env`
- `/opt/docker-appdata/compose-files/wireguard/config/`
- `/opt/docker-appdata/compose-files/wireguard/secrets/`

---

### **Implementation workflow**

#### 1. Create stack layout

```shell
mkdir -p /opt/docker-appdata/compose-files/wireguard/{config,secrets}
cd /opt/docker-appdata/compose-files/wireguard
chmod 700 secrets
```

#### 2. Create compose baseline (`wg-easy`)

Example `docker-compose.yml`:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - TZ=Europe/Oslo
      - WG_HOST=vpn.example.com
      - PASSWORD_HASH=$2y$10$replace-with-bcrypt-hash
      - WG_DEFAULT_DNS=10.0.20.21
      - WG_ALLOWED_IPS=10.0.1.0/24,10.0.20.0/24,10.0.30.0/24
      - WG_PERSISTENT_KEEPALIVE=25
    volumes:
      - ./config:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

Use non-secret values in `.env`, and keep secrets/credentials aligned with [Secrets Management](../security/secrets-management.md).

#### 3. Start stack and create peers

```shell
docker compose config --quiet
docker compose up -d
docker compose ps
```

Create peers from `wg-easy` UI and distribute config/QR securely.

#### 4. Restrict reachable networks

Do not advertise all RFC1918 ranges by default.

Recommended initial `WG_ALLOWED_IPS` scope:

- Management VLAN (for admin paths)
- Required service VLANs only
- Avoid direct exposure of storage/quarantine VLANs

#### 5. Enforce firewall boundary for VPN clients

Add explicit gateway ACL policy for WireGuard client subnet:

- Allow only required target VLANs and ports
- Deny-by-default to sensitive networks (storage/quarantine)

---

### **Validation checklist**

From VPN client:

```shell
ping 10.0.20.21
nslookup grafana.internal.rognheim.no 10.0.20.21
curl -Ik https://grafana.internal.rognheim.no
```

On WireGuard host:

```shell
docker logs wg-easy --tail 100
wg show
ss -uapn | rg 51820
```

Expected:

- Peer handshake is established.
- Internal DNS resolves via Technitium.
- Only intended networks are reachable.

---

### **Troubleshooting**

1. No handshake

- Verify UDP port forward (`51820`) on gateway.
- Confirm public DNS resolves to current WAN IP.
- Confirm ISP path is not blocking UDP.

2. Handshake works but no internal access

- Check allowed IPs on server and peer config.
- Check gateway ACLs for VPN client subnet.
- Check host IP forwarding and NAT behavior.

3. DNS resolves externally but not internal names

- Ensure `WG_DEFAULT_DNS` points to Technitium.
- Confirm firewall allows DNS from VPN subnet to `10.0.20.21:53`.

---

### **Operational guardrails**

- Keep WireGuard as remote admin path, not a broad flat-network tunnel.
- Require strong UI/admin credential hygiene for `wg-easy`.
- Rotate peer keys for lost devices or personnel changes.
- Keep management and storage access restricted even over VPN.
