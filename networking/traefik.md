---
title: Traefik
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :simple-traefikproxy:


## **Traefik**

---

### **Introduction**

This page documents the public-ingress baseline for the platform. In this environment, Traefik is the canonical reverse proxy for Docker Compose workloads, TLS termination, and middleware-driven policy enforcement at the edge.

---

### **Scope**

This page owns:

- the Traefik container baseline and file layout
- static and dynamic configuration examples used by the platform
- Cloudflare DNS-challenge TLS flow and shared edge middleware hooks
- ingress-side validation and debugging checks

This page does not own identity policy, threat-detection tuning, or upstream application configuration.

---

### **Dependencies / related pages**

- [Docker](../infrastructure-tools/docker.md)
- [Authelia](authelia.md)
- [CrowdSec](crowdsec.md)
- [Cloudflare](cloudflare.md)
- [Firewall](firewall.md)

---

### **Configuration files**

<details>
<summary>docker-compose.yml</summary>

```yaml
---
# Traefik | https://github.com/traefik/traefik/ | https://doc.traefik.io/traefik/

secrets:
  cf_api_email:
    file: ./config/secrets/cf_api_email
  cf_api_key:
    file: ./config/secrets/cf_api_key

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    secrets:
      - cf_api_email
      - cf_api_key
    environment:
      TZ: Europe/Oslo
      CF_API_EMAIL_FILE: etc/traefik/secrets/cf_api_email
      CF_API_KEY_FILE: etc/traefik/secrets/cf_api_key
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./config/acme.json:/etc/traefik/acme.json
      - ./config/config.yml:/etc/traefik/config.yml:ro
      - ./config/secrets:/etc/traefik/secrets:ro
      - logs:/var/log
    networks:
      - traefik
    ports:
      # - 80:80
      - 443:443
    labels:
      - traefik.enable=true # Enable routing through traefik
      - traefik.http.routers.traefik.entrypoints=websecure
      # - traefik.http.routers.traefik.rule=Host(`traefik.rognheim.no`)
      # - traefik.http.routers.traefik.service=api@internal # Needed to load the dashboard
      - traefik.http.routers.traefik.tls=true
      - traefik.http.routers.traefik.tls.certresolver=cloudflare
      - traefik.http.routers.traefik.tls.domains[0].main=rognheim.no
      - traefik.http.routers.traefik.tls.domains[0].sans=*.rognheim.no
      # - traefik.http.routers.traefik.tls.domains[1].main=example.no
      # - traefik.http.routers.traefik.tls.domains[1].sans=*.example.no
      - traefik.http.routers.traefik.middlewares=authelia@docker # Needed to enable authelia forward proxy authentication
      - diun.enable=true # Enable to get notified when updates are available
      - com.centurylinklabs.watchtower.enable=true

volumes:
  logs:

networks:
  traefik:
    driver: bridge

```
</details>

<details>
<summary>traefik.yml</summary>

```yaml
# Static configuration (traefik.yml)

global:
  sendAnonymousUsage: false

# Dashboard
api:
  dashboard: false # true/false. If you want to enable the traefik dashboard.
  insecure: false

serversTransport:
  insecureSkipVerify: false # true/false. Enable for local services routing.

entryPoints:
  web:
    # address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    address: :443
    forwardedHeaders:
      insecure: false
      trustedIPs:
        - 10.0.1.0/24 # MGMT VLAN IP Range
        - 173.245.48.0/20 # Cloudflare IPv4 Ranges
        - 103.21.244.0/22
        - 103.22.200.0/22
        - 103.31.4.0/22
        - 141.101.64.0/18
        - 108.162.192.0/18
        - 190.93.240.0/20
        - 188.114.96.0/20
        - 197.234.240.0/22
        - 198.41.128.0/17
        - 162.158.0.0/15
        - 104.16.0.0/13
        - 104.24.0.0/14
        - 172.64.0.0/13
        - 131.0.72.0/22
        - 2400:cb00::/32 # Cloudflare IPv6 Ranges
        - 2606:4700::/32
        - 2803:f800::/32
        - 2405:b500::/32
        - 2405:8100::/32
        - 2a06:98c0::/29
        - 2c0f:f248::/32
    http: # Enables the listed middlewares for all containers
      middlewares:
        - crowdsec@file
        - app-ratelimit@file
        - secure-headers@file
  # tcp:
  #   address: :3179
  # udp:
  #   address: :3179/udp

log:
  level: INFO # INFO|DEBUG|ERROR
  filePath: /var/log/traefik.log
  format: json

accessLog:
  filePath: /var/log/access.log
  format: json
  bufferingSize: 100
  # filters: # Disabled to give crowdsec complete visibility
  #   statusCodes:
  #     - 204-299
  #     - 400-499
  #     - 500-599
  #   retryAttempts: true
  #   minDuration: 10ms

providers:
  docker:
    endpoint: unix:///var/run/docker.sock
    exposedByDefault: false
  file:
    filename: /etc/traefik/config.yml

certificatesResolvers:
  cloudflare:
    acme:
      email: user@example.no
      storage: /etc/traefik/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - 1.1.1.1:53
          - 1.0.0.1:53
```
</details>

<details>
<summary>config.yml</summary>

```yaml
# Dynamic configuration (config.yml)

### Only for local DNS (Not proxied through Cloudflare)

http:
  middlewares:
    secure-headers:
      headers:
        customResponseHeaders:
          X-Robots-Tag: "none,noarchive,nosnippet,notranslate,noimageindex"
          server: ""
        referrerPolicy: "same-origin"
        contentTypeNosniff: true
        frameDeny: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsSeconds: 63072000
        stsPreload: true

    authelia:
      forwardAuth:
        address: http://authelia:9091/api/authz/forward-auth
        trustForwardHeader: true
        authResponseHeaders:
          - Remote-User
          - Remote-Groups
          - Remote-Email
          - Remote-Name

    crowdsec:
      forwardauth:
        address: http://crowdsec_bouncer:8080/api/v1/forwardAuth
        trustForwardHeader: true

    app-ratelimit:
      rateLimit:
        average: 200
        burst: 100

  # routers:
  #   plex:
  #     entryPoints:
  #       - websecure
  #     rule: "Host(`plex.rognheim.no`)"
  #     service: plex
  #     tls: {}
  #     middlewares:
  #       - crowdsec
  #       - secure-headers

  # services:
  #   plex:
  #     loadBalancer:
  #       servers:
  #         - url: "http://10.0.40.20:32400"
  #       passHostHeader: true

tls: # Only matters if you are not proxying your sites through Cloudflare DNS. Enable these settings in the cloudflare GUI under SSL/TLS -> Edge Certificates
  options:
    default:
      minVersion: VersionTLS13
      sniStrict : true
```
</details>


---

### **Implementation**

- [Official Documentation](https://doc.traefik.io/traefik/) | Traefik.io

!!! Warning "If you want to follow this guide you should make sure you already have docker and docker compose installed"

Navigate to '/opt/docker-appdata/compose-files/traefik'

create the config folder

```shell
mkdir config
```

Change directory into the newly made config folder

```shell
cd config
```

Create the acme.json file

```shell
touch acme.json
```

Change the permissions of the acme.json file

```shell
chmod 600 acme.json
```

Create the static configuration file

```shell
nano traefik.yml
```

Enter the content from the [traefik.yml] template and make the necessary edits

Create the config.yml file

```shell
nano config.yml
```

Enter the content from the [config.yml] template and make the necessary edits

Navigate back to the traefik folder and create the docker-compose.yml file

```shell
nano docker-compose.yml
```

Enter the content from the [docker-compose.yml] template and make the necessary edits

create the secrets folder

```shell
mkdir secrets
```

the path should be /config/secrets/

Create the 'cf_api_email' and 'cf_api_key' files

```shell
nano cf_api_email
nano cf_api_key
```

paste your cloudflare credentials

Spin up the container

```shell
docker compose up -d
```

### **Operations / validation**

#### **Validation checklist**

- `docker compose ps` shows the Traefik container healthy
- `curl -I https://app.example.com` returns the expected certificate and response headers
- Traefik access logs show the real client IP, not only the proxy hop
- the intended middleware chain is attached to the router
- a protected app redirects correctly through Authelia when required

Useful checks:

```shell
docker compose logs traefik --tail 100
tail -f /var/lib/docker/volumes/<traefik_logs_volume>/_data/access.log
```

#### **Troubleshooting checklist**

- If certificates are not issued:
  - verify the Cloudflare API token or secret file
  - verify the requested hostname exists in Cloudflare DNS
  - confirm the resolver is pointing at the intended domain
- If requests work but client IPs are wrong:
  - verify `forwardedHeaders.trustedIPs`
  - confirm Cloudflare and internal proxy ranges are current for your environment
- If a router does not appear:
  - confirm `traefik.enable=true`
  - confirm the service is attached to the same Docker network as Traefik
  - confirm the rule and service port labels match the application

---

### **Known issues**

- Real client IP handling depends on correctly trusting only the upstream proxy ranges you actually use.
- It is easy to let middleware policy drift between apps. Keep the shared file-based middleware set small and deliberate.
