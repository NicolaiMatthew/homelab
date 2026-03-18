# Media Server Stack

## Overview

A fully automated self-hosted media server running on Proxmox VE. Downloads are handled automatically via the ARR stack, routed through a Mullvad VPN, and served via Jellyfin with Intel QuickSync hardware transcoding.

## Stack

|Service|Purpose|Port|
|---|---|---|
|Gluetun|VPN tunnel (Mullvad WireGuard)|—|
|qBittorrent|Download client (behind VPN)|8080|
|Prowlarr|Indexer aggregator|9696|
|FlareSolverr|Cloudflare bypass proxy|8191|
|Sonarr|TV show automation|8989|
|Radarr|Movie automation|7878|
|Bazarr|Subtitle automation|6767|
|Jellyfin|Media server|8096|
|Jellyseerr|Media request portal|5055|
|Scraparr|ARR metrics exporter|7100|
|Profilarr|Quality profile manager|6868|

---

## Prerequisites

- Proxmox VE
- Mullvad VPN account (mullvad.net)
- Terramaster D4-320 DAS (or any USB storage)
- Intel CPU with QuickSync (for hardware transcoding)

---

## Storage Setup

### Create LVM Volume (2x 4TB → 7.3TB)

```bash
apt install lvm2 -y
pvcreate /dev/sdb /dev/sdc
vgcreate das-vg /dev/sdb /dev/sdc
lvcreate -l 100%FREE -n media das-vg
mkfs.ext4 /dev/das-vg/media
```

### Mount Persistently

```bash
mkdir -p /mnt/das
blkid /dev/das-vg/media   # copy UUID
```

Add to `/etc/fstab`:

```
UUID=your-uuid  /mnt/das  ext4  defaults,nofail  0  2
```

```bash
mount -a
```

### Fix Permissions

> ⚠️ This is an unprivileged LXC — use UID 101000 on the host, not 1000.

```bash
chown -R 101000:101000 /mnt/das
chmod -R 755 /mnt/das
```

---

## Proxmox LXC Setup

Use the [Proxmox Community Helper Scripts](https://github.com/community-scripts/ProxmoxVE) to create a Docker LXC:

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/docker.sh)"
```

### LXC Config (`/etc/pve/lxc/112.conf`)

```
arch: amd64
cores: 4
features: nesting=1,keyctl=1
hostname: docker
memory: 4096
mp0: /mnt/das,mp=/mnt/data
onboot: 1
ostype: debian
rootfs: local-lvm:vm-112-disk-0,size=16G
timezone: America/New_York
unprivileged: 1

# TUN device for VPN
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file

# Intel QuickSync GPU passthrough
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card1 dev/dri/card1 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

> **Note:** Replace `card1` with `card0` if that's what `ls /dev/dri` shows on your host.

### Make GPU Permissions Persistent

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

---

## Folder Structure

```bash
mkdir -p /mnt/data/media/{movies,tv}
mkdir -p /mnt/data/torrents/{movies,tv,incomplete}
mkdir -p /opt/mediastack/{gluetun,qbittorrent,sonarr,radarr,prowlarr,jellyfin,bazarr,jellyseerr,scraparr,profilarr}
```

```
/mnt/das/
└── data/                    ← bind mounted at /mnt/data in LXC
    ├── media/
    │   ├── movies/
    │   └── tv/
    └── torrents/
        ├── movies/
        ├── tv/
        └── incomplete/
```

> **Hardlinks:** Both torrents and media are on the same filesystem so Sonarr/Radarr use hardlinks instead of copying — instant, zero extra disk usage.

---

## Docker Compose

Full `docker-compose.yml` at `/opt/mediastack/docker-compose.yml`:

```yaml
services:

  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=mullvad
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=YOUR_PRIVATE_KEY
      - WIREGUARD_ADDRESSES=10.x.x.x/32
      - WIREGUARD_MTU=1280
      - FIREWALL_OUTBOUND_SUBNETS=192.168.x.x/xx
      - HEALTH_SERVER_ADDRESS=127.0.0.1:9999
      - UPDATER_PERIOD=0
    ports:
      - 8080:8080
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - WEBUI_PORT=8080
    volumes:
      - /opt/mediastack/qbittorrent:/config
      - /mnt/data/torrents:/data/torrents
    depends_on:
      - gluetun
    restart: unless-stopped

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
    ports:
      - 8191:8191
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediastack/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediastack/sonarr:/config
      - /mnt/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediastack/radarr:/config
      - /mnt/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediastack/bazarr:/config
      - /mnt/data:/data
    ports:
      - 6767:6767
    restart: unless-stopped

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card1:/dev/dri/card1
    volumes:
      - /opt/mediastack/jellyfin:/config
      - /mnt/data/media:/data/media
    ports:
      - 8096:8096
    restart: unless-stopped

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - /opt/mediastack/jellyseerr:/app/config
    ports:
      - 5055:5055
    restart: unless-stopped

  scraparr:
    image: thegameprofi/scraparr:latest
    container_name: scraparr
    volumes:
      - /opt/mediastack/scraparr/config.yaml:/scraparr/config/config.yaml
    ports:
      - 7100:7100
    restart: unless-stopped

  profilarr:
    image: santiagosayshey/profilarr:latest
    container_name: profilarr
    ports:
      - 6868:6868
    volumes:
      - /opt/mediastack/profilarr:/config
    environment:
      - TZ=America/New_York
    restart: unless-stopped
```

---

## VPN Setup (Mullvad WireGuard)

1. Log into mullvad.net → Account → WireGuard configuration
2. Generate a key → copy the **Private Key**
3. Note the **Address** field for `WIREGUARD_ADDRESSES`
4. Paste both into `docker-compose.yml`

**Verify VPN is working:**

```bash
docker exec qbittorrent curl -s ifconfig.me
# Should return a Mullvad IP, not your home IP
```

---

## Service Configuration

### qBittorrent

- Default Save Path: `/data/torrents`
- Incomplete torrents: `/data/torrents/incomplete`
- Default login: check logs → `docker logs qbittorrent 2>&1 | grep -i password`

### Prowlarr

- Add indexers (YTS, EZTV, 1337x, TPB, etc.)
- FlareSolverr proxy host: `http://flaresolverr:8191/`
- Connect to Sonarr and Radarr via Settings → Apps

### Sonarr & Radarr

- Enable **Use Hardlinks instead of Copy** ✅
- Root folders: `/data/media/tv` and `/data/media/movies`
- Download client: qBittorrent at `localhost:8080`
- Categories: `tv` and `movies`

> **Anime:** Set Series Type to `Anime` in Sonarr, otherwise searches miss most results.

### Bazarr

- Connect to Sonarr and Radarr with API keys
- Add providers: OpenSubtitles, Subscene, etc.
- Use SRT format — avoids transcoding in Jellyfin

### Jellyseerr

- Sign in with Jellyfin credentials
- Connect Radarr and Sonarr
- Share URL with users — they request via Jellyfin login

---

## Intel QuickSync Hardware Transcoding

### Enable in Jellyfin

Dashboard → Playback → Transcoding:

- Hardware Acceleration: **Intel QuickSync (QSV)**
- Enable all codecs: H.264, HEVC, MPEG2, VP9, AV1 ✅
- Enable Hardware Encoding ✅

### Verify

While playing → Dashboard → Active Streams → look for `(hw)` next to codec.

### Monitor GPU

```bash
apt install intel-gpu-tools -y
intel_gpu_top
```

---

## SMART Drive Monitoring

### Script (`/usr/local/bin/smartmon.sh`)

Auto-detects all drives and writes metrics for Prometheus:

```bash
#!/bin/bash
OUTPUT_FILE="/var/lib/prometheus/node-exporter/smartmon.prom"
TEMP_FILE="/var/lib/prometheus/node-exporter/smartmon.prom.tmp"
mkdir -p /var/lib/prometheus/node-exporter

DISKS=$(lsblk -d -o NAME,TYPE | awk '$2=="disk" {print "/dev/"$1}')

{
  for disk in $DISKS; do
    name=$(basename "$disk")
    health_output=$(smartctl -H "$disk" 2>/dev/null)
    if echo "$health_output" | grep -q "PASSED"; then
      health=1
    else
      health=0
    fi
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

### Crontab (runs every 5 minutes)

```
*/5 * * * * /usr/local/bin/smartmon.sh
```

### node_exporter Config (`/etc/default/prometheus-node-exporter`)

```
ARGS="--collector.textfile.directory=/var/lib/prometheus/node-exporter --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run|snap|boot/efi)($|/)"
```

---

## Troubleshooting

### TUN device not found

Add to LXC config, or enable via Proxmox UI → LXC → Options → Features → TUN device ✅

### Permission denied on /mnt/data

Always fix from Proxmox host using UID 101000 (unprivileged LXC offset):

```bash
chown -R 101000:101000 /mnt/das
chmod -R 755 /mnt/das
```

### Gluetun restart loop

```yaml
- HEALTH_SERVER_ADDRESS=127.0.0.1:9999
- UPDATER_PERIOD=0
- WIREGUARD_MTU=1280
```

### qBittorrent erroring after DAS mount

Right-click errored torrents → Set Location → `/data/torrents/tv` or `/data/torrents/movies` → Force Recheck → Resume

### Sonarr not finding content

1. Prowlarr → Settings → Apps → force sync
2. Verify indexers appear in Sonarr → Settings → Indexers
3. For anime set Series Type to `Anime`
4. Use Interactive Search to see raw results

---

## Related

- [[Proxmox LXC Setup]]
- [[Homepage Dashboard]]
- [[Monitoring Stack]]