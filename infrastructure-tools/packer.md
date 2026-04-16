---
title: Packer
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-packer:


## **Packer**

---

### **Introduction**

Packer automates the creation and versioning of your Proxmox Debian 13 Cloud-Init base template.

In this platform, Packer is used to:

- Build template artifacts with predictable inputs
- Validate template quality before use in Terraform
- Publish a machine-readable manifest for handoff

---

### **Prerequisites**

- Packer installed on an automation host
- API access to Proxmox node
- Debian 13 cloud image URL and checksum source
- Template ID reservation strategy (example: `800` for `main` baseline)

---

### **Repository layout (recommended)**

```text
infrastructure/
  packer/
    proxmox/
      debian13.pkr.hcl
      debian13.auto.pkrvars.hcl
      scripts/
        create_template.sh
        validate_template.sh
    artifacts/
      manifest.json
```

---

### **Key files**

- `packer/proxmox/debian13.pkr.hcl`
- `packer/proxmox/debian13.auto.pkrvars.hcl`
- `packer/proxmox/scripts/create_template.sh`
- `packer/proxmox/scripts/validate_template.sh`
- `packer/artifacts/manifest.json`

---

### **Implementation workflow**

#### 1. Install Packer on Debian 13 automation host

```shell
sudo apt update
sudo apt install -y wget unzip
wget -O /tmp/packer.zip https://releases.hashicorp.com/packer/1.11.0/packer_1.11.0_linux_amd64.zip
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
packer version
```

#### 2. Define Packer build file

Example `debian13.pkr.hcl`:

```hcl
packer {
  required_version = ">= 1.10.0"
}

variable "template_id" {
  type    = string
  default = "800"
}

variable "proxmox_node" {
  type = string
}

variable "storage" {
  type    = string
  default = "local-zfs"
}

variable "image_url" {
  type    = string
  default = "https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2"
}

source "null" "debian13_template" {
  communicator = "none"
}

build {
  name    = "debian13-cloudinit-template"
  sources = ["source.null.debian13_template"]

  provisioner "shell-local" {
    inline = [
      "bash packer/proxmox/scripts/create_template.sh ${var.template_id} ${var.proxmox_node} ${var.storage} ${var.image_url}",
      "bash packer/proxmox/scripts/validate_template.sh ${var.template_id}"
    ]
  }

  post-processor "manifest" {
    output = "packer/artifacts/manifest.json"
  }
}
```

This pattern keeps Proxmox `qm` logic in scripts while Packer controls repeatable execution and artifact tracking.

#### 3. Create template script

Example `scripts/create_template.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

TEMPLATE_ID="$1"
NODE="$2"
STORAGE="$3"
IMAGE_URL="$4"
IMAGE_FILE="/root/debian-13-generic-amd64.qcow2"

wget -O "$IMAGE_FILE" "$IMAGE_URL"

qm stop "$TEMPLATE_ID" >/dev/null 2>&1 || true
qm destroy "$TEMPLATE_ID" --purge >/dev/null 2>&1 || true

qm create "$TEMPLATE_ID" \
  --name debian-13-cloudinit \
  --memory 4096 \
  --cores 4 \
  --cpu host \
  --machine q35 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,firewall=1

qm set "$TEMPLATE_ID" --scsi0 "${STORAGE}:0,import-from=${IMAGE_FILE},discard=on,iothread=1,ssd=1"
qm set "$TEMPLATE_ID" --boot order=scsi0
qm disk resize "$TEMPLATE_ID" scsi0 32G
qm set "$TEMPLATE_ID" --bios ovmf
qm set "$TEMPLATE_ID" --efidisk0 "${STORAGE}:1,format=raw,efitype=4m,pre-enrolled-keys=1"
qm set "$TEMPLATE_ID" --ide2 "${STORAGE}:cloudinit"
qm set "$TEMPLATE_ID" --agent enabled=1 --ostype l26 --onboot 1
qm template "$TEMPLATE_ID"
```

#### 4. Create validation script

Example `scripts/validate_template.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

TEMPLATE_ID="$1"

qm config "$TEMPLATE_ID" | rg -n "template: 1|ide2:|agent: 1|scsi0:" >/dev/null
qm config "$TEMPLATE_ID" | rg -n "name: debian-13-cloudinit" >/dev/null

echo "Template ${TEMPLATE_ID} validation passed"
```

#### 5. Define variables and run build

Example `debian13.auto.pkrvars.hcl`:

```hcl
template_id  = "800"
proxmox_node = "pve-01"
storage      = "local-zfs"
```

Run:

```shell
packer init packer/proxmox/debian13.pkr.hcl
packer fmt -check packer/proxmox/debian13.pkr.hcl
packer validate -var-file=packer/proxmox/debian13.auto.pkrvars.hcl packer/proxmox/debian13.pkr.hcl
packer build -var-file=packer/proxmox/debian13.auto.pkrvars.hcl packer/proxmox/debian13.pkr.hcl
```

---

### **Artifact handoff to Terraform**

After successful build:

- Publish the template identifier (`vmid`, name, revision tag)
- Commit `packer/artifacts/manifest.json` in infra change records
- Use manifest values as Terraform input (`template_name`, template revision)

See [Terraform](terraform.md) for consumption flow.

---

### **Validation checklist**

```shell
qm config 800
qm list | grep "800|debian-13-cloudinit"
cat packer/artifacts/manifest.json
```

Expected:

- Template exists and is marked `template: 1`.
- Cloud-Init drive and guest agent are configured.
- Manifest output exists for downstream reference.

---

### **Troubleshooting**

1. Build fails because VM ID already exists

- Ensure old instance/template is destroyed before recreate.
- Reserve non-overlapping ID ranges for template pipelines.

2. Packer succeeds but template is not usable

- Confirm `ide2` Cloud-Init drive is present.
- Confirm template conversion happened at end of script (`qm template`).

3. First clone has missing network/SSH data

- Template build is fine; issue is clone-level Cloud-Init settings.
- Validate with [Cloud-Init](cloud-init.md) clone workflow.

---

### **Operational guardrails**

- Treat Packer output as immutable artifact; avoid manual edits to live template.
- Promote template revisions through `stage` before production use.
- Keep script logic in repo and code-reviewed like application code.
- Align checksum and integrity controls with [Hashing](../security/hashing.md).
