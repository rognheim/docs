---
title: Add Monitoring to a Containerized Application
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-chart-line:


## **Add Monitoring to a Containerized Application**

---

### **Goal**

Use this guide when you want a new or existing containerized application to meet the platform minimum for health checks, metrics visibility, log visibility, and alertable failure conditions.

---

### **When to use this guide**

- A new service is about to be exposed to other users.
- An existing service has poor troubleshooting visibility.
- You want to promote a service from "it runs" to "it can be operated safely."

This guide complements, rather than replaces:

- [Docker](../infrastructure-tools/docker.md)
- [Prometheus](../observability/prometheus.md)
- [Grafana](../observability/grafana.md)
- [Loki](../observability/loki.md)

---

### **Prerequisites**

- The application already runs through the platform Docker workflow.
- You know whether the app exposes a `/metrics` endpoint or only a health endpoint.
- Prometheus, Grafana, and Loki are already available somewhere in the platform.
- The service owner, purpose, backup path, and exposure model are documented.

---

### **Inputs**

Before you start, decide these values:

| Item | Example |
| --- | --- |
| Service name | `seerr` |
| Internal service address | `http://seerr:5055` |
| Health endpoint | `/api/v1/status` or `/health` |
| Metrics endpoint | `/metrics` if the app supports it |
| Owner | `media` |
| Severity for outage alert | `warning` or `critical` |

---

### **Step-by-step workflow**

#### 1. Add a healthcheck to the Compose service

At minimum, every user-facing or important internal app should expose one health signal.

```yaml
services:
  app:
    image: ghcr.io/example/app:stable
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
```

If the app does not expose a dedicated health endpoint, use the lightest safe HTTP check available.

#### 2. Make logs easy to collect

Prefer writing operational logs to standard output or standard error so the host collector can ingest them.

Good pattern:

- JSON or clearly structured text logs
- request or correlation identifiers where supported
- no secrets, tokens, or full credentials in logs

Avoid:

- logging only to an internal file path that the host never collects
- dumping full stack traces for routine user errors

#### 3. Add a Prometheus scrape target if the app exposes metrics

If the application already exports Prometheus metrics, add it to the scrape config.

```yaml
- job_name: "app"
  metrics_path: /metrics
  static_configs:
    - targets:
        - app:8080
```

If the app does not expose metrics:

- keep the healthcheck
- use container status and reverse-proxy health as the minimum signal
- only add exporters when they materially improve operations

#### 4. Decide the minimum dashboard view

Every important app should be visible in Grafana through a very small dashboard or shared overview panel:

- up/down state
- request or worker error rate if available
- latency or response time if available
- recent logs filtered to the app name

Do not wait for a perfect dashboard before adding the first useful one.

#### 5. Create at least one actionable alert

Start with one or two alerts you know how to act on.

Good first alerts:

- service healthcheck failing for several minutes
- Prometheus scrape target down
- application error rate far above normal

Avoid noisy alerts such as:

- every single container restart without context
- every warning log line
- alerts with no owner or no runbook

#### 6. Record the operational checklist before exposing the service widely

Use this minimum review:

- owner and purpose documented
- backup and restore path validated
- logs visible in Loki
- health visible in Prometheus or through container state
- one dashboard or shared panel available in Grafana
- one meaningful alert configured

That checklist is intentionally derived from the archived onboarding and readiness checklists in this repository.

---

### **Validation**

Run these checks after wiring the app into monitoring:

```shell
docker compose ps
docker inspect --format '{{json .State.Health }}' <container-name>
curl -fsS http://<service-host-or-container>:<port>/<health-endpoint>
```

Then validate in the observability stack:

- Prometheus target shows `UP`
- Grafana can display the new signal
- Loki shows recent app logs
- Alertmanager receives a test alert or a safe synthetic failure

If the app is public-facing, also confirm Traefik and CrowdSec can see requests to it.

---

### **Rollback / failure modes**

- If a new healthcheck causes restarts, loosen the timing or point it at a cheaper endpoint.
- If scraping introduces load, reduce scrape frequency or remove the target until the exporter is fixed.
- If logs become too noisy, reduce verbosity at the application rather than filtering everything downstream.
- If an alert is not actionable, disable it until the response path is clear.
