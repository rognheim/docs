---
title: Linux Hardening
status: draft
last_reviewed: 2026-03-08
owner: Rognheim
---

# :material-shield-check:


## **Linux Hardening**

---

### **Introduction**

This is the canonical hardening runbook for Debian 13 VMs deployed from the Proxmox Cloud-Init template.  
It consolidates host, runtime, and operational controls for your platform:

- Proxmox as hypervisor
- ZFS-backed storage
- Debian 13 VM baseline
- Docker Compose workloads
- VLAN-segmented network with internet-facing DMZ workloads

---

### **Prerequisites and Scope**

Applies to:

- Debian 13 application VMs
- Docker hosts that run `docker-compose.yml` services
- Internal and DMZ workloads, with role-specific firewall policies

Does not replace:

- Gateway/firewall ACL configuration
- Proxmox host hardening tasks
- Application-specific auth and authorization controls

---

### **Configuration files**

Primary files managed in this baseline:

- `/etc/ssh/sshd_config`
- `/etc/ssh/sshd_config.d/99-hardening.conf`
- `/etc/ufw/user.rules` and `/etc/ufw/user6.rules`
- `/etc/fail2ban/jail.local`
- `/etc/audit/rules.d/hardening.rules`
- `/etc/sysctl.d/99-hardening.conf`
- `/etc/apt/apt.conf.d/20auto-upgrades`
- `/etc/apt/apt.conf.d/50unattended-upgrades`

---

### **Implementation**

#### 1. Install and enable baseline tooling

```shell
sudo apt update
sudo apt install -y ufw fail2ban auditd audispd-plugins unattended-upgrades apt-listchanges needrestart
sudo systemctl enable --now ufw fail2ban auditd unattended-upgrades
```

#### 2. Harden SSH

Create `/etc/ssh/sshd_config.d/99-hardening.conf`:

```sshconfig
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowTcpForwarding no
MaxAuthTries 3
MaxSessions 4
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
```

Validate and reload:

```shell
sudo sshd -t
sudo systemctl reload ssh
```

#### 3. Set host firewall baseline (UFW)

Default policy:

```shell
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Management access (recommended from VLAN 1 only):

```shell
sudo ufw allow from 10.0.1.0/24 to any port 22 proto tcp comment 'Mgmt SSH'
```

If the VM hosts reverse proxy services in DMZ:

```shell
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
```

Enable firewall:

```shell
sudo ufw enable
sudo ufw status verbose
```

#### 4. Configure fail2ban for SSH

Create `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
maxretry = 4
```

Apply:

```shell
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

#### 5. Enable unattended security updates

Set `/etc/apt/apt.conf.d/20auto-upgrades`:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Dry-run test:

```shell
sudo unattended-upgrade --dry-run --debug
```

#### 6. Add audit baseline

Create `/etc/audit/rules.d/hardening.rules`:

```text
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k privilege
-w /etc/ssh/sshd_config -p wa -k sshd
-w /etc/ssh/sshd_config.d/ -p wa -k sshd
```

Load and validate:

```shell
sudo augenrules --load
sudo auditctl -l
```

#### 7. Apply safe kernel/sysctl network hardening

Create `/etc/sysctl.d/99-hardening.conf`:

```text
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.tcp_syncookies=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
net.ipv6.conf.all.accept_redirects=0
net.ipv6.conf.default.accept_redirects=0
```

Apply:

```shell
sudo sysctl --system
```

#### 8. Docker host hardening controls

For compose services on Debian hosts:

- Use explicit container users (`user: "1000:1000"` when supported).
- Avoid `privileged: true` unless documented and approved.
- Use `read_only: true` where possible.
- Add `security_opt: ["no-new-privileges:true"]`.
- Keep public-exposed services on a dedicated public network and private services on internal networks.
- Avoid direct host socket mounts (`/var/run/docker.sock`) unless required.

Minimal compose hardening pattern:

```yaml
services:
  app:
    image: ghcr.io/example/app:stable
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
```

---

### **Operational Verification Checklist**

Run after provisioning and after major updates:

```shell
sudo systemctl --failed
sudo sshd -t
sudo ufw status numbered
sudo fail2ban-client status
sudo auditctl -s
sudo apt list --upgradable
docker ps --format 'table {{.Names}}\t{{.RunningFor}}\t{{.Ports}}'
```

Review points:

- SSH does not allow password or root login.
- UFW rules match VM role and VLAN placement.
- fail2ban is banning repeated failed SSH attempts.
- auditd is active and rules are loaded.
- No unexpected internet-exposed Docker ports.

---

### **Post-install tasks**

- Add this baseline to your VM template build pipeline (Cloud-Init + Ansible or equivalent).
- Document per-service exceptions (ports, capabilities, privileged modes) with owner and expiry.
- Re-run checklist after kernel updates and after Docker/Compose upgrades.
- Confirm controls still align with VLAN policy and gateway ACL expectations.

### **Known issues**

- Docker-published ports can bypass expected firewall behavior if deployment review is weak.
- Overly strict sysctl/firewall settings can break service discovery if not tested per role.
- Some containers require additional capabilities; keep deviations explicit and reviewed.
