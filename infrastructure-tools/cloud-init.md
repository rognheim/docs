---
title: Cloud-Init
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-cloud-download:


## **Cloud-Init**

---

### **Introduction**

This page defines the canonical Proxmox workflow for building and cloning the Debian 13 Cloud-Init VM template used across the platform.

Outcome:

- Reproducible base VM template
- Consistent first-boot identity/networking configuration
- Clean handoff into Terraform and Ansible workflows

---

### **Prerequisites**

- Proxmox VE node shell access
- Storage IDs available for VM disks and Cloud-Init drive (examples use `local-zfs`)
- Management bridge available (example: `vmbr0`)
- SSH public key for admin user injection

---

### **Configuration files and artifacts**

- Debian cloud image: `debian-13-generic-amd64.qcow2`
- Proxmox VM config: `/etc/pve/qemu-server/<vmid>.conf`
- Cloud-Init runtime logs on guest:
  - `/var/log/cloud-init.log`
  - `/var/log/cloud-init-output.log`

---

### **Implementation workflow**

#### 1. Download and verify Debian 13 cloud image

```shell
cd /root
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2
wget https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum -c SHA512SUMS 2>/dev/null | grep debian-13-generic-amd64.qcow2
```

If the checksum is verified the output should look like this:

```shell
debian-13-generic-amd64.qcow2: OK
```

For integrity policy and hashing guidance: [Hashing](../security/hashing.md).

#### 2. Create base VM shell

Example template VM ID: `8000`.

```shell
qm create 8000 \
  --name debian-13-cloud-init-template \
  --memory 4096 \
  --cores 4 \
  --cpu host \
  --machine q35 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,firewall=1
```

#### 3. Import image and set boot disk

```shell
qm set 8000 --scsi0 local-zfs:0,import-from=/root/debian-13-generic-amd64.qcow2,discard=on,iothread=1,ssd=1
qm set 8000 --boot order=scsi0
qm disk resize 8000 scsi0 32G
```

#### 4. Enable firmware and Cloud-Init drive

```shell
qm set 8000 --bios ovmf
qm set 8000 --efidisk0 local-zfs:1,format=raw,efitype=4m,pre-enrolled-keys=1
qm set 8000 --ide2 local-zfs:cloudinit
```

#### 5. Enable guest agent and Linux profile

```shell
qm set 8000 --agent enabled=1 --ostype l26 --onboot 1
```

#### 6. Configure Cloud-Init defaults

In Proxmox UI for VM `8000` -> **Cloud-Init**:

- User: `user`
- Password: disabled or temporary one-time setup only
- SSH public key: paste operator/team key(s)
- IP config: DHCP for template
- DNS: platform DNS resolver(s)

Click `Regenerate image` to regenerate Cloud-Init config after you have made the edits.

#### 7. First boot verification before templating

Start VM once, then verify inside guest:

```shell
sudo cloud-init status --wait
sudo systemctl status qemu-guest-agent
hostnamectl
ip -brief addr
```

Install guest agent if missing:

```shell
sudo apt update
sudo apt install -y qemu-guest-agent
```

Shutdown cleanly:

```shell
sudo poweroff
```

#### 8. Convert to template

```shell
qm template 8000
```

Template is now ready for cloning and Terraform consumption.

---

### **Clone workflow**

#### 1. Create clone

```shell
qm clone 8000 2021 --name app-01 --full true --target <proxmox-node>
```

#### 2. Set per-VM network and identity

For static IP in SRV VLAN (`10.0.20.0/24`):

```shell
qm set 2021 --ipconfig0 ip=10.0.20.21/24,gw=10.0.20.1
qm set 2021 --nameserver 10.0.1.10 --searchdomain rognheim.no
qm set 2021 --ciuser user
```

Optional CPU/memory tuning:

```shell
qm set 2021 --cores 4 --memory 8192
```

#### 3. Regenerate and boot

```shell
qm cloudinit update 2021
qm start 2021
```

---

### **Validation checklist**

On Proxmox node:

```shell
qm config 2021
qm guest cmd 2021 network-get-interfaces
```

From management workstation:

```shell
ssh user@10.0.20.21 "hostname && cloud-init status"
```

Expected:

- VM boots from imported cloud disk.
- Cloud-Init applies user, key, and network settings on first boot.
- Guest agent reports IP and responds to Proxmox guest commands.

---

### **Troubleshooting**

1. No IP assigned on first boot

- Confirm `ipconfig0` format and gateway.
- Confirm bridge/VLAN mapping in Proxmox network config.
- Check guest logs: `/var/log/cloud-init.log`.

2. SSH key not injected

- Verify `ciuser` and Cloud-Init user match expected login user.
- Re-run `qm cloudinit update <vmid>` before booting.

3. Clone still has stale metadata

- Ensure clone was powered off before Cloud-Init changes.
- Regenerate Cloud-Init config and reboot clone.

---

### **Operational guardrails**

- Treat template `8000` as immutable; rebuild via documented flow instead of manual drift edits.
- Keep template package set minimal; apply role-specific packages through Ansible.
- Link host hardening execution to [Linux Hardening](../security/linux-hardening.md).
- Use [Terraform](terraform.md) for repeatable VM lifecycle after template baseline is stable.
