---
title: GitOps Workflow
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-source-branch-sync:


## **GitOps Workflow**

---

### **Introduction**

This page defines how desired state in Git is reconciled into running Docker Compose environments.

In this platform, GitOps is used for:

- Controlled promotion from `stage` to `main`
- Deterministic deploys from immutable commit SHAs/tags
- Drift detection and recovery

---

### **Scope and model**

GitOps applies to:

- Compose stack repositories
- Host-level deploy scripts
- Gitea Actions deployment jobs

Git remains the source of truth for:

- `docker-compose.yml`
- stack configuration templates
- deployment scripts and policies

Secrets remain outside Git per [Secrets Management](../security/secrets-management.md).

---

### **Repository structure (recommended)**

```text
stacks/
  traefik/
    docker-compose.yml
    env/
      stage.env
      prod.env
  authelia/
    docker-compose.yml
  crowdsec/
    docker-compose.yml
scripts/
  deploy.sh
  verify.sh
  rollback.sh
```

---

### **Branch and promotion policy**

- `feature/*` -> change development
- `stage` -> automatic stage deployment
- `main` -> production candidate
- `vYYYY.MM.DD-N` tags -> production release points

Promotion rule:

1. Merge feature into `stage`.
2. Validate in stage environment.
3. Promote same commit to `main`.
4. Tag and deploy.

---

### **Reconciliation workflow**

#### 1. Deploy script contract

Example `scripts/deploy.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

ENVIRONMENT="$1"
STACK_PATH="/opt/docker-appdata/compose-files/${ENVIRONMENT}"

cd "$STACK_PATH"
git fetch --all --prune
git checkout "$GIT_SHA"
git reset --hard "$GIT_SHA"

docker compose pull
docker compose up -d --remove-orphans
```

Set `GIT_SHA` from CI job context to enforce immutable deploys.

#### 2. Verification script

Example `scripts/verify.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

docker compose ps

docker ps --format '{{.Names}} {{.Status}}' | rg -v 'Up'
```

Extend with route/auth checks for public stacks.

#### 3. Drift detection

Run scheduled job:

```shell
cd /opt/docker-appdata/compose-files/prod
git fetch
git status --porcelain
docker compose config --quiet
```

Drift signals:

- local uncommitted changes
- checked-out commit differs from release tag/SHA
- running compose config diverges from Git state

#### 4. Rollback flow

Rollback to previous good tag:

```shell
cd /opt/docker-appdata/compose-files/prod
git fetch --tags
git checkout v2026.03.09-1
docker compose pull
docker compose up -d --remove-orphans
```

Record rollback reason and follow-up action.

---

### **Validation checklist**

- Deployment logs include commit SHA/tag.
- Host checkout matches intended release reference.
- Post-deploy checks pass (health + routing + auth).
- Drift checks run on schedule and alert on mismatch.

---

### **Troubleshooting**

1. Deploy script updates wrong commit

- Ensure CI exports explicit immutable SHA.
- Avoid branch-based deploy commands in production.

2. Stage and production behavior differ

- Compare env files and mounted config paths.
- Validate external dependencies (DNS, certificates, middleware chain).

3. Frequent drift detected

- Eliminate manual edits on deployment hosts.
- Route all changes through pull request and pipeline.

---

### **Operational guardrails**

- Keep deploy scripts idempotent and non-interactive.
- Never edit production compose files directly on hosts.
- Tie every deploy event to commit SHA + operator/pipeline identity.
- Integrate with [CI/CD Overview](ci-cd-overview.md) and [Gitea Actions](gitea-actions.md).
