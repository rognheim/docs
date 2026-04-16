---
title: Tautulli
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-playlist-check:


## **Tautulli**

---

### **Role summary**

Tautulli provides usage analytics and playback visibility for Plex. It is mainly an operational companion service for tracking activity, troubleshooting streams, and driving notifications.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Internal admin-facing companion service for Plex
- Persists historical playback and user-activity data separately from Plex itself

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Plex Media Server](plex.md)
- [Seerr](seerr.md)

---

### **Post-install checklist**

- Connect the Plex token and verify library visibility.
- Configure notification channels and event triggers.
- Set retention expectations for historical activity data.
- Confirm privacy expectations before exposing reports to other users.

---

### **Related canonical pages**

- [Plex Media Server](plex.md)
- [Seerr](seerr.md)
- [Observability](../observability/index.md)
