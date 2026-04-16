---
title: SSH
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-key-variant:


## **SSH**

---

### **Introduction**

SSH is the standard access method for Debian 13 VMs provisioned from your Proxmox Cloud-Init template.

This page focuses on:

- SSH key lifecycle on operator devices
- Cloud-Init key injection for VM provisioning
- Access model for management networks
- Rotation and revocation workflows

Host hardening details (`sshd_config`, UFW, fail2ban, audit) are maintained in [Linux Hardening](../security/linux-hardening.md).

---

### **Prerequisites**

- Debian 13 VM template/clone in Proxmox
- Management VLAN reachability (typically `10.0.1.0/24`)
- Terminal access from macOS, Linux, or Windows 11 WSL
- Existing admin account or console access for break-glass recovery

---

### **Configuration files**

Client-side:

- `~/.ssh/id_ed25519` (private key)
- `~/.ssh/id_ed25519.pub` (public key)
- `~/.ssh/config`
- `~/.ssh/known_hosts`

Server-side:

- `~/.ssh/authorized_keys` for each admin user
- `/etc/ssh/sshd_config` and `/etc/ssh/sshd_config.d/*.conf` (see Linux Hardening)

---

### **Implementation workflow**

#### 1. Generate operator keys

On each admin endpoint:

```shell
ssh-keygen -t ed25519 -a 256
```

What this command does:

`ssh-keygen`

- The standard program used to create and manage SSH keys.

`-t ed25519`

- Specifies the **type** of key to generate: **ed25519**.
- **ed25519** is a modern elliptic-curve algorithm that is:
    1. fast
    2. secure
    3. compact
    4. generally recommended over older RSA keys for most SSH use cases

`-a 256`

- Sets the number of **KDF rounds** (key derivation function rounds) to **256**.  
- This matters when your private key is protected with a passphrase.

More specifically:

- SSH stores the private key in an encrypted format if you set a passphrase.
- The KDF makes brute-forcing the passphrase harder by increasing the computational work required to test each guess.
- A higher number means better resistance against password guessing attacks, but also slightly slower unlocking of the key.

So `-a 256` means:

- stronger protection for the private key if stolen
- a small performance cost when entering the passphrase

Recommended:

- Use a unique key per operator device.
- Protect keys with passphrases.
- Keep a secure inventory of key owners and intended host scope.

#### 2. Register key in Proxmox Cloud-Init

For template or clone:

1. Open VM in Proxmox UI.
2. Go to **Cloud-Init**.
3. Set `User` (example: `admin`).
4. Paste `id_ed25519.pub` into **SSH public key**.
5. Configure network (`DHCP` for template, static per clone where required).
6. Click **Regenerate image/config** before first boot.

#### 3. Standardize SSH client host entries

Create `~/.ssh/config` entries per environment:

```sshconfig
Host pve-app-01
  HostName 10.0.20.21
  User admin
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  ServerAliveInterval 60
  ServerAliveCountMax 3

Host pve-dmz-01
  HostName 10.0.30.21
  User admin
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

Then connect using aliases:

```shell
ssh pve-app-01
ssh pve-dmz-01
```

#### 4. Enforce access model

Recommended policy:

- Management access enters from VLAN 1 (`10.0.1.0/24`) only.
- Direct admin access to DMZ hosts is restricted and audited.
- Emergency recovery uses Proxmox console, not password SSH fallback.

Network and hardening enforcement:

- [Firewall](../networking/firewall.md)
- [Linux Hardening](../security/linux-hardening.md)

#### 5. Rotate and revoke keys

Rotation workflow (quarterly or on role/device changes):

1. Generate new key pair on operator device.
2. Add new public key to template and active VMs.
3. Validate login with new key.
4. Remove old key from `authorized_keys` and Cloud-Init records.
5. Record rotation date and owner in ops notes.

Immediate revocation workflow:

1. Remove compromised key from all `authorized_keys` files.
2. Remove it from template/Cloud-Init defaults.
3. Reload SSH service (if user-level keys unchanged, reload is optional).
4. Review auth logs and rotate related credentials.

---

### **Validation checklist**

Run from client:

```shell
ssh -G pve-app-01 | rg -n "user|hostname|identityfile"
ssh -i ~/.ssh/id_ed25519 admin@10.0.20.21 "hostname && whoami"
```

Run on host:

```shell
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Expected:

- SSH connects with key auth only.
- `authorized_keys` exists with owner-only write permissions.
- Management path follows expected VLAN restrictions.

---

### **Troubleshooting**

1. `Permission denied (publickey)`

- Confirm key exists locally and in `authorized_keys`.
- Check private key permissions:

```shell
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh
```

- Debug with verbose SSH:

```shell
ssh -vvv admin@10.0.20.21
```

2. Host key mismatch warning

```shell
ssh-keygen -R 10.0.20.21
ssh admin@10.0.20.21
```

3. Cloud-Init did not apply keys

- Confirm Cloud-Init drive exists and config was regenerated.
- Confirm correct Cloud-Init user was set in Proxmox before first boot.
- Use console access to inspect `/var/log/cloud-init.log`.

---

### **Operational guardrails**

- Do not share private keys across operators.
- Do not keep password SSH enabled as a fallback path.
- Keep one break-glass process documented via Proxmox console.
- Delegate daemon hardening changes to [Linux Hardening](../security/linux-hardening.md) to avoid drift between runbooks.
