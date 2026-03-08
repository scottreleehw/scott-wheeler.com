---
title: "Centralized Logging for a Homelab with Grafana, Loki, and Alloy"
description: "How I set up a lightweight logging pipeline to aggregate syslog and DNS query logs from my Proxmox homelab into Grafana dashboards."
date: 2026-03-01
tags: ["homelab", "grafana", "loki", "monitoring", "security"]
---

After getting my homelab organized, the next step was visibility. I wanted a single place to search logs across all my containers and the Proxmox host itself — especially auth failures, DNS queries, and errors. Here's how I built it.

## Why Loki Over the ELK Stack

The classic answer for centralized logging is Elasticsearch + Logstash + Kibana, but that's heavy for a homelab. Loki is purpose-built for logs, indexes only labels (not full text), and runs comfortably in ~256-512MB of RAM. Paired with Grafana (which I'd want anyway) and Alloy (Grafana's new log collector), it's a lightweight stack that punches above its weight.

## Architecture

```
Log sources ──rsyslog UDP──→  Logging CT
                              rsyslog → /var/log/remote/{host}/{program}.log
                              Alloy tails files → Loki → Grafana
```

The key design decision: **rsyslog receives the syslog, not Alloy.** Alloy's native `loki.source.syslog` component only supports RFC5424 and is strict about compliance. Most Linux systems send RFC3164 (BSD syslog) by default. Rather than fighting format conversion, I let rsyslog handle reception and write to files organized by host and program name. Alloy then tails those files and pushes them into Loki with appropriate labels. Much more reliable.

## The Stack

Everything runs in Docker Compose on a dedicated LXC container with 1GB RAM and 32GB disk on NVMe storage.

### Docker Compose

```yaml
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  alloy:
    image: grafana/alloy:latest
    volumes:
      - ./alloy-config.alloy:/etc/alloy/config.alloy
      - /var/log/remote:/var/log/remote:ro
    command: run /etc/alloy/config.alloy
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

volumes:
  loki-data:
  grafana-data:
```

### Alloy Config

Alloy's config language is declarative and reads well:

```
local.file_match "remote_logs" {
  path_targets = [{"__path__" = "/var/log/remote/**/*.log"}]
}

loki.source.file "remote_logs" {
  targets    = local.file_match.remote_logs.targets
  forward_to = [loki.process.add_labels.receiver]
}

loki.process "add_labels" {
  stage.regex {
    expression = "/var/log/remote/(?P<host>[^/]+)/(?P<program>[^/]+)\\.log"
    source     = "filename"
  }
  stage.labels {
    values = {
      host    = "",
      program = "",
    }
  }
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

The regex stage extracts `host` and `program` from the file path that rsyslog creates. This gives you clean labels in Grafana without any manual tagging.

### Loki Retention

Set 30-day retention with automatic cleanup:

```yaml
limits_config:
  retention_period: 30d

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
  delete_request_store: filesystem
```

That last line (`delete_request_store`) is easy to miss — Loki won't start with retention enabled without it.

## Adding Log Sources

For any Debian-based container or VM:

```bash
apt install -y rsyslog
echo "*.* @<logging-server-ip>:<port>" >> /etc/rsyslog.conf
systemctl restart rsyslog
logger "test from $(hostname)"
```

For Pi-hole DNS query logs specifically, FTL logs to its own file rather than syslog. Use rsyslog's `imfile` module to tail and forward it:

```
module(load="imfile")

input(type="imfile"
  File="/var/log/pihole/pihole.log"
  Tag="pihole-dns:"
  Facility="local7")

local7.* @<logging-server-ip>:<port>
```

## The Dashboard

I built a Grafana dashboard with panels for:

- **Log volume by host** — time series showing ingestion rate per host
- **Error rate** — filtered for error, fail, and critical keywords
- **Top programs by volume** — pie chart of the noisiest services
- **Auth failures** — failed passwords, invalid users, access denied
- **Pi-hole DNS activity** — live stream of DNS queries
- **Recent errors and warnings** — cross-host error feed

A dashboard-level `host` variable with multi-select lets you filter everything at once.

## Useful LogQL Queries

```
# All logs from a specific host
{host="myhost"}

# Search for errors across everything
{host=~".+"} |= "error"

# Auth failures
{host=~".+"} |~ "(?i)(authentication failure|failed password|invalid user)"

# DNS queries from a specific client
{program="pihole-dns"} |= "from 10.0.0."

# Exclude noisy programs
{host="myhost", program!="noisyservice"}
```

## Gotchas

- **Unprivileged LXC containers can't bind to port 514** — use a high port (I used 1514) for syslog reception
- **Pi-hole DNS logs are chatty** — expect ~50MB/day; monitor disk usage
- **Alloy's syslog receiver vs file tailing** — file tailing is significantly more reliable for mixed RFC3164/5424 environments
- **Log rotation** — set up logrotate for `/var/log/remote/` on the logging server to prevent disk fill from the raw syslog files

## What's Next

- Grafana alerting for specific patterns (failed SSH, disk warnings)
- Adding more log sources as the homelab grows
- Exploring Grafana's AI plugin for natural language log queries
