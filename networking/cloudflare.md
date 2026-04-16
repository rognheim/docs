---
title: Cloudflare
last_reviewed: 2026-04-16
owner: Rognheim
---

# :fontawesome-brands-cloudflare:

## **Cloudflare**

---

### **Introduction**

Cloudflare is the public DNS and edge entry point for services that leave the private network. In this platform it owns DNS records, TLS edge behavior, optional proxying, and optionally Cloudflare Tunnel for services that should not expose the origin directly.

---

### **Scope**

This page owns:

- DNS records for public services.
- API tokens used by reverse proxy and automation.
- Basic Cloudflare Tunnel workflow.
- Validation steps after DNS or edge changes.

This page does not own:

- Reverse proxy routing inside the stack. See [Traefik](traefik.md).
- Authentication and access policy inside the app path. See [Authelia](authelia.md).
- Bot blocking and remediation inside the origin stack. See [CrowdSec](crowdsec.md).

---

### **Prerequisites**

- A domain delegated to Cloudflare.
- A working public origin, typically routed by Traefik.
- A decision on whether the service should be:
  - DNS only, with traffic going directly to the origin.
  - Proxied through Cloudflare.
  - Published through Cloudflare Tunnel.

---

### **Dependencies / related pages**

- [Traefik](traefik.md)
- [Expose and Secure a Public-Facing Containerized Application](../guides/traefik-authelia-crowdsec.md)
- [Linux Hardening](../security/linux-hardening.md)

---

### **Configuration files**

- Cloudflare dashboard:
  - DNS records
  - SSL/TLS mode
  - Zero Trust tunnels
- Local secrets store for API tokens
- Optional `cloudflared` container or host config if using tunnels

---

### **Implementation**

#### **1. Choose the edge pattern**

Use one of these patterns and keep it consistent per service:

- `DNS only`: best for direct origin control and simpler debugging.
- `Proxied`: best when the service is normal HTTP or HTTPS and benefits from Cloudflare caching, WAF, or origin masking.
- `Tunnel`: best when the origin should not open inbound ports at all.

#### **2. Create a least-privilege API token**

Create a dedicated token for automation instead of using the global API key.

Recommended minimum scopes for DNS automation:

- `Zone - DNS - Edit`
- `Zone - Zone - Read`

Store the token in the same secret source used by Traefik or deployment automation. Do not commit it into compose files or shell history.

#### **3. Create the DNS records**

Common patterns:

- Public reverse proxy on the root domain:

```text
Type: A
Name: @
Content: <public IPv4 of reverse proxy>
Proxy status: Proxied or DNS only
```

- Public reverse proxy on a subdomain:

```text
Type: A
Name: apps
Content: <public IPv4 of reverse proxy>
Proxy status: Proxied or DNS only
```

- Service routed through Traefik on a subdomain:

```text
Type: CNAME
Name: plex
Target: apps.example.com
Proxy status: Proxied or DNS only
```

Use `Proxied` only for HTTP or HTTPS services that are expected to sit behind Cloudflare. For raw TCP, UDP, or non-web protocols, keep the record `DNS only` unless a Cloudflare product explicitly supports that traffic.

#### **4. Confirm TLS behavior**

If the origin already serves valid certificates, prefer end-to-end encryption. A practical baseline is:

- Cloudflare edge: `Full (strict)`
- Origin: valid certificate on Traefik or the application

Avoid loose TLS modes as a long-term state. If you must use one briefly during migration, document it and remove it after the origin certificate is fixed.

#### **5. Publish a service through Cloudflare Tunnel when direct inbound access is not wanted**

This is the simpler pattern when the origin should stay private:

```bash
cloudflared tunnel login
cloudflared tunnel create homelab
cloudflared tunnel route dns homelab app.example.com
```

Minimal tunnel ingress example:

```yaml
tunnel: <tunnel-id>
credentials-file: /etc/cloudflared/<tunnel-id>.json

ingress:
  - hostname: app.example.com
    service: https://traefik:443
    originRequest:
      noTLSVerify: false
  - service: http_status:404
```

Use tunnels for admin surfaces, temporary migrations, and environments where opening ports on the edge router is not desirable.

#### **6. Record ownership**

For every public hostname, record:

- origin service owner
- expected exposure type
- whether Authelia is required
- whether Cloudflare proxying is enabled
- where the certificate is terminated

This prevents “mystery DNS” and makes later incident work much easier.

---

### **Validation / troubleshooting**

#### **Basic validation**

Check that the record resolves:

```bash
dig +short app.example.com
```

Check the HTTP response headers from outside the network:

```bash
curl -I https://app.example.com
```

If using a tunnel, check tunnel status:

```bash
cloudflared tunnel list
cloudflared tunnel info homelab
```

#### **Troubleshooting checklist**

- If DNS resolves but the app does not load:
  - confirm Traefik has a router for the hostname
  - confirm the service container is on the expected proxy network
  - confirm the origin certificate matches the hostname when using strict TLS
- If the app works on `DNS only` but breaks on `Proxied`:
  - check for non-HTTP traffic
  - check websocket or large-upload behavior
  - confirm the origin sees the correct forwarded headers
- If tunnel routes exist but traffic still fails:
  - verify the `cloudflared` container can reach Traefik over the internal network
  - verify the ingress hostnames exactly match the DNS hostnames

---

### **Known issues**

- Cloudflare proxying is not a universal transport for every protocol. For non-web traffic, keep the record `DNS only` unless the traffic is explicitly supported.
- It is easy to forget whether TLS terminates at Cloudflare, Traefik, or both. Document the expected path when onboarding a new service.
