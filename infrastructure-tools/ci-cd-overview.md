---
title: CI/CD Overview
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-source-branch:


## **CI/CD Overview**

---

### **Introduction**

This page defines the baseline CI/CD model for infrastructure, docs, and Docker Compose deployments.

Design goals:

- Fail fast in pull requests
- Require explicit promotion for production
- Keep rollback path simple and deterministic

---

### **Scope in this platform**

CI/CD pipelines cover:

- Documentation (`mkdocs`)
- IaC repositories (Packer/Terraform/Ansible)
- Compose stack repositories
- GitOps reconciliation triggers

Primary runner/orchestrator reference: [Gitea Actions](gitea-actions.md).

---

### **Pipeline model (gated)**

#### Stage 1: Source and policy checks

- Branch naming convention check
- Secret scanning and policy checks
- YAML/JSON/HCL linting

Examples:

```shell
rg -n "(password|token|secret)\s*=" .
ansible-lint
terraform fmt -check -recursive
```

#### Stage 2: Build and static validation

- `mkdocs build --strict`
- `docker compose config --quiet` for changed stacks
- `terraform validate`
- `packer validate`

#### Stage 3: Environment plan generation

- Terraform `plan` artifact for target environment
- Compose change summary (`docker compose config` diff)
- Ansible dry-run (`--check`) where applicable

#### Stage 4: Controlled deployment

- Automatic deploy to `stage` after merge to `stage`
- Manual approval gate for `prod`
- Serialized deployments for DMZ-facing stacks

#### Stage 5: Post-deploy verification

- Container/service health checks
- Route/auth policy checks (Traefik + Authelia)
- Security signal checks (CrowdSec decisions/metrics)

---

### **Promotion and rollback model**

Promotion path:

1. Merge feature branch into `stage`.
2. Validate `stage` deployment.
3. Promote identical commit SHA to `main`.
4. Deploy from signed tag (`vYYYY.MM.DD-N`).

Rollback path:

1. Identify last known good tag.
2. Redeploy exact tag to target environment.
3. Verify service health and auth path.
4. Document incident and corrective action.

---

### **Required quality gates**

Minimum mandatory gates before production deploy:

- Docs: `mkdocs build --strict`
- Compose: `docker compose config --quiet`
- Terraform: `terraform validate` + reviewed `plan`
- Ansible: syntax check + idempotency check in stage
- Security: no committed secrets, no critical policy failures

---

### **Cross-domain integration**

Do not duplicate security/network baselines inside pipelines. Reference canonical pages:

- [Linux Hardening](../security/linux-hardening.md)
- [Secrets Management](../security/secrets-management.md)
- [Encryption](../security/encryption.md)
- [Hashing](../security/hashing.md)
- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)
- [CrowdSec](../networking/crowdsec.md)

---

### **Validation checklist**

For each pipeline run:

- Artifact set includes validation outputs (lint logs, plan outputs).
- Deployment references immutable commit SHA/tag.
- Stage and production promotion records are traceable.
- Rollback command path is documented and tested.

---

### **Troubleshooting**

1. Pipeline passes but deploy fails

- Check environment-specific variables/secrets mapping.
- Compare stage vs prod runtime configs.

2. Compose deploy introduces route/auth regression

- Validate router labels and middleware order.
- Re-run route/auth smoke tests post-deploy.

3. Terraform plan noise from drift

- Investigate manual infra changes outside Terraform.
- Reconcile desired state before applying further changes.

---

### **Operational guardrails**

- No direct deploy from local workstation to production.
- No production deploy without reviewed plan artifacts.
- Keep CI logic versioned in repository.
- Keep deployment jobs deterministic and idempotent.
