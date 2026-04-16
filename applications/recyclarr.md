---
title: Recyclarr
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-television:

## **Recyclarr**

---

### **Introduction**

Recyclarr keeps Radarr and Sonarr aligned with the chosen TRaSH-based quality and custom-format policy. Its value is consistency: instead of hand-editing profile logic in multiple apps, define the policy once, sync it, and keep manual changes limited to the few settings Recyclarr does not manage.

---

### **Scope**

This page owns:

- Recyclarr deployment
- the parts of Radarr and Sonarr configuration that should be automated
- the operator workflow for syncing and validating configuration

This page does not own:

- root folders and downloader wiring in the `*arr` apps
- indexer management. See [Prowlarr](prowlarr.md)

---

### **Prerequisites**

- [Docker](../infrastructure-tools/docker.md)
- A working Radarr and Sonarr instance with API keys available
- A decision on whether language behavior should prefer English, original language, or another profile strategy

---

### **Dependencies / related pages**

- [Radarr](radarr.md)
- [Sonarr](sonarr.md)
- [Prowlarr](prowlarr.md)

---

### **Configuration files**

- `docker-compose.yml`
- `config/recyclarr.yml`

---

### **Implementation**

#### **1. Deploy Recyclarr as a small utility container**

Example Compose service:

```yaml
services:
  recyclarr:
    image: ghcr.io/recyclarr/recyclarr:latest
    container_name: recyclarr
    restart: unless-stopped
    environment:
      - TZ=Europe/Oslo
    volumes:
      - ./config:/config
```

Start it:

```bash
docker compose up -d
```

#### **2. Keep a single source of truth in `recyclarr.yml`**

Minimal example:

```yaml
radarr:
  movies:
    base_url: http://radarr:7878
    api_key: RADARR_API_KEY

sonarr:
  series:
    base_url: http://sonarr:8989
    api_key: SONARR_API_KEY
```

The exact profile includes can evolve, but the operating principle stays the same: keep policy in versioned config and sync it into the applications instead of rebuilding it by hand.

#### **3. Be explicit about what Recyclarr does and does not own**

Recyclarr should own:

- quality profiles
- custom formats
- release-policy style synchronization

Recyclarr should not be expected to own:

- root folders
- download clients
- remote path mappings
- connect notifications
- indexers

Keeping this boundary clear prevents confusion when a setting “didn’t sync.”

#### **4. Run sync commands as an operator workflow**

Useful commands:

```bash
docker compose exec recyclarr recyclarr sync --debug
docker compose exec recyclarr recyclarr sync radarr --debug
docker compose exec recyclarr recyclarr sync sonarr --debug
```

Use `--debug` when first rolling out a config or when reconciling drift after manual edits.

#### **5. Decide language policy deliberately**

One of the most important manual decisions is language preference. If the synced profile assumes English but the library includes a lot of foreign-language content, review that before calling the configuration complete.

---

### **Validation / troubleshooting**

#### **Basic validation**

- a sync completes without authentication or schema errors
- Radarr and Sonarr show the expected profiles and custom formats
- manual root-folder and downloader settings remain untouched

#### **Troubleshooting checklist**

- If sync fails:
  - verify base URLs on the Docker network
  - verify API keys
  - inspect the debug output for schema or include errors
- If a setting is missing after sync:
  - check whether Recyclarr is actually supposed to manage that class of setting
- If profile behavior looks wrong for foreign-language content:
  - review the language assumptions in the synced profiles before blaming the download client or indexers

---

### **Known issues**

- Recyclarr improves consistency, but it cannot fix a broken filesystem layout or bad downloader configuration.
- Manual edits in Radarr and Sonarr can create drift. When in doubt, treat the config file as the source of truth and sync again.
