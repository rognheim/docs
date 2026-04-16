---
title: Kubernetes
last_reviewed: 2026-03-09
owner: Rognheim
---

# :simple-kubernetes:


## **Kubernetes**

---

### **Introduction**

Kubernetes is kept as a reference page only.

Primary runtime standard remains:

- Docker Engine
- Docker Compose
- `docker-compose.yml`-based service deployment

Use [Docker](docker.md) for production-aligned runbooks in this environment.

---

### **When Kubernetes is useful here**

Use Kubernetes only when a workload explicitly requires:

- Cluster-level scheduling across multiple workers
- Native Kubernetes controllers/operators
- Advanced service mesh or multi-tenant isolation requirements

If those needs are absent, Compose is the preferred operational model.

---

### **Minimal operational reference**

#### Context and cluster checks

```shell
kubectl config get-contexts
kubectl config use-context <context>
kubectl cluster-info
kubectl get nodes -o wide
```

#### Namespace and workload checks

```shell
kubectl get ns
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl get svc -A
```

#### Debugging quick commands

```shell
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --tail=200
kubectl rollout status deployment/<name> -n <namespace>
```

---

### **Interoperability notes**

- Keep networking/security principles consistent with Compose environment.
- If exposing services publicly, retain equivalent edge controls (TLS, auth, abuse protection).
- Do not mix Kubernetes and Compose for the same application unless ownership and boundaries are explicit.

Reference controls:

- [Traefik](../networking/traefik.md)
- [Authelia](../networking/authelia.md)
- [CrowdSec](../networking/crowdsec.md)
- [Linux Hardening](../security/linux-hardening.md)

---

### **Operational guardrails**

- Treat Kubernetes as optional capability, not default platform path.
- Require explicit decision record before introducing new production Kubernetes workloads.
- Keep deployment ownership and rollback process documented before onboarding workloads.
