---
title: Debian
last_reviewed: 2026-03-13
owner: Rognheim
---

# :fontawesome-brands-debian:


## **Debian**

---

### **Introduction**

This page defines the Debian 13 VM baseline used for platform workloads.

Scope:

- Debian 13 clones from the Proxmox Cloud-Init template
- First-boot baseline verification
- Host hardening and Docker readiness
- Resolver and network baseline alignment

---

### **Prerequisites**

- Proxmox template available from [Cloud-Init](../infrastructure-tools/cloud-init.md)
- VLAN and addressing plan available from [VLAN](../networking/vlan.md)
- Automation path available via [Terraform](../infrastructure-tools/terraform.md) and [Ansible](../infrastructure-tools/ansible.md)

---

### **Configuration files**

Common OS-level files for this baseline:

- `/etc/hostname`
- `/etc/hosts`
- `/etc/resolv.conf`
- `/etc/ssh/sshd_config.d/99-hardening.conf`
- `/etc/ufw/*`
- `/etc/apt/apt.conf.d/20auto-upgrades`
- `/etc/apt/apt.conf.d/50unattended-upgrades`

Runtime path contract for application hosts:

- `/opt/docker-appdata/compose-files/<stack>/docker-compose.yml`

---

### **Implementation workflow**

#### 1. Clone from Debian 13 Cloud-Init template

Example (SRV VLAN):

```shell
qm clone 800 901 --name srv-app-01 --full true
qm set 901 --ipconfig0 ip=10.0.20.2/24,gw=10.0.20.1
qm set 901 --nameserver 10.0.20.21 --searchdomain internal.rognheim.no
qm cloudinit update 901
qm start 901
```

#### 2. First-boot verification

```shell
cloud-init status --wait
hostnamectl
ip -brief addr
sudo systemctl status qemu-guest-agent
```

Confirm:

- Cloud-Init completed successfully
- Correct hostname and IP configuration
- Guest agent active

#### 3. Apply baseline packages and updates

```shell
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent curl ca-certificates sudo
sudo systemctl enable --now qemu-guest-agent
```

#### 4. Apply host hardening baseline

Apply controls from [Linux Hardening](../security/linux-hardening.md):

- SSH hardening
- UFW baseline
- fail2ban
- unattended upgrades
- auditd and sysctl baseline

#### 5. Prepare Docker runtime hosts (if role requires)

Use [Docker](../infrastructure-tools/docker.md) runbook to install Docker Engine + Compose plugin and enforce compose contract.

#### 6. Standardize DNS resolver behavior

- Keep primary resolver pointed at Technitium (`10.0.20.21` example)
- Avoid static per-app resolver overrides
- Validate local zone resolution for platform services

---

### **Validation checklist**

```shell
sudo sshd -t
sudo ufw status verbose
docker --version
docker compose version
dig @10.0.20.21 grafana.internal.rognheim.no +short
```

Expected:

- SSH config valid and hardened
- Host firewall active for intended role
- Docker tools available on application hosts
- Internal DNS resolution succeeds

---

### **Troubleshooting**

1. Cloud-Init clone boots without expected network

- Verify `ipconfig0` and gateway settings.
- Verify VLAN tag on Proxmox NIC.
- Check `/var/log/cloud-init.log`.

2. SSH key login fails

- Verify Cloud-Init user and injected key.
- Validate authorized key and permission model.
- Verify hardening did not remove expected admin user access.

3. Docker install succeeds but services fail to start

- Check host firewall and DNS resolution.
- Check available disk space and ownership on `/opt/docker-appdata`.

---

### **Operational guardrails**

- Treat Debian VM configuration as code-managed after bootstrap.
- Keep manual changes minimal and fold durable changes into Ansible roles.
- Keep role separation between DMZ edge, internal services, and storage hosts.
- Avoid mixing testing and production workloads on the same host without policy justification.

---

### **Upgrade from Debian 12 to Debian 13**

!!! Warning

    This runbook covers an in-place upgrade from Debian 12 (`bookworm`) to Debian 13 (`trixie`).
    Perform the upgrade from a local console when possible, or from a resilient remote session such as `tmux` or `screen`.
    Take a tested backup, hypervisor snapshot, or VM snapshot before changing APT sources.

Use this workflow for existing Debian 12 hosts that need to move to Debian 13 in place. For production systems, schedule a maintenance window and assume one or more reboots may be required.

#### Before you begin

Confirm the host is currently on Debian 12:

```shell
cat /etc/os-release
```

Confirm:

- `VERSION_CODENAME=bookworm`
- The host has console access or a persistent remote session
- A recent backup or snapshot exists and has been validated

Check free space before the upgrade:

```shell
df -h /
df -h /var
df -h /boot
```

Review APT configuration and identify any non-Debian repositories:

```shell
grep -R "^[[:space:]]*deb " /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
find /etc/apt/sources.list.d -maxdepth 1 -type f 2>/dev/null
```

Before proceeding:

- Disable or remove third-party repositories that are not confirmed for Debian 13 yet.
- Export or document any role-specific services that may need vendor-specific upgrade steps.
- Make sure you have root or `sudo` access even if SSH does not come back immediately after reboot.

#### Update Debian 12 fully

Bring Debian 12 fully up to date before changing repositories:

```shell
sudo apt update
sudo apt upgrade --without-new-pkgs
sudo apt full-upgrade
sudo apt autoremove --purge
```

If the upgrade installed a new kernel, core libraries, or systemd components, reboot before moving on:

```shell
sudo reboot
```

After the reboot, reconnect and confirm the host is healthy before changing release names.

#### Review installed software and package holds

Check for held, broken, or partially configured packages:

```shell
apt-mark showhold
sudo dpkg --audit
sudo dpkg --configure -a
sudo apt --fix-broken install
```

Review packages that may not have a normal Debian upgrade path:

```shell
apt list "~o"
dpkg -l | awk "/^rc/ { print \$2 }"
```

Review services that may be sensitive to a major OS upgrade, for example:

- Database servers
- Container runtimes
- VPN agents
- Monitoring agents
- Third-party package repositories such as Docker, Tailscale, Grafana, or vendor-specific software

If this host runs site-specific applications, review their vendor upgrade notes before continuing. The OS upgrade should not be the first time you discover an application-level compatibility issue.

#### Switch APT sources from bookworm to trixie

Edit Debian-managed repository definitions and replace `bookworm` with `trixie`.
Keep third-party repositories disabled until they are confirmed to support Debian 13.

Example `/etc/apt/sources.list`:

```shell
deb https://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb https://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

If your host uses deb822-style source files such as `/etc/apt/sources.list.d/debian.sources`, update the `Suites:` entries there instead of editing `sources.list`.
For any Debian-managed files under `/etc/apt/sources.list.d/`, switch only the Debian release names to `trixie`.

Example `/etc/apt/sources.list.d/debian.sources`:

```shell
Types: deb
URIs: https://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://security.debian.org/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

If the file currently references `bookworm`, replace only the suite names with `trixie`, `trixie-updates`, and `trixie-security`.
Do not remove `Signed-By` unless you intentionally use a different Debian archive keyring path.

A quick review after editing:

```shell
grep -R "bookworm\\|trixie" /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
```

Expected:

- Debian repositories now reference `trixie`
- Unsupported third-party repositories remain disabled or commented out

#### Run the staged upgrade

Refresh package metadata from the Debian 13 repositories:

```shell
sudo apt update
```

Run the minimal upgrade first:

```shell
sudo apt upgrade --without-new-pkgs
```

Then run the full distribution upgrade:

```shell
sudo apt full-upgrade
```

During the upgrade:

- Read prompts carefully before accepting package maintainer versions of config files.
- Pay attention to service restart prompts and packages that are removed.
- Do not interrupt `dpkg` or reboot the host mid-upgrade unless you are explicitly following Debian recovery guidance.

If the upgrade stops in an unusual state, resolve package configuration before rebooting:

```shell
sudo dpkg --configure -a
sudo apt --fix-broken install
```

#### Reboot into Debian 13

After the full upgrade completes successfully, reboot into the new userspace and kernel:

```shell
sudo reboot
```

After reconnecting, confirm the release:

```shell
cat /etc/os-release
uname -r
hostnamectl
```

Expected:

- `VERSION_CODENAME=trixie`
- The system boots normally and reaches its expected multi-user state

#### Post-upgrade cleanup and validation

Remove packages that are no longer needed:

```shell
sudo apt autoremove --purge
sudo apt autoclean
```

Run a general health check:

```shell
systemctl --failed
journalctl -p err -b
ip -brief addr
resolvectl status || cat /etc/resolv.conf
sudo sshd -t
systemctl list-timers --all
```

If the host provides additional services, validate them now:

```shell
sudo ufw status verbose
docker --version
docker compose version
```

Post-upgrade checklist:

- Confirm networking, DNS, SSH, and firewall behavior
- Confirm application services start and remain reachable
- Confirm container workloads, scheduled jobs, and monitoring agents still function
- Re-enable third-party repositories only after confirming Debian 13 support from the vendor
- Run `sudo apt update` again after re-enabling any approved third-party repositories

#### Common issues to check after reboot

1. SSH-managed systems

- OpenSSH upgrades can change daemon behavior or authentication expectations.
- Keep a console path available until you have verified fresh SSH logins.

2. `sysctl` baseline changes

- `/etc/sysctl.conf` may not be the only active source of kernel tuning after the upgrade.
- Reapply and verify expected settings with `sudo sysctl --system`.

3. Network interface naming changes

- Some systems can come back with different interface names after a major release upgrade.
- If networking is missing, compare `ip -brief link` output to your network configuration.

4. Incomplete upgrade state

- If packages remain unpacked or services are not fully configured, finish package recovery before repeated reboots.
- Re-run `sudo dpkg --configure -a` and `sudo apt --fix-broken install`, then review `journalctl -b`.
