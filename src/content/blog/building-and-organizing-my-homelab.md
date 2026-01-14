---
title: "Building and Organizing My Homelab: A Complete Guide"
description: "How I spent a day transforming a chaotic mess of services into a production-ready homelab with Proxmox, Pi-hole, Caddy, and more."
date: 2026-01-12
tags: ["homelab", "self-hosted", "proxmox", "networking", "linux"]
---

*How I spent a day transforming a chaotic mess of services into a production-ready homelab*

If you've ever found yourself scrolling through self-hosted software lists at 2 AM, wondering if you really *need* your own media server, DNS sinkhole, and documentation wiki, you're not alone. That curiosity eventually led me down the homelab rabbit hole—and I haven't looked back since.

But here's the thing nobody tells you: setting up services is the easy part. The hard part? Keeping everything organized, backed up, and maintainable. After months of "I'll organize it later" and IP addresses scribbled on sticky notes, I finally dedicated a full day to transforming my homelab from a functional mess into something I'm genuinely proud of.

This guide walks through everything: the why, the what, and the hard-won lessons from reorganizing my entire setup.

## Why Build a Homelab?

Before diving into the technical details, let's address the obvious question: why bother?

**Learning and skill development.** There's no better way to learn networking, Linux administration, containerization, and infrastructure concepts than by running your own services. Every problem you encounter teaches something new. When Pi-hole stops resolving DNS and your entire network goes dark, you'll learn more about DNS in 20 minutes of panicked troubleshooting than in any certification course.

**Privacy and control.** Your data stays on hardware you physically own. No cloud provider mining your media library metadata, no subscription services deciding to "sunset" features you depend on. When you run Jellyfin instead of Plex Pass, you're not at the mercy of corporate product decisions.

**Practical daily use.** This isn't just a learning exercise—my homelab provides genuine value every day. Network-wide ad blocking means no more YouTube pre-roll ads on my TV. A personal documentation wiki means my notes are searchable and accessible from any device. Vehicle maintenance tracking helps me stay on top of oil changes across multiple cars.

**Cost savings over time.** Yes, there's upfront hardware cost. But eliminating multiple subscription services while gaining more features and control? That math works out pretty quickly.

## Hardware Overview

My setup proves you don't need enterprise-grade equipment to run a capable homelab. Here's what powers everything:

### The Server: Lenovo ThinkCentre Mini PC

These small form-factor business PCs are homelab gold. Mine has a multi-core processor, 16GB of RAM, and an NVMe SSD for the operating system. They're quiet, power-efficient (running 24/7 costs just a few dollars a month in electricity), and widely available refurbished for reasonable prices.

The mini PC runs **Proxmox VE 8.2.7**, a free and open-source virtualization platform. Proxmox lets me run multiple isolated containers and virtual machines on a single physical host. Think of it as having multiple computers in one box—except they're all software-defined and easily managed through a web interface.

### Storage Configuration

- **System SSD (66GB):** Holds Proxmox itself, ISO images, and container templates
- **Container Storage SSD (233GB):** LVM-thin provisioned storage for container and VM disks
- **2TB USB External Drive:** Dedicated backup storage

The external drive was a critical addition during my reorganization day. Internal storage is great for performance, but having backups on physically separate media means I can survive a complete host failure.

### Networking Equipment

- **Netgear Nighthawk RAX45:** Primary router, WiFi access point, and gateway at 10.0.0.2
- **Ubiquiti EdgeRouter X:** Repurposed as a simple gigabit switch (these make excellent dumb switches when you don't need their routing features)

The network topology is straightforward: ISP fiber modem in bridge mode feeds the Nighthawk, which handles routing and WiFi. The EdgeRouter provides additional wired ports to the Proxmox server and other devices.

## Services Running

Here's what's actually running on this modest hardware—five LXC containers and one virtual machine:

### Pi-hole (Container 100)

**Purpose:** Network-wide DNS filtering and ad blocking

Pi-hole intercepts DNS queries from every device on my network and blocks requests to known advertising and tracking domains. The result? No ads on YouTube, no tracking beacons on websites, and faster page loads since ad content never downloads.

**Resource allocation:** 2 CPU cores, 512MB RAM, 8GB storage

The beauty of Pi-hole is its network-wide coverage. Unlike browser extensions that only work on one device, Pi-hole blocks ads on smart TVs, phones, tablets—anything using DNS on your network.

### Caddy (Container 102)

**Purpose:** Reverse proxy with automatic HTTPS

Caddy sits in front of all my other services and handles SSL/TLS termination. Instead of accessing services via `http://10.0.0.55:8080`, I use friendly URLs like `https://lube.scott-wheeler.com` with valid certificates.

**Resource allocation:** 1 CPU core, 512MB RAM, 8GB storage

### Home Assistant (VM 103)

**Purpose:** Home automation

Home Assistant runs as a full virtual machine rather than a container because it benefits from direct hardware access for Zigbee/Z-Wave dongles. It integrates with smart home devices across different ecosystems and provides unified automation.

**Resource allocation:** 2 CPU cores, 4GB RAM, 32GB storage

### Bookstack (Container 105)

**Purpose:** Self-hosted documentation wiki

All my notes, procedures, and documentation live here. It's searchable, supports Markdown, and organizes content into books, chapters, and pages. The documentation files that informed this blog post started as Bookstack pages.

**Resource allocation:** 1 CPU core, 1GB RAM, 4GB storage

### Lubelogger (Container 101)

**Purpose:** Vehicle maintenance tracking

This tracks service history, fuel economy, and maintenance schedules for all my vehicles. It runs via Docker Compose with a PostgreSQL database backend. Not the flashiest service, but genuinely useful for staying on top of oil changes and tire rotations.

**Resource allocation:** 1 CPU core, 512MB RAM, 32GB storage

### Jellyfin (Container 107)

**Purpose:** Media streaming server

Jellyfin serves my personal media library to any device on the network. It's a fully open-source alternative to Plex, with no premium tiers or account requirements.

**Resource allocation:** 2 CPU cores, 2GB RAM, 75GB storage

## Network Architecture

One of the biggest improvements during my reorganization was implementing a logical IP addressing scheme. Before: random addresses scattered across the subnet, documented nowhere except my memory. After: a structured system that makes sense.

### The IP Allocation Scheme

```
10.0.0.0/24 Network

Infrastructure (10.0.0.1-49):
  10.0.0.2   - Netgear Router (Gateway)
  10.0.0.3   - Pi-hole (DNS)
  10.0.0.10-49 - Reserved for future infrastructure

Servers (10.0.0.50-99):
  10.0.0.50  - Proxmox Host
  10.0.0.52  - Caddy (Reverse Proxy)
  10.0.0.53  - Home Assistant
  10.0.0.54  - Jellyfin
  10.0.0.55  - Lubelogger
  10.0.0.56  - Bookstack
  10.0.0.57-99 - Reserved for future containers

Static Devices (10.0.0.100-149):
  Reserved for workstations and devices needing permanent IPs

DHCP Pool (10.0.0.150-254):
  Dynamic assignment for phones, tablets, IoT devices
```

This scheme provides several benefits:

- **Predictability:** I know a server will always be in the 50-99 range
- **Room for growth:** Reserved ranges prevent address conflicts as I add services
- **Easy firewall rules:** I can create rules based on IP ranges (e.g., block all 10.0.0.100-149 from accessing the internet during certain hours)
- **Quick troubleshooting:** When something's wrong, I immediately know what type of device is at an address

### The Migration Process

Migrating IP addresses on a live network is nerve-wracking. One wrong move and you're locked out of your own infrastructure. Here's how I approached it:

1. **Proxmox host first:** Changed `/etc/network/interfaces` from the old address to 10.0.0.50. This took about 5 seconds of downtime.

2. **Container by container:** Stopped each container, updated its network configuration via `pct set`, then started it again. Most took under a minute.

3. **Application-specific updates:** Some services (like Bookstack) store URLs in their databases. These required additional commands to update internal references.

4. **Pi-hole last—actually, deferred:** Since Pi-hole is the most critical service (DNS failure = network-wide outage), I kept it at its legacy address. Stability over consistency.

Total migration time for 5 containers and the host: under 30 minutes with cumulative downtime under 5 minutes.

## HTTPS Everywhere with Caddy

Running services over plain HTTP feels wrong in 2026, even on a private network. Caddy makes HTTPS trivially easy with its automatic certificate management.

### Why Not Let's Encrypt?

The typical approach for HTTPS involves Let's Encrypt certificates validated via DNS challenges or exposed HTTP endpoints. For internal-only services, this creates complications:

- Requires exposing ports to the internet (even temporarily)
- Needs public DNS records or DNS provider API access
- Certificates must be renewed externally

Instead, I use Caddy's local certificate authority feature. Caddy generates its own CA and issues certificates for any domain I configure. The certificates are just as cryptographically valid—the only difference is the trust anchor.

### Setting Up Trust

The local CA approach requires trusting Caddy's root certificate on devices that will access the services. On my Arch Linux workstation:

```bash
# Copy Caddy's root CA certificate
sudo cp caddy-root.crt /etc/ca-certificates/trust-source/anchors/
sudo trust extract-compat
```

After this one-time setup, all HTTPS connections to my homelab services show as fully secure in browsers.

### The Caddyfile Configuration

Caddy's configuration is refreshingly simple:

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

Each domain gets its own block. Caddy automatically provisions certificates and handles renewals. The `tls_insecure_skip_verify` for Proxmox is necessary because Proxmox uses its own self-signed certificate—we're still encrypting the connection, just not validating Proxmox's cert.

## DNS and Ad Blocking with Pi-hole

Pi-hole is the unsung hero of my homelab. It handles two critical functions: local DNS resolution and network-wide ad blocking.

### How Pi-hole Works

When any device on your network requests a domain name (like `ads.facebook.com`), the request goes to Pi-hole. Pi-hole checks its blocklists—community-maintained lists of known advertising, tracking, and malicious domains. If the domain is on a blocklist, Pi-hole returns a null response, and the connection never happens.

For legitimate domains, Pi-hole forwards the request to upstream DNS servers (I use Cloudflare's 1.1.1.1) and caches responses for faster subsequent lookups.

### Local DNS Records

Beyond blocking ads, Pi-hole serves as my local DNS server. Instead of maintaining `/etc/hosts` files on every device, I add records to Pi-hole once:

```
10.0.0.52 caddy.scott-wheeler.com
10.0.0.52 proxmox.scott-wheeler.com
10.0.0.52 lube.scott-wheeler.com
10.0.0.52 pihole.scott-wheeler.com
10.0.0.52 bookstack.scott-wheeler.com
10.0.0.52 jellyfin.scott-wheeler.com
```

All domains point to Caddy (10.0.0.52), which then routes requests to the appropriate backend service based on the hostname.

### Maintenance Lessons Learned

Pi-hole's query database can grow surprisingly large. During my cleanup, I discovered 1.4GB of accumulated query logs. The fix:

1. Configure `MAXDBDAYS=90` in `/etc/pihole/pihole-FTL.conf` to auto-purge old queries
2. Run `sqlite3 /etc/pihole/pihole-FTL.db 'VACUUM;'` to reclaim space
3. Set systemd journal limits to prevent log accumulation

After cleanup, Pi-hole's container went from 75% full to 23%.

## Automated Backup Strategy

Backups were the most important addition during my reorganization. Running services is worthless if a drive failure means starting from scratch.

### The Setup

The 2TB external USB drive is formatted as ext4 and mounted at `/mnt/backups`. Proxmox sees it as a storage location specifically for backup files.

**Weekly backup schedule:**
- **Time:** Sunday at 2:00 AM
- **Scope:** All containers and VMs
- **Compression:** ZSTD (excellent compression ratio with reasonable CPU usage)
- **Retention:** 14 backups (approximately 3 months)

**Estimated storage requirements:**
- Pi-hole: ~2-3 GB
- Lubelogger: ~6-7 GB
- Caddy: ~1-2 GB
- Home Assistant: ~4-5 GB
- Bookstack: ~1-2 GB
- Jellyfin: ~15-20 GB
- **Total per week:** ~30-40 GB
- **14-week retention:** ~420-560 GB

With 2TB available, I have plenty of headroom for growth.

### Manual Backup Verification

Automated backups are useless if they're corrupt. Monthly, I verify backup integrity:

```bash
# List backups
ls -lh /mnt/backups/dump/

# Check last backup status
grep "Backup job finished" /var/log/vzdump.log | tail -1
```

I also perform periodic test restores on non-critical containers to verify the complete restore process works.

## The Transformation Process

The January 12th reorganization day addressed accumulated technical debt from months of "quick fixes" and "temporary" configurations.

### Before: The Mess

- IP addresses randomly assigned (10.0.0.181, 10.0.0.219, 10.0.0.146...)
- No backup strategy beyond Proxmox's internal snapshots
- Services accessible only via IP:port combinations
- No HTTPS anywhere
- Documentation scattered across note apps
- Several unused containers consuming storage

### After: Production Ready

- Logical IP scheme with documented ranges
- Automated weekly backups to external storage
- All services accessible via friendly HTTPS URLs
- Comprehensive documentation in Bookstack
- ~100GB of storage reclaimed from deleted containers
- Configured log rotation to prevent future storage bloat

### Space Reclaimed

| Action | Space Freed |
|--------|-------------|
| Deleted ScriptingEnv container | 32 GB |
| Deleted OpenSpeedTest container | 64 GB |
| Pi-hole database vacuum | ~1 GB |
| Pi-hole journal cleanup | 220 MB |
| Lubelogger journal cleanup | 3.1 GB |
| **Total** | **~100 GB** |

The SSD storage pool went from 75% utilization to 47%.

## Challenges and Solutions

### Challenge: Bookstack URL Migration

Bookstack stores URLs in its database. Simply changing the container's IP address left the application redirecting to the old address.

**Solution:** Run the built-in URL update command:
```bash
php artisan bookstack:update-url http://10.0.0.146 http://10.0.0.56
php artisan cache:clear
```

**Lesson:** Database-backed applications often store their own URLs. Always check documentation for URL migration procedures.

### Challenge: Pi-hole Web Interface 403 Errors

After setting up Caddy to proxy Pi-hole, accessing `pihole.scott-wheeler.com` returned "403 Access Denied."

**Solution:** Pi-hole's web interface lives at `/admin`, not root. Added a redirect in Caddy:
```
redir / /admin
```

**Lesson:** Always test the entire user flow, not just whether traffic reaches the backend.

### Challenge: Proxmox Enterprise Repository Errors

Updates were failing because the system was configured to use Proxmox's enterprise repository (which requires a subscription).

**Solution:** Switch to the no-subscription repository:
```bash
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

**Lesson:** Verify repository configuration on any new Proxmox installation.

### Challenge: Client DNS Caching

After migrating IPs, some devices continued trying to reach old addresses.

**Solution:** Clear DNS cache on affected devices. Also, configuring Pi-hole as the network's DNS server ensures all devices get updated records.

**Lesson:** Plan for DNS propagation time and client-side caching when making network changes.

## Lessons Learned

### Document Everything—Before You Forget

The best time to document a configuration is immediately after setting it up. The second-best time is during a reorganization day when you're forced to figure out what you configured six months ago.

### Static IPs Need a System

Random IP assignment works until you have more than three services. Plan your IP scheme before your network grows.

### Backups Aren't Real Until Tested

Having backup files isn't enough. Periodically restore a container to verify the backup process actually works.

### Log Management Is Not Optional

System logs will consume all available space if you let them. Configure retention policies on day one:
- Systemd journals: 500MB maximum, 7-day retention
- Docker containers: 10MB per file, 3 files maximum
- Application logs: Whatever makes sense for the service

### Keep Critical Services Stable

During the IP migration, I intentionally left Pi-hole at its original address. DNS failure means network-wide outage. Some services are worth protecting even at the cost of a less elegant configuration.

## Future Plans

No homelab is ever "done." Here's what's next:

### Short Term
- Migrate Pi-hole to the new IP scheme (10.0.0.51) with careful coordination
- Configure router to use Pi-hole as its DNS server (enabling network-wide ad blocking for WiFi devices)
- Add monitoring with Uptime Kuma or Netdata

### Medium Term
- Implement 3-2-1 backup strategy (offsite copy of backup drive)
- Add Wireguard VPN for secure remote access
- Explore Jellyfin hardware transcoding options

### Long Term
- VLAN segmentation (separate IoT devices from servers and workstations)
- Additional storage node or NAS
- High-availability for critical services like Pi-hole

## Start Building Your Own

If you've read this far, you're probably considering your own homelab. Here's my honest advice:

**Start small.** A Raspberry Pi running Pi-hole is a legitimate homelab. You don't need enterprise hardware to learn.

**Solve real problems.** The best homelab services are ones you'll actually use. Network-wide ad blocking is a game-changer. A media server you never watch isn't.

**Accept imperfection.** My homelab ran for months with random IPs and no backups. It still worked. Don't let perfect be the enemy of good—especially when you're learning.

**Document as you go.** Future you will thank present you for writing down what that obscure configuration option does.

**Budget for mistakes.** You will break things. You will accidentally delete containers. You will lock yourself out of your own server. This is part of learning. External backups make these mistakes survivable.

The homelab community is remarkably welcoming. Subreddits like r/homelab and r/selfhosted are full of people who've solved the exact problem you're facing. Don't hesitate to ask questions.

My homelab started as curiosity and became an invaluable learning tool and daily utility. Yours can too.
