# # 🏠 Homelab

Personal homelab running on a Lenovo ThinkCentre M920q with Proxmox VE. This repo documents every service, configuration, and design decision.

---

## 🖥️ Infrastructure

|Component|Detail|
|---|---|
|**Host**|Lenovo ThinkCentre M920q|
|**CPU**|Intel 8th/9th Gen (QuickSync)|
|**Hypervisor**|Proxmox VE|
|**Storage**|Terramaster D4-320 DAS — 2x4TB → 7.3TB LVM|

---

## 🗺️ Service Map

```
Proxmox VE
├── 🌐 Networking
│   └── Tailscale          → VPN exit node + subnet router
│
├── 📺 Media
│   └── Docker LXC
│       ├── Gluetun        → Mullvad VPN tunnel
│       ├── qBittorrent    → Download client (behind VPN)
│       ├── Prowlarr       → Indexer aggregator
│       ├── FlareSolverr   → Cloudflare bypass
│       ├── Sonarr         → TV automation
│       ├── Radarr         → Movie automation
│       ├── Bazarr         → Subtitle automation
│       ├── Jellyfin       → Media server
│       ├── Jellyseerr     → Request portal
│       ├── Scraparr       → ARR metrics exporter
│       └── Profilarr      → Quality profile manager
│
├── 📊 Monitoring
│   ├── Prometheus         → Metrics collection
│   ├── PVE Exporter       → Proxmox metrics
│   ├── Node Exporter      → Host hardware + SMART
│   ├── Alertmanager       → Alert routing
│   └── Grafana            → Dashboards
│
├── 🤖 AI & Automation
│   ├── OpenWebUI          → AI chat interface
│   └── n8n                → Workflow automation
│
├── 🛠️ Dev & Tools
│   ├── Code Server        → VS Code in browser
│   ├── Crafty Controller  → Minecraft server manager
│   └── File Share         → File sharing
│
└── 🎛️ Dashboard
    └── Homepage           → Unified service dashboard
```

---

## 🔗 Quick Access

|Service|URL|Docs|
|---|---|---|
|Proxmox|`https://PROXMOX-IP:8006`|[[proxmox-setup]]|
|Homepage|`http://HOMEPAGE-IP:3000`|[[homepage-dashboard]]|
|Jellyfin|`http://DOCKER-IP:8096`|[[media-server]]|
|Jellyseerr|`http://DOCKER-IP:5055`|[[media-server]]|
|Grafana|`http://GRAFANA-IP:3000`|[[grafana]]|
|Prometheus|`http://PROMETHEUS-IP:9090`|[[prometheus-stack]]|
|Sonarr|`http://DOCKER-IP:8989`|[[media-server]]|
|Radarr|`http://DOCKER-IP:7878`|[[media-server]]|
|qBittorrent|`http://DOCKER-IP:8080`|[[media-server]]|

---

## 📁 Documentation Structure

```
homelab/
├── Infrastructure/
│   ├── proxmox-setup.md      → Proxmox config, LXC best practices
│   └── lvm-storage.md        → DAS setup, LVM configuration
├── Media/
│   └── media-server.md       → Full ARR stack setup
├── Monitoring/
│   ├── prometheus-stack.md   → Prometheus, exporters, alert rules
│   ├── smart-monitoring.md   → Drive health monitoring
│   └── grafana.md            → Dashboards and queries
├── Networking/
│   └── tailscale.md          → VPN exit node setup
├── Dashboard/
│   └── homepage-dashboard.md → Homepage config
└── Templates/
    └── service-template.md   → Template for new service docs
```

---

## 🔔 Alerting

Prometheus alerts route through Alertmanager for:

- LXC/VM going offline
- Disk filling up (>85%)
- High memory usage (>90%)
- Drive SMART health failure
- Drive temperature warnings
- DAS storage filling up (>80%)

See [[prometheus-stack]] for full alert rules.

---

## 📝 Notes

- All LXCs are **unprivileged** — use UID `101000` for bind mount permissions on host
- qBittorrent routes through Gluetun — all torrent traffic exits via Mullvad
- Jellyfin uses Intel QuickSync hardware transcoding
- DAS drives are used (~12,700hrs) — SMART monitoring active

---

## 🗓️ Changelog

|Date|Change|
|---|---|
|2026-03|Initial homelab documentation|
|2026-03|Added DAS storage (Terramaster D4-320)|
|2026-03|Set up SMART monitoring with Prometheus alerts|
|2026-03|Added Obsidian + GitHub documentation workflow|