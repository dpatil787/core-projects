# 📋 Rsyslog Centralized Logging with Loki, Promtail, Prometheus & Grafana

![Linux](https://img.shields.io/badge/Linux-RHEL%209-black?style=for-the-badge&logo=linux&logoColor=white)
![Rsyslog](https://img.shields.io/badge/Rsyslog-Centralized%20Logging-orange?style=for-the-badge)
![Loki](https://img.shields.io/badge/Grafana-Loki-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Dashboard-F46800?style=for-the-badge&logo=grafana&logoColor=white)

---

## Project Overview

In a production environment, logs are scattered across many different servers. It is not practical to log into every machine individually to troubleshoot issues. This project builds a complete centralized logging and monitoring solution from scratch — collecting logs from multiple client machines, making them searchable through Loki, and visualizing them alongside system metrics in Grafana.

---

## What This Project Demonstrates

- Setting up Rsyslog as a centralized log collection server to receive logs from multiple clients over TCP
- Configuring dynamic log storage using Rsyslog templates, automatically creating separate folders per client hostname
- Implementing disk-assisted queues on client machines to ensure zero log loss during server downtime
- Automating log rotation and disk management with Logrotate
- Installing and configuring Loki as a log aggregation and indexing engine
- Deploying Promtail on the log server to read local log files and forward them to Loki
- Deploying Prometheus to collect metrics and Grafana to visualize both logs and metrics from a single dashboard
- Querying logs in Grafana using LogQL with label-based filtering

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Log Collection Layer                        │
│                                                                  │
│   VM2 (loki) ──────────────────────────────────────────────┐    │
│                  TCP 514          VM1 (rsyslogserver)       │    │
│   VM3 (prometheus) ─────────────► Rsyslog stores logs      │    │
│                                   /var/log/%HOSTNAME%/      │    │
└──────────────────────────────────────────────────────────── │ ──┘
                                                              │
┌─────────────────────────────────────────────────────────────▼───┐
│                     Forwarding Layer (VM1)                        │
│                                                                   │
│              Promtail reads local log files                       │
│              /var/log/{rsyslogserver,loki,prometheus}/*.log       │
└──────────────────────────────────────────┬────────────────────── ┘
                                           │ HTTP Push
                                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Storage & Visualization Layer                   │
│                                                                    │
│   VM2 (loki)          VM3 (prometheus + grafana)                  │
│   Loki indexes ──────► Grafana displays logs + metrics            │
│   logs                 Prometheus scrapes metrics                  │
└────────────────────────────────────────────────────────────────── ┘
```

| VM | IP Address | Role |
|---|---|---|
| VM1 — rsyslogserver | 192.168.253.128 | Rsyslog Server + Promtail |
| VM2 — loki | 192.168.253.129 | Loki (Log Storage & Indexing) |
| VM3 — prometheus | 192.168.253.130 | Prometheus + Grafana |

---

## Production Use

This setup maps directly to how centralized logging is handled in real production environments:

| This Lab | Production Equivalent |
|---|---|
| Rsyslog on VM1 collecting logs | Centralized syslog server receiving logs from 100+ nodes |
| Disk-assisted queues on clients | No log loss during network outages or server restarts |
| Logrotate managing /var/log | Automated log lifecycle management preventing disk fill |
| Promtail reading local files | Agents deployed on every node forwarding to central Loki |
| Loki indexing by hostname, job | Log streams queryable by service, environment, region |
| Grafana LogQL queries | SRE/DevOps teams filtering incidents by time + severity |
| Prometheus scraping Promtail | Monitoring the health of the logging infrastructure itself |

---

## Technology Stack

| Tool | Version | Purpose |
|---|---|---|
| Rsyslog | Default RHEL 9 | Centralized log collection and storage |
| Promtail | v2.9.9 | Log file reader — reads and pushes logs to Loki |
| Loki | v3.x | Log aggregation system with indexing and search |
| Prometheus | v3.10.0 | Metrics collection and storage |
| Grafana | Latest OSS | Visualization and dashboard |
| Logrotate | System default | Automated log compression and rotation |

---

## Project Phases

### Phase 1 — Basic Centralized Logging

Configured Rsyslog server on VM1 to receive logs from client machines via TCP port 514. Configured VM2 and VM3 as clients to forward all logs to VM1. Created dynamic log storage using Rsyslog templates so each client gets its own folder automatically under `/var/log/<hostname>/`.

### Phase 2 — Reliability with Disk-Assisted Queues

Configured disk queues on client machines. When the central server goes down, logs are saved to the client's local disk and sent automatically once the server is back. Completely eliminates log loss during network failures or server restarts.

### Phase 3 — Log Rotation with Logrotate

Configured logrotate to automatically compress, archive, and delete old logs on a daily schedule. Retains the last 7 days of logs. Prevents disk from filling up over time.

### Phase 4 — Log Indexing and Search (Loki + Promtail)

Installed Loki on VM2 for log storage and indexing. Installed Promtail on VM1 (where all logs are stored) to read log files locally and push them to Loki. Logs are now fully searchable by time range, hostname, job, and content through Grafana Explore.

### Phase 5 — Metrics and Dashboard (Prometheus + Grafana)

Installed Prometheus on VM3 for metrics collection including Promtail health metrics. Installed Grafana and configured both Prometheus and Loki as data sources. Grafana provides a unified view of logs and metrics from a single location.

---

## Setup and Implementation

### Phase 1 — Rsyslog Server (VM1)

Enable TCP input in `/etc/rsyslog.conf`:
```
module(load="imtcp")
input(type="imtcp" port="514")
```

Dynamic log storage template:
```
$template RemoteLogs, "/var/log/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
```

Open firewall:
```bash
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --reload
```

---

### Phase 1 — Rsyslog Client Configuration (VM2 and VM3)

Forward all logs to server in `/etc/rsyslog.conf`:
```
*.* @@192.168.253.128:514
```

---

### Phase 2 — Disk Queue Configuration on Clients

```
*.* action(type="omfwd"
    target="192.168.253.128"
    port="514"
    protocol="tcp"
    action.resumeRetryCount="-1"
    queue.type="LinkedList"
    queue.filename="rsyslog_queue"
    queue.maxDiskSpace="1g"
    queue.saveOnShutdown="on")
```

---

### Phase 3 — Logrotate Config on VM1

File: `/etc/logrotate.d/rsyslog_custom`
```
/var/log/loki/*.log
/var/log/prometheus/*.log
{
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
}
```

---

### Phase 4 — Loki Config (VM2)

File: `/etc/loki-config.yaml`
```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /var/lib/loki
  ring:
    kvstore:
      store: inmemory
  replication_factor: 1
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  allow_structured_metadata: false
```

---

### Phase 4 — Promtail Config (VM1)

File: `/etc/promtail-config.yaml`
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://192.168.253.129:3100/loki/api/v1/push

scrape_configs:
  - job_name: rsyslog_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: rsyslog
          __path__: /var/log/{rsyslogserver,loki,prometheus}/*.log
    pipeline_stages:
      - match:
          selector: '{job="rsyslog"}'
          stages:
            - regex:
                expression: '^.*\/(?P<hostname>[^\/]+)\/.*$'
                source: filename
            - labels:
                hostname:
```

---

### Phase 5 — Prometheus Scrape Config (VM3)

Addition in `/etc/prometheus/prometheus.yml`:
```yaml
  - job_name: 'promtail'
    static_configs:
      - targets: ['192.168.253.128:9080']
```

Grafana Data Sources:
- Prometheus: `http://192.168.253.130:9090`
- Loki: `http://192.168.253.129:3100`

---

## Output

### Phase 1 — Dynamic Log Folders Created Automatically

```
/var/log/
├── loki/
│   ├── rsyslogd.log
│   ├── sshd.log
│   └── root.log
├── prometheus/
│   ├── rsyslogd.log
│   ├── sshd.log
│   └── root.log
└── rsyslogserver/
    └── rsyslogd.log
```

### Phase 2 — Disk Queue Files on Client (when server is down)

```
/var/lib/rsyslog/
└── rsyslog_queue.qi   ← queue index file created when server unreachable
```

### Phase 3 — Logrotate Output

```
/var/log/loki/
├── root.log            ← current active log
├── root.log.1          ← yesterday's rotated log
└── root.log.2.gz       ← day before compressed
```

### Phase 4 — Loki Labels Confirmed

```bash
curl -s "http://localhost:3100/loki/api/v1/labels" | python3 -m json.tool
```
```json
{
    "status": "success",
    "data": [
        "filename",
        "hostname",
        "job",
        "service_name"
    ]
}
```

### Phase 5 — Grafana LogQL Queries Working

```logql
{job="rsyslog"}                                       # All logs
{job="rsyslog", hostname="rsyslogserver"}             # Specific host
{job="rsyslog"} |= "error"                           # Error logs only
{job="rsyslog"} |= "DEBUG"                           # Debug logs
{job="rsyslog", hostname="prometheus"} |= "error"    # Host + severity filter
```

---

## Key Files and Directories

| File / Directory | VM | Purpose |
|---|---|---|
| `/etc/rsyslog.conf` | VM1, VM2, VM3 | Main Rsyslog configuration |
| `/etc/logrotate.d/rsyslog_custom` | VM1 | Custom logrotate rules |
| `/var/log/%HOSTNAME%/` | VM1 | Dynamic log storage per client |
| `/var/lib/rsyslog/` | VM2, VM3 | Disk queue storage location |
| `/etc/loki-config.yaml` | VM2 | Loki server configuration |
| `/var/lib/loki/chunks/` | VM2 | Loki log chunk storage |
| `/etc/promtail-config.yaml` | VM1 | Promtail scrape and forward config |
| `/var/lib/promtail/positions.yaml` | VM1 | Tracks log lines already sent to Loki |
| `/etc/prometheus/prometheus.yml` | VM3 | Prometheus scrape configuration |
| `/etc/systemd/system/loki.service` | VM2 | Loki systemd service |
| `/etc/systemd/system/promtail.service` | VM1 | Promtail systemd service |
| `/etc/systemd/system/prometheus.service` | VM3 | Prometheus systemd service |

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| Logs not arriving on VM1 | Firewall blocking port 514 | `firewall-cmd --permanent --add-port=514/tcp --reload` on VM1 |
| Loki service fails with 203/EXEC | Wrong SELinux context after moving binary from /tmp | `restorecon -v /usr/local/bin/loki` |
| Loki config validation errors | Schema or store version mismatch for Loki 3.x | Use `schema: v13`, `store: tsdb`, add `allow_structured_metadata: false` |
| Promtail permission denied errors | Path too broad — targeting system log folders | Restrict `__path__` to `/var/log/{rsyslogserver,loki,prometheus}/*.log` |
| Port 3100 already in use | Stale Loki process from previous failed start | `kill -9 <pid>` then start via systemd |
| Loki "Ingester not ready" on first start | Normal startup delay | Wait 15-20 seconds, then curl again |
| No logs in Grafana query | Time range too narrow | Change from "Last 1 hour" to "Last 24 hours" |
| Grafana cannot reach Loki | Wrong URL in data source | Use `http://192.168.253.129:3100` not localhost |
| promtool check fails | YAML indentation error | Check spacing — YAML is space-sensitive |

---

## What's Next

This setup can be further extended with:

- **Alerting** — Configure Grafana alerting rules to notify when error log rate crosses a threshold or when a client stops sending logs
- **TLS Encryption** — Secure log transmission between Rsyslog clients and server using certificates
- **Additional Exporters** — Add Node Exporter on VM1 and VM2 to monitor CPU, memory, and disk alongside logs
- **Replication Monitoring** — Add a replica database server and monitor replication lag through Prometheus
- **Ansible Automation** — Automate Promtail and Rsyslog client deployment across multiple nodes using Ansible playbooks

---

*Document prepared as part of DevOps Home Lab*
