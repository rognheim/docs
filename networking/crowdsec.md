---
title: Crowdsec
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-security-network:

## **CrowdSec**

---

### **Introduction**

CrowdSec is the log-driven detection and remediation layer for exposed services. In this platform it primarily watches Traefik access logs and host logs, identifies abusive behavior, and feeds decisions to the reverse-proxy edge so repeat attackers are blocked quickly.

---

### **Scope**

This page owns:

- the CrowdSec service itself
- log acquisition for Traefik and host sources
- hub collections, scenarios, and local overrides
- validation of parser, scenario, and decision flow

This page does not own:

- reverse proxy routing. See [Traefik](traefik.md)
- user authentication and SSO. See [Authelia](authelia.md)
- firewall baseline hardening. See [Linux Hardening](../security/linux-hardening.md)

---

### **Prerequisites**

- Docker and Docker Compose available on the edge host.
- Traefik access logging enabled and written to a file that CrowdSec can read.
- A decision on where remediation happens:
  - Traefik bouncer
  - firewall bouncer
  - application-specific integration

---

### **Dependencies / related pages**

- [Traefik](traefik.md)
- [Expose and Secure a Public-Facing Containerized Application](../guides/traefik-authelia-crowdsec.md)
- [Linux Hardening](../security/linux-hardening.md)

---

### **Configuration files**

- `docker-compose.yml`
- `config/acquis.yaml`
- `config/profiles.yaml` if local profiles are needed
- `config/notifications/` if alerting is added later
- local scenario overrides under `/etc/crowdsec/scenarios/` inside the container

---

### **Implementation**

#### **1. Deploy CrowdSec with persistent state**

Minimal Compose example:

```yaml
services:
  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    restart: unless-stopped
    environment:
      COLLECTIONS: "crowdsecurity/traefik crowdsecurity/linux"
      GID: "1000"
    volumes:
      - ./config:/etc/crowdsec
      - ./data:/var/lib/crowdsec/data
      - /var/log:/var/log:ro
      - /opt/docker-appdata/logs/traefik:/logs/traefik:ro
```

The important part is persistence for `/var/lib/crowdsec/data` and read-only mounts for every log source you expect CrowdSec to parse.

#### **2. Define acquisitions explicitly**

Example `config/acquis.yaml`:

```yaml
filenames:
  - /logs/traefik/access.log
labels:
  type: traefik
---
filenames:
  - /var/log/auth.log
labels:
  type: syslog
```

Keep this file small and intentional. If CrowdSec is not reading a log source, it cannot detect anything from it.

#### **3. Install the collections you actually use**

Inside the container:

```bash
docker exec crowdsec cscli collections install crowdsecurity/traefik
docker exec crowdsec cscli collections install crowdsecurity/linux
docker exec crowdsec cscli hub update
docker exec crowdsec cscli hub upgrade
```

Start with the base collections that match the platform, then add more only when there is a real data source and a clear remediation plan.

#### **4. Connect remediation to the edge**

If Traefik is the public edge, the simplest pattern is a Traefik bouncer. The exact deployment may vary, but the principle is always the same:

- CrowdSec reads logs and creates decisions.
- The bouncer asks CrowdSec whether a source IP should be allowed.
- Traefik denies requests for banned sources before the request reaches the application.

Document the bouncer token in the same secret source used for the rest of the edge stack.

#### **5. Tune noisy scenarios locally instead of disabling protection wholesale**

Some modern apps trigger crawler-style detections during normal browsing. Prefer a local override over disabling the whole collection.

Inspect a scenario:

```bash
docker exec crowdsec cscli scenarios inspect crowdsecurity/http-crawl-non_statics
```

Example local override:

```yaml
type: leaky
name: crowdsecurity/http-crawl-non_statics
description: "Tuned threshold for interactive apps behind Traefik"
filter: "evt.Meta.log_type == 'http_access-log' && evt.Meta.service == 'traefik'"
capacity: 300
leakspeed: 30s
groupby: evt.Meta.source_ip
labels:
  service: http
  type: crawl
  remediation: true
```

After adding an override, restart CrowdSec and watch new decisions closely before calling the rule “done.”

#### **6. Keep an operator command set nearby**

Useful day-two commands:

```bash
docker exec crowdsec cscli metrics
docker exec crowdsec cscli alerts list
docker exec crowdsec cscli decisions list
docker exec crowdsec cscli parsers list
docker exec crowdsec cscli scenarios list
```

These are more useful than pasting long static command output into the page because they reflect the live state of the instance you are operating.

---

### **Validation / troubleshooting**

#### **Basic validation**

Confirm the service is healthy:

```bash
docker compose ps
docker logs crowdsec --tail 100
```

Confirm acquisitions are active:

```bash
docker exec crowdsec cscli metrics
```

You want to see lines proving that Traefik and host logs are being read and parsed.

Confirm decisions are flowing:

```bash
docker exec crowdsec cscli alerts list
docker exec crowdsec cscli decisions list
```

#### **Troubleshooting checklist**

- If there are no alerts:
  - verify the log file path in `acquis.yaml`
  - verify the mounted log path inside the container actually contains data
  - verify Traefik access logging is enabled
- If alerts exist but bans do not happen:
  - verify the bouncer token and bouncer connectivity
  - verify the bouncer is attached to the same Traffic path as the public app
- If trusted clients are banned during normal use:
  - inspect the triggering scenario
  - prefer a threshold override or whitelist instead of turning off the entire collection

---

### **Known issues**

- CrowdSec quality depends entirely on log quality. If Traefik access logs are incomplete or missing, HTTP detections will also be incomplete.
- Aggressive default thresholds can be too noisy for modern self-hosted apps with APIs, live search, or image-heavy browsing. Tune locally and re-test instead of accepting false positives.
