---
title: Proxmox
last_reviewed: 2026-04-16
owner: Rognheim
# icon: lucide/rocket
---

# :simple-proxmox:


## **Proxmox**

---

### **Introduction**

This page defines the Proxmox VE baseline for hosting Debian 13 VM workloads in this platform. It is the hypervisor-side reference for host lifecycle, VM attachment patterns, and the handoff into template, IaC, and guest-configuration workflows.

---

### **Scope**

This page owns:

- Proxmox host baseline and update policy
- bridge, VLAN, and storage attachment patterns for VMs
- Debian 13 Cloud-Init template handoff
- VM lifecycle guardrails before guest-level automation takes over

This page does not own guest hardening, compose deployment, or application-specific configuration.

---

### **Dependencies / related pages**

- [Cloud-Init](../infrastructure-tools/cloud-init.md)
- [Packer](../infrastructure-tools/packer.md)
- [Terraform](../infrastructure-tools/terraform.md)
- [Ansible](../infrastructure-tools/ansible.md)
- [VLAN](../networking/vlan.md)
- [ZFS Overview](../file-systems/zfs-overview.md)

---

### **Prerequisites**

- Proxmox VE installed on supported hardware
- Management network reachable from admin workstation
- Storage pool(s) prepared (ZFS recommended)
- VLAN-capable gateway/switch in place

---

### **Configuration files and artifacts**

- Host network: `/etc/network/interfaces`
- Proxmox APT sources:
  - `/etc/apt/sources.list`
  - `/etc/apt/sources.list.d/*.list`
- VM config: `/etc/pve/qemu-server/<vmid>.conf`
- Cluster/global config under `/etc/pve/`

---

### **Implementation workflow**

#### 1. Apply host update baseline

Use supported Proxmox and Debian repositories appropriate for your subscription model.

Update cycle:

```shell
apt update
apt full-upgrade -y
reboot
```

Keep this host-level baseline distinct from guest OS update procedures.

#### 2. Validate storage baseline

Recommended checks:

```shell
zpool status
zpool list
pvesm status
```

Ensure storage IDs used in VM/template automation match your infrastructure-tool runbooks (for example `local-zfs`).

#### 3. Configure bridge and VLAN model

Baseline pattern:

- `vmbr0` as primary VM bridge
- VLAN tagging enabled through VM NIC definitions
- Gateway/firewall enforces inter-VLAN controls

Example VM VLAN mapping:

```shell
qm set 901 --net0 virtio,bridge=vmbr0,tag=20,firewall=1
qm set 930 --net0 virtio,bridge=vmbr0,tag=30,firewall=1
```

Network policy details are documented in [VLAN](../networking/vlan.md) and [Firewall](../networking/firewall.md).

#### 4. Build Debian 13 Cloud-Init template

Do not maintain ad-hoc handcrafted base VMs.

Use canonical template flow from:

- [Cloud-Init](../infrastructure-tools/cloud-init.md)
- [Packer](../infrastructure-tools/packer.md)

Expected outcome:

- Versioned Debian 13 template
- Guest agent enabled
- Cloud-Init drive attached
- Consistent first-boot identity/network configuration

#### 5. Provision VMs through IaC handoff

Split responsibilities:

1. Packer builds template artifact.
2. Terraform clones and sets compute/network/storage parameters.
3. Ansible applies host baseline and role configuration.
4. Compose deploys workload stacks.

References:

- [Terraform](../infrastructure-tools/terraform.md)
- [Ansible](../infrastructure-tools/ansible.md)
- [Docker](../infrastructure-tools/docker.md)

#### 6. Enable safe operational defaults

Recommended Proxmox VM defaults for Debian workload guests:

- Machine type `q35`
- CPU type `host`
- `virtio-scsi-single` controller
- discard + iothread for storage where supported
- guest agent enabled
- startup order and on-boot behavior documented per service role

---

### **Validation checklist**

```shell
pveversion -v
qm list
qm config 800
qm config 901
pvesh get /nodes/$(hostname)/network
```

Expected:

- Host and package state are healthy.
- Template VM exists and is marked as template.
- Workload VMs show expected VLAN tags and hardware profile.
- Network interfaces/bridges match documented intent.

---

### **Configuring IOMMU (PCI Passthrough)**

!!! Info "Make sure the Motherboard and Processor supports IOMMU before continuing. These settings also needs to be turned on in the BIOS!"

Edit `/etc/default/grub`

```shell
# GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```

Run

```shell
update-grub
```
Edit `/etc/kernel/cmdline`

```shell
root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt
```

Edit `/etc/modules` and add the following lines:

```shell
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Run

```shell
update-initramfs -u -k all
```

```shell
reboot
```

---

### **Windows VirtIO driver**

Download the latest stable VirtIO Drivers for Windows from [Proxmox](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)

You can go to local storage in Proxmox or wherever you save your ISO Images and "Download from URL"

---

### **Finding out if CPU supports x86_64-v3**

Enter the shell

```shell
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 --help
```

Output should be something like this:

```shell
This program interpreter self-identifies as: /lib64/ld-linux-x86-64.so.2

Shared library search path:
  (libraries located via /etc/ld.so.cache)
  /lib/x86_64-linux-gnu (system search path)
  /usr/lib/x86_64-linux-gnu (system search path)
  /lib (system search path)
  /usr/lib (system search path)

Subdirectories of glibc-hwcaps directories, in priority order:
  x86-64-v4
  x86-64-v3 (supported, searched)
  x86-64-v2 (supported, searched)

Legacy HWCAP subdirectories under library search path directories:
  haswell (AT_PLATFORM; supported, searched)
  tls (supported, searched)
  avx512_1
  x86_64 (supported, searched
```

---

### **Setting up alerts**

Start by updating the system

```shell
apt update
```

Install mail notification dependencies

```shell
apt install libsasl2-modules mailutils
```

Create/login to your google account and generate an app password

Configure postfix

```shell
echo "smtp.gmail.com your-email@gmail.com:YourAppPassword" > /etc/postfix/sasl_passwd
```

Hash the file

```shell
postmap hash:/etc/postfix/sasl_passwd
```

Set chmod 600 on the password file

```shell
chmod 600 /etc/postfix/sasl_passwd
```

Make changes to the postfix configuration file

```shell
nano /etc/postfix/main.cf
```

Comment out the default relayhost and add the google credentials to the bottom of the file

```shell
..
..
# relayhost =
..
..
..

compatability_level = 2

# google mail configuration

relayhost = smtp.gmail.com:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options =
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/Entrust_Root_Certification_Authority.pem
smtp_tls_session_cache_database = btree:/var/lib/postfix/smtp_tls_session_cache
smtp_tls_session_cache_timeout = 3600s
```

Reload postfix

```shell
postfix reload
```

Test if it works by sending yourself an test notification

!!! Warning "The e-mail will most likely be put in your spam folder"

```shell
echo "This is a test message sent from postfix on my Proxmox Server" | mail -s "Test Email from Proxmox" your-email@gmail.com
```

If you want to change the name of the sender you need to install another dependency

```shell
apt update
apt install postfix-pcre
```

Edit the new configuration file

```shell
nano /etc/postfix/smtp_header_checks
```

Add the following text

```shell
/^From:.*/ REPLACE From: example-name <example@example.com>
```

Hash the file

```shell
postmap hash:/etc/postfix/smtp_header_checks
```

Verify the contents

```shell
cat /etc/postfix/smtp_header_checks.db
```

Open the postfix configuration file

```shell
nano /etc/postfix/main.cf
```

Add the new module to the bottom of the config

```shell
smtp_header_checks = pcre:/etc/postfix/smtp_header_checks
```

Reload the postfix service

```shell
postfix reload
```

Add the email-account you want to receive notifications from Proxmox to by going to Datacenter -> Permissions -> Users -> "username" -> Edit -> E-Mail: "example@example.com"

Enable alerts on your chosen services by adding your e-mail to the "Send email to:" input box for the various services which support it.

- Backups
- S.M.A.R.T
- ZFS

Check if ZFS alerts is enabled

```shell
nano /etc/zfs/zed.d/zed.rc
```

If ZED_EMAIL_ADDR="your_proxmox_user" (probably "root") is not commented out, then everything is good to go!

---

### **(Optional) Check Network Speed on Network Adapter**

Check Network Adapter Settings: Your network adapter settings might have change during some updates. So you may need to ensure they are set to the correct bandwidth. If it was altered manually or automatically limited to 100 Mbps, you will need to adjust it back to support higher speeds.

**1.** Open Terminal application.

**2.** List all network interfaces

```
ip addr show
```

Output:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp6s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master vmbr0 state UP group default qlen 1000
    link/ether 74:56:3c:0f:00:94 brd ff:ff:ff:ff:ff:ff
3: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 74:56:3c:0f:00:94 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.9/24 scope global vmbr0
       valid_lft forever preferred_lft forever
    inet6 fe80::7656:3cff:fe0f:94/64 scope link 
       valid_lft forever preferred_lft forever
```

**3.** Find the details of a specific network interface.

```
ethtool enp6s0
```

Output:

```
Settings for enp6s0:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
                                2500baseT/Full
        Supported pause frame use: Symmetric
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
                                2500baseT/Full
        Advertised pause frame use: Symmetric
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Speed: 100Mb/s
        Duplex: Full
        Auto-negotiation: on
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        MDI-X: off (auto)
        Supports Wake-on: pumbg
        Wake-on: g
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: yes
```

Based on the output, the network adapter's speed is indeed set to 100Mbps.

The Network adapter auto-negotiation is turned on, so it should be able to negotiate for the fastest speed available. But it can sometimes be constrained by the network infrastructure that lies behind it - routers, switches, or the quality of the physical cables used.

**4.** Force Speed and Duplex Settings: If your network adapter and cable support higher speeds but the problem persists, you can force the speed and duplex settings by using ethtool.

```
ethtool -s enp6s0 speed 2500 duplex full autoneg off 
```

```
ethtool enp6s0
```

```
Settings for enp6s0:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
                                2500baseT/Full
        Supported pause frame use: Symmetric
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
                                2500baseT/Full
        Advertised pause frame use: Symmetric
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Speed: 1000Mb/s
        Duplex: Full
        Auto-negotiation: on
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        MDI-X: off (auto)
        Supports Wake-on: pumbg
        Wake-on: g
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: yes
```

**5.** (Optional) Adding this command to crontab to ensure it is always applied, even after a reboot.

```
which ethtool
```

output:

```
/usr/sbin/ethtool
```

```
crontab -e
```

Add the following line:

```
@reboot /usr/sbin/ethtool -s enp6s0 speed 2500 duplex full autoneg off
@reboot /usr/sbin/ethtool -K enp6s0 rx off tx off

# Every hour, make sure the correct speed is set.
0 * * * * /usr/sbin/ethtool -s enp6s0 speed 2500 duplex full autoneg off
```

---

### **Troubleshooting**

1. VM does not receive expected IP

- Verify VM NIC tag and bridge assignment.
- Verify upstream switch trunk config and gateway VLAN interface.
- Verify clone Cloud-Init network settings.

2. Cloud-Init data not applied on clone

- Ensure Cloud-Init drive exists on clone.
- Re-run `qm cloudinit update <vmid>` before first boot.
- Check guest `/var/log/cloud-init.log`.

3. Performance regression on storage-heavy guests

- Verify storage backend health and saturation.
- Review disk flags (`discard`, `iothread`, cache mode).
- Separate noisy workloads by dataset/pool strategy.

4. Unexpected guest drift from desired config

- Stop editing guest hardware/network parameters manually.
- Reconcile desired state through Terraform and associated review process.

---

#### Guest VM hangs at 100% CPU Usage

First try

```shell
qm stop 101 (VM_ID)
```

Output from proxmox:

`can't lock file '/var/lock/qemu-server/lock-101.conf' - got timeout`

Then run

```shell
fuser /var/lock/qemu-server/lock-101.conf
```

Output:

`/run/lock/qemu-server/lock-101.conf: 3600250`

Run

```shell
kill 3600250
```

Then run 

```shell
qm stop 101
```

That should do the trick, now you can either start the VM again or do a full reboot.

---

### **Operational guardrails**

- Keep Proxmox host role focused on virtualization and storage orchestration.
- Keep guest lifecycle changes driven by IaC and reviewed workflows.
- Treat templates as immutable artifacts; rebuild rather than patching in place.
- Keep firewall and VLAN enforcement at network control layer, not per-VM assumptions.
