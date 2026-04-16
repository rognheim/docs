---
title: Loki
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-grafana:


## **Loki**

---

### **Introduction**

This page defines the Loki log platform baseline with Grafana Alloy as the canonical log collector.

Design goals:

- Centralize service and host logs
- Keep labels intentional and low-cardinality
- Make logs explorable in Grafana next to metrics and alerts

---

### **Prerequisites**

- Observability stack host with Docker Compose
- Network access from Alloy to log sources and Loki endpoint
- Grafana datasource integration from [Grafana](grafana.md)

---

### **Configuration files and paths**

- `/opt/docker-appdata/compose-files/observability/docker-compose.yml`
- `/opt/docker-appdata/compose-files/observability/config/loki/loki-config.yml`
- `/opt/docker-appdata/compose-files/observability/config/alloy/config.alloy`
- `/opt/docker-appdata/compose-files/observability/data/loki/`

---

### **Implementation workflow**

#### 1. Deploy Loki service

Example service block:

```yaml
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    command: ["-config.file=/etc/loki/loki-config.yml"]
    volumes:
      - ./config/loki/loki-config.yml:/etc/loki/loki-config.yml:ro
      - ./data/loki:/loki
    networks:
      - observability
```

Example `loki-config.yml`:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules

schema_config:
  configs:
    - from: 2026-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: loki_index_
        period: 24h

limits_config:
  allow_structured_metadata: true
  volume_enabled: true

compactor:
  working_directory: /loki/compactor

ruler:
  alertmanager_url: http://alertmanager:9093
```

#### 2. Deploy Grafana Alloy collector

Example service block:

```yaml
services:
  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    restart: unless-stopped
    command: ["run", "/etc/alloy/config.alloy"]
    volumes:
      - ./config/alloy/config.alloy:/etc/alloy/config.alloy:ro
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - observability
```

Example `config.alloy` (minimal pipeline):

```hcl
local.file_match "varlog" {
  path_targets = [{
    __path__ = "/var/log/*.log",
    job      = "host-varlog",
    host     = "srv-monitor-01",
  }]
}

loki.source.file "varlog" {
  targets    = local.file_match.varlog.targets
  forward_to = [loki.write.default.receiver]
}

loki.source.docker "containers" {
  host       = "unix:///var/run/docker.sock"
  targets    = []
  forward_to = [loki.write.default.receiver]
  labels = {
    job = "docker",
  }
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

#### 3. Standardize label strategy

Recommended required labels:

- `job`
- `host`
- `service` (where known)
- `env` (for stage/prod split)

Avoid high-cardinality labels (request IDs, user IDs, random tokens).

#### 4. Keep log access behind auth path

If Loki or Grafana log views are exposed on web routes, protect access through [Traefik](../networking/traefik.md) and [Authelia](../networking/authelia.md).

---

### **Validation checklist**

```shell
curl -fsS http://localhost:3100/ready
docker logs alloy --tail 100
docker logs loki --tail 100
```

In Grafana Explore:

- Query `{job="docker"}` and `{job="host-varlog"}`
- Confirm recent logs are present

---

### **Troubleshooting**

1. No logs arriving in Loki

- Verify Alloy has read permissions for mounted log paths.
- Verify `loki.write` endpoint URL and connectivity.
- Inspect Alloy logs for pipeline errors.

2. Loki query timeouts or slow responses

- Review retention and storage performance.
- Reduce unbounded query ranges.
- Review label strategy for index pressure.

3. Logs missing expected metadata labels

- Verify Alloy pipeline label assignment.
- Ensure labels are added before write stage.

---

### **Operational guardrails**

- Keep log retention and storage growth monitored.
- Keep collector config version-controlled with peer review.
- Avoid exposing sensitive logs to broad user groups.
- Keep label schema stable to preserve dashboard/query consistency.
