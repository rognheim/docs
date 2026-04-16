---
title: Hypervisors
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-vector-intersection:


## **Hypervisors**

---

### **Purpose**

This section defines the virtualization baseline for the platform and the handoff from hypervisor-managed infrastructure to guest-level automation.

---

### **Primary path**

- Proxmox VE is the only active production hypervisor baseline.
- Debian 13 Cloud-Init templates are the standard guest starting point.
- Packer, Terraform, and Ansible handle template-to-runtime handoff.

---

### **What belongs here**

- Proxmox host baseline and lifecycle guardrails
- VM network and storage attachment patterns
- Template boundaries before IaC and guest configuration take over
- Hypervisor-side operational notes that should not live in guest OS pages

---

### **Canonical pages**

- [Proxmox](proxmox.md)
- [Cloud-Init](../infrastructure-tools/cloud-init.md)
- [Packer](../infrastructure-tools/packer.md)
- [Terraform](../infrastructure-tools/terraform.md)
- [Ansible](../infrastructure-tools/ansible.md)

---

### **Secondary references**

- [VLAN](../networking/vlan.md)
- [Firewall](../networking/firewall.md)
- [ZFS Overview](../file-systems/zfs-overview.md)

---

### **Drafts / planned**

- No active draft hypervisor pages are exposed in this section right now.

---

### **Guardrails**

- Keep Proxmox VE as the only production hypervisor baseline.
- Treat templates as immutable artifacts instead of ad-hoc starting points.
- Keep hypervisor ownership separate from guest operating-system ownership.
- Keep manual VM drift out of the standard IaC workflow.
