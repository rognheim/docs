---
title: Prowlarr
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-calendar-text:


## **Prowlarr**

---

### **Role summary**

Prowlarr centralizes indexer management for the media-automation stack. It reduces duplication by synchronizing indexer configuration across services such as Sonarr, Radarr, and Lidarr.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Internal stack component rather than a user-facing application
- Best treated as the indexer control plane for the media workflow

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [qBittorrent](qbittorrent.md)
- [Sonarr](sonarr.md)
- [Radarr](radarr.md)
- [Lidarr](lidarr.md)

---

### **Post-install checklist**

- Add and test indexers before syncing them to downstream apps.
- Configure category mappings and application connections intentionally.
- Review sync behavior so one bad change does not propagate everywhere.
- Keep API credentials and secrets out of tracked files.

---

### **Related canonical pages**

- [Sonarr](sonarr.md)
- [Radarr](radarr.md)
- [qBittorrent](qbittorrent.md)
