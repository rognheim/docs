---
title: Encryption
status: draft
last_reviewed: 2026-03-08
owner: Rognheim
---

# :simple-letsencrypt:


## **Encryption**

---

### **Introduction**

This page covers encryption controls for your stack at two layers:

- Data in transit (TLS at the public edge and between services where required)
- Data at rest (ZFS datasets, VM storage, and backup targets)

In this environment, internet-facing encryption is terminated at Traefik, while authentication and session boundaries are enforced by Authelia.

---

### **Prerequisites and Scope**

Applies to:

- Public applications routed through Traefik in DMZ (`10.0.30.0/24`)
- Internal service traffic with sensitive credentials
- ZFS and backup datasets on storage networks

Assumptions:

- DNS for public services is correctly delegated
- Traefik ACME flow is configured
- Secrets are handled through the Secrets Management baseline

---

### **Configuration files**

- Traefik static config (`traefik.yml`)
- Traefik dynamic config (`config.yml` or file provider fragments)
- Traefik `acme.json` certificate store
- Authelia `configuration.yml`
- ZFS dataset properties and host backup config

---

### **Implementation**

#### 1. Enforce TLS at the edge (Traefik)

Baseline requirements:

- Redirect HTTP (`:80`) to HTTPS (`:443`)
- Disable weak TLS versions/ciphers
- Use ACME certificates for all public hostnames
- Route authentication-sensitive apps through Authelia policy

Traefik checks:

```shell
docker logs traefik --tail 200 | grep -Ei "acme|certificate|renew"
```

Validate endpoint certificate:

```shell
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

#### 2. Protect auth boundary between Traefik and Authelia

Baseline:

- Keep Authelia behind Traefik
- Enforce `default_policy: deny` and explicit allow rules
- Trust forwarded headers only from known reverse proxy path

Quick checks:

```shell
docker logs authelia --tail 100
docker logs traefik --tail 100
```

#### 3. Encrypt sensitive data at rest

Use one or more of:

- ZFS native encryption for sensitive datasets
- Encrypted backup targets for off-host/off-site data
- Application-level encryption where available (database/file-layer)

For ZFS-sensitive datasets, verify:

```shell
zfs get encryption,keyformat,keylocation <pool/dataset>
```

#### 4. Key and certificate lifecycle

Operational policy:

- Protect private keys with strict filesystem permissions
- Rotate API tokens and encryption keys on schedule or incident
- Monitor certificate expiration before outage windows
- Keep documented owners for each cert/key domain

Check file permissions:

```shell
ls -l /opt/docker-appdata/compose-files/public/traefik/config/acme.json
```

---

### **Validation Checklist**

- Public app endpoints only serve HTTPS certificates expected for each hostname.
- HTTP requests are redirected to HTTPS.
- Authelia-protected services enforce expected one-factor/two-factor policy.
- Sensitive storage locations are encrypted or explicitly documented as not encrypted.
- Certificate renewal logs show successful ACME renewal behavior.

---

### **Post-install tasks**

- Add periodic certificate expiry checks to monitoring/alerts.
- Review TLS and auth policies after adding new DMZ services.
- Validate backup restore workflow for encrypted datasets.
- Document exceptions where encryption is intentionally not enabled.

---

### **Known issues**

- TLS handshake issues commonly come from DNS mismatch or stale ACME state.
- Double TLS termination can hide client IP/header expectations if proxy chain is misconfigured.
- Encryption without tested key recovery can create availability risk during incidents.
