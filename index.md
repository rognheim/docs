---
title: Home
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-information-box:


## **Rognheim Server Documentation**

This site is a mixed notebook with a clear operational center: server infrastructure, networking, security, storage, automation, and application runbooks for a Proxmox and Debian based platform.

!!! Warning "Treat these pages as operator notes, not universal truth."
    Validate commands, versions, and assumptions before applying them to your own environment.

---

### **Start here**

- Build or rebuild a workload host: [Proxmox](hypervisors/proxmox.md), [Cloud-Init](infrastructure-tools/cloud-init.md), [Terraform](infrastructure-tools/terraform.md), [Ansible](infrastructure-tools/ansible.md)
- Standardize a service host: [Debian](operating-systems/debian.md), [Docker](infrastructure-tools/docker.md), [Secrets Management](security/secrets-management.md)
- Expose a service safely: [Traefik](networking/traefik.md), [Authelia](networking/authelia.md), [CrowdSec](networking/crowdsec.md), [Cloudflare](networking/cloudflare.md)
- Plan storage and backups: [ZFS Overview](file-systems/zfs-overview.md), [ZFS Settings](file-systems/zfs-settings.md), [Proxmox Backup Server](operating-systems/proxmox-backup-server.md)
- Monitor and troubleshoot: [Observability](observability/index.md), [Prometheus](observability/prometheus.md), [Grafana](observability/grafana.md), [Loki](observability/loki.md)
- Deploy or operate apps: [Applications](applications/index.md), [Guides](guides/index.md)

---

### **Top-level groups**

| Group | What it covers |
| --- | --- |
| Home | Site framing, contact details, and how to use the documentation |
| Platform | Hypervisor, storage, and operating-system baselines |
| Toolchain | Provisioning, automation, deployment, and source-control workflows |
| Edge & Security | Networking, trust boundaries, public ingress, and control layers |
| Operations | Observability and task-oriented operational guides |
| Applications | Service catalog, major runbooks, and app-specific operational notes |
| Reference | Shell, scripting, Markdown, and terminology notes that support the main platform docs |

---

### **How to read this site**

- Canonical pages define the preferred platform path and are the best starting point for repeatable work.
- Draft pages capture active work that still needs end-to-end verification or cleanup.
- Personal and peripheral notebook material may still exist in the repository, but it is intentionally outside the primary navigation.

Related context:

- [About This Site](meta/about-this-site.md)
- [Changelog](meta/changelog.md)
