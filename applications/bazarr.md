---
title: Bazarr
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-subtitles:

## **Bazarr**

---

### **Introduction**

Bazarr manages subtitle acquisition for the existing movie and TV libraries. It should be added after Radarr and Sonarr are already stable, because Bazarr depends on their library metadata, naming, and path consistency.

---

### **Scope**

This page owns:

- Bazarr deployment
- subtitle-provider and language-profile baseline settings
- the integration points with Radarr and Sonarr

This page does not own:

- movie library automation. See [Radarr](radarr.md)
- TV automation. See [Sonarr](sonarr.md)

---

### **Prerequisites**

- [Docker](../infrastructure-tools/docker.md)
- A working Radarr and Sonarr installation
- Shared library paths mounted consistently into the media applications

---

### **Dependencies / related pages**

- [Radarr](radarr.md)
- [Sonarr](sonarr.md)
- [Prowlarr](prowlarr.md)

---

### **Configuration files**

- `docker-compose.yml`
- Bazarr application config under the mounted `/config` volume

---

### **Implementation**

#### **1. Deploy Bazarr with the same library view**

Example Compose service:

```yaml
services:
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - TZ=Europe/Oslo
      - PUID=1000
      - PGID=1000
    volumes:
      - config:/config
      - /mnt/media:/data
    ports:
      - 6767:6767
    networks:
      - local

volumes:
  config:

networks:
  local:
    external: true
```

Start it:

```bash
docker compose up -d
```

#### **2. Add Radarr and Sonarr first**

Before adding subtitle providers, connect Bazarr to:

- Radarr
- Sonarr

Use the internal Docker hostname and API keys from each application. The important part is that Bazarr sees the same `/data/movies` and `/data/tv` paths they do.

#### **3. Create a deliberate language profile**

Simple baseline:

- default language filter: English
- custom profile: `English`
- include normal and hearing-impaired variants only if you actually want both

The goal is to prevent Bazarr from pulling every possible subtitle variant just because it can.

#### **4. Add providers conservatively**

Start with one working provider before adding more.

Suggested baseline decisions:

- analytics: disabled
- embedded subtitles: disabled if you prefer external subtitle management
- post-processing: enabled
- subtitle synchronization: enabled only if you have a real mismatch problem and accept the extra processing cost

#### **5. Keep path mappings simple**

If Bazarr, Radarr, and Sonarr all mount the same media root as `/data`, you usually do not need custom path mappings. Add them only when one service truly sees a different path.

---

### **Validation / troubleshooting**

#### **Basic validation**

- Bazarr connects successfully to both Radarr and Sonarr
- one movie and one series are visible in the Bazarr library views
- a test subtitle search succeeds and writes the subtitle next to the media file

#### **Troubleshooting checklist**

- If libraries appear empty:
  - verify the API connection to Radarr and Sonarr
  - compare the media paths exposed inside each container
- If providers fail repeatedly:
  - start with one provider and validate credentials before adding more
- If subtitles are downloaded but do not match expectations:
  - review the language profile and post-processing rules first

---

### **Known issues**

- Bazarr inherits most path problems from the rest of the media stack. If Radarr and Sonarr are inconsistent, Bazarr will also be inconsistent.
- Subtitle sync and post-processing can add noticeable work on large libraries. Enable only the features that solve a real problem.
