---
title: Secret Rotation Playbook
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-lock-reset:

## **Secret Rotation Playbook**

---

### **Goal**

Rotate secrets safely with minimal service disruption and without losing track of dependent systems, active sessions, or upstream credentials.

---

### **When to use this guide**

- scheduled credential rotation
- emergency response after suspected secret exposure
- replacing bootstrap credentials with durable ones

---

### **Prerequisites**

- an inventory of the secret and the systems that depend on it
- access to the upstream system that issued the credential
- a rollback or re-issue path if authentication breaks

---

### **Inputs**

- secret name and purpose
- service or stack that consumes it
- upstream provider or identity system
- expected restart or session impact

---

### **Step-by-step workflow**

#### **1. Map the dependency chain**

List:

- which service reads the secret
- where the secret is stored
- what upstream system validates it
- whether active sessions depend on it

Do not rotate blind. Most secret-rotation mistakes are dependency mistakes.

#### **2. Stage the replacement secret**

Create the new secret with restrictive permissions in the correct stack directory.

Example:

```bash
umask 077
openssl rand -base64 48 > /opt/docker-appdata/compose-files/<stack>/secrets/<secret_name>.new
chmod 600 /opt/docker-appdata/compose-files/<stack>/secrets/<secret_name>.new
```

#### **3. Update the consuming service**

Choose one pattern:

- replace the existing secret file contents in place
- point the service to the new secret file and then remove the old one later

Prefer the method with the smallest restart and rollback surface for that service.

#### **4. Restart only the affected services**

Example:

```bash
docker compose up -d <service-name>
```

Avoid restarting unrelated services just because they share a host.

#### **5. Revoke the old credential upstream**

Only after the new credential is confirmed working:

- revoke the old API token
- rotate the old password
- invalidate old sessions if the secret signs sessions or tokens

#### **6. Record the rotation**

Capture:

- date
- owner
- systems affected
- next rotation target
- any sessions or automation that were intentionally disrupted

---

### **Validation**

Confirm:

- the service authenticates successfully with the new secret
- the old secret no longer works
- logs contain no repeated authentication failures
- dependent automation still functions

---

### **Rollback / failure modes**

- If the new secret fails, restore the previous secret immediately and re-test before revoking anything upstream.
- If rotation breaks active sessions, verify whether that was expected and document it for the next run.
- If the dependency chain was incomplete, update the secret inventory before attempting another rotation.
