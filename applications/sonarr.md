---
title: Sonarr
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-television:

## **Sonarr**

---

### **Introduction**

Sonarr is the TV-series automation part of the media stack. Like Radarr, it works best when it shares the same `/data` view as the downloader and when manual settings are kept small and intentional.

---

### **Scope**

This page owns:

- Sonarr deployment
- TV library and download-path decisions
- the manual settings that remain after Recyclarr

This page does not own:

- indexer management. See [Prowlarr](prowlarr.md)
- downloader setup. See [qBittorrent](qbittorrent.md)
- profile synchronization. See [Recyclarr](recyclarr.md)

---

### **Prerequisites**

- [Docker](../infrastructure-tools/docker.md)
- qBittorrent already writing to `/data/downloads/tv`
- A TV library path at `/data/tv`

---

### **Dependencies / related pages**

- [qBittorrent](qbittorrent.md)
- [Prowlarr](prowlarr.md)
- [Recyclarr](recyclarr.md)
- [Bazarr](bazarr.md)

---

### **Configuration files**

- `docker-compose.yml`
- Sonarr application config under the mounted `/config` volume

---

### **Implementation**

#### **1. Deploy Sonarr with the shared `/data` mount**

Example Compose service:

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - TZ=Europe/Oslo
      - PUID=1000
      - PGID=1000
    volumes:
      - config:/config
      - /mnt/media:/data
    ports:
      - 8989:8989
    networks:
      - local

volumes:
  config:

networks:
  local:
    external: true
```

Start the service:

```bash
docker compose up -d
```

#### **2. Set the TV root folder**

Inside Sonarr, use:

```text
/data/tv
```

Keep downloads under `/data/downloads/tv`. The library and the download area should be different paths on the same filesystem, not the same folder.

#### **3. Add qBittorrent as the download client**

Recommended values:

- download client: qBittorrent
- category: `tv`
- completed download handling: enabled

As with Radarr, leave Remote Path Mapping empty unless the downloader truly sees a different filesystem path.

#### **4. Let Recyclarr own the profile logic**

Use Recyclarr for:

- quality profile sync
- custom formats
- release-profile style policy

Keep these manual Sonarr settings intentional:

- root folder
- download client
- metadata and connect integrations
- naming and season-folder preferences

#### **5. Apply the manual naming and media-management settings**

Recommended baseline:

- Replace illegal characters: enabled
- Colon replacement: `Smart Replace`
- Season folder format: `Season {season:00}`
- Multi-episode style: `Prefixed Range`
- Propers and repacks: `Do Not Prefer` when custom formats already manage preference

These keep the library stable and readable without fighting the profile automation layer.

#### **6. Validate the first import before scaling up**

Healthy flow:

1. Sonarr sends a grab to qBittorrent with category `tv`
2. qBittorrent writes to `/data/downloads/tv`
3. Sonarr imports into `/data/tv`
4. the source files remain seedable when hardlinks are expected

---

### **Validation / troubleshooting**

#### **Basic validation**

- grab one test episode or series
- confirm the torrent lands in qBittorrent category `tv`
- confirm Sonarr imports into `/data/tv`
- confirm season folders are created as expected

#### **Troubleshooting checklist**

- If Sonarr reports missing or unmapped completed downloads:
  - inspect the download path shown in Activity
  - compare it with the path visible inside the Sonarr container
- If imports copy instead of hardlink:
  - verify both paths are on the same filesystem
- If naming keeps changing unexpectedly:
  - check whether manual edits in Sonarr are fighting Recyclarr-managed profile behavior

---

### **Known issues**

- Sonarr import failures are usually path problems, not indexer problems.
- Remote Path Mapping should be rare in a clean containerized media stack. If you need it everywhere, the mount design is probably wrong.
