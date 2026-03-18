# LVM Storage (DAS)

---

## port: ~ description: Terramaster D4-320 DAS — 2x4TB combined via LVM status: mounted

## Overview

Two 4TB HDDs in a Terramaster D4-320 USB 3.2 Gen2 DAS enclosure, combined into a single 7.3TB LVM volume mounted at `/mnt/das` on the Proxmox host.

## Hardware

|Component|Detail|
|---|---|
|Enclosure|Terramaster D4-320|
|Connection|USB 3.2 Gen2 (10Gbps)|
|Drives|2x 4TB HDD|
|Usable Space|~7.3TB (LVM)|
|Mount Point|`/mnt/das`|

---

## Setup

### 1. Verify Drives Detected

```bash
fdisk -l | grep -E "Disk /dev/sd|Disk /dev/nvme"
```

### 2. Create LVM Volume

```bash
apt install lvm2 -y
pvcreate /dev/sdb /dev/sdc
vgcreate das-vg /dev/sdb /dev/sdc
lvcreate -l 100%FREE -n media das-vg
mkfs.ext4 /dev/das-vg/media
```

### 3. Mount Persistently

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
df -h | grep das
```

### 4. Fix Permissions (Unprivileged LXC)

```bash
chown -R 101000:101000 /mnt/das
chmod -R 755 /mnt/das
```

### 5. Bind Mount into Docker LXC

Add to `/etc/pve/lxc/112.conf`:

```
mp0: /mnt/das,mp=/mnt/data
```

---

## Folder Structure

```
/mnt/das/
├── media/
│   ├── movies/
│   └── tv/
└── torrents/
    ├── movies/
    ├── tv/
    └── incomplete/
```

---

## LVM Commands Reference

```bash
pvs          # show physical volumes
vgs          # show volume groups
lvs          # show logical volumes
df -h        # check mounted space
```

---

## Related

- [[proxmox-setup]]
- [[media-server]]
- [[smart-monitoring]]