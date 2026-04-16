---
title: Observability
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-chart-line:


## **Observability**

---

### **Purpose**

This section defines the platform observability baseline for metrics, logs, dashboards, and actionable alerting.

---

### **Primary path**

- Prometheus for metrics collection and alert rule evaluation
- Alertmanager for grouping, routing, and deduplication
- Loki and Alloy for centralized log collection and search
- Grafana for dashboards and investigation workflows

---

### **What belongs here**

- Monitoring and alerting coverage expectations
- Metrics and log pipeline boundaries
- Signal quality guidance for labels, dashboards, and alert noise
- Access-control expectations for observability systems

---

### **Canonical pages**

- [Prometheus](prometheus.md)
- [Alertmanager](alertmanager.md)
- [Loki](loki.md)
- [Grafana](grafana.md)

---

### **Secondary references**

- [Docker](../infrastructure-tools/docker.md)
- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)

---

### **Drafts / planned**

- No additional observability drafts are promoted in the active navigation during this pass.

---

### **Guardrails**

- Monitor at least host health, container health, ingress health, and backup-related services.
- Keep alert rules actionable and noise-resistant.
- Keep label cardinality controlled in Prometheus and Loki.
- Keep dashboards and alert rules version-controlled with stack changes.
