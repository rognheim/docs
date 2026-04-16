---
title: Home Assistant OS
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-homeassistant:

## **Home Assistant OS**

---

### **Introduction**

Home Assistant OS is treated here as an appliance VM: one guest that owns home automation, integrations, and its own update lifecycle. The preferred pattern is to run it as a dedicated VM on Proxmox rather than mixing it into the general Docker stack.

---

### **Scope**

This page owns:

- the recommended Proxmox VM pattern for Home Assistant OS
- first-boot and onboarding validation
- optional USB and Bluetooth passthrough notes

This page does not own:

- general Proxmox host setup. See [Proxmox](../hypervisors/proxmox.md)
- reverse proxy exposure or external access strategy

---

### **Prerequisites**

- A Proxmox host with enough free RAM and storage for a dedicated appliance VM.
- The Home Assistant OS disk image downloaded to the Proxmox host.
- Any USB coordinator or Bluetooth adapter you intend to pass through identified ahead of time.

---

### **Dependencies / related pages**

- [Proxmox](../hypervisors/proxmox.md)

---

### **Configuration files**

- Proxmox VM config, for example `/etc/pve/qemu-server/250.conf`
- Optional Proxmox host modprobe blacklist file when passing through a combined Wi-Fi and Bluetooth device:
  - `/etc/modprobe.d/bluetooth-blacklist.conf`

---

### **Implementation**

#### **1. Create the VM shell in Proxmox**

Example values:

```bash
VMID=250
VMNAME=home-assistant-os
STORAGE=local-lvm
BRIDGE=vmbr0
```

Create the VM:

```bash
qm create "$VMID" --name "$VMNAME" --memory 4096 --cores 2 --machine q35 --bios ovmf --scsihw virtio-scsi-pci --net0 virtio,bridge="$BRIDGE"
qm set "$VMID" --efidisk0 "$STORAGE":0,pre-enrolled-keys=1
qm set "$VMID" --serial0 socket --vga serial0
```

This gives Home Assistant OS a simple UEFI VM with virtio networking and enough resources for a normal homelab deployment.

#### **2. Import the Home Assistant OS disk**

From the Proxmox host, in the directory containing the downloaded image:

```bash
unxz haos_ova-*.qcow2.xz
qm importdisk "$VMID" haos_ova-*.qcow2 "$STORAGE"
qm set "$VMID" --scsi0 "$STORAGE":vm-"$VMID"-disk-0,ssd=1
qm set "$VMID" --boot order=scsi0
```

If your storage backend uses a different naming convention, adjust the imported disk path after `qm importdisk` completes.

#### **3. Start the VM and complete onboarding**

```bash
qm start "$VMID"
qm terminal "$VMID"
```

Once the guest is up, open:

```text
http://homeassistant.local:8123
```

If local mDNS does not resolve, use the VM's DHCP address directly:

```text
http://<vm-ip>:8123
```

Complete the initial Home Assistant onboarding before adding passthrough hardware or advanced integrations.

#### **4. Add USB passthrough only after the appliance is healthy**

For Zigbee, Z-Wave, or Bluetooth adapters, attach the device to the VM from Proxmox after first boot and then confirm the integration appears inside Home Assistant.

Keep passthrough notes in the VM documentation so the next hardware migration is not guesswork.

#### **5. Optional: pass through a combined Wi-Fi and Bluetooth card**

If Proxmox host drivers claim the Bluetooth side of a combo card before passthrough works reliably, blacklist the Bluetooth modules on the host:

```bash
sudo nano /etc/modprobe.d/bluetooth-blacklist.conf
```

Add:

```ini
blacklist btintel
blacklist btusb
```

Rebuild initramfs and reboot the host:

```bash
sudo update-initramfs -k all -u
sudo reboot
```

Only do this when the adapter is intended primarily for passthrough. It can affect Bluetooth availability on the Proxmox host itself.

---

### **Validation / troubleshooting**

#### **Basic validation**

- the VM boots without dropping into the UEFI shell
- the web UI answers on port `8123`
- Home Assistant reports healthy storage and network access
- passthrough USB devices appear under the expected integrations

#### **Troubleshooting checklist**

- If the VM boots to UEFI instead of Home Assistant:
  - confirm the imported disk is attached as `scsi0`
  - confirm the boot order points to `scsi0`
- If the UI does not load:
  - confirm the VM has a DHCP address
  - check the VM console for first-boot storage expansion or recovery output
- If Bluetooth passthrough is unstable:
  - confirm the device is not still claimed by the Proxmox host
  - review any host-side blacklist changes after kernel updates

---

### **Known issues**

- Home Assistant OS behaves best as a dedicated appliance. Mixing it with unrelated server duties usually makes upgrades and troubleshooting harder.
- Bluetooth and USB passthrough can be sensitive to host kernel changes, so hardware-dependent integrations should be revalidated after major Proxmox updates.
