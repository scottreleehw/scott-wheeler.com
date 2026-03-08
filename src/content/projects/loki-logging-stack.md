---
title: "Centralized Logging Stack"
description: "Grafana + Loki + Alloy logging pipeline for homelab infrastructure — syslog aggregation, DNS query tracking, and security monitoring."
tags: ["grafana", "loki", "docker", "security", "monitoring"]
featured: true
date: 2026-03-01
---

## Overview

A lightweight centralized logging stack built on Grafana, Loki, and Alloy, running in Docker Compose on a Proxmox LXC container. Aggregates syslog from all homelab hosts and Pi-hole DNS query logs into a single searchable dashboard.

## Architecture

```
Pi-hole ──rsyslog UDP──→  loki-stack CT
Proxmox ──rsyslog UDP──→  rsyslog → /var/log/remote/{host}/{program}.log
                          Alloy tails files → Loki → Grafana
```

Clients forward syslog via rsyslog to the loki-stack container on UDP 1514. rsyslog on the server writes logs to per-host directories. Alloy tails those files and pushes them into Loki with `host` and `program` labels. Grafana provides the query and dashboard UI.

## Dashboard

Built a custom Grafana dashboard with 8 panels:

- **Log Volume by Host** — time series of ingestion rate
- **Error Rate by Host** — errors, failures, and critical events over time
- **Top Programs by Volume** — pie chart of noisiest services
- **Auth Failures** — filtered view of failed logins and access denials
- **Pi-hole DNS Activity** — DNS query log stream
- **Recent Errors & Warnings** — cross-host error feed
- **All Logs** — full log stream with host dropdown filter

## Why This Stack

- **Loki** uses ~256-512MB RAM — fits easily in a 1GB container
- **Alloy** is Grafana's new collector (replaces Promtail) with a cleaner config language
- **rsyslog as the receiver** instead of Alloy's native syslog input — avoids RFC5424/RFC3164 parsing headaches
- **30-day retention** with automatic compaction and cleanup

## What I Learned

- Alloy's native syslog receiver is strict about RFC5424 compliance, which most Linux systems don't send by default — using rsyslog to write files and having Alloy tail them is more reliable
- Pi-hole DNS logs are chatty (~50MB/day) — worth monitoring disk usage
- Setting up Pi-hole as DHCP server was necessary to get per-device DNS visibility since the Netgear router always hands out its own IP as DNS
