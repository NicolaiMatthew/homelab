# Proxmox Setup

---

## port: 8006 description: Proxmox VE Hypervisor status: running

## Overview

Proxmox VE hypervisor running on a Lenovo ThinkCentre M920q. Hosts all LXCs and VMs for the homelab.

## Hardware

|Component|Detail|
|---|---|
|Machine|Lenovo ThinkCentre M920q|
|CPU|Intel 8th/9th Gen (QuickSync capable)|
|Hypervisor|Proxmox VE|
|Boot Drive|/dev/sda|

## Access

- Web UI: `https://PROXMOX-IP:8006`
- Default port: `8006`

---

## LXC Best Practices

### Privileged vs Unprivileged

Always use **unprivileged** LXCs unless hardware passthrough requires otherwise.

> ⚠️ Unprivileged LXCs remap UIDs — UID 1000 inside maps to UID 101000 on the host. Always `chown` bind mounts from the host using the offset UID.

```bash
# Fix permissions for unprivileged LXC bind mounts
chown -R 101000:101000 /your/mount/path
```

### TUN Device (Required for VPN containers)

Add to LXC config before starting:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Or via UI: LXC → Options → Features → TUN device ✅

### GPU Passthrough (Intel QuickSync)

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card1 dev/dri/card1 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

Make permissions persistent:

```bash
nano /etc/udev/rules.d/99-dri.rules
```

```
SUBSYSTEM=="drm", KERNEL=="card1", MODE="0666"
SUBSYSTEM=="drm", KERNEL=="renderD128", MODE="0666"
```

```bash
udevadm control --reload-rules && udevadm trigger
```

### Bind Mounts

Add storage mounts to LXC config:

```
mp0: /mnt/host-path,mp=/mnt/container-path
```

---

## LXC Inventory

|ID|Name|Purpose|
|---|---|---|
|100|tailscale|VPN exit node|
|101|file-share|File sharing|
|102|n8n|Workflow automation|
|103|upsnap|Wake on LAN|
|105|crafty-controller|Minecraft server manager|
|106|prometheus|Metrics collection|
|107|prometheus-pve-exporter|Proxmox metrics exporter|
|108|grafana|Metrics dashboard|
|109|prometheus-alertmanager|Alert routing|
|110|openwebui|AI chat interface|
|111|code|VS Code in browser|
|112|docker|Media stack|

---

## API Token (for Homepage widget)

1. Datacenter → Permissions → API Tokens → Add
2. User: `root@pam`, Token ID: `homepage`
3. Uncheck Privilege Separation
4. Copy token secret — shown once only

---

## Related

- [[lvm-storage]]
- [[media-server]]
- [[smart-monitoring]]