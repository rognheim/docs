---
title: Docker
last_reviewed: 2026-03-09
owner: Rognheim
---

# :fontawesome-brands-docker:


## **Docker**

---

### **Introduction**

Docker + Docker Compose is the primary application runtime for this platform.

This page defines:

- Debian 13 Docker host installation
- Canonical stack directory and `docker-compose.yml` contracts
- Reverse proxy integration with Traefik/Authelia/CrowdSec
- Backup, restore, and migration workflows

---

### **Prerequisites**

- Debian 13 VM host provisioned from Cloud-Init template
- Host hardening baseline applied
- Storage paths prepared for app data
- Traefik public network available on edge/application hosts where needed

---

### **Configuration files and paths**

- Docker daemon config: `/etc/docker/daemon.json` (optional tuning)
- Compose stacks:
  - `/opt/docker-appdata/compose-files/<stack>/docker-compose.yml`
  - `/opt/docker-appdata/compose-files/<stack>/.env`
  - `/opt/docker-appdata/compose-files/<stack>/secrets/*`
  - `/opt/docker-appdata/compose-files/<stack>/config/*`
  - `/opt/docker-appdata/compose-files/<stack>/data/*`

---

### **Implementation workflow**

#### 1. Install Docker Engine and Compose plugin (Debian 13)

```shell
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Validate:

```shell
docker --version
docker compose version
sudo docker run --rm hello-world
```

**(Optional) Manage Docker as a non-root user**

```shell
sudo groupadd docker
```

```shell
sudo usermod -aG docker $USER
```

Reboot the VM.

Validate:

```shell
docker run hello-world
```

#### 2. Prepare canonical stack layout

```shell
sudo mkdir -p /opt/docker-appdata
sudo chown -R "$USER:$USER" /opt/docker-appdata
```

Navigate to `/opt/docker-appdata`

```shell
cd /opt/docker-appdata
```

Either start from scratch or clone a Git repository with your saved compose-files into the newly created folder.

```shell
git clone https://git.internal.rognheim.no/dev/platform-docker.git
```

Per stack example:

```shell
mkdir -p /opt/docker-appdata/git-repo/compose-files/traefik/{config,secrets,data}
chmod 700 /opt/docker-appdata/git-repo/compose-files/traefik/secrets
```

Secret rules are defined in [Secrets Management](../security/secrets-management.md).

#### 3. Create shared networks

Create reverse proxy external network once per host:

```shell
docker network create traefik_public
```

Internal per-stack networks are declared in each compose file.

#### 4. Apply canonical Compose contract

Example `docker-compose.yml` for public app:

```yaml
services:
  app:
    image: ghcr.io/example/app:stable
    container_name: app
    restart: unless-stopped
    user: "1000:1000"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    env_file:
      - .env
    environment:
      - TZ=Europe/Oslo
      - APP_SECRET_FILE=/run/secrets/app_secret
    secrets:
      - app_secret
    volumes:
      - ./config:/config
      - ./data:/data
    networks:
      - app_internal
      - traefik_public
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik_public
      - traefik.http.routers.app.entrypoints=websecure
      - traefik.http.routers.app.rule=Host(`app.example.com`)
      - traefik.http.routers.app.tls=true
      - traefik.http.routers.app.middlewares=authelia@docker,crowdsec@file,secure-headers@file
      - traefik.http.services.app.loadbalancer.server.port=8080
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3

secrets:
  app_secret:
    file: ./secrets/app_secret

networks:
  app_internal:
    name: app_internal
  traefik_public:
    external: true
```

#### 5. Operate stacks safely

```shell
cd /opt/docker-appdata/compose-files/<stack>
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

For controlled restart after config change:

```shell
docker compose up -d --no-deps <service>
```

#### 6. Backup and restore pattern

Bind mount backup:

```shell
cd /opt/docker-appdata/compose-files/<stack>
tar -czf /tmp/<stack>-$(date +%F).tar.gz config data
```

Restore:

```shell
tar -xzf /tmp/<stack>-YYYY-MM-DD.tar.gz -C /opt/docker-appdata/compose-files/<stack>
docker compose up -d
```

For named volumes:

```shell
docker run --rm -v <volume_name>:/source -v "$PWD":/backup alpine tar czf /backup/<volume_name>.tar.gz -C /source .
```

#### 7. Migrate stack to another host

1. Export stack directory tarball.
2. Transfer with `scp` to target host.
3. Recreate secret files with correct permissions.
4. Recreate required external networks (`traefik_public`).
5. Run `docker compose up -d` on target host.

---

### **Validation checklist**

```shell
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Networks}}'
docker compose -f /opt/docker-appdata/compose-files/<stack>/docker-compose.yml config --quiet
ss -tulpn | rg ":80|:443"
```

Expected:

- Containers healthy and attached to intended networks.
- Compose file validates without interpolation errors.
- Only edge services expose public ports.

---

### **Troubleshooting**

1. Container cannot resolve upstream services

- Verify service and network names in Compose.
- Check per-stack internal network attachment.

2. App not reachable through Traefik

- Verify labels and `traefik_public` network.
- Check Traefik logs and router/service resolution.
- Validate DNS and TLS settings in [Traefik](../networking/traefik.md).

3. Authentication not enforced

- Verify Authelia middleware on router labels.
- Confirm policy rules in [Authelia](../networking/authelia.md).

4. CrowdSec blocks normal traffic

- Review decisions and scenario tuning in [CrowdSec](../networking/crowdsec.md).

---

### **Operational guardrails**

- Keep one stack per directory with explicit ownership.
- Do not place plaintext secrets in `.env` or Compose files.
- Use CI to validate compose syntax and policy before deploy.
- Keep host-level hardening in [Linux Hardening](../security/linux-hardening.md), not per-stack ad-hoc changes.
