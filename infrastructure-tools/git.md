---
title: Git
last_reviewed: 2026-03-09
owner: Rognheim
---

# :material-git:


## **Git**

---

### **Introduction**

This page standardizes Git usage for infrastructure repositories in this platform:

- Debian 13 administration hosts
- Infrastructure-as-code repositories (Packer/Terraform/Ansible)
- Docker Compose stack repositories
- CI/CD and GitOps workflows with Gitea Actions

---

### **Prerequisites**

- Debian 13 VM with SSH access
- User account with `sudo`
- Remote Git provider account (Gitea/GitHub/GitLab)
- SSH key pair available (see [SSH](ssh.md))

---

### **Configuration files**

- System config: `/etc/gitconfig`
- User config: `~/.gitconfig`
- Repo config: `.git/config`
- Optional user SSH client config: `~/.ssh/config`

Inspect active settings with source locations:

```shell
git config --list --show-origin
```

---

### **Implementation workflow**

#### 1. Install Git on Debian 13

```shell
sudo apt update
sudo apt install -y git ca-certificates openssh-client
```

Verify:

```shell
git --version
```

#### 2. Set baseline user configuration

```shell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global merge.conflictstyle zdiff3
git config --global pull.ff only
git config --global fetch.prune true
git config --global core.autocrlf input
```

#### 3. Configure SSH remotes (recommended)

Generate a key if needed:

```shell
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
```

Add the key to your Git provider and define host aliases in `~/.ssh/config`:

```sshconfig
Host gitea-prod
  HostName git.example.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

Test connectivity:

```shell
ssh -T gitea-prod
```

Use alias in clone URLs:

```shell
git clone gitea-prod:infrastructure/homelab.git
```

#### 4. Branch and release conventions

Use this pattern for infrastructure repos:

- `main`: production-ready state
- `stage`: pre-production validation
- `feature/<topic>`: active changes
- `hotfix/<topic>`: urgent fixes
- Tags: `vYYYY.MM.DD-N` for deployment points

Example:

```shell
git checkout -b feature/proxmox-template-hardening
git add .
git commit -m "harden template baseline"
git push -u origin feature/proxmox-template-hardening
```

After review/merge, create release tag from `main`:

```shell
git checkout main
git pull
git tag v2026.03.09-1
git push origin v2026.03.09-1
```

#### 5. Daily operations for infrastructure repos

Update local view safely:

```shell
git fetch --all --prune
git status
git log --oneline --decorate --graph -20
```

Rebase feature branch on latest `main`:

```shell
git checkout feature/example
git rebase origin/main
```

Compare pending change:

```shell
git diff --stat origin/main...HEAD
git diff origin/main...HEAD
```

---

### **Validation checklist**

```shell
git config --global --list
git remote -v
git branch -vv
ssh -T gitea-prod
```

Expected:

- Global config includes identity and branch defaults.
- Remotes use SSH aliases or approved HTTPS endpoints.
- Current branch tracks intended upstream.
- SSH auth works without password prompts.

---

### **Troubleshooting**

1. `server certificate verification failed`

```shell
sudo apt install -y ca-certificates
sudo update-ca-certificates
```

2. `Permission denied (publickey)`

```shell
ls -l ~/.ssh/id_ed25519*
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh -Tv gitea-prod
```

3. `detected dubious ownership in repository`

```shell
sudo chown -R "$USER:$USER" /path/to/repository
```

4. Push rejected due to divergence

- Fetch latest refs and rebase branch before pushing.
- Avoid force pushes on shared `main`/`stage` branches.

---

### **Operational guardrails**

- Keep all infra changes in pull requests; avoid direct commits to `main`.
- Sign and tag deployment points for rollback clarity.
- Store secrets outside Git and enforce ignore rules.
- Use CI validation gates from [CI/CD Overview](ci-cd-overview.md) before merge.
