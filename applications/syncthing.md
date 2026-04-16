---
title: Syncthing
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-file-sync:


## **Syncthing**

---

### **Role summary**

Syncthing provides peer-to-peer file synchronization between trusted devices. It is best used for controlled replication workflows where ownership, versioning, and delete behavior are clearly understood.

---

### **Deployment model**

- Compose-managed service on a Debian host
- Typically internal-only or reachable over a trusted private-access path
- Folder-level permissions and retention settings matter more than the initial install itself

---

### **Required dependencies**

- [Docker](../infrastructure-tools/docker.md)
- [WireGuard](../networking/wireguard.md)
- [Tailscale](../networking/tailscale.md)

---

### **Post-install checklist**

- Pair devices and verify trust fingerprints carefully.
- Define folder-level read/write ownership before syncing real data.
- Enable versioning or recovery paths for accidental deletes.
- Check firewall and remote-access assumptions before exposing sync traffic.

---

### **Related canonical pages**

- [File Browser](filebrowser.md)
- [WireGuard](../networking/wireguard.md)
- [Tailscale](../networking/tailscale.md)
