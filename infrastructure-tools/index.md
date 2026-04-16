---
title: Infrastructure tools
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-toolbox:


## **Infrastructure tools**

---

### **Purpose**

This section defines the delivery toolchain from base image creation to running Docker Compose workloads.

---

### **Primary path**

- Build Debian 13 template artifacts on Proxmox.
- Provision VMs and placement through Terraform.
- Apply host configuration through Ansible.
- Deploy and operate services with Docker Compose and Git-based workflows.

---

### **What belongs here**

- Provisioning and deployment toolchain contracts
- Image, VM, host, and runtime handoff boundaries
- Source-control and automation patterns for repeatable operations
- Compose expectations for application stacks

---

### **Canonical pages**

- [Git](git.md)
- [SSH](ssh.md)
- [Cloud-Init](cloud-init.md)
- [Packer](packer.md)
- [Terraform](terraform.md)
- [Ansible](ansible.md)
- [Docker](docker.md)
- [Gitea Actions](gitea-actions.md)
- [GitOps Workflow](gitops-workflow.md)

---

### **Secondary references**

- [CI/CD Overview](ci-cd-overview.md)
- [Kubernetes](kubernetes.md)
- [Applications](../applications/index.md)

---

### **Drafts / planned**

- No active draft toolchain pages are promoted in the main navigation during this pass.

---

### **Guardrails**

- Keep deployments tied to versioned artifacts and reviewed Git changes.
- Keep template build, VM provisioning, host configuration, and runtime responsibilities separated.
- Keep compose stacks aligned with the documented directory and secrets contract.
- Keep rollback paths tested for both IaC and compose-managed workloads.
