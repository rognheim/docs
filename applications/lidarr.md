---
title: Lidarr
status: draft
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-music:


## **Lidarr**

---

### **Role summary**

Lidarr manages music acquisition and library organization inside the media stack. In this environment it remains a draft catalog page because metadata and retagging behavior need extra care in hardlinked seeding workflows.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Usually paired with download clients, indexers, and shared media paths
- Requires careful path mapping when hardlinks or seeding workflows are involved

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [Prowlarr](prowlarr.md)
- [qBittorrent](qbittorrent.md)
- [Recyclarr](recyclarr.md)

---

### **Post-install checklist**

- Configure indexers and download clients first.
- Validate root folders, path mappings, and hardlink behavior.
- Review metadata and retagging settings before enabling automation.
- Treat this page as draft until the workflow is verified end to end.

---

### **Related canonical pages**

- [Prowlarr](prowlarr.md)
- [qBittorrent](qbittorrent.md)
- [Sonarr](sonarr.md)
