# Tailscale Exit Node

## Overview

Tailscale VPN running as an exit node and subnet router inside a Proxmox LXC. Allows remote access to the home network and routes traffic through the home connection from anywhere.

## Prerequisites

- Proxmox VE
- Tailscale account (tailscale.com)

---

## Step 1 — Create the LXC

1. Open Proxmox web UI
2. Navigate to your storage (usually `local`) → **CT Templates** → **Templates**
3. Sort by package, download **Debian 12**
4. Click **Create CT** and configure:
    - **Hostname:** `tailscale`
    - **ID:** your choice
    - **Password:** set a root password
    - **Template:** Debian 12
    - **Cores:** 4
    - **Memory:** 8192 MB
    - **IP:** DHCP
5. Click **Finish** — do not start yet

---

## Step 2 — Edit LXC Config

On the Proxmox host shell, edit the LXC config before starting:

```bash
nano /etc/pve/lxc/[ID].conf
```

Add these lines at the bottom:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Then start the LXC:

```bash
pct start [ID]
```

---

## Step 3 — Install Tailscale

Open the LXC shell and run:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Confirm it appears in your Tailscale dashboard at tailscale.com/admin.

---

## Step 4 — Enable IP Forwarding

```bash
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

---

## Step 5 — Find Your Home Subnet

```bash
ip route | grep default
```

Example output:

```
default via 192.168.68.1 dev eth0
```

If your IP is `192.168.68.x` then your subnet is `192.168.68.0/24`.

---

## Step 6 — Advertise as Exit Node

```bash
tailscale up --advertise-exit-node --advertise-routes=192.168.68.0/24 --reset
sysctl -w net.ipv4.ip_forward=1
```

Then go to **Tailscale Admin Console → Machines** → find your node → **Edit route settings** → approve the exit node and subnet route.

---

## Troubleshooting

### UDP GRO Forwarding Warning

```
UDP GRO forwarding is suboptimally configured on eth0
```

Fix:

```bash
apt update && apt install ethtool -y
ethtool -K eth0 rx-udp-gro-forwarding on rx-gro on
```

---

## Related

- [[Proxmox LXC Setup]]
- [[Networking Overview]]