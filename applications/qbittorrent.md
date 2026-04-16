---
title: qBittorrent
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-download-network:

## **qBittorrent**

---

### **Introduction**

qBittorrent is the downloader in the media-automation stack. Its job is simple: receive downloads from the `*arr` applications, write them into the shared `/data/downloads` tree, and stay out of the way of hardlink-safe imports into the library.

---

### **Scope**

This page owns:

- qBittorrent deployment and filesystem layout
- the category model used by the media stack
- validation steps that prevent broken imports later

This page does not own:

- indexer management. See [Prowlarr](prowlarr.md)
- movie automation. See [Radarr](radarr.md)
- TV automation. See [Sonarr](sonarr.md)

---

### **Prerequisites**

- [Docker](../infrastructure-tools/docker.md)
- A single shared media root mounted into every media application as `/data`
- Enough free space for incomplete and completed downloads

---

### **Dependencies / related pages**

- [Prowlarr](prowlarr.md)
- [Radarr](radarr.md)
- [Sonarr](sonarr.md)
- [Recyclarr](recyclarr.md)

---

### **Configuration files**

- `docker-compose.yml`
- qBittorrent application config under the mounted `/config` volume

---

### **Implementation**

#### **1. Keep the media stack on one shared path model**

Use the same container mount everywhere:

```text
Host path:   /mnt/media
Container:   /data
```

Expected structure:

```text
/data/downloads
/data/downloads/movies
/data/downloads/tv
/data/downloads/music
/data/movies
/data/tv
/data/music
```

This is the most important design choice in the stack. If qBittorrent sees one path and Radarr or Sonarr see another, imports become fragile and hardlinks stop working.

#### **2. Deploy qBittorrent with the shared `/data` mount**

Example Compose service:

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    environment:
      - TZ=Europe/Oslo
      - PUID=1000
      - PGID=1000
      - WEBUI_PORT=8080
    volumes:
      - config:/config
      - /mnt/media:/data
    ports:
      - 8080:8080
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

#### **3. Set sane download paths inside qBittorrent**

Recommended defaults:

- Default save path: `/data/downloads`
- Incomplete torrents: optional `/data/downloads/.incomplete`
- Keep incomplete torrents in: enabled only if you actually use the separate incomplete path
- Torrent Management Mode: `Automatic`

#### **4. Create the shared category model**

Create these categories in qBittorrent:

```text
movies -> /data/downloads/movies
tv     -> /data/downloads/tv
music  -> /data/downloads/music
```

Those category names should match what Radarr, Sonarr, and Lidarr send to the downloader.

#### **5. Wire the application into the rest of the stack**

- Add qBittorrent as a download client in Radarr and Sonarr.
- Use the same hostname and port visible on the Docker network.
- Do not configure Remote Path Mapping unless the downloader truly has a different filesystem view than the `*arr` app.

If every container mounts the same host media root as `/data`, Remote Path Mapping should usually stay empty.

#### **6. Optional: Debian LXC deployment**

If qBittorrent runs in its own Debian LXC instead of Docker, keep the same path contract and group permissions.

Minimal service setup:

```bash
sudo apt-get update
sudo apt-get install -y qbittorrent-nox
sudo adduser --system --group qbittorrent-nox
```

Example service unit:

```ini
[Unit]
Description=qBittorrent Command Line Client
After=network.target

[Service]
Type=forking
User=qbittorrent-nox
Group=qbittorrent-nox
UMask=007
ExecStart=/usr/bin/qbittorrent-nox -d --webui-port=8080
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now qbittorrent-nox
```

If you use an SMB or NFS mount in an unprivileged LXC, make sure the qBittorrent service user actually owns the mounted files from inside the container.

---

### **Validation / troubleshooting**

#### **Basic validation**

- qBittorrent Web UI loads on port `8080`
- category directories are created under `/data/downloads`
- a test torrent completes into the expected category path
- Radarr or Sonarr can see the completed download without a Remote Path Mapping hack

#### **Troubleshooting checklist**

- If imports fail:
  - compare the exact path seen by qBittorrent with the path seen by the `*arr` container
  - verify both applications mount the same host media root at `/data`
- If hardlinks do not work:
  - verify downloads and final libraries live on the same filesystem
  - avoid separate mounts like `/downloads` and `/movies` when they point to different underlying filesystems
- If permissions break in an LXC deployment:
  - inspect the UID and GID of the qBittorrent service user
  - confirm the mounted share maps ownership correctly inside the container

---

### **Known issues**

- Remote Path Mapping is often used to paper over inconsistent mounts. Fix the filesystem layout first.
- Hardlink-safe imports depend on downloads and final media paths living on the same filesystem boundary.
