# Nginx Proxy Manager

## Overview
Nginx Proxy Manager provides reverse proxy and SSL termination for publicly
accessible services. Runs natively (no Docker) in a dedicated LXC.

## Infrastructure
| Component | Detail |
|---|---|
| **LXC ID** | 115 |
| **IP** | 192.168.68.115 |
| **Admin UI** | http://192.168.68.115:81 |
| **OS** | Debian 12 |
| **Install method** | community-scripts helper |

## Architecture
```
Internet (HTTPS)
    ↓
Cloudflare (SSL terminated here)
    ↓ HTTP
cloudflared service (inside NPM LXC)
    ↓ HTTP to localhost:80
Nginx Proxy Manager
    ↓ HTTP
Backend service (e.g. Jellyfin at 192.168.68.106:8096)
```

## Key Decisions
- NPM runs natively, not in Docker — avoids iptables conflicts that Docker
  creates which block connections to the LXC's own IP
- Cloudflare terminates SSL — NPM does NOT use Force SSL since traffic
  arrives from cloudflared already decrypted
- Cloudflare tunnel uses `localhost:80` not the LXC IP — LXCs cannot
  connect to their own external IP from within themselves

## Cloudflare Tunnel
- Tunnel name: `homelab`
- Cloudflared installed as systemd service
- Token stored in systemd service file at `/etc/systemd/system/cloudflared.service`
- Tunnel points to `localhost:80` (NPM)
```bash
# Check tunnel status
systemctl status cloudflared

# Restart tunnel
systemctl restart cloudflared
```

## Adding a New Service

### Step 1 — Cloudflare Tunnel
1. Zero Trust → Tunnels → homelab → Configure → Public Hostnames → Add
2. Subdomain: `servicename`
3. Domain: `nicolaimatthew.com`
4. Type: `HTTP`
5. URL: `localhost:80`

### Step 2 — NPM Proxy Host
1. NPM → Hosts → Proxy Hosts → Add Proxy Host
2. Domain: `servicename.nicolaimatthew.com`
3. Scheme: `http`
4. Forward Hostname: `SERVICE_IP`
5. Forward Port: `SERVICE_PORT`
6. Enable Websockets if needed
7. SSL tab — select existing wildcard cert (do NOT enable Force SSL)

## Current Services

| Subdomain | Internal Target | Notes |
|---|---|---|
| jellyfin.nicolaimatthew.com | 192.168.68.106:8096 | Media server |

## Security
- Cloudflare Bot Fight Mode: enabled
- Force SSL in NPM: DISABLED (Cloudflare handles SSL)
- Cloudflare Access: pending setup

## Gotchas
- **Do NOT enable Force SSL in NPM** — causes redirect loop since
  Cloudflare already handles HTTPS
- **Use localhost:80 in tunnel config** — not the LXC IP, LXCs can't
  self-connect via external IP
- **Docker breaks iptables** — never install Docker in this LXC, it
  creates firewall rules that block nginx connections
- **NPM admin port 81** — never expose this through the tunnel