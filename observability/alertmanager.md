---
title: Alertmanager
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-prometheus:


## **Alertmanager**

---

### **Introduction**

This page defines Alertmanager baseline routing, grouping, and deduplication for platform alerts.

Primary receiver model:

- Discord/webhook as primary notification channel

---

### **Prerequisites**

- Prometheus alerts configured in [Prometheus](prometheus.md)
- Reachable webhook endpoint for Discord delivery
- Optional webhook relay service if direct Discord payload transform is required

---

### **Configuration files and paths**

- `/opt/docker-appdata/compose-files/observability/docker-compose.yml`
- `/opt/docker-appdata/compose-files/observability/config/alertmanager/alertmanager.yml`
- `/opt/docker-appdata/compose-files/observability/secrets/discord_webhook_url`

---

### **Implementation workflow**

#### 1. Deploy Alertmanager service

Example service block:

```yaml
services:
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager
    volumes:
      - ./config/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - ./data/alertmanager:/alertmanager
    networks:
      - observability
```

#### 2. Define routing and grouping baseline

Example `alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: discord-default
  group_by: ["alertname", "job", "severity"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - matchers:
        - severity="critical"
      receiver: discord-critical
      repeat_interval: 30m

receivers:
  - name: discord-default
    webhook_configs:
      - url_file: /run/secrets/discord_webhook_url
        send_resolved: true

  - name: discord-critical
    webhook_configs:
      - url_file: /run/secrets/discord_webhook_url
        send_resolved: true

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal: ["alertname", "instance"]
```

If direct Discord webhook format is not accepted, send Alertmanager output to a small internal relay endpoint that transforms payloads for Discord.

#### 3. Attach secret file in compose

```yaml
secrets:
  discord_webhook_url:
    file: ./secrets/discord_webhook_url

services:
  alertmanager:
    secrets:
      - discord_webhook_url
```

Keep secret permissions aligned with [Secrets Management](../security/secrets-management.md).

#### 4. Connect Prometheus to Alertmanager

In `prometheus.yml`:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]
```

---

### **Validation checklist**

```shell
curl -fsS http://localhost:9093/-/healthy
curl -fsS http://localhost:9093/api/v2/status | jq '.configYAML != null'
```

Trigger test alert from Prometheus rule and verify:

- Grouping behavior works as expected
- Alert reaches Discord/webhook receiver
- Resolved notification behavior is acceptable

---

### **Troubleshooting**

1. Alerts visible in Prometheus but not in Alertmanager

- Verify Alertmanager target in Prometheus config.
- Verify Prometheus can resolve and connect to `alertmanager:9093`.

2. Alertmanager receives alerts but Discord gets nothing

- Verify webhook secret file content and permissions.
- Verify outbound connectivity from alertmanager container.
- Verify whether payload transformation relay is required.

3. Notification storm/noise

- Adjust `group_by`, `group_interval`, and `repeat_interval`.
- Add inhibition for lower-severity duplicates.
- Remove or tune noisy rules in Prometheus.

---

### **Operational guardrails**

- Keep routing tree simple and severity-driven.
- Keep receiver secrets out of git and `.env` plaintext.
- Keep every critical alert tied to a remediation path/runbook.
- Review noisy alerts and stale routes after each major platform change.
