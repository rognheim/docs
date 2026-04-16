---
title: Network UPS Tools
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-battery-charging-outline:

## **Network UPS Tools**

---

### **Introduction**

Network UPS Tools (NUT) provides shared UPS state to the rest of the platform so critical hosts can monitor battery status and shut down cleanly during a power event. In this environment the main pattern is one NUT server attached to the UPS over USB, with Proxmox and other important systems acting as NUT clients.

---

### **Scope**

This page owns:

- the server and client split for NUT
- the minimum config needed to expose UPS state
- graceful-shutdown validation steps

This page does not own:

- host-specific shutdown sequencing for every guest workload
- generator, router, or PDU automation outside the UPS chain

---

### **Prerequisites**

- A supported UPS connected to one always-on host over USB or serial.
- Network connectivity between the NUT server and client hosts.
- A clear decision on which host is the authoritative NUT server.

---

### **Dependencies / related pages**

- [Proxmox](../hypervisors/proxmox.md)
- [Debian](../operating-systems/debian.md)

---

### **Configuration files**

On the NUT server:

- `/etc/nut/nut.conf`
- `/etc/nut/ups.conf`
- `/etc/nut/upsd.conf`
- `/etc/nut/upsd.users`
- `/etc/nut/upsmon.conf`

On NUT clients:

- `/etc/nut/nut.conf`
- `/etc/nut/upsmon.conf`
- optional `/etc/nut/upssched.conf`
- optional `/etc/nut/upssched-cmd`

---

### **Implementation**

#### **1. Build the NUT server**

Install the packages on the host physically connected to the UPS:

```bash
sudo apt-get update
sudo apt-get install nut -y
```

Set server mode in `/etc/nut/nut.conf`:

```ini
MODE=netserver
```

Example `/etc/nut/ups.conf`:

```ini
[ups]
  driver = usbhid-ups
  port = auto
  desc = "Rack UPS"
```

Start by using the simplest viable driver and only add vendor-specific tuning when the basic driver is not stable.

#### **2. Expose the UPS over the network**

Allow the monitoring network in `/etc/nut/upsd.conf`:

```ini
LISTEN 0.0.0.0 3493
```

Create a monitoring user in `/etc/nut/upsd.users`:

```ini
[monuser]
  password = strong-password-here
  upsmon primary
```

Point `/etc/nut/upsmon.conf` on the server at the local UPS:

```ini
MONITOR ups@localhost 1 monuser strong-password-here primary
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POWERDOWNFLAG /etc/killpower
```

Restart the service after each config change:

```bash
sudo systemctl restart nut-server
sudo systemctl restart nut-monitor
```

#### **3. Add a NUT client**

Install the package on each client:

```bash
sudo apt-get install nut-client -y
```

Set client mode in `/etc/nut/nut.conf`:

```ini
MODE=netclient
```

Point `/etc/nut/upsmon.conf` to the server:

```ini
MONITOR ups@192.168.10.20 1 monuser strong-password-here secondary
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POWERDOWNFLAG /etc/killpower
```

Restart monitoring:

```bash
sudo systemctl restart nut-monitor
```

#### **4. Add scheduled actions if the client needs extra logic**

If a client should run a script before shutdown, use `upssched`.

Example ownership fix:

```bash
sudo chmod +x /etc/nut/upssched-cmd
sudo chown root:nut /etc/nut/upssched-cmd
```

Keep the script small and deterministic. It should log its actions and avoid trying to orchestrate the whole environment.

---

### **Validation / troubleshooting**

#### **Basic validation**

On the NUT server:

```bash
upsc ups@localhost
```

On a client:

```bash
upsc ups@192.168.10.20
systemctl status nut-monitor
```

You want to see battery charge, input voltage, runtime, and status fields. If `upsc` cannot read them, the rest of the stack is not ready yet.

#### **Shutdown validation**

- confirm the server sees `OL` while utility power is normal
- test a monitored power event during a maintenance window
- confirm clients enter the expected delayed shutdown path
- verify Proxmox host shutdown order matches workload importance

#### **Troubleshooting checklist**

- If the UPS is not detected:
  - check USB passthrough and cabling
  - verify the selected driver in `ups.conf`
- If clients cannot connect:
  - verify port `3493` is reachable
  - verify `upsd.users` credentials
  - verify the UPS name in `MONITOR` matches the section name in `ups.conf`
- If `upssched-cmd` does not run:
  - verify execute permissions
  - verify ownership is `root:nut`

---

### **Known issues**

- Shutdown testing is easy to postpone, but a NUT setup is not trustworthy until at least one controlled power-event test has been completed.
- UPS naming mismatches are a common source of client failures. The identifier before `@host` must exactly match the section name in `ups.conf`.
