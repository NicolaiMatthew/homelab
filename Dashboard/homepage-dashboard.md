# Homepage Dashboard

---
## 📝 Writing

- [How I Built a Production-Grade Homelab on a $200 Mini PC](https://dev.to/nicolaimatthew/how-i-built-a-production-grade-homelab-on-a-200-mini-pc-2154)
## port: 3000 description: Unified homelab dashboard status: running

## Overview

Homepage running in its own dedicated LXC, serving as a unified dashboard for all homelab services. Shows live widgets for media stack, monitoring, and infrastructure services.

## Access

- URL: `http://HOMEPAGE-LXC-IP:3000`

---

## Installation

Via Proxmox Community Helper Scripts:

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/homepage.sh)"
```

---

## Config Files (`/opt/homepage/config/`)

|File|Purpose|
|---|---|
|`settings.yaml`|Title, theme, layout|
|`services.yaml`|Service tiles and widgets|
|`widgets.yaml`|Top bar widgets|
|`bookmarks.yaml`|Quick links|

Restart after any config change:

```bash
systemctl restart homepage
```

---

## settings.yaml

```yaml
title: My Homelab
theme: dark
color: slate
headerStyle: clean
providers:
  proxmox:
    url: https://PROXMOX-IP:8006
    allowInsecure: true
layout:
  Media:
    style: row
    columns: 4
  Monitoring:
    style: row
    columns: 3
  Infrastructure:
    style: row
    columns: 3
  Network:
    style: row
    columns: 2
```

---

## widgets.yaml

```yaml
- resources:
    cpu: true
    memory: true
    expanded: true
    disk:
      - /
      - /mnt/das

- search:
    provider: google
    target: _blank

- datetime:
    text_size: xl
    format:
      timeStyle: short
      dateStyle: short
```

---

## DAS Storage Visibility

The DAS must be bind mounted into the Homepage LXC:

Add to Homepage LXC config (`/etc/pve/lxc/HOMEPAGE-ID.conf`):

```
mp0: /mnt/das,mp=/mnt/das
```

---

## Icon Reference

Icons use the [dashboard-icons](https://github.com/walkxcode/dashboard-icons) library. For any missing icon use a direct URL:

```yaml
icon: https://raw.githubusercontent.com/walkxcode/dashboard-icons/main/png/SERVICE-NAME.png
```

---

## API Keys Required

|Service|Where to find|
|---|---|
|Sonarr|Settings → General → API Key|
|Radarr|Settings → General → API Key|
|Prowlarr|Settings → General → API Key|
|Bazarr|Settings → General → Security → API Key|
|Jellyfin|Dashboard → API Keys → +|
|Jellyseerr|Settings → General → API Key|
|Proxmox|Datacenter → Permissions → API Tokens (format: `root@pam!tokenid`)|

---

## Related

- [[proxmox-setup]]
- [[media-server]]
- [[prometheus-stack]]