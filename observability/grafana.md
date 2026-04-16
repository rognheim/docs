---
title: Grafana
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-grafana:


## **Grafana**

---

### **Introduction**

This page defines Grafana baseline operation for dashboards, exploration, and alert visualization in the observability stack.

Grafana integrates:

- Prometheus for metrics
- Loki for logs
- Alert visibility from Prometheus + Alertmanager flow

---

### **Prerequisites**

- Prometheus baseline from [Prometheus](prometheus.md)
- Loki + Alloy baseline from [Loki](loki.md)
- Controlled web access via [Traefik](../networking/traefik.md) + [Authelia](../networking/authelia.md)

---

### **Configuration files and paths**

- `/opt/docker-appdata/compose-files/observability/docker-compose.yml`
- `/opt/docker-appdata/compose-files/observability/config/grafana/provisioning/datasources/datasources.yml`
- `/opt/docker-appdata/compose-files/observability/config/grafana/provisioning/dashboards/dashboards.yml`
- `/opt/docker-appdata/compose-files/observability/config/grafana/dashboards/*.json`
- `/opt/docker-appdata/compose-files/observability/data/grafana/`

---

### **Implementation workflow**

#### 1. Deploy Grafana service

Example service block:

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    user: "472:472"
    environment:
      - TZ=Europe/Oslo
      - GF_SERVER_ROOT_URL=https://grafana.example.com
      - GF_SECURITY_ALLOW_EMBEDDING=false
    volumes:
      - ./data/grafana:/var/lib/grafana
      - ./config/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./config/grafana/dashboards:/var/lib/grafana/dashboards:ro
    networks:
      - observability
      - traefik_public
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik_public
      - traefik.http.routers.grafana.entrypoints=websecure
      - traefik.http.routers.grafana.rule=Host(`grafana.example.com`)
      - traefik.http.routers.grafana.tls=true
      - traefik.http.routers.grafana.middlewares=authelia@docker,crowdsec@file,secure-headers@file
      - traefik.http.services.grafana.loadbalancer.server.port=3000
```

#### 2. Provision datasources

Example `datasources.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

#### 3. Provision baseline dashboards

Example `dashboards.yml`:

```yaml
apiVersion: 1

providers:
  - name: platform
    orgId: 1
    folder: Platform
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

Start with dashboards for:

- Host CPU/memory/disk
- Container health and restarts
- Traefik request/error profile
- Prometheus target health
- Loki ingestion volume

#### 4. Access and identity policy

- Keep Grafana behind Traefik route and Authelia policy.
- Keep local admin use minimal and break-glass only.
- Map regular access via SSO/forward-auth model where feasible.

---

### **Validation checklist**

```shell
curl -fsS http://localhost:3000/api/health
```

UI checks:

- Prometheus datasource test succeeds
- Loki datasource test succeeds
- Provisioned dashboards appear after startup

---

### **Troubleshooting**

1. Datasource connection fails

- Verify service names and network attachments in compose.
- Verify target containers are healthy and reachable.

2. Dashboards not auto-loaded

- Verify dashboard provider path and file mount.
- Verify JSON files are readable by Grafana user.

3. Login or auth loop on public route

- Verify Traefik middleware chain order.
- Verify Authelia access policy and cookie/session domain settings.

---

### **Operational guardrails**

- Keep dashboard provisioning and datasource config in version control.
- Keep high-value dashboards minimal and actionable.
- Keep Grafana route protected by edge auth controls.
- Review dashboard and alert noise quarterly and remove stale panels.
