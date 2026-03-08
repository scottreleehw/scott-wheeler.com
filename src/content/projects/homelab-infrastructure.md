---
title: "Homelab Infrastructure"
description: "Proxmox-based homelab running LXC containers for self-hosted services — Pi-hole DNS, Caddy reverse proxy, Jellyfin, Bookstack, and more."
tags: ["proxmox", "linux", "networking", "self-hosted"]
featured: true
date: 2026-01-12
---

## Overview

A single Lenovo ThinkCentre Mini PC running Proxmox VE, hosting everything in lightweight LXC containers. Designed around a clean IP scheme, automated backups, and internal HTTPS via Caddy with a local CA.

## Architecture

```
10.0.0.0/24

Infrastructure (1-49):
  .2   - Gateway (Netgear Nighthawk RAX45)
  .3   - Pi-hole DNS + DHCP

Servers (50-99):
  .50  - Proxmox host
  .52  - Caddy reverse proxy
  .54  - Jellyfin media server
  .55  - Lubelogger
  .56  - Bookstack wiki

Logging (100-149):
  .121 - Loki logging stack

DHCP (150-254):
  Dynamic pool (served by Pi-hole)
```

## Services

| Container | Purpose |
|-----------|---------|
| Pi-hole | DNS filtering, ad blocking, DHCP server |
| Caddy | Reverse proxy with local CA HTTPS |
| Jellyfin | Media server |
| Bookstack | Internal documentation wiki |
| Lubelogger | Vehicle maintenance tracking |
| Loki Stack | Centralized logging (Grafana + Loki + Alloy) |

## Hardware

- **Server:** Lenovo ThinkCentre Mini PC, 16GB RAM
- **Storage:** NVMe SSD (system + containers) + SSD (overflow) + 2TB USB (backups)
- **Network:** Netgear Nighthawk RAX45 + Ubiquiti EdgeRouter X (switch mode)

## Key Design Decisions

- **LXC over VMs** — lower overhead, faster boot, sufficient isolation for homelab use
- **Pi-hole as DHCP server** — Netgear router can't customize DNS handed to DHCP clients, so Pi-hole handles both DNS and DHCP for per-device visibility
- **Caddy with local CA** — internal HTTPS without needing Let's Encrypt or exposing services to the internet
- **Weekly automated backups** — ZSTD-compressed snapshots to external USB, 14-backup retention

## What I Learned

- Reorganizing IP schemes on a live network is tedious but worth doing early
- Unprivileged LXC containers can't bind to ports below 1024 — plan around it
- Pi-hole's query database will balloon if you don't set retention limits
- Journal logs in containers accumulate fast — set `SystemMaxUse` and `MaxRetentionSec`
