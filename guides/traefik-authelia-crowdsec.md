---
title: Expose and Secure a Public-Facing Containerized Application
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-shield-lock:


## **Expose and Secure a Public-Facing Containerized Application**

---

### **Goal**

Publish a containerized application through the platform edge stack so it is reachable from the internet, authenticated where needed, and protected by the standard reverse-proxy and abuse-resistance controls.

---

### **When to use this guide**

- A service needs a public hostname.
- The app is moving from internal-only access to controlled external access.
- You want one repeatable sequence for DNS, Traefik, Authelia, and CrowdSec.

Use the deeper runbooks for component-specific detail:

- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)
- [CrowdSec](../networking/crowdsec.md)
- [Cloudflare](../networking/cloudflare.md)
- [Secrets Management](../security/secrets-management.md)

---

### **Prerequisites**

- Docker deployment for the application already works internally.
- Cloudflare already manages the public zone.
- Traefik is running and handling TLS for the platform.
- Authelia and CrowdSec are already integrated into the edge stack.
- You know whether the app should be fully public or protected behind login.

---

### **Inputs**

Decide these values before editing the stack:

| Item | Example |
| --- | --- |
| Public hostname | `app.rognheim.no` |
| Service port | `8080` |
| Internal container name | `app` |
| Auth requirement | `Authelia-protected` |
| Public DNS mode | `Cloudflare proxied` |
| Health endpoint | `/health` |

---

### **Step-by-step workflow**

#### 1. Confirm the application is healthy before exposing it

Do not publish a service that only "sort of" works internally.

Minimum Compose pattern:

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
```

Validate from the host first:

```shell
docker compose ps
curl -fsS http://localhost:8080/health
```

#### 2. Prepare the DNS side in Cloudflare

Create the public DNS record before or alongside the Traefik router:

- proxied `A` or `AAAA` record if the hostname points directly to the edge host
- proxied `CNAME` if your routing model uses a stable alias

Use a least-privilege Cloudflare API token for any automation. Keep it in a secret file, not in `.env` or the compose file.

#### 3. Attach the app to the Traefik public network

For a public app on the same Docker host as Traefik:

```yaml
services:
  app:
    networks:
      - app_internal
      - traefik_public

networks:
  app_internal:
    name: app_internal
  traefik_public:
    external: true
```

If the service is on another host or runtime, use the exposure model documented in the relevant runtime page instead of pretending Docker labels will reach across hosts.

#### 4. Add the Traefik router labels

Example protected route:

```yaml
services:
  app:
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik_public
      - traefik.http.routers.app.entrypoints=websecure
      - traefik.http.routers.app.rule=Host(`app.rognheim.no`)
      - traefik.http.routers.app.tls=true
      - traefik.http.routers.app.middlewares=authelia@docker
      - traefik.http.services.app.loadbalancer.server.port=8080
```

In this platform, CrowdSec and shared security headers are typically applied globally or through the file provider. Keep the app labels small unless the service needs an exception.

#### 5. Decide whether the app belongs behind Authelia

Use Authelia for:

- admin UIs
- personal services
- anything that should not be anonymously reachable

Avoid forcing login in front of services that must remain publicly consumable, such as some media or webhook endpoints, unless the app’s user flow explicitly supports it.

If the app needs partial protection, split routes intentionally instead of mixing private and public paths under one careless rule.

#### 6. Make sure CrowdSec can see the traffic

CrowdSec only helps when the right logs reach it.

Confirm:

- Traefik access logs are enabled
- CrowdSec ingests those logs
- remediation is wired through the Traefik plugin or another supported enforcement component

This step is especially important for new public apps with higher crawl or abuse risk.

#### 7. Apply and test the change

```shell
docker compose config --quiet
docker compose up -d
docker compose ps
```

Then test:

- anonymous access behavior
- Authelia login flow if enabled
- TLS certificate presentation
- application health after going through the public hostname

---

### **Validation**

Use at least these checks:

```shell
curl -I https://app.rognheim.no
openssl s_client -connect app.rognheim.no:443 -servername app.rognheim.no </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
docker logs traefik --tail 100
```

Confirm:

- DNS resolves to the intended Cloudflare-managed path
- TLS is valid for the hostname
- the app route appears in Traefik without router errors
- Authelia protects the route when expected
- CrowdSec can see traffic and does not immediately ban normal browsing

---

### **Rollback / failure modes**

- If the app is unhealthy through Traefik but healthy locally, revert the router labels and inspect the service port, network, and middleware chain.
- If login loops occur, remove the route from public use until Authelia cookie domain, upstream headers, and policy are corrected.
- If Cloudflare points to the wrong target, disable or correct the record before debugging the app itself.
- If CrowdSec blocks normal usage, tune the offending scenario rather than removing the whole protection layer blindly.
