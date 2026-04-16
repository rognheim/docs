---
title: Seerr
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-movie-plus:


## **Seerr**

---

### **Role summary**

Seerr is the request-management front end for the media stack. It sits between end users and the acquisition pipeline, coordinating requests with Plex, Sonarr, and Radarr.

---

### **Deployment model**

- Compose-managed service on a Debian host
- User-facing application that normally lives behind the standard edge stack
- Depends on a working Plex and media-automation backend

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Plex Media Server](plex.md)
- [Sonarr](sonarr.md)
- [Radarr](radarr.md)

---

### **Post-install checklist**

- Configure Plex, Sonarr, and Radarr integrations.
- Define request, approval, and quota policies for users.
- Validate notifications, webhooks, and sign-in behavior.
- Confirm that request paths match real library ownership and naming rules.

---

### **Related canonical pages**

- [Plex Media Server](plex.md)
- [Sonarr](sonarr.md)
- [Radarr](radarr.md)
