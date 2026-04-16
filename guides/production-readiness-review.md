---
title: Production Readiness Review
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-clipboard-check-multiple-outline:

## **Production Readiness Review**

---

### **Goal**

Use this review before calling a service production-ready. The aim is to test whether the service can survive routine operations, faults, and operator turnover, not just whether it worked once during setup.

---

### **When to use this guide**

- promoting a service from `working` to `production`
- re-reviewing a critical service after a major redesign
- checking whether an older service still meets today’s operational baseline

---

### **Prerequisites**

- the service is already deployed and functional
- baseline docs exist for the service
- the owner can answer operational questions or gather missing evidence

---

### **Inputs**

- service documentation
- deployment location
- backup and monitoring notes
- known risks and exceptions

---

### **Step-by-step workflow**

#### **1. Review service ownership and scope**

Confirm:

- the owner is current
- the purpose is documented
- upstream and downstream dependencies are known

#### **2. Review reliability**

Check:

- startup behavior
- health checks
- restart expectations
- expected failure modes

If the service is fragile after routine restarts or host reboots, it is not ready.

#### **3. Review security controls**

Check:

- auth model
- secret handling
- public exposure path
- host hardening and network boundaries

#### **4. Review backup and restore readiness**

Confirm:

- important state is backed up
- restore notes exist
- at least one restore path has been tested or is easy to test during a maintenance window

#### **5. Review observability**

Check:

- operators can find logs quickly
- the basic health state is visible
- important failures trigger alerts or at least clear operator signals

#### **6. Review change safety**

Confirm:

- updates have a defined path
- rollback steps exist
- config changes are traceable

#### **7. Review capacity and lifecycle risks**

Check:

- storage growth
- memory and CPU headroom
- external dependencies such as DNS providers, APIs, certificates, or storage appliances

#### **8. Record an explicit decision**

Choose one:

- promote to `production`
- keep as `working`
- demote until gaps are addressed

Write down the reason. “Feels stable” is not enough.

---

### **Validation**

A service is ready for `production` only when:

- owner, purpose, and dependencies are documented
- backup and restore expectations are credible
- monitoring and troubleshooting are workable
- security controls match the exposure level
- known risks are either accepted or scheduled for correction

---

### **Rollback / failure modes**

- If a critical control is missing, keep the service at `working` and create follow-up tasks.
- If restore readiness is weak, do not promote based only on uptime.
- If public exposure exists without matched security controls, treat that as a blocker rather than a minor gap.
