---
title: Diun
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-vibrate:


## **Diun**

---

### **Role summary**

Diun monitors container registries and notifies operators when new image tags are published. It is the lower-risk companion to Watchtower because it surfaces update awareness without forcing deployment changes.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Internal operator-facing utility with outbound notification access
- Best used as an update signal in front of manual or Git-driven change management

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Watchtower](watchtower.md)
- [Gitea Actions](../infrastructure-tools/gitea-actions.md)

---

### **Post-install checklist**

- Verify notification credentials and delivery targets.
- Confirm polling schedule and include or exclude filters.
- Trigger a test event to validate message formatting.
- Decide which services should only alert and which may auto-update elsewhere.

---

### **Related canonical pages**

- [Docker](../infrastructure-tools/docker.md)
- [Watchtower](watchtower.md)
- [GitOps Workflow](../infrastructure-tools/gitops-workflow.md)
