---
title: Tailscale
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-tailscale:

## **Tailscale**

---

### **Introduction**

Tailscale is the private access path into the homelab. In this platform it is used for remote administration, private service access, and optional subnet routing or exit-node duties without exposing those systems to the public internet.

---

### **Scope**

This page owns:

- Debian installation and enrollment
- LXC-specific host changes for `/dev/net/tun`
- subnet router and exit-node setup
- DNS and forwarding notes that commonly matter in the homelab

This page does not own:

- public service exposure. See [Traefik](traefik.md)
- edge DNS for public hostnames. See [Cloudflare](cloudflare.md)

---

### **Prerequisites**

- A Tailscale account and tailnet.
- A Debian host, VM, or LXC with outbound internet access.
- A decision on the node role:
  - client only
  - subnet router
  - exit node

---

### **Dependencies / related pages**

- [Debian](../operating-systems/debian.md)
- [Proxmox](../hypervisors/proxmox.md)

---

### **Configuration files**

- `/etc/apt/sources.list.d/tailscale.list`
- `/etc/sysctl.d/99-tailscale.conf` when enabling forwarding
- Proxmox LXC config, for example `/etc/pve/lxc/123.conf`, when running inside an LXC container

---

### **Implementation**

#### **1. Install on Debian**

```bash
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt-get update
sudo apt-get install tailscale
```

#### **2. Enroll the node**

For an interactive first login:

```bash
sudo tailscale up
```

For a node that should accept DNS from the tailnet:

```bash
sudo tailscale up --accept-dns
```

Remember that `tailscale up` replaces the active settings with the flags you provide at that moment. When changing behavior later, re-run the full intended command instead of only adding one new flag.

#### **3. If the node runs in a Proxmox LXC, pass through TUN support**

On the Proxmox host:

```bash
nano /etc/pve/lxc/123.conf
```

Add:

```ini
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Then restart the container before retrying `tailscale up`.

#### **4. Enable forwarding for subnet-router or exit-node roles**

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

#### **5. Advertise routes or exit-node capability**

Advertise a local LAN as a subnet router:

```bash
sudo tailscale up --accept-dns --advertise-routes=192.168.10.0/24
```

Advertise both subnet routes and exit-node capability:

```bash
sudo tailscale up --accept-dns --advertise-routes=192.168.10.0/24 --advertise-exit-node
```

If the node should also consume routes published by other nodes:

```bash
sudo tailscale up --accept-dns --accept-routes --advertise-routes=192.168.10.0/24 --advertise-exit-node
```

Approve advertised routes or exit-node capability in the Tailscale admin console after the node comes online.

#### **6. Re-authenticate or move the node between accounts**

```bash
sudo tailscale up --force-reauth
```

#### **7. Optional performance tuning for high-throughput nodes**

Install the helper packages:

```bash
sudo apt-get install ethtool networkd-dispatcher -y
```

Apply the GRO settings to the default route interface:

```bash
NETDEV=$(ip route show 0/0 | cut -f5 -d' ')
sudo ethtool -K "$NETDEV" rx-udp-gro-forwarding on rx-gro-list off
```

Persist it for future boot and link events:

```bash
printf '#!/bin/sh\n\nethtool -K %s rx-udp-gro-forwarding on rx-gro-list off\n' "$(ip route show 0/0 | cut -f5 -d' ')" | sudo tee /etc/networkd-dispatcher/routable.d/50-tailscale
sudo chmod 755 /etc/networkd-dispatcher/routable.d/50-tailscale
sudo /etc/networkd-dispatcher/routable.d/50-tailscale
```

Only keep this optimization on nodes where it measurably improves throughput.

---

### **Validation / troubleshooting**

#### **Basic validation**

Check local status:

```bash
tailscale status
tailscale ip -4
tailscale ping <peer-name>
```

For subnet routers and exit nodes:

```bash
tailscale status --json
```

Confirm the expected routes or exit-node flags appear in the node metadata.

#### **Troubleshooting checklist**

- If `tailscale up` fails inside an LXC:
  - verify the host container config includes TUN access
  - restart the container after changing the config
- If subnet routes are advertised but clients cannot use them:
  - confirm IP forwarding is enabled
  - confirm the routes were approved in the admin console
  - confirm the local firewall allows forwarding
- If DNS behavior is surprising:
  - check whether the node was enrolled with `--accept-dns`
  - review tailnet DNS settings before overriding local resolver behavior

---

### **Known issues**

- Re-running `tailscale up` with a partial set of flags can silently replace previous settings. Keep the intended command documented for each node role.
- LXC deployments are easy to misconfigure if TUN support is not passed through from the Proxmox host.
