---
title: Service Onboarding Checklist
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-check-decagram-outline:

## **Service Onboarding Checklist**

---

### **Goal**

Use this checklist before introducing a new service into the platform so ownership, exposure, recovery, and day-two operations are defined before the service becomes “important by accident.”

---

### **When to use this guide**

- onboarding a new containerized application
- introducing a new VM-hosted service
- exposing an internal-only tool to more users
- formalizing an older service that was previously undocumented

---

### **Prerequisites**

- a proposed service name and owner
- a clear purpose for the service
- a basic deployment target, such as Docker, an LXC container, or a VM

---

### **Inputs**

- service name
- owner or responsible operator
- deployment model
- exposure model
- storage and backup requirements

---

### **Step-by-step workflow**

#### **1. Define ownership and purpose**

Document:

- what the service does
- who owns it
- who can approve major changes
- whether it is experimental, working, or production-ready

If no owner exists, the service is not ready to join the main platform.

#### **2. Define the network and access model**

Decide:

- internal only or public
- whether Traefik is required
- whether Authelia is required
- which VLAN or network zone it belongs to

Write down the intended hostname and whether Cloudflare or Tailscale are part of the access path.

#### **3. Define the data model**

Record:

- config path
- persistent data path
- backup requirement
- recovery target

If the service stores important state, it must have a documented backup and restore path before promotion beyond a test deployment.

#### **4. Define observability**

Minimum expectations:

- logs can be reached quickly
- basic health checks exist
- important failures are visible in monitoring or operator workflow

Use [Add Monitoring to a Containerized Application](container-application-monitoring.md) if the service needs a deeper observability pass.

#### **5. Define the update and rollback path**

Document:

- how updates are applied
- whether downtime is expected
- how to roll back
- what validation proves the service is healthy after a change

#### **6. Define the security baseline**

Check:

- secrets are not stored in Git or plaintext `.env` files
- least-privilege access is used where possible
- public services sit behind the expected edge controls

#### **7. Record final acceptance notes**

Before calling the onboarding complete, capture:

- related docs pages
- known risks
- next review date

---

### **Validation**

A service is onboarded only when:

- the owner is known
- the deployment path is documented
- the backup or restore path is defined
- the exposure model is explicit
- update and rollback notes exist

---

### **Rollback / failure modes**

- If ownership is unclear, stop onboarding and assign it before proceeding.
- If the restore path is unknown, keep the service in draft or lab-only state.
- If the service needs public exposure but the auth or edge path is undecided, keep it internal until the design is complete.
