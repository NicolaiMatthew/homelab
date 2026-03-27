# 🏠 Homelab Infrastructure

> A production-grade self-hosted homelab built on Proxmox VE, featuring containerized services, automated media pipelines, comprehensive monitoring, and an offsite backup strategy.

![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=flat&logo=proxmox&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-A81D33?style=flat&logo=debian&logoColor=white)

---
## 📝 Writing

- [How I Built a Production-Grade Homelab on a $200 Mini PC](https://dev.to/nicolaimatthew/how-i-built-a-production-grade-homelab-on-a-200-mini-pc-2154)


## 📋 Overview

This homelab runs on a **Lenovo ThinkCentre M920q** and serves as a practical environment for developing and demonstrating skills in Linux administration, containerization, infrastructure monitoring, and DevOps practices.

Every decision in this build was made deliberately — from choosing unprivileged LXC containers for security isolation, to implementing hardware-accelerated transcoding via Intel QuickSync passthrough, to building a custom SMART monitoring pipeline that caught a real conflict between a systemd timer and a cron-based textfile collector.

---

## 🖥️ Hardware

| Component | Spec |
|---|---|
| **Host** | Lenovo ThinkCentre M920q (SFF) |
| **CPU** | Intel Core i5/i7 8th/9th Gen (QuickSync capable) |
| **Hypervisor** | Proxmox VE 9.x |
| **Primary Storage** | Local SSD (Proxmox OS + LXC rootfs) |
| **Media Storage** | Terramaster D4-320 DAS - 2x4TB HDD → 7.3TB LVM volume |

---

## 🗺️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Proxmox VE Host                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Networking  │  │   Media      │  │   Monitoring     │  │
│  │              │  │   Stack      │  │   Stack          │  │
│  │  Tailscale   │  │  Docker LXC  │  │  Prometheus      │  │
│  │  Exit Node   │  │  (LXC 112)   │  │  Grafana         │  │
│  │  + Subnet    │  │              │  │  Alertmanager    │  │
│  │  Router      │  │  Gluetun ──► │  │  PVE Exporter    │  │
│  └──────────────┘  │  qBittorrent │  │  Node Exporter   │  │
│                    │  Prowlarr    │  └──────────────────┘  │
│  ┌──────────────┐  │  Sonarr      │                        │
│  │  AI & Tools  │  │  Radarr      │  ┌──────────────────┐  │
│  │              │  │  Bazarr      │  │   Dashboard      │  │
│  │  OpenWebUI   │  │  Jellyfin    │  │                  │  │
│  │  n8n         │  │  Jellyseerr  │  │  Homepage        │  │
│  │  Code Server │  │  Scraparr    │  │  (LXC 113)       │  │
│  └──────────────┘  │  Profilarr   │  └──────────────────┘  │
│                    └──────────────┘                        │
│                           │                                │
│              ┌────────────▼────────────┐                   │
│              │   DAS — 7.3TB LVM       │                   │
│              │   /mnt/das              │                   │
│              │   Bind mounted →        │                   │
│              │   /mnt/data in LXC      │                   │
│              └─────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Technical Highlights

### Container Strategy
All services run in **unprivileged LXC containers** on Proxmox. This was a deliberate security decision — unprivileged containers remap UIDs so even a container escape would land in an unprivileged user context on the host. The tradeoff is bind mount complexity: UID 1000 inside maps to UID 101000 on the host, requiring careful `chown` management for shared storage.

### VPN Kill Switch
The entire torrent pipeline routes through a Gluetun WireGuard container. qBittorrent uses `network_mode: service:gluetun` in Docker Compose, meaning it physically cannot route traffic except through the VPN tunnel. If Gluetun stops, qBittorrent loses network access entirely — no configuration required, it's enforced at the network namespace level.

### Hardware Transcoding
Intel QuickSync is passed through to the Docker LXC via `lxc.mount.entry` directives in the Proxmox container config. Jellyfin uses QSV hardware decoding/encoding for H.264, HEVC, VP9 and AV1, offloading transcoding from the CPU entirely.

### Custom SMART Monitoring
Built a bash-based SMART metrics exporter that writes to Prometheus node_exporter's textfile collector. During development discovered a conflict: the built-in `prometheus-node-exporter-smartmon.service` systemd timer runs every 15 minutes and overwrites the textfile with zeros — causing false alerts. Fixed by using a `custom_smart_` metric prefix and disabling the conflicting timer. Alert rules fire after 20 minutes of sustained failure to avoid transient scrape issues.

### Storage Architecture
Two 4TB HDDs combined into a single 7.3TB LVM logical volume. Mounted on the Proxmox host at `/mnt/das` and bind mounted into the Docker LXC at `/mnt/data`. Sonarr/Radarr download and media folders are on the same filesystem, enabling **hardlinks** instead of file copies — imported media is instantaneous with zero additional disk usage.

---

## 📦 Full Service Inventory

| Service | Purpose | Location |
|---|---|---|
| **Tailscale** | VPN exit node + subnet router | LXC 100 |
| **File Share** | Network file sharing | LXC 101 |
| **n8n** | Workflow automation | LXC 102 |
| **Upsnap** | Wake-on-LAN | LXC 103 |
| **Crafty Controller** | Minecraft server management | LXC 105 |
| **Prometheus** | Metrics collection + alerting | LXC 106 |
| **PVE Exporter** | Proxmox-specific metrics | LXC 107 |
| **Grafana** | Metrics visualization | LXC 108 |
| **Alertmanager** | Alert routing + notifications | LXC 109 |
| **OpenWebUI** | Self-hosted AI interface | LXC 110 |
| **Code Server** | Browser-based VS Code | LXC 111 |
| **Gluetun** | WireGuard VPN tunnel (Mullvad) | Docker LXC 112 |
| **qBittorrent** | Torrent client (VPN-only network namespace) | Docker LXC 112 |
| **Prowlarr** | Indexer aggregation | Docker LXC 112 |
| **FlareSolverr** | Cloudflare challenge bypass | Docker LXC 112 |
| **Sonarr** | TV show automation | Docker LXC 112 |
| **Radarr** | Movie automation | Docker LXC 112 |
| **Bazarr** | Subtitle automation | Docker LXC 112 |
| **Jellyfin** | Media server (QuickSync HW transcode) | Docker LXC 112 |
| **Jellyseerr** | Media request portal | Docker LXC 112 |
| **Scraparr** | ARR stack Prometheus exporter | Docker LXC 112 |
| **Profilarr** | Quality profile management | Docker LXC 112 |
| **Navidrome** | Self-hosted music streaming | Docker LXC 112 |
| **Audiobookshelf** | Audiobook + podcast server | Docker LXC 112 |
| **Stirling PDF** | Self-hosted PDF processing | Docker LXC 112 |
| **IT Tools** | Developer utility toolkit | Docker LXC 112 |
| **Homepage** | Unified service dashboard | LXC 113 |

---

## 📊 Monitoring Stack

Prometheus scrapes metrics from four sources:

```
Proxmox Host
├── node_exporter (port 9100)
│   ├── CPU, RAM, disk, network
│   ├── Filesystem metrics (/mnt/das)
│   └── Custom textfile collector
│       ├── custom_smart_health
│       ├── custom_smart_temperature
│       ├── custom_smart_reallocated_sectors
│       ├── custom_smart_power_on_hours
│       └── r2_bucket_size_bytes (offsite backup monitoring)
│
PVE Exporter (LXC 107)
└── Per-LXC CPU, RAM, disk usage
    └── VM/LXC up/down state

Scraparr (Docker LXC)
└── Sonarr, Radarr, Prowlarr, Bazarr, Jellyseerr metrics
```

### Alert Rules

| Alert | Condition | Severity |
|---|---|---|
| VirtualGuestDown | LXC/VM offline >1m | Critical |
| ProxmoxDiskFilling | Disk >85% | Warning |
| ProxmoxGuestHighMemory | RAM >90% | Warning |
| DriveHealthFailed | SMART health=0 for 20m | Critical |
| DriveReallocatedSectors | Any reallocated sectors | Warning |
| DriveTemperatureHigh | Temp >55°C for 10m | Warning |
| DASStorageFillingUp | DAS >80% | Warning |
| DASStorageCritical | DAS >95% | Critical |
| R2BackupSizeWarning | R2 bucket >8GB | Warning |
| R2BackupSizeCritical | R2 bucket >9GB | Critical |

---

## 💾 Backup Strategy

Three-tier backup approach:

```
Tier 1 — Daily (Local)
├── /opt/mediastack configs → DAS tarball
└── /etc/pve/lxc/*.conf → DAS

Tier 2 — Weekly (Local Snapshots)
└── All LXCs/VMs → Proxmox VZDump → DAS

Tier 3 — Weekly (Offsite)
└── rclone sync → Cloudflare R2 (free tier)
    ├── LXC configs
    ├── Docker Compose + service configs
    ├── Prometheus alert rules
    ├── Homepage config
    └── Custom scripts
```

Restore time estimate: ~1 hour from complete hardware failure using DAS snapshots + R2 config backup + this documentation.

---

## 🔑 Key Engineering Decisions & Lessons Learned

**Unprivileged LXCs over VMs** — Lower overhead than full VMs, better security than privileged containers. The UID remapping complexity (101000 offset) is worth the security isolation.

**Docker Compose over individual containers** — Managing 15+ containers as a single stack with shared networking makes the VPN kill switch trivial to implement and the whole stack easy to update atomically.

**Hardlinks over copies** — Keeping download and media directories on the same filesystem enables Sonarr/Radarr to use hardlinks. A 30GB season pack is "moved" instantaneously with zero extra disk usage.

**Custom metric prefix to avoid collector conflicts** — The built-in `smartmon` collector in prometheus-node-exporter conflicted with custom textfile metrics, causing false critical alerts every 15 minutes. Using a `custom_smart_` prefix completely eliminated the conflict without disabling any system services.

**Atomic file writes for Prometheus textfile** — Writing metrics to a temp file and `mv`-ing atomically prevents Prometheus from scraping a partially-written file and generating false alerts during the write window.

**PATH in cron** — Cron runs with a minimal environment. Discovered this when smartctl couldn't be found during cron execution despite working fine interactively. Fixed by explicitly exporting `PATH` at the top of all cron-executed scripts.

---

## 📁 Repository Structure

```
homelab/
├── Infrastructure/
│   ├── proxmox-setup.md      → LXC config, GPU passthrough, UID mapping
│   └── lvm-storage.md        → DAS setup, LVM provisioning, bind mounts
├── Media/
│   └── media-server.md       → Full ARR stack, VPN kill switch, hardlinks
├── Monitoring/
│   ├── prometheus-stack.md   → Scrape configs, exporters, alert rules
│   ├── smart-monitoring.md   → Custom SMART exporter, timer conflict fix
│   └── grafana.md            → Dashboard queries, R2 monitoring panels
├── Networking/
│   └── tailscale.md          → Exit node setup, subnet routing
├── Dashboard/
│   └── homepage-dashboard.md → Homepage YAML config, widget setup
└── Templates/
    └── service-template.md   → Standard template for new service docs
```

---

## 🗓️ Changelog

| Date | Change |
|---|---|
| 2026-03 | Initial build — Proxmox, Docker stack, ARR media pipeline |
| 2026-03 | DAS storage — Terramaster D4-320, LVM, LXC bind mount |
| 2026-03 | Monitoring stack — Prometheus, Grafana, Alertmanager, PVE Exporter |
| 2026-03 | Custom SMART monitoring — textfile collector, systemd timer conflict resolution |
| 2026-03 | Offsite backup — rclone + Cloudflare R2, R2 size monitoring in Grafana |
| 2026-03 | Documentation workflow — Obsidian vault synced to GitHub |
| 2026-03 | Expanded stack — Navidrome, Audiobookshelf, Stirling PDF, IT Tools |
