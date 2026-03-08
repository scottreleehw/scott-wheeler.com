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
| System Drive | NVMe SSD |
| Container Storage | SSD, LVM-thin |
| Backup Drive | 2TB USB 3.0 External |
| Router | Consumer Wi-Fi 6 router |
| Switch | Ubiquiti EdgeRouter X (dumb switch mode) |

**Hypervisor:** Proxmox VE (latest stable)

## Container/VM Inventory

| Name | Type | CPU | RAM | Disk | Purpose |
|------|------|-----|-----|------|---------|
| pi-hole | LXC | 2 | 512MB | 8GB | DNS/ad-blocking |
| lubelogger | LXC | 1 | 512MB | 32GB | Vehicle maintenance |
| caddy | LXC | 1 | 512MB | 8GB | Reverse proxy |
| bookstack | LXC | 1 | 1GB | 4GB | Documentation wiki |
| jellyfin | LXC | 2 | 2GB | 75GB | Media server |

## Network Architecture

I organized the `/24` subnet into logical blocks:

```
Infrastructure (1-49):
  Gateway, Pi-hole DNS

Servers (50-99):
  Proxmox host, Caddy, Jellyfin, Lubelogger, Bookstack

Static Devices (100-149):
  Reserved for future use

DHCP (150-254):
  Dynamic pool
```

### IP Migration

Migrating containers is straightforward:

```bash
# Stop container
pct stop <vmid>

# Update network config
pct set <vmid> --net0 name=eth0,bridge=vmbr0,ip=<new-ip>/24,gw=<gateway>

# Start container
pct start <vmid>
```

For the Proxmox host itself, edit `/etc/network/interfaces`:

```
auto vmbr0
iface vmbr0 inet static
    address <host-ip>/24
    gateway <gateway-ip>
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

Apply with `ifreload -a`. ~5 seconds downtime.

## Caddy Reverse Proxy

Using Caddy's internal CA for HTTPS on the local network — no Let's Encrypt needed since services aren't internet-exposed.

### Caddyfile

```
{
    local_certs
}

proxmox.example.com {
    reverse_proxy <proxmox-ip>:8006 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}

service.example.com {
    reverse_proxy <service-ip>:<port>
}
```

The pattern is the same for each service — a hostname that Caddy matches, proxied to the container's internal IP and port. Proxmox needs `tls_insecure_skip_verify` because its built-in web UI uses a self-signed cert.

### Trust the Local CA

Root cert location: `/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt`

On Arch Linux:
```bash
sudo cp caddy-root.crt /etc/ca-certificates/trust-source/anchors/
sudo trust extract-compat
```

## Pi-hole Configuration

### Local DNS Records

All internal hostnames point to Caddy's IP in Pi-hole's local DNS records (`/etc/pihole/custom.list` or via web UI). Caddy then routes by hostname to the appropriate container.

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

Container journals can grow fast. Set limits:

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

# Add to /etc/fstab (use your drive's UUID from blkid)
UUID=<your-uuid> /mnt/backups ext4 defaults 0 2

# Mount
mount -a

# Add to Proxmox storage
pvesm add dir backup-usb --path /mnt/backups --content backup
```

### Backup Schedule

- **When:** Sunday 02:00
- **Compression:** ZSTD
- **Retention:** 14 backups
- **Storage:** 2TB external USB

Estimated weekly backup: ~30-40GB. 14 weeks retention = ~500GB.

### Manual Backup Commands

```bash
# Backup specific container
vzdump <vmid> --storage backup-usb --mode snapshot --compress zstd

# Check backup logs
tail -100 /var/log/vzdump.log

# List backups
ls -lh /mnt/backups/dump/
```

## Gotchas

### Bookstack URL Migration

Bookstack stores URLs in the database. After an IP change:

```bash
cd /opt/bookstack
php artisan bookstack:update-url http://OLD_IP http://NEW_IP
php artisan cache:clear
systemctl restart apache2
```

### Pi-hole 403 on Root

Pi-hole web UI lives at `/admin`. Add a redirect in Caddy:

```
redir / /admin
```

### Proxmox No-Subscription Repo

Default enterprise repo requires a subscription. Switch to community:

```bash
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update
```

## Storage Cleanup Results

| Action | Space Freed |
|--------|-------------|
| Deleted unused containers | ~96 GB |
| Pi-hole DB vacuum | ~1 GB |
| Pi-hole journal cleanup | 220 MB |
| Lubelogger journal cleanup | 3.1 GB |
| **Total** | **~100 GB** |

SSD pool went from 75% to 47% utilization.

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
pct set <vmid> --net0 name=eth0,bridge=vmbr0,ip=<ip>/24,gw=<gateway>
```

## Future Plans

- VLAN segmentation for IoT isolation
- VPN for remote access
- Uptime monitoring
- Offsite backup rotation (3-2-1 strategy)
