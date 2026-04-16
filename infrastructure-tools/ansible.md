---
title: Ansible
status: working
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-ansible:


## **Ansible**

---

### **Introduction**

Ansible applies consistent configuration to Debian 13 VMs after provisioning.

In this platform, Ansible is used for:

- Baseline host configuration and package state
- Security baseline enforcement
- Docker runtime preparation
- App host role configuration before Compose deployments

---

### **Prerequisites**

- Provisioned Debian 13 VMs (typically via Terraform)
- SSH key-based access from control node
- Python available on target hosts (default on Debian 13)
- Sudo privileges for automation account

---

### **Repository layout (recommended)**

```text
infrastructure/
  ansible/
    ansible.cfg
    inventories/
      prod/
        hosts.yml
        group_vars/
          all.yml
          srv.yml
          dmz.yml
      stage/
        hosts.yml
    roles/
      baseline/
      docker_host/
      traefik_edge/
    playbooks/
      bootstrap.yml
      baseline.yml
      docker.yml
      edge.yml
```

---

### **Key files**

- `ansible/ansible.cfg`
- `ansible/inventories/<env>/hosts.yml`
- `ansible/playbooks/*.yml`
- `ansible/roles/*`

---

### **Implementation workflow**

#### 1. Install Ansible on control node

```shell
sudo apt update
sudo apt install -y ansible sshpass
ansible --version
```

#### 2. Configure Ansible defaults

Example `ansible.cfg`:

```ini
[defaults]
inventory = inventories/prod/hosts.yml
host_key_checking = True
forks = 20
stdout_callback = yaml
interpreter_python = auto_silent
retry_files_enabled = False

[ssh_connection]
pipelining = True
```

#### 3. Build inventory by role and VLAN

Example `inventories/prod/hosts.yml`:

```yaml
all:
  children:
    srv:
      hosts:
        srv-app-01:
          ansible_host: 10.0.20.21
        srv-monitor-01:
          ansible_host: 10.0.20.31
    dmz:
      hosts:
        dmz-edge-01:
          ansible_host: 10.0.30.10
```

Example `group_vars/all.yml`:

```yaml
ansible_user: admin
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
timezone: Europe/Oslo
```

#### 4. Bootstrap playbook (first-run idempotent)

Example `playbooks/bootstrap.yml`:

```yaml
- name: Bootstrap Debian hosts
  hosts: all
  become: true
  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install baseline packages
      ansible.builtin.apt:
        name:
          - qemu-guest-agent
          - curl
          - ca-certificates
          - sudo
        state: present

    - name: Ensure qemu-guest-agent is enabled
      ansible.builtin.service:
        name: qemu-guest-agent
        enabled: true
        state: started
```

#### 5. Baseline and hardening playbook

```yaml
- name: Apply platform baseline
  hosts: all
  become: true
  roles:
    - baseline
```

`baseline` role should implement controls from [Linux Hardening](../security/linux-hardening.md) instead of duplicating ad-hoc commands per host.

#### 6. Docker host playbook

```yaml
- name: Prepare Docker hosts
  hosts: srv
  become: true
  roles:
    - docker_host
```

Role outcomes:

- Docker Engine + Compose plugin installed
- `/opt/docker-appdata/compose-files` created
- permission model aligned to secrets and runtime requirements

#### 7. Edge-specific playbook

```yaml
- name: Configure reverse proxy edge hosts
  hosts: dmz
  become: true
  roles:
    - traefik_edge
```

Role includes only host/runtime prerequisites. Stack config remains in Compose repositories.

#### 8. Run sequence

```shell
cd infrastructure/ansible
ansible all -m ping
ansible-playbook playbooks/bootstrap.yml
ansible-playbook playbooks/baseline.yml
ansible-playbook playbooks/docker.yml
ansible-playbook playbooks/edge.yml
```

Limit by host/group when needed:

```shell
ansible-playbook playbooks/docker.yml -l srv-app-01
```

---

### **Validation checklist**

```shell
ansible all -m ping
ansible srv -m command -a "docker --version"
ansible dmz -m command -a "systemctl is-active docker"
```

Expected:

- SSH connectivity works for all target hosts.
- Playbooks run idempotently (`changed=0`) on second run.
- Docker and baseline services are active where expected.

---

### **Troubleshooting**

1. SSH unreachable/permission denied

- Verify key and user in inventory/group vars.
- Test raw SSH outside Ansible first.

2. Sudo failures in playbooks

- Confirm automation user has sudo privileges.
- Use `--ask-become-pass` only when policy requires interactive escalation.

3. Drift between manual changes and role state

- Re-run relevant playbook and review differences.
- Move persistent manual changes into role defaults/tasks.

---

### **Operational guardrails**

- Keep roles small and single-purpose.
- Enforce idempotency; avoid shell tasks when native modules exist.
- Split environment inventory (`stage`, `prod`) and apply separately.
- Use CI lint checks (`ansible-lint`) before merge in [CI/CD Overview](ci-cd-overview.md).
