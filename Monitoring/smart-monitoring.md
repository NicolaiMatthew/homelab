# SMART Drive Monitoring

---

## port: 9100 description: Drive health monitoring via node_exporter textfile collector status: running

## Overview

Custom SMART monitoring using smartmontools and a bash script that writes metrics to a textfile picked up by node_exporter. Metrics are scraped by Prometheus and trigger alerts via Alertmanager.

## How It Works

```
smartmon.sh (cron, every 1min)
    ↓ writes custom_smart_* metrics
/var/lib/prometheus/node-exporter/smartmon.prom
    ↓ read by
node_exporter (textfile collector)
    ↓ scraped by
Prometheus
    ↓ evaluated against
Alert Rules → Alertmanager → notifications
```

---

## Script (`/usr/local/bin/smartmon.sh`)

```bash
#!/bin/bash
OUTPUT_FILE="/var/lib/prometheus/node-exporter/smartmon.prom"
TEMP_FILE="/var/lib/prometheus/node-exporter/smartmon.prom.tmp"
mkdir -p /var/lib/prometheus/node-exporter

DISKS=$(lsblk -d -o NAME,TYPE | awk '$2=="disk" {print "/dev/"$1}')

{
  for disk in $DISKS; do
    name=$(basename "$disk")

    # Run health check twice to avoid false positives
    health=0
    for i in 1 2; do
      health_output=$(smartctl -H "$disk" 2>/dev/null)
      if echo "$health_output" | grep -q "PASSED"; then
        health=1
        break
      fi
      sleep 2
    done
    echo "custom_smart_health{disk=\"${name}\"} ${health}"

    smartctl -A "$disk" 2>/dev/null | awk -v disk="${name}" '
      /Reallocated_Sector_Ct/  {print "custom_smart_reallocated_sectors{disk=\""disk"\"} "$10}
      /Pending_Sector_Count/   {print "custom_smart_pending_sectors{disk=\""disk"\"} "$10}
      /Uncorrectable_Sector/   {print "custom_smart_uncorrectable_sectors{disk=\""disk"\"} "$10}
      /Temperature_Celsius/    {print "custom_smart_temperature{disk=\""disk"\"} "$10}
      /Power_On_Hours/         {print "custom_smart_power_on_hours{disk=\""disk"\"} "$10}
    '
  done
} > "$TEMP_FILE"

mv "$TEMP_FILE" "$OUTPUT_FILE"
```

---

## node_exporter Config (`/etc/default/prometheus-node-exporter`)

```
ARGS="--collector.textfile.directory=/var/lib/prometheus/node-exporter --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run|snap|boot/efi)($|/)"
```

---

## Crontab

```
* * * * * /usr/local/bin/smartmon.sh
```

---

## Metrics Exposed

|Metric|Description|
|---|---|
|`custom_smart_health`|1 = PASSED, 0 = FAILED|
|`custom_smart_reallocated_sectors`|Bad sectors remapped|
|`custom_smart_pending_sectors`|Sectors awaiting reallocation|
|`custom_smart_uncorrectable_sectors`|Unrecoverable errors|
|`custom_smart_temperature`|Drive temp in °C|
|`custom_smart_power_on_hours`|Total drive runtime|

---

## Current Drive Status

|Drive|Role|Health|Temp|Hours|
|---|---|---|---|---|
|sda|Proxmox boot|✅|~40°C|~1,300hrs|
|sdb|DAS drive 1|✅|~30°C|~12,700hrs|
|sdc|DAS drive 2|✅|~31°C|~13,100hrs|

> ⚠️ sdb and sdc are used drives with ~1.5 years of runtime. Monitor closely.

---

## Troubleshooting

### Metrics showing 0 for all drives

Check for built-in smartmon collector conflict:

```bash
curl http://localhost:9100/metrics | grep smartmon | head -5
```

If built-in metrics appear alongside custom ones, they may conflict. The script uses `custom_smart_` prefix to avoid this.

### Script not generating metrics

```bash
# Run manually and check output
/usr/local/bin/smartmon.sh
cat /var/lib/prometheus/node-exporter/smartmon.prom

# Check node_exporter is reading it
curl http://localhost:9100/metrics | grep custom_smart
```

---

## Related

- [[prometheus-stack]]
- [[lvm-storage]]
- [[grafana]]