---
title: Secrets Management
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-key-chain-variant:


## **Secrets Management**

---

### **Introduction**

This page defines how secrets are stored, consumed, rotated, and recovered for your Docker Compose-first homelab platform.

Default pattern for this environment:

- Use Docker Compose file-based secrets
- Keep secret material outside git-tracked files
- Keep strict host permissions on secret paths
- Rotate on schedule and after incidents

---

### **Scope**

Applies to:

- Traefik, Authelia, CrowdSec, and other compose-managed services
- Debian 13 VM service hosts
- credentials used by internal and DMZ services

Out of scope:

- external enterprise vault integration
- end-user password manager guidance

---

### **Prerequisites**

- Compose-managed services with writable stack directories
- A secret storage path outside tracked repository content
- Service images that support Docker secrets or `_FILE`-style environment variables whenever possible

---

### **Dependencies / related pages**

- [Docker](../infrastructure-tools/docker.md)
- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)
- [CrowdSec](../networking/crowdsec.md)
- [Linux Hardening](linux-hardening.md)

---

### **Secret Classes and Storage Locations**

| Secret class | Examples | Storage location | Notes |
| --- | --- | --- | --- |
| Runtime service secrets | API keys, DB passwords, JWT secrets | Per-stack `./secrets/*` files referenced by Compose | Never commit to git |
| Bootstrap/admin credentials | Initial service admin passwords | Offline password manager + secure notes | Rotate after initial setup |
| Infrastructure credentials | DNS tokens, SMTP creds, backup keys | Dedicated secret files with root-restricted permissions | Minimize scope and privileges |
| Identity secrets | Authelia session/JWT/encryption keys | Authelia secret files mounted as Docker secrets | Regenerate on compromise |

Recommended layout per stack:

```text
/opt/docker-appdata/compose-files/<stack>/
  docker-compose.yml
  .env                # non-secret values only
  secrets/
    <secret_name>     # one file per secret
```

---

### **Configuration files**

- `<stack>/docker-compose.yml`
- `<stack>/.env`
- `<stack>/secrets/*`
- `.gitignore` (must exclude secret paths and override files)

---

### **Implementation**

#### 1. Create a restricted secret directory

```shell
mkdir -p /opt/docker-appdata/compose-files/<stack>/secrets
chmod 700 /opt/docker-appdata/compose-files/<stack>/secrets
```

Create a secret file:

```shell
umask 077
openssl rand -base64 48 > /opt/docker-appdata/compose-files/<stack>/secrets/jwt_secret
chmod 600 /opt/docker-appdata/compose-files/<stack>/secrets/jwt_secret
```

If you get errors or are unable to use the secret file make sure there are no trailing newlines. Alternatively use the following command:

```shell
printf "mypassword" > jwt_secret
```

#### 2. Consume secrets with Docker Compose

```yaml
secrets:
  jwt_secret:
    file: ./secrets/jwt_secret

services:
  app:
    image: ghcr.io/example/app:stable
    secrets:
      - jwt_secret
    environment:
      - APP_JWT_SECRET_FILE=/run/secrets/jwt_secret
```

#### 3. Keep `.env` non-sensitive

Allowed in `.env`:

- Public hostnames
- Non-sensitive feature flags
- Time zone and locale values

Not allowed in `.env`:

- Passwords
- API tokens
- Private keys
- Session/JWT master secrets

#### 4. Enforce git hygiene

Example `.gitignore` lines:

```gitignore
**/secrets/
**/*.key
**/*.pem
**/.env.override
```

#### 5. Verify effective secret permissions

```shell
find /opt/docker-appdata/compose-files -type d -name secrets -exec ls -ld {} \;
find /opt/docker-appdata/compose-files -path "*/secrets/*" -type f -exec ls -l {} \;
```

---

### **Rotation Workflow**

Use this sequence for planned rotation:

1. Generate new secret file with restrictive permissions.
2. Update service config to reference same secret filename (content swap) or new secret file.
3. Restart only dependent services.
4. Validate authentication/session behavior.
5. Revoke old credential upstream (DNS provider, SMTP provider, API platform).
6. Record date, owner, and next rotation date.

Minimum recommended intervals:

- Public-edge tokens and API keys: every 90 days
- Internal service credentials: every 180 days
- Emergency rotation: immediately after leakage suspicion

---

### **Incident Response (Secret Leakage)**

If a secret is exposed:

1. Assume compromise and rotate immediately.
2. Revoke old token/key in upstream provider.
3. Invalidate active sessions if the secret affects auth/session signing.
4. Audit access logs around exposure time window.
5. Document root cause and prevention action (permissions, repo hygiene, process change).

---

### **Validation / troubleshooting**

#### **Validation checklist**

- secret files exist only under the intended `secrets/` paths
- file permissions are restrictive enough for the host role
- Compose stacks read secrets through mounted files instead of plaintext environment variables where supported
- a restore test brings the service back without writing plaintext secrets into logs

#### **Troubleshooting checklist**

- If a container cannot read a secret:
  - check file ownership and mode first
  - confirm the image expects `/run/secrets/<name>` or the specific `_FILE` path you configured
- If secret rotation breaks login or API access:
  - verify the upstream credential was updated as well
  - restart only the dependent services instead of bouncing the whole host
- If secret material keeps appearing in repo diffs:
  - tighten `.gitignore`
  - stop storing secrets in `.env`
  - add review checks for `secrets/`, `.pem`, and `.key` files

---

### **Known issues**

- Some images do not support `_FILE` env patterns and require workarounds.
- Running Compose as root increases blast radius for badly permissioned secret files.
- Secret rotation can invalidate sessions and break automation if not coordinated.
