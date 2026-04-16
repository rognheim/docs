---
title: VS Code Server
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-microsoft-visual-studio-code:


## **VS Code Server**

---

### **Role summary**

VS Code Server provides a browser-based maintenance and light-development environment hosted inside the platform. It is best treated as an operator tool rather than a primary production workload.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Usually internal-only or protected behind the standard edge stack
- Mount only the directories that genuinely need remote editing access

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Git](../infrastructure-tools/git.md)
- [SSH](../infrastructure-tools/ssh.md)

---

### **Post-install checklist**

- Configure authentication and disable anonymous access.
- Persist settings, extensions, and workspace state intentionally.
- Limit mounted paths to reduce accidental write access.
- Verify shell tools and extensions match the target host workflow.

---

### **Related canonical pages**

- [Docker](../infrastructure-tools/docker.md)
- [Git](../infrastructure-tools/git.md)
- [SSH](../infrastructure-tools/ssh.md)
