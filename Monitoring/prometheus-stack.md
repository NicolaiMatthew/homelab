# Prometheus Monitoring Stack

---

## port: 9090 description: Metrics collection and alerting status: running

## Overview

Full Prometheus monitoring stack with PVE exporter for Proxmox metrics, node_exporter for host hardware metrics, Alertmanager for alert routing, and Grafana for visualization.

## Components

|Service|LXC|Port|Purpose|
|---|---|---|---|
|Prometheus|106|9090|Metrics collection and alert evaluation|
|PVE Exporter|107|9221|Proxmox VE metrics|
|Alertmanager|109|9093|Alert routing and notifications|
|Node Exporter|Proxmox host|9100|Host hardware metrics + SMART data|
|Scraparr|Docker LXC|7100|ARR stack metrics|

---

## Prometheus Config (`/etc/prometheus/prometheus.yml`)

```yaml
scrape_configs:
  - job_name: 'proxmox-pve'
    static_configs:
      - targets: ['PVE-EXPORTER-IP:9221']

  - job_name: 'proxmox-node'
    static_configs:
      - targets: ['PROXMOX-HOST-IP:9100']

  - job_name: 'scraparr'
    static_configs:
      - targets: ['DOCKER-LXC-IP:7100']

rule_files:
  - /etc/prometheus/alert_rules.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['ALERTMANAGER-IP:9093']
```

---

## Node Exporter Setup (Proxmox Host)

```bash
apt install smartmontools prometheus-node-exporter -y
```

Config (`/etc/default/prometheus-node-exporter`):

```
ARGS="--collector.textfile.directory=/var/lib/prometheus/node-exporter --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run|snap|boot/efi)($|/)"
```

Crontab for SMART metrics:

```
* * * * * /usr/local/bin/smartmon.sh
```

---

## Alert Rules (`/etc/prometheus/alert_rules.yml`)

### ProxmoxEnvironment Alerts

- **VirtualGuestDown** — LXC/VM offline for 1m
- **ProxmoxDiskFilling** — disk >85% full for 10m
- **ProxmoxGuestHighMemory** — RAM >90% for 5m

### SMART Alerts

- **DriveHealthFailed** — SMART health == 0 for 20m
- **DriveReallocatedSectors** — any reallocated sectors for 5m
- **DrivePendingSectors** — any pending sectors for 5m
- **DriveUncorrectableSectors** — any uncorrectable sectors for 5m
- **DriveTemperatureHigh** — temp >55°C for 10m
- **DriveTemperatureCritical** — temp >65°C for 5m
- **DASStorageFillingUp** — DAS >80% full for 5m
- **DASStorageCritical** — DAS >95% full for 5m
- **DASInodesFilling** — inodes <10% remaining for 5m

---

## Grafana Dashboard IDs

|Dashboard|ID|
|---|---|
|Node Exporter Full|1860|
|Scraparr ARR Metrics|22934|
|Proxmox via PVE Exporter|10347|

---

## Related

- [[grafana]]
- [[smart-monitoring]]
- [[proxmox-setup]]