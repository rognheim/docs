---
title: Terraform
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-terraform:


## **Terraform**

---

### **Introduction**

Terraform manages VM lifecycle on Proxmox using the Debian 13 Cloud-Init template as source.

In this platform, Terraform is responsible for:

- VM cloning and sizing
- Network and VLAN placement
- Cloud-Init identity and IP configuration
- Declarative drift-aware lifecycle control

---

### **Prerequisites**

- Existing Proxmox template from [Cloud-Init](cloud-init.md)/[Packer](packer.md)
- Proxmox API token with least privilege
- Terraform CLI installed on automation host
- SSH public keys for Cloud-Init user

---

### **Repository layout (recommended)**

```text
infrastructure/
  terraform/
    envs/
      stage/
        main.tf
        variables.tf
        terraform.tfvars
      prod/
        main.tf
        variables.tf
        terraform.tfvars
    modules/
      vm/
        main.tf
        variables.tf
        outputs.tf
```

---

### **Key files**

- `terraform/modules/vm/main.tf`
- `terraform/modules/vm/variables.tf`
- `terraform/envs/<env>/main.tf`
- `terraform/envs/<env>/terraform.tfvars`

---

### **Implementation workflow**

#### 1. Install Terraform

```shell
sudo apt update
sudo apt install -y wget unzip
wget -O /tmp/terraform.zip https://releases.hashicorp.com/terraform/1.11.0/terraform_1.11.0_linux_amd64.zip
sudo unzip -o /tmp/terraform.zip -d /usr/local/bin
terraform version
```

#### 2. Define VM module interface

Example `modules/vm/variables.tf`:

```hcl
variable "vm_id" { type = number }
variable "name" { type = string }
variable "target_node" { type = string }
variable "template_name" { type = string }
variable "storage" { type = string }
variable "cores" { type = number }
variable "memory" { type = number }
variable "disk_size" { type = string }
variable "bridge" { type = string }
variable "vlan_tag" { type = number }
variable "cloudinit_user" { type = string }
variable "ssh_public_key" { type = string }
variable "ip_cidr" { type = string }
variable "gateway" { type = string }
```

#### 3. Implement Proxmox VM resource

Example `modules/vm/main.tf`:

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "Telmate/proxmox"
      version = "~> 2.9"
    }
  }
}

resource "proxmox_vm_qemu" "this" {
  vmid        = var.vm_id
  name        = var.name
  target_node = var.target_node
  clone       = var.template_name
  full_clone  = true

  os_type = "cloud-init"
  agent   = 1
  cores   = var.cores
  memory  = var.memory
  scsihw  = "virtio-scsi-single"

  disk {
    slot    = "scsi0"
    storage = var.storage
    type    = "disk"
    size    = var.disk_size
    iothread = 1
  }

  network {
    model    = "virtio"
    bridge   = var.bridge
    tag      = var.vlan_tag
    firewall = true
  }

  ciuser    = var.cloudinit_user
  sshkeys   = var.ssh_public_key
  ipconfig0 = "ip=${var.ip_cidr},gw=${var.gateway}"
}
```

#### 4. Compose environment stacks

Example `envs/prod/main.tf`:

```hcl
provider "proxmox" {
  pm_api_url          = var.pm_api_url
  pm_api_token_id     = var.pm_api_token_id
  pm_api_token_secret = var.pm_api_token_secret
  pm_tls_insecure     = false
}

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

Use separate tfvars for each environment and keep secrets out of Git.

#### 5. Run plan/apply safely

```shell
cd terraform/envs/prod
terraform init
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

#### 6. Controlled destroy flow

For retire/decommission only:

```shell
terraform plan -destroy -out=destroy.tfplan
terraform apply destroy.tfplan
```

---

### **State and variable strategy**

- Keep `prod` and `stage` state isolated.
- Store sensitive inputs via environment variables or external secret injection.
- Pin provider versions and module contracts to prevent uncontrolled drift.

Operational minimum:

- `TF_VAR_pm_api_url`
- `TF_VAR_pm_api_token_id`
- `TF_VAR_pm_api_token_secret`

---

### **Validation checklist**

```shell
terraform state list
terraform show -json tfplan | jq '.resource_changes[].change.actions'
qm list | rg "901|srv-app-01"
```

Expected:

- State matches intended VM resources.
- Plan actions are explicit (`create`, `update`, `delete`).
- Proxmox VM properties align with declared Terraform values.

---

### **Troubleshooting**

1. API auth failures

- Validate token ID/secret and API URL.
- Confirm token role has VM clone/create/network permissions.

2. Clone failure from template

- Confirm template exists and is accessible on target node/storage.
- Confirm template is Cloud-Init-enabled.

3. Terraform drift after manual Proxmox edits

- Stop manual VM edits outside IaC.
- Reconcile with `terraform plan` and apply desired state.

---

### **Operational guardrails**

- Never apply Terraform changes to production without reviewed plan output.
- Use pull requests and CI checks before `apply`.
- Keep VM application configuration out of Terraform; hand off to [Ansible](ansible.md).
- Treat template selection as a controlled input from [Packer](packer.md) artifact output.
