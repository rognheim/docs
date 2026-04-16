---
title: Radarr
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-movie:

## **Radarr**

---

### **Introduction**

Radarr owns movie acquisition and import. In this platform it should not be treated as a standalone app. It works correctly only when the downloader, library paths, indexers, and quality-profile automation all agree on the same media-stack contract.

---

### **Scope**

This page owns:

- Radarr deployment
- movie library and download-path decisions
- the small set of manual settings that still matter after Recyclarr

This page does not own:

- indexer management. See [Prowlarr](prowlarr.md)
- downloader setup. See [qBittorrent](qbittorrent.md)
- profile synchronization. See [Recyclarr](recyclarr.md)

---

### **Prerequisites**

- [Docker](../infrastructure-tools/docker.md)
- qBittorrent already writing to `/data/downloads/movies`
- A movie library path at `/data/movies`

---

### **Dependencies / related pages**

- [qBittorrent](qbittorrent.md)
- [Prowlarr](prowlarr.md)
- [Recyclarr](recyclarr.md)
- [Bazarr](bazarr.md)

---

### **Configuration files**

- `docker-compose.yml`
- Radarr application config under the mounted `/config` volume

---

### **Implementation**

#### **1. Deploy Radarr with the shared `/data` mount**

Example Compose service:

```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - TZ=Europe/Oslo
      - PUID=1000
      - PGID=1000
    volumes:
      - config:/config
      - /mnt/media:/data
    ports:
      - 7878:7878
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

#### **2. Set the movie root folder**

Inside Radarr, use:

```text
/data/movies
```

Do not point Radarr at the downloads directory. Its job is to import from downloads into the library, not to use downloads as the library itself.

#### **3. Add qBittorrent as the download client**

Recommended values:

- download client: qBittorrent
- category: `movies`
- completed download handling: enabled

Only add Remote Path Mapping if the downloader sees a truly different filesystem path than Radarr. If both containers mount the same host root as `/data`, leave it empty.

#### **4. Let Recyclarr manage profiles and custom formats**

Use Recyclarr for:

- quality profiles
- custom formats
- release-profile style synchronization

Keep these manual items in Radarr itself:

- root folders
- download client connectivity
- connect notifications
- import behavior that depends on your filesystem

#### **5. Apply the manual media-management settings that still matter**

Recommended baseline:

- Replace illegal characters: enabled
- Colon replacement: `Space Dash Space`
- Propers and repacks: `Do Not Prefer` when custom formats already handle preference logic

These are small settings, but they prevent noisy rename churn and keep the library readable.

#### **6. Validate hardlink-safe imports early**

A healthy import flow looks like this:

1. qBittorrent downloads to `/data/downloads/movies/<release>`
2. Radarr imports into `/data/movies/<movie>`
3. Seeding can continue because the source and destination share the same filesystem

If that flow is broken on day one, fix it before adding a full wanted list.

---

### **Validation / troubleshooting**

#### **Basic validation**

- test a movie search and grab
- confirm the torrent lands in the `movies` category in qBittorrent
- confirm Radarr imports into `/data/movies`
- confirm the source torrent can continue seeding after import if hardlinks are expected

#### **Troubleshooting checklist**

- If Radarr can download but not import:
  - compare the exact completed-download path reported in Activity
  - inspect whether Radarr and qBittorrent really share the same `/data` mount
- If imports copy instead of hardlink:
  - confirm downloads and library paths are on the same filesystem
- If quality settings drift over time:
  - re-run [Recyclarr](recyclarr.md) and inspect any manual edits made directly in Radarr

---

### **Known issues**

- Broken path design is the root cause of most “Radarr is not importing” problems.
- Recyclarr improves consistency, but it does not replace the need to set root folders, download clients, and filesystem-aware import behavior correctly.
