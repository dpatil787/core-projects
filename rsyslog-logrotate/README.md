# Rsyslog Centralized Logging with Loki, Promtail, Prometheus & Grafana

## Project Overview

In a production environment, logs are scattered across many different servers. It is not practical to log into every machine individually to troubleshoot. This project sets up a complete centralized logging and monitoring solution that collects logs from multiple client machines, indexes them for searching, and visualizes them through a dashboard.

---

## Architecture

```
VM1 (rsyslogserver) 192.168.253.128   — Rsyslog Server + Promtail
VM2 (loki)          192.168.253.129   — Loki (Log Storage & Indexing)
VM3 (prometheus)    192.168.253.130   — Prometheus + Grafana (Metrics & Dashboard)
```

**Full Data Flow:**
```
VM2 & VM3 → send logs via TCP 514 → VM1 Rsyslog (stores logs as files)
                                          ↓
                              VM1 Promtail (reads local log files)
                                          ↓
                              VM2 Loki (indexes logs)
                                          ↓
                              VM3 Grafana (search + visualization)
```

---

## Technology Stack

| Tool | Role | VM |
|---|---|---|
| Rsyslog | Centralized log collection and storage | VM1 |
| Promtail | Log file reader — reads and forwards logs to Loki | VM1 |
| Loki | Log aggregation system with indexing and search | VM2 |
| Prometheus | Metrics collection and storage | VM3 |
| Grafana | Visualization and dashboard for logs and metrics | VM3 |

---

## Project Phases

### Phase 1 — Basic Centralized Logging
Configured Rsyslog server on VM1 to receive logs from client machines via TCP port 514. Configured VM2 and VM3 as clients to forward all logs to VM1. Created dynamic log storage using templates so each client gets its own folder (`/var/log/<hostname>/`).

### Phase 2 — Reliability with Disk-Assisted Queues
Configured disk queues on client machines. When the central server is unreachable, logs are saved to local disk and automatically sent once the server is back online. Eliminates log loss during network failures.

### Phase 3 — Log Rotation with Logrotate
Configured logrotate to automatically compress, archive, and delete old logs daily. Keeps the last 7 days of logs and prevents disk from filling up.

### Phase 4 — Log Indexing and Search (Loki + Promtail)
Installed Loki on VM2 for log storage and indexing. Installed Promtail on VM1 (where logs are stored) to read log files locally and push them to Loki. Logs are now searchable by time range, hostname, job, and content through Grafana.

### Phase 5 — Metrics and Dashboard (Prometheus + Grafana)
Installed Prometheus on VM3 for metrics collection. Installed Grafana on VM3 and configured both Prometheus and Loki as data sources. Grafana dashboard provides visibility into both metrics and logs from a single location.

---

## Setup and Implementation

### VM1 — Rsyslog Server Configuration

**Install and enable Rsyslog:**
```bash
dnf install -y rsyslog
systemctl start rsyslog
systemctl enable rsyslog
```

**Enable TCP input in `/etc/rsyslog.conf`:**
```
module(load="imtcp")
input(type="imtcp" port="514")
```

**Dynamic log storage template in `/etc/rsyslog.conf`:**
```
$template RemoteLogs, "/var/log/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
```

**Open firewall:**
```bash
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --reload
```

---

### VM2 & VM3 — Client Configuration

**Forward logs to server in `/etc/rsyslog.conf`:**
```
*.* @@192.168.253.128:514
```

**Disk queue configuration in `/etc/rsyslog.conf`:**
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

### VM1 — Logrotate Configuration

**File: `/etc/logrotate.d/rsyslog_custom`**
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

### VM2 — Loki Configuration

**File: `/etc/loki-config.yaml`**
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

### VM1 — Promtail Configuration

**File: `/etc/promtail-config.yaml`**
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

### VM3 — Prometheus Configuration

**Scrape config addition in `/etc/prometheus/prometheus.yml`:**
```yaml
  - job_name: 'promtail'
    static_configs:
      - targets: ['192.168.253.128:9080']
```

**Grafana Data Sources:**
- Prometheus: `http://192.168.253.130:9090`
- Loki: `http://192.168.253.129:3100`

---

## Output / Verification

**Verify Rsyslog is receiving logs:**
```bash
ls /var/log/loki/
ls /var/log/prometheus/
```

**Verify Loki is working:**
```bash
curl http://localhost:3100/ready
# Expected: ready
```

**Verify Promtail is working:**
```bash
curl http://localhost:9080/metrics
journalctl -u promtail -f
# Look for: msg="Adding target" with correct log paths
```

**Verify logs reached Loki:**
```bash
curl -s "http://localhost:3100/loki/api/v1/labels" | python3 -m json.tool
# Expected labels: filename, hostname, job, service_name
```

**Grafana LogQL Queries:**
```logql
{job="rsyslog"}                                      # All logs
{job="rsyslog", hostname="rsyslogserver"}            # Specific host
{job="rsyslog"} |= "error"                          # Error logs only
{job="rsyslog"} |= "DEBUG"                          # Debug logs
```

---

## Troubleshooting

**Rsyslog not receiving logs from clients:**
- Check firewall — port 514/tcp must be open on VM1
- Check SELinux — set to permissive in lab
- Run `ss -tlnp | grep 514` on VM1 to confirm Rsyslog is listening

**Loki service fails with 203/EXEC:**
- Wrong SELinux context on binary after moving from `/tmp`
- Run `restorecon -v /usr/local/bin/loki`

**Loki config validation errors:**
- Ensure `schema: v13` and `store: tsdb` — Loki 3.x requires these
- Ensure `allow_structured_metadata: false` is present in `limits_config`

**Promtail permission denied errors:**
- `__path__` was too broad — pointed to system folders like `/var/log/anaconda/`
- Restrict path to specific folders: `/var/log/{rsyslogserver,loki,prometheus}/*.log`

**No logs in Grafana:**
- Change time range from "Last 1 hour" to "Last 24 hours"
- Verify Loki data source URL is correct
- Switch to Code mode and use a simple query: `{job="rsyslog"}`

---

## Key Learnings

- Promtail should be installed on the same machine where logs are stored — reads locally, pushes remotely. Installing on a different machine would require NFS mount and adds unnecessary complexity.
- Loki 3.x requires `schema: v13` and `store: tsdb` — older configs cause validation failures.
- Binaries moved from `/tmp` must have SELinux context restored using `restorecon` — otherwise systemd throws `203/EXEC` error.
- Disk queues on Rsyslog clients ensure zero log loss during server downtime.
- Positions file in Promtail prevents duplicate logs from being sent after restarts.

---

## Part of Linux Server Configuration Series

This project is part of the Linux Server Configurations learning series.

Repository: [dpatil787/linux-server-configurations](https://github.com/dpatil787/linux-server-configurations)
