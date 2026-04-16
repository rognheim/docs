---
title: Applications
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-application-brackets:


## **Applications**

---

### **Purpose**

This section is the application catalog for compose-managed services in the platform.

---

### **Primary path**

- Use this section to understand service roles, dependencies, and app-specific operational checks.
- Use the Toolchain, Networking, and Security sections for shared platform rules that apply across multiple apps.

---

### **What belongs here**

- Application-level install and post-install notes
- Service-specific dependencies and known constraints
- Catalog pages that describe how each app fits into the wider platform

---

### **Shared media stack contract**

The media automation pages in this section assume one consistent path model:

- All media-stack containers mount the same host media root at `/data` inside the container.
- Downloads live under `/data/downloads`.
- Final libraries live under `/data/movies`, `/data/tv`, and `/data/music`.
- qBittorrent categories match the automation tools: `movies`, `tv`, and `music`.
- Remote Path Mapping is only needed when the downloader does not share the same `/data` view as the *arr applications.

---

### **Canonical pages**

- [Plex Media Server](plex.md)
- [qBittorrent](qbittorrent.md)
- [Radarr](radarr.md)
- [Sonarr](sonarr.md)
- [Bazarr](bazarr.md)

---

### **Secondary references**

Catalog notes and shorter app references:

- [Portainer](portainer.md)
- [VS Code Server](vscode.md)
- [MkDocs Material](mkdocs-material.md)
- [Watchtower](watchtower.md)
- [Diun](diun.md)
- [Prowlarr](prowlarr.md)
- [Recyclarr](recyclarr.md)
- [Seerr](seerr.md)
- [Tautulli](tautulli.md)
- [Matrix](matrix.md)
- [Syncthing](syncthing.md)
- [File Browser](filebrowser.md)

Shared platform dependencies:

- [Docker](../infrastructure-tools/docker.md)
- [Secrets Management](../security/secrets-management.md)
- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)
- [CrowdSec](../networking/crowdsec.md)

---

### **Drafts / planned**

- [Lidarr](lidarr.md)
- [Application Template](template.md) is retained as a hidden scaffold, not a navigational destination.

---

### **Guardrails**

- Keep every application deployed and versioned through the documented compose workflow.
- Keep secrets out of Git and out of `.env` files.
- Keep public routes behind standardized edge middleware and TLS policy.
- Keep backup expectations, data paths, and health checks documented per app.
