---
title: Prometheus
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-prometheus:


## **Prometheus**

---

### **Introduction**

This page defines the Prometheus baseline for metrics collection and alert rule evaluation in the platform.

Prometheus responsibilities:

- Scrape platform and service metrics
- Evaluate alert rules
- Send alerts to Alertmanager

---

### **Prerequisites**

- Debian 13 VM host with Docker Compose runtime
- Network access from Prometheus to scrape targets
- Alert routing destination in [Alertmanager](alertmanager.md)
- Dashboard consumer in [Grafana](grafana.md)

---

### **Configuration files and paths**

Recommended observability stack layout:

- `/opt/docker-appdata/compose-files/observability/docker-compose.yml`
- `/opt/docker-appdata/compose-files/observability/config/prometheus/prometheus.yml`
- `/opt/docker-appdata/compose-files/observability/config/prometheus/rules/*.yml`

---

### **Implementation workflow**

#### 1. Deploy Prometheus in observability stack

Example service block:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --web.enable-lifecycle
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./config/prometheus/rules:/etc/prometheus/rules:ro
      - ./data/prometheus:/prometheus
    networks:
      - observability
```

#### 2. Define scrape contract

Example `prometheus.yml`:

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: node-exporter
    static_configs:
      - targets:
          - 10.0.20.21:9100
          - 10.0.20.31:9100

  - job_name: cadvisor
    static_configs:
      - targets:
          - 10.0.20.21:8080

  - job_name: traefik
    static_configs:
      - targets: ["10.0.30.10:8082"]
```

Keep target count and labels controlled to avoid unnecessary cardinality growth.

#### 3. Add alert rule baseline

Example `rules/platform.yml`:

```yaml
groups:
  - name: platform-health
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Target is down"
          description: "{{ $labels.job }} on {{ $labels.instance }} is unreachable"

      - alert: HostHighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High CPU usage"
          description: "{{ $labels.instance }} CPU > 90% for 10m"
```

#### 4. Expose Prometheus through controlled access path

- Keep Prometheus private by default.
- If web UI is published, route via [Traefik](../networking/traefik.md) and enforce [Authelia](../networking/authelia.md).

---

### **Validation checklist**

```shell
docker compose -f /opt/docker-appdata/compose-files/observability/docker-compose.yml config --quiet
curl -fsS http://localhost:9090/-/healthy
curl -fsS http://localhost:9090/-/ready
curl -fsS http://localhost:9090/api/v1/rules | jq '.status'
```

Expected:

- Prometheus is healthy and ready.
- Scrape targets appear in `up` metric.
- Rule groups load without parse errors.

---

### **Troubleshooting**

1. Targets show `DOWN`

- Verify network reachability and exporter ports.
- Verify firewall allows source from observability host.
- Verify target endpoint path/metrics format.

2. Rule file fails to load

- Validate YAML syntax and expression correctness.
- Check Prometheus logs for parser errors.

3. Alerts never appear in Alertmanager

- Confirm `alerting.alertmanagers` target is correct.
- Confirm rule expressions are firing in expression browser.

---

### **Operational guardrails**

- Keep scrape jobs explicit and ownership-documented.
- Keep label cardinality constrained.
- Keep alert rules actionable and tied to runbook context.
- Keep retention sizing aligned with disk capacity and restore expectations.
