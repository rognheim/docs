---
title: Firewall
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-security:


## **Firewall**

---

### **Introduction**

This page defines the network firewall baseline and policy lifecycle for the platform.

Goals:

- Enforce VLAN trust boundaries
- Minimize east-west exposure
- Keep internet-facing services isolated
- Maintain deterministic exception handling

---

### **Prerequisites**

- VLAN design finalized from [VLAN](vlan.md)
- DNS baseline using [Technitium](dns.md)
- Security control model defined in [Security](../security/index.md)
- Public edge stack standardized on [Traefik](traefik.md), [Authelia](authelia.md), [Crowdsec](crowdsec.md)

---

### **Configuration surfaces**

- Gateway firewall ACLs (UniFi Network / UXG-Fiber)
- Network and group objects:
  - RFC1918 group
  - VLAN network objects
  - management-port groups
  - service endpoint objects
- Optional host-level controls on Debian VMs via [Linux Hardening](../security/linux-hardening.md)

---

### **Implementation workflow**

#### 1. Apply base rule ordering

Top-down order baseline:

1. Allow established/related
2. Allow management-to-target (management ports only)
3. Service-specific allow exceptions
4. Explicit deny policies (management protection, inter-VLAN blocks, quarantine)

#### 2. Define mandatory deny boundaries

Mandatory deny controls:

- Block any non-management VLAN traffic into management VLAN
- Block broad RFC1918 lateral traffic unless explicitly allowed
- Block DMZ to internal RFC1918 except explicit service exceptions
- Block quarantine VLAN to any destination

#### 3. Implement explicit service-path exceptions

Use narrowly scoped allow rules similar to your existing pattern:

- DMZ -> SRV app-specific ports for required integrations
- SRV -> Storage/PBS for backup/storage protocols
- Trusted -> selected service endpoints (Plex/Home Assistant)
- IoT/Lab -> DNS resolver (`10.0.20.21:53`) and approved app endpoints only

Keep each rule specific on source group, destination object, protocol, and destination port.

#### 4. Align firewall policy with reverse proxy model

- Expose public applications through Traefik routing, not ad-hoc open ports.
- Keep direct inbound ports on service VLANs closed unless documented exception.
- Enforce auth-sensitive flows through Authelia policies.

#### 5. Track exceptions and lifecycle

For each non-default allow rule, document:

- owner
- service dependency
- source/destination object
- expiry/review date

---

### **Policy reference pattern**

Your current ruleset is a strong baseline for this structure:

- State-aware accept first
- Management restrictions enforced
- DMZ and untrusted VLAN containment
- Explicit app-level cross-VLAN allows
- Final inter-VLAN drop

Keep this as the default model for future additions.

---

### **Validation checklist**

- Stateful return traffic works without broad allow-any rules.
- Management VLAN reaches required admin ports only.
- Untrusted VLANs cannot laterally traverse RFC1918 by default.
- DMZ cannot reach internal networks except defined exceptions.
- Quarantine VLAN is fully isolated.

Example validation commands:

```shell
nc -zv 10.0.20.21 53
nc -zv 10.0.20.13 27878
curl -Ik https://app.example.com
```

---

### **Troubleshooting**

1. New service unreachable across VLANs

- Confirm matching allow rule exists and is above deny rule.
- Confirm destination object/IP is current.
- Confirm protocol and destination port are correct.

2. Unexpected access between VLANs

- Look for overly broad allow rules to RFC1918.
- Check inherited policy groups and object overlap.
- Confirm final inter-VLAN drop rule is active.

3. DNS failures from IoT/Lab

- Confirm allow rule targets Technitium IP/port specifically.
- Check if outbound DNS to public resolvers is intentionally blocked.

---

### **Operational guardrails**

- Use deny-by-default for inter-VLAN traffic.
- Prefer hostnames and service objects over hard-to-track ad-hoc IP/port sprawl.
- Pair every firewall exception with a documented application dependency.
- Re-test high-risk paths after firmware upgrades or major policy edits.
