---
title: Watchtower
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-progress-download:


## **Watchtower**

---

### **Role summary**

Watchtower automates container image update checks and can recreate containers when new images appear. In this platform it is a convenience tool that needs clear limits, especially for stateful or user-facing services.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Usually attached to the Docker socket for inspection and rollout actions
- Best suited to low-risk or explicitly approved auto-update targets

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Diun](diun.md)
- [GitOps Workflow](../infrastructure-tools/gitops-workflow.md)

---

### **Post-install checklist**

- Decide whether the service is `monitor-only` or allowed to roll out updates.
- Exclude stateful, fragile, or publicly critical services by default.
- Configure notifications and maintenance expectations.
- Verify restart behavior and health checks after an update event.

---

### **Related canonical pages**

- [Docker](../infrastructure-tools/docker.md)
- [Diun](diun.md)
- [GitOps Workflow](../infrastructure-tools/gitops-workflow.md)
