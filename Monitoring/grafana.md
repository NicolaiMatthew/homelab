# Grafana

---

## port: 3000 description: Metrics visualization and dashboards status: running

## Overview

Grafana running in its own LXC, connected to Prometheus as a datasource. Displays dashboards for Proxmox host metrics, ARR stack stats, and drive health.

## Access

- URL: `http://GRAFANA-LXC-IP:3000`

---

## Datasources

|Name|Type|URL|
|---|---|---|
|Prometheus|Prometheus|`http://PROMETHEUS-IP:9090`|

---

## Imported Dashboards

|Dashboard|ID|Source|
|---|---|---|
|Node Exporter Full|1860|Host CPU, RAM, disk, SMART|
|Scraparr ARR Metrics|22934|Sonarr, Radarr, Prowlarr stats|
|Proxmox via PVE Exporter|10347|LXC/VM resource usage|

### Import a Dashboard

1. Dashboards → Import
2. Enter dashboard ID → Load
3. Select Prometheus datasource → Import

---

## Custom SMART Panels

Useful queries for custom panels:

```promql
# Drive temperatures
custom_smart_temperature

# Drive health (1=healthy, 0=failed)
custom_smart_health

# Power on hours
custom_smart_power_on_hours

# DAS usage percentage
(node_filesystem_size_bytes{mountpoint="/mnt/das"} - node_filesystem_free_bytes{mountpoint="/mnt/das"}) / node_filesystem_size_bytes{mountpoint="/mnt/das"} * 100
```

---

## Related

- [[prometheus-stack]]
- [[smart-monitoring]]