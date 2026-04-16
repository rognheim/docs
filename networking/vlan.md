---
title: VLAN
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-lan:


## **VLAN**

---

### **Introduction**

This page defines VLAN segmentation and allowed traffic patterns for the platform.

Primary objective: reduce blast radius and lateral movement while preserving required service flows.

---

### **Prerequisites**

- Managed gateway and switch with VLAN support (UniFi stack)
- Proxmox bridges and VM NIC tags mapped to target VLANs
- DNS and firewall runbooks in place:
  - [DNS](dns.md)
  - [Firewall](firewall.md)
- Security control model from [Security](../security/index.md)

---

### **VLAN and subnet baseline**

| VLAN | Subnet | Role | Internet | Notes |
| --- | --- | --- | --- | --- |
| 1 | 10.0.1.0/24 | Management | Yes | Admin plane only |
| 10 | 10.0.10.0/24 | Trusted clients | Yes | User devices |
| 20 | 10.0.20.0/24 | Internal services (SRV) | Restricted | App and monitoring VMs |
| 30 | 10.0.30.0/24 | External services (DMZ) | Yes | Reverse-proxy entry workloads |
| 40 | 10.0.40.0/24 | Media | Yes | Media-focused services |
| 50 | 10.0.50.0/24 | Storage/PBS | No | Backup/storage boundary |
| 60 | 10.0.60.0/24 | IoT/Lab | Yes | Treated as untrusted |
| 99 | 10.0.99.0/25 | Quarantine | No | Isolation network |

---

### **Configuration files and control points**

- UniFi network/VLAN definitions in controller UI
- Proxmox host network config: `/etc/network/interfaces`
- VM network tags in Proxmox VM config: `/etc/pve/qemu-server/<vmid>.conf`
- Gateway ACL policy set (documented in [Firewall](firewall.md))

---

### **Implementation workflow**

#### 1. Define VLANs on gateway and switch

Create VLAN IDs and subnet interfaces first, then assign switch ports as tagged/untagged per device role.

Baseline examples:

- Access points: trunk/tagged for client SSIDs mapped to VLANs
- Hypervisor uplink: trunk/tagged for VM VLAN mobility
- Dedicated management port: untagged management VLAN where appropriate

#### 2. Map Proxmox guest NICs to VLAN tags

For VM in SRV VLAN:

```shell
qm set 901 --net0 virtio,bridge=vmbr0,tag=20,firewall=1
```

For DMZ edge VM:

```shell
qm set 930 --net0 virtio,bridge=vmbr0,tag=30,firewall=1
```

Provisioning lifecycle references:

- [Cloud-Init](../infrastructure-tools/cloud-init.md)
- [Terraform](../infrastructure-tools/terraform.md)

#### 3. Apply trust-boundary ACL model

High-level rules:

- Allow established/related traffic.
- Allow management VLAN to required management ports only.
- Deny broad RFC1918 east-west by default.
- Add explicit exceptions per service dependency.
- Keep quarantine VLAN fully denied except incident-response access.

#### 4. Implement explicit service exceptions

Use exception rules for required paths only, for example:

- DMZ -> specific SRV service port (application API callback)
- SRV -> Storage/PBS for backup and data paths
- IoT/Lab -> DNS resolver only
- Trusted -> selected media or home automation endpoints

This aligns with your current ACL style and preserves minimum necessary connectivity.

#### 5. Document service-to-VLAN placement

Maintain per-service mapping in operations docs:

- Reverse proxy/auth stack in DMZ or tightly controlled edge segment
- Monitoring services in SRV
- Storage endpoints in Storage/PBS VLAN

---

### **Validation checklist**

- VM gets expected gateway and DNS for its VLAN.
- Cross-VLAN probes fail unless explicitly allowed.
- Required application dependencies are reachable.
- Quarantine VLAN hosts cannot reach internal networks.

Example checks:

```shell
ip -brief addr
ip route
ping -c 2 10.0.20.13
nc -zv 10.0.20.21 53
```

---

### **Troubleshooting**

1. VM in wrong network

- Check Proxmox NIC `tag` value.
- Confirm switch port is trunk/tagged correctly.
- Confirm gateway has VLAN interface and DHCP scope if used.

2. Legitimate service flow blocked

- Confirm explicit allow rule exists above broad deny.
- Confirm destination IP/port object is correct.
- Validate protocol (TCP/UDP) and state handling.

3. Unexpected lateral access

- Review any broad allow rules to RFC1918 ranges.
- Check for overlapping address objects.
- Re-apply default inter-VLAN deny and exceptions.

---

### **Operational guardrails**

- Default-deny inter-VLAN remains mandatory.
- Every allow rule must map to a documented service dependency.
- Keep management access separated from user/device VLANs.
- Review VLAN exception set after each new service onboarding.
