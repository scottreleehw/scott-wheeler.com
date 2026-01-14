---
title: "Building and Organizing My Homelab: A Complete Guide"
description: "Technical deep-dive into my Proxmox homelab: LXC containers, Caddy reverse proxy, Pi-hole DNS, automated backups, and network architecture."
date: 2026-01-12
tags: ["homelab", "self-hosted", "proxmox", "networking", "linux"]
---

After months of random IP assignments and "I'll document it later," I spent a day reorganizing my entire homelab. This is the technical breakdown.

## Hardware

| Component | Spec |
|-----------|------|
| Server | Lenovo ThinkCentre Mini PC |
| CPU | Multi-core (low TDP) |
| RAM | 16GB (7.8GB allocated to containers) |
| System Drive | NVMe SSD, 66GB |
| Container Storage | SSD, 233GB LVM-thin |
| Backup Drive | 2TB USB 3.0 External |
| Router | Netgear Nighthawk RAX45 |
| Switch | Ubiquiti EdgeRouter X (dumb switch mode) |

**Hypervisor:** Proxmox VE 8.2.7, Kernel 6.8.12-2-pve

## Container/VM Inventory

| ID | Name | IP | Type | CPU | RAM | Disk | Purpose |
|----|------|-----|------|-----|-----|------|---------|
| 100 | pi-hole | 10.0.0.3 | LXC | 2 | 512MB | 8GB | DNS/ad-blocking |
| 101 | lubelogger | 10.0.0.55 | LXC | 1 | 512MB | 32GB | Vehicle maintenance |
| 102 | caddy | 10.0.0.52 | LXC | 1 | 512MB | 8GB | Reverse proxy |
| 105 | bookstack | 10.0.0.56 | LXC | 1 | 1GB | 4GB | Documentation wiki |
| 107 | jellyfin | 10.0.0.54 | LXC | 2 | 2GB | 75GB | Media server |

## Network Architecture

```
10.0.0.0/24

Infrastructure (1-49):
  .2   - Gateway (Nighthawk)
  .3   - Pi-hole DNS

Servers (50-99):
  .50  - Proxmox host
  .52  - Caddy
  .54  - Jellyfin
  .55  - Lubelogger
  .56  - Bookstack

Static Devices (100-149):
  Reserved

DHCP (150-254):
  Dynamic pool
```

### IP Migration

Migrating containers is straightforward:

```bash
# Stop container
pct stop 105

# Update network config
pct set 105 --net0 name=eth0,bridge=vmbr0,ip=10.0.0.56/24,gw=10.0.0.2

# Start container
pct start 105
```

For Proxmox host, edit `/etc/network/interfaces`:

```
auto vmbr0
iface vmbr0 inet static
    address 10.0.0.50/24
    gateway 10.0.0.2
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

Apply with `ifreload -a`. ~5 seconds downtime.

## Caddy Reverse Proxy

Using Caddy's internal CA for HTTPS on local network - no Let's Encrypt needed since services aren't internet-exposed.

### Caddyfile

```
{
    email scott@scott-wheeler.com
    local_certs
}

proxmox.scott-wheeler.com {
    reverse_proxy 10.0.0.50:8006 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}

lube.scott-wheeler.com {
    reverse_proxy 10.0.0.55:8080
}

pihole.scott-wheeler.com {
    redir / /admin
    reverse_proxy 10.0.0.3:80
}

bookstack.scott-wheeler.com {
    reverse_proxy 10.0.0.56:80
}

jellyfin.scott-wheeler.com {
    reverse_proxy 10.0.0.54:8096
}
```

### Trust the Local CA

Root cert location: `/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt`

On Arch Linux:
```bash
sudo cp caddy-root.crt /etc/ca-certificates/trust-source/anchors/
sudo trust extract-compat
```

## Pi-hole Configuration

### Local DNS Records

Add to Pi-hole (`/etc/pihole/custom.list` or via web UI):

```
10.0.0.52 caddy.scott-wheeler.com
10.0.0.52 proxmox.scott-wheeler.com
10.0.0.52 lube.scott-wheeler.com
10.0.0.52 pihole.scott-wheeler.com
10.0.0.52 bookstack.scott-wheeler.com
10.0.0.52 jellyfin.scott-wheeler.com
```

All domains point to Caddy, which routes by hostname.

### Prevent Database Bloat

Pi-hole's query DB grew to 1.4GB. Fix:

```bash
# Set retention in /etc/pihole/pihole-FTL.conf
MAXDBDAYS=90

# Reclaim space
sqlite3 /etc/pihole/pihole-FTL.db 'VACUUM;'

# Restart
systemctl restart pihole-FTL
```

### Journal Limits

```bash
mkdir -p /etc/systemd/journald.conf.d/
cat > /etc/systemd/journald.conf.d/00-max-size.conf << 'EOF'
[Journal]
SystemMaxUse=100M
MaxRetentionSec=7day
EOF

systemctl restart systemd-journald
```

## Backup Strategy

### External Drive Setup

```bash
# Format
mkfs.ext4 -L proxmox-backups /dev/sdb1

# Get UUID
blkid /dev/sdb1

# Add to /etc/fstab
UUID=21b64a95-b691-411a-ab11-50b3e7c2f8f8 /mnt/backups ext4 defaults 0 2

# Mount
mount -a

# Add to Proxmox storage (via CLI)
pvesm add dir backup-usb --path /mnt/backups --content backup
```

### Backup Schedule

- **When:** Sunday 02:00
- **Compression:** ZSTD
- **Retention:** 14 backups
- **Storage:** backup-usb (2TB external)

Estimated weekly backup: ~30-40GB. 14 weeks retention = ~500GB.

### Manual Backup Commands

```bash
# Backup specific container
vzdump 100 --storage backup-usb --mode snapshot --compress zstd

# Check backup logs
tail -100 /var/log/vzdump.log

# List backups
ls -lh /mnt/backups/dump/
```

## Gotchas

### Bookstack URL Migration

Bookstack stores URLs in the database. After IP change:

```bash
cd /opt/bookstack
php artisan bookstack:update-url http://OLD_IP http://NEW_IP
php artisan cache:clear
systemctl restart apache2
```

### Pi-hole 403 on Root

Pi-hole web UI lives at `/admin`. Add redirect in Caddy:

```
redir / /admin
```

### Proxmox No-Subscription Repo

Default enterprise repo requires subscription. Switch to community:

```bash
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update
```

## Storage Cleanup Results

| Action | Space Freed |
|--------|-------------|
| Deleted unused containers (104, 106) | 96 GB |
| Pi-hole DB vacuum | ~1 GB |
| Pi-hole journal cleanup | 220 MB |
| Lubelogger journal cleanup | 3.1 GB |
| **Total** | **~100 GB** |

SSD pool: 75% → 47%

## Useful Commands

```bash
# Container management
pct list                          # List containers
pct enter <vmid>                  # Shell into container
pct exec <vmid> -- df -h          # Run command in container
pct config <vmid>                 # View config

# Storage
pvesm status                      # Storage pool status
lvs -o lv_name,lv_size,data_percent

# Backups
vzdump <vmid> --storage backup-usb --mode snapshot --compress zstd
pct restore <vmid> /path/to/backup.tar.zst

# Network
pct set <vmid> --net0 name=eth0,bridge=vmbr0,ip=10.0.0.X/24,gw=10.0.0.2
```

## Future Plans

- Migrate Pi-hole to 10.0.0.51 (currently at legacy .3 for stability)
- VLAN segmentation for IoT isolation
- Wireguard for remote access
- Uptime Kuma or Netdata for monitoring
- Offsite backup rotation (3-2-1 strategy)
