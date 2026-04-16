---
title: Provision a New Production VM
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-server-plus:


## **Provision a New Production VM**

---

### **Goal**

Provision a Debian 13 VM from the standard Proxmox Cloud-Init template, then hand it off cleanly through Terraform and Ansible until it is ready for application deployment.

---

### **When to use this guide**

- You are creating a new service host or utility VM.
- You want repeatable provisioning instead of a manually cloned one-off VM.
- You need a checklist that spans template, IaC, first boot, and baseline configuration.

Use the canonical runbooks for deeper detail:

- [Proxmox](../hypervisors/proxmox.md)
- [Cloud-Init](../infrastructure-tools/cloud-init.md)
- [Terraform](../infrastructure-tools/terraform.md)
- [Ansible](../infrastructure-tools/ansible.md)
- [Debian](../operating-systems/debian.md)

---

### **Prerequisites**

- The Debian 13 Cloud-Init template already exists in Proxmox.
- Terraform access to the Proxmox API is working.
- Ansible can reach target hosts with SSH keys.
- VLAN, DNS, and IP allocation for the new VM are already decided.

---

### **Inputs**

Capture these before you write or apply anything:

| Field | Example |
| --- | --- |
| VM name | `srv-app-01` |
| VM ID | `901` |
| Proxmox node | `pve-01` |
| VLAN tag | `20` |
| IP/CIDR | `10.0.20.21/24` |
| Gateway | `10.0.20.1` |
| Resolver | `10.0.20.21` |
| vCPU / RAM / disk | `4 / 8192 / 64G` |
| Host role | `internal app host` |

---

### **Step-by-step workflow**

#### 1. Confirm the template and handoff path are ready

Validate the template before touching Terraform:

```shell
qm list
qm config 8000
```

Confirm:

- the Debian 13 template exists
- guest agent is enabled
- Cloud-Init is attached
- the template is the same baseline your Terraform module expects

#### 2. Define the VM in Terraform

Add the VM to the correct environment stack.

Example:

```hcl
module "srv_app_01" {
  source = "../../modules/vm"

  vm_id         = 901
  name          = "srv-app-01"
  target_node   = "pve-01"
  template_name = "debian-13-cloudinit"
  storage       = "local-zfs"
  cores         = 4
  memory        = 8192
  disk_size     = "64G"
  bridge        = "vmbr0"
  vlan_tag      = 20

  cloudinit_user = "admin"
  ssh_public_key = file("~/.ssh/id_ed25519.pub")
  ip_cidr        = "10.0.20.21/24"
  gateway        = "10.0.20.1"
}
```

Keep naming, tagging, and IP assignment aligned with the platform naming scheme before planning.

#### 3. Plan and apply Terraform cleanly

```shell
cd terraform/envs/prod
terraform init
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

At this point Terraform should:

- clone the VM from the template
- apply CPU, memory, disk, network, and Cloud-Init settings
- boot the VM or leave it ready according to your module behavior

#### 4. Verify first boot before running Ansible

Use either the Proxmox console or SSH:

```shell
ssh admin@10.0.20.21 "cloud-init status --wait && hostnamectl && ip -brief addr"
```

Confirm:

- Cloud-Init finished successfully
- hostname and IP match the declared values
- guest agent is running
- SSH key access works without fallback passwords

#### 5. Run Ansible in layers

Start with connectivity:

```shell
cd infrastructure/ansible
ansible all -m ping -l srv-app-01
```

Then apply the baseline sequence:

```shell
ansible-playbook playbooks/bootstrap.yml -l srv-app-01
ansible-playbook playbooks/baseline.yml -l srv-app-01
ansible-playbook playbooks/docker.yml -l srv-app-01
```

Use the edge playbook only if the VM is supposed to host edge services:

```shell
ansible-playbook playbooks/edge.yml -l srv-app-01
```

#### 6. Validate the host role before handing it to applications

For a typical app host:

```shell
ssh admin@10.0.20.21 "\
  systemctl is-active qemu-guest-agent && \
  docker --version && \
  docker compose version && \
  sudo ufw status verbose"
```

Make sure the host is ready for its actual workload before you deploy any stack.

#### 7. Record the service onboarding facts

Before calling the VM "done", document:

- owner and purpose
- exposure model
- backup path
- monitoring expectation
- update strategy

This check comes directly from the archived onboarding and readiness checklists in the repository and should be treated as part of provisioning, not an optional afterthought.

---

### **Validation**

A VM provisioned through this guide is ready when:

- Terraform shows no unexpected drift after the first apply
- Cloud-Init completed successfully
- Ansible can run idempotently against the host
- host hardening and Docker prerequisites are in place
- DNS, routing, and firewall behavior match the intended role

Optional final check:

```shell
terraform plan
ansible-playbook playbooks/baseline.yml -l srv-app-01 --check
```

---

### **Rollback / failure modes**

- If Terraform cloned the wrong settings, fix the module inputs and re-run `terraform plan` before making manual VM edits.
- If Cloud-Init failed on first boot, stop and fix the template or module values rather than layering manual fixes into the guest.
- If Ansible fails during bootstrap, correct the role or inventory and rerun the playbook; do not drift the host by hand unless it is an emergency.
- If the VM is irreparably wrong and no application data exists yet, destroy and recreate it through Terraform rather than repairing it ad hoc.
