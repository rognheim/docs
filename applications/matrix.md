---
title: Matrix
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-matrix:


## **Matrix**

---

### **Role summary**

Matrix is the collaboration and messaging service reference in the application catalog. It is a higher-complexity workload because it combines persistent data, federation concerns, and public-facing identity and TLS requirements.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Usually exposed publicly with careful DNS, TLS, and reverse-proxy configuration
- Persistent database, media, and config storage should be separated intentionally

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Traefik](../networking/traefik.md)
- [Cloudflare](../networking/cloudflare.md)
- [Secrets Management](../security/secrets-management.md)

---

### **Post-install checklist**

- Complete admin bootstrap and initial homeserver configuration.
- Validate federation, trusted proxies, and TLS behavior.
- Set media limits, retention expectations, and backup coverage.
- Keep this page aligned with the real deployment before expanding it into a full runbook.

---

### **Related canonical pages**

- [Traefik](../networking/traefik.md)
- [DNS](../networking/dns.md)
- [Secrets Management](../security/secrets-management.md)
