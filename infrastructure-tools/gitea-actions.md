---
title: Gitea Actions
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-gitea:


## **Gitea Actions**

---

### **Introduction**

Gitea Actions runs CI/CD workflows for docs, infrastructure, and Compose repositories.

This page covers:

- Runner deployment on Debian 13 with Docker Compose
- Runner hardening and network placement
- Workflow patterns for lint/build/validate/deploy

---

### **Prerequisites**

- Running Gitea instance (example: `https://git.example.com`)
- Runner registration token from Gitea
- Debian 13 runner VM with Docker/Compose installed
- Network egress from runner to Gitea and required registries

---

### **Configuration files**

- Runner stack:
  - `/opt/docker-appdata/compose-files/gitea-runner/docker-compose.yml`
  - `/opt/docker-appdata/compose-files/gitea-runner/config.yaml`
  - `/opt/docker-appdata/compose-files/gitea-runner/data/`
- Repository workflows:
  - `.gitea/workflows/*.yml`

---

### **Implementation workflow**

#### 1. Create runner stack directory

```shell
mkdir -p /opt/docker-appdata/compose-files/gitea-runner/{data,config}
cd /opt/docker-appdata/compose-files/gitea-runner
```

#### 2. Generate runner config template

```shell
docker run --rm gitea/act_runner:latest act_runner generate-config > config/config.yaml
```

#### 3. Register runner with Gitea

```shell
docker run --rm -v "$PWD/config:/config" gitea/act_runner:latest \
  act_runner register \
  --instance https://git.example.com \
  --token <registration-token> \
  --name runner-prod-01 \
  --labels "debian13:docker://catthehacker/ubuntu:act-latest"
```

This writes registration credentials under `config/` or `data/` depending on runner version/config.

#### 4. Deploy runner via Compose

Example `docker-compose.yml`:

```yaml
services:
  runner:
    image: gitea/act_runner:latest
    container_name: gitea-runner
    restart: unless-stopped
    command: daemon --config /config/config.yaml
    volumes:
      - ./config:/config
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Europe/Oslo
```

Start runner:

```shell
docker compose up -d
docker compose ps
```

#### 5. Harden runner host and execution model

Recommended controls:

- Use dedicated runner VM in SRV VLAN (`10.0.20.0/24`).
- Restrict inbound access to SSH from management VLAN only.
- Limit outbound traffic to Gitea, registries, and required deploy targets.
- Keep runner host patched and monitored.
- Separate stage and production runners.

Host controls: [Linux Hardening](../security/linux-hardening.md).

#### 6. Implement baseline workflows

Docs validation workflow (`.gitea/workflows/docs.yml`):

```yaml
name: docs
on:
  pull_request:
  push:
    branches: [stage, main]

jobs:
  mkdocs:
    runs-on: debian13
    steps:
      - uses: actions/checkout@v4
      - run: pip install mkdocs mkdocs-material
      - run: mkdocs build --strict
```

Compose validation + deploy workflow (`.gitea/workflows/compose.yml`):

```yaml
name: compose
on:
  pull_request:
  push:
    branches: [stage, main]

jobs:
  validate:
    runs-on: debian13
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f docker-compose.yml config --quiet

  deploy_stage:
    if: github.ref == 'refs/heads/stage'
    needs: validate
    runs-on: debian13
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy.sh stage

  deploy_prod:
    if: github.ref == 'refs/heads/main'
    needs: validate
    runs-on: debian13
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy.sh prod
```

---

### **Validation checklist**

```shell
docker logs gitea-runner --tail 100
docker exec gitea-runner act_runner daemon --version
```

In Gitea UI, confirm runner status:

- Runner appears online.
- Expected labels are available.
- Workflows are assigned to intended runner group.

---

### **Troubleshooting**

1. Runner offline

- Validate registration token/instance URL.
- Check container logs and DNS connectivity to Gitea.

2. Workflows stuck waiting for runner

- Verify `runs-on` labels match registered runner labels.
- Confirm runner scope (repo/org/global) includes target repo.

3. Docker steps fail in workflow

- Verify `/var/run/docker.sock` mount and host daemon state.
- Confirm workflow image has required CLI dependencies.

---

### **Operational guardrails**

- Keep separate runners for production deployments.
- Avoid broad credentials in runner environment.
- Treat workflow files as code-reviewed production assets.
- Use [CI/CD Overview](ci-cd-overview.md) gates for deployment policy.
