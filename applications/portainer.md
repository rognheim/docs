---
title: Portainer
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-portainer:


## **Portainer**

---

### **Role summary**

Portainer is the convenience UI for routine Docker administration in this platform. It is useful for inspection and low-risk operational tasks, but Git-managed compose files and CLI workflows remain the source of truth.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Admin-facing service, preferably kept internal or protected behind the standard edge stack
- Persistent Portainer state stored separately from application stacks

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)

---

### **Post-install checklist**

- Complete admin bootstrap and rotate default credentials.
- Restrict endpoints, teams, and stack-level permissions.
- Back up Portainer settings before using it for recurring admin tasks.
- Keep Git and compose files as the canonical deployment record.

---

### **Related canonical pages**

- [Docker](../infrastructure-tools/docker.md)
- [GitOps Workflow](../infrastructure-tools/gitops-workflow.md)
- [Watchtower](watchtower.md)
