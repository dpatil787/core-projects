# Nginx Observability Stack — Reverse Proxy, Load Balancing & Full Monitoring

![Linux](https://img.shields.io/badge/Linux-000000?style=for-the-badge&logo=linux&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-4CAF50?style=for-the-badge&logo=grafana&logoColor=white)

---

## Project Overview

This project builds a complete web application infrastructure with Nginx as reverse proxy and load balancer, backend web servers hosting a custom website, and a full observability stack including metrics, logs, and alerts. The entire system is monitored in real time, and failures trigger automatic alerts.

> I built a complete web application stack with Nginx as reverse proxy and load balancer, backend web servers with SSL, and full observability using Prometheus, Grafana, Loki, and Alertmanager. When I simulated a backend failure, the system detected it within 30 seconds, showed error logs in Grafana, and sent a Slack alert — exactly how production systems are monitored.

---

## Tech Stack

| Tool | Role |
|---|---|
| Nginx | Web server, Reverse proxy, Load balancer |
| Promtail | Log collection agent |
| Loki | Log storage and indexing |
| Prometheus | Metrics collection |
| Node Exporter | System metrics — CPU, RAM, disk (all 4 VMs) |
| Grafana | Visualization and dashboards |
| Alertmanager | Alert routing and Slack integration |
| OS | CentOS Stream 9 |

---

## Architecture

```
VM1 (nginxwebserver)   192.168.253.128   Backend web server, serves HTML/CSS, handles HTTPS
VM2 (lbserver)         192.168.253.129   Reverse proxy, load balancer, Promtail installed
VM3 (backendserver2)   192.168.253.130   Second backend server, same website and SSL as VM1
VM4 (monitoring)       192.168.253.136   Prometheus, Grafana, Loki, Alertmanager
```

**Traffic Flow:**
```
Internet → VM2 (Reverse Proxy + Load Balancer) → VM1 and VM3 (Backend Web Servers)
VM2 Promtail → VM4 Loki → VM4 Grafana (logs)
VM4 Prometheus scrapes Node Exporter on all 4 VMs (metrics)
VM4 Alertmanager → Slack #alerts (notifications)
```

---

## What This Project Demonstrates

| Area | Detail |
|---|---|
| Nginx as web server | Serving custom HTML website with SSL on backend servers |
| Nginx as reverse proxy | Single entry point hiding backend servers from internet |
| Nginx as load balancer | Round Robin traffic distribution across VM1 and VM3 |
| SSL termination | HTTPS configured on both proxy and backend layers |
| Log pipeline | Promtail reads VM2 Nginx logs and pushes to Loki |
| Log visualization | Grafana Loki dashboard showing real time Nginx logs |
| Metrics collection | Node Exporter on all 4 VMs scraped by Prometheus |
| System monitoring | Grafana Dashboard 1860 showing CPU, RAM, disk across all VMs |
| Alerting | Prometheus alert rule detects Nginx down within 30 seconds |
| Slack integration | Alertmanager sends firing and resolved notifications to Slack |
| Failure simulation | Simulated backend failure and observed full alert lifecycle |

---

## Production Use Cases

| Scenario | How This Project Covers It |
|---|---|
| High traffic websites | Load balancer distributes requests across multiple backends |
| SSL termination | Reverse proxy handles all HTTPS, backends run HTTP internally |
| Error detection | Promtail captures 502 errors in real time, visible in Grafana |
| On-call alerting | Alertmanager sends Slack alert when Nginx goes down |
| Log analysis | Loki indexes logs by label, filterable by time and severity |
| Infrastructure visibility | Node Exporter gives per-VM CPU, memory, disk metrics |

---

## Phase 1 — Backend Web Servers (VM1 and VM3)

**Why two backend servers?**
A load balancer needs at least two backend servers to distribute traffic between them. VM1 is the first backend. VM3 is the second backend with identical configuration. After both are ready, VM2 will load balance between them.

> Perform all steps in this phase on both VM1 (192.168.253.128) and VM3 (192.168.253.130). Use the respective IP address of each VM in `server_name` inside the config.

### Step 1 — Install Nginx

```bash
dnf install nginx -y
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

Open firewall for HTTP and HTTPS and reload:

```bash
firewall-cmd --permanent --add-service=http --zone=public
firewall-cmd --permanent --add-service=https --zone=public
firewall-cmd --reload
```

### Step 2 — Deploy Website Template

Download a free HTML template from templatemo.com (Infinite Loop template). Remove the default Nginx welcome page from `/usr/share/nginx/html` and paste all template files into `/usr/share/nginx/html`.

```bash
firewall-cmd --reload
systemctl reload nginx
```

Verify: Open `http://192.168.253.128` and `http://192.168.253.130` in browser. Both should load the website.

### Step 3 — Add Self-Signed SSL Certificate

```bash
dnf install openssl -y
```

Generate self-signed certificate:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt
```

During generation, OpenSSL asks for country, organization, and common name (CN). In production, CN must match your domain name exactly. In this lab: `CN = www.infinitesite.com`

Copy certificate and key to default system locations on RHEL/CentOS:

```bash
cp server.key /etc/pki/tls/private/
cp server.crt /etc/pki/tls/certs/
```

### Step 4 — Configure Nginx for HTTPS

Nginx currently serves on HTTP port 80 only. We need to tell Nginx to use the SSL certificate and automatically redirect HTTP visitors to HTTPS.

Take backup of original config first:

```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bkp
vim /etc/nginx/nginx.conf
```

Add two server blocks inside the existing `http { }` block. Do not add another `http { }` wrapper — one already exists in nginx.conf.

```nginx
server {
listen 80;
listen [::]:80;
server_name 192.168.253.128;
return 301 https://$host$request_uri;
}

server {
listen 443 ssl;
listen [::]:443 ssl;
server_name 192.168.253.128;
ssl_certificate /etc/pki/tls/certs/server.crt;
ssl_certificate_key /etc/pki/tls/private/server.key;
ssl_protocols TLSv1.2 TLSv1.3;
root /usr/share/nginx/html;
index index.html;
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;
}
```

Test config syntax before applying — never skip this:

```bash
nginx -t
```

Fix SELinux context on certificate files:

```bash
restorecon -Rv /etc/pki/tls/
```

Reload Nginx:

```bash
systemctl reload nginx
```

Verify HTTPS is working:

```bash
curl -k https://192.168.253.128
```

If HTML content appears in output, HTTPS is working.

**Official Nginx Documentation:**
- HTTP module: https://nginx.org/en/docs/http/ngx_http_core_module.html
- SSL module: https://nginx.org/en/docs/http/ngx_http_ssl_module.html

### Step 5 — Local DNS Entry

```bash
vim /etc/hosts
```

Add:

```
192.168.253.128 www.infinitesite.com
192.168.253.130 www.infinitesite.com
```

Open `https://www.infinitesite.com` in browser. Website loads with SSL on both VMs. Add the same entry on your Windows host machine if testing from there.

---

## Phase 2 — Reverse Proxy and Load Balancer (VM2)

**What is a Reverse Proxy?**
A reverse proxy sits between internet users and your actual web servers. When a user types `www.infinitesite.com`, the request goes to the reverse proxy first. The reverse proxy forwards to the actual backend and returns the response to the user. The user never communicates directly with the backend server.

```
Internet → VM2 (reverse proxy) → VM1 or VM3 → response back to user
```

### Step 1 — Install Nginx on VM2

```bash
dnf install nginx -y
systemctl enable nginx
systemctl start nginx
systemctl status nginx
firewall-cmd --permanent --add-service=http --zone=public
firewall-cmd --permanent --add-service=https --zone=public
firewall-cmd --reload
```

### Step 2 — Set SELinux to Permissive on VM2

```bash
vim /etc/selinux/config
```

Change `SELINUX=enforcing` to `SELINUX=permissive` and reboot.

Or without reboot (temporary until next boot):

```bash
setenforce 0
```

### Step 3 — Generate SSL Certificate on VM2

VM2 is the entry point for all HTTPS traffic. It needs its own SSL certificate so browsers can establish a secure connection to VM2. VM2 then communicates with backends using the backends' own self-signed certificates internally.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/server.key -out /etc/pki/tls/certs/server.crt
```

When asked for Common Name (CN) enter: `www.infinitesite.com`

Set permissions:

```bash
chmod 600 /etc/pki/tls/private/server.key
chmod 644 /etc/pki/tls/certs/server.crt
```

### Step 4 — Configure Reverse Proxy with Load Balancing

```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bkp
vim /etc/nginx/nginx.conf
```

Add inside the existing `http { }` block:

```nginx
upstream backend_servers {
server 192.168.253.128:443;
server 192.168.253.130:443;
}

server {
listen 80;
listen [::]:80;
listen 443 ssl;
listen [::]:443 ssl;
server_name www.infinitesite.com;
ssl_certificate /etc/pki/tls/certs/server.crt;
ssl_certificate_key /etc/pki/tls/private/server.key;
location / {
proxy_pass https://backend_servers;
proxy_ssl_verify off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
}
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;
}
```

Test and reload:

```bash
nginx -t
systemctl reload nginx
```

Add to `/etc/hosts` on VM2:

```
192.168.253.129 www.infinitesite.com
```

Verify:

```bash
curl -I -k https://www.infinitesite.com
```

Expected: `HTTP/1.1 200 OK`

### Step 5 — Load Balancing Verification

**Load balancing methods in Nginx:**

| Method | How It Works | When to Use |
|---|---|---|
| Round Robin (default) | Requests go to each server in turn | Equal capacity servers |
| Least Connections | Sends to server with fewest active connections | Servers with different capacities |
| IP Hash | Same user IP always goes to same backend | Session data stored locally on backends |

**Verification:**

Open two terminal windows simultaneously.

On VM1:
```bash
tail -f /var/log/nginx/access.log
```

On VM3:
```bash
tail -f /var/log/nginx/access.log
```

Open `https://www.infinitesite.com` in incognito mode and refresh multiple times. Both VM1 and VM3 logs should show entries with `192.168.253.129` (VM2 IP) as the source. This confirms VM2 is forwarding requests to both backends alternately — Round Robin working correctly.

Example log entry:
```
192.168.253.129 - - "GET /img/infinite-loop-01.jpg" 200
```

---

## Phase 3 — Loki on VM4

Loki is a log aggregation system. It stores and indexes logs pushed to it by Promtail. Grafana queries Loki to display and search logs. Loki does not scrape logs — it only receives logs pushed from Promtail.

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/latest/download/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
mv loki-linux-amd64 /usr/local/bin/loki
chmod +x /usr/local/bin/loki
restorecon -v /usr/local/bin/loki
useradd --no-create-home --shell /usr/sbin/nologin loki
mkdir -p /var/lib/loki/chunks
mkdir -p /var/lib/loki/rules
chown -R loki:loki /var/lib/loki
firewall-cmd --permanent --add-port=3100/tcp
firewall-cmd --reload
```

Create config file at `/etc/loki-config.yaml`:

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

Official documentation: https://grafana.com/docs/loki/next/configure/

Create service at `/etc/systemd/system/loki.service`:

```ini
[Unit]
Description=Loki Service
After=network.target

[Service]
User=loki
Group=loki
WorkingDirectory=/var/lib/loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki-config.yaml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable loki
systemctl start loki
systemctl status loki
```

Verify:

```bash
curl http://192.168.253.136:3100/ready
```

Expected: `ready`

Check API before Promtail is installed:

```bash
curl http://192.168.253.136:3100/loki/api/v1/labels
```

Expected: `{"status":"success"}` — labels will appear after Promtail sends logs.

---

## Phase 4 — Promtail on VM2

**Why Promtail on VM2 and not VM4?**

VM2 is the load balancer. All incoming traffic passes through VM2 first. VM2 Nginx logs every request including 502 errors when backends go down. Installing Promtail on VM2 reads these logs directly from local disk — no network dependency, no NFS mount needed.

```
VM2 Promtail reads local logs → pushes to VM4 Loki over network → Grafana displays
```

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.9/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
sudo restorecon -v /usr/local/bin/promtail
sudo useradd --no-create-home --shell /usr/sbin/nologin promtail
sudo mkdir -p /var/lib/promtail
sudo mkdir -p /etc/promtail
sudo chown -R promtail:promtail /var/lib/promtail
sudo chown -R promtail:promtail /etc/promtail
sudo usermod -a -G nginx promtail
sudo chmod 755 /var/log/nginx/
```

Create config at `/etc/promtail-config.yaml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
positions:
  filename: /var/lib/promtail/positions.yaml
clients:
  - url: http://192.168.253.136:3100/loki/api/v1/push
scrape_configs:
  - job_name: nginx_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          host: lbserver
          __path__: /var/log/nginx/*.log
```

Create service at `/etc/systemd/system/promtail.service`:

```ini
[Unit]
Description=Promtail Log Collector
After=network.target

[Service]
User=promtail
Group=promtail
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail-config.yaml
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable promtail
systemctl start promtail
systemctl status promtail
firewall-cmd --permanent --add-port=9080/tcp
firewall-cmd --reload
```

Verify Promtail is reading logs:

```bash
journalctl -u promtail -f
```

Generate traffic by refreshing `https://www.infinitesite.com` in browser. Look for `Seeked` and `tail routine started` messages.

Verify labels reached Loki:

```bash
curl http://192.168.253.136:3100/loki/api/v1/labels
```

Expected: `{"status":"success","data":["filename","host","job"]}`

Verify positions file (created by Promtail automatically on first run):

```bash
cat /var/lib/promtail/positions.yaml
```

Expected output:
```
/var/log/nginx/access.log: "150208"
/var/log/nginx/error.log: "4217"
```

---

## Phase 5 — Prometheus on VM4

Prometheus is a metrics collection and storage system. It scrapes metrics from targets at regular intervals. Node Exporter exposes system metrics from each VM. Prometheus pulls these and stores them as time series data.

```bash
useradd --no-create-home --shell /usr/sbin/nologin prometheus
mkdir /etc/prometheus
mkdir /var/lib/prometheus
chown -R prometheus:prometheus /etc/prometheus
chown -R prometheus:prometheus /var/lib/prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.10.0/prometheus-3.10.0.linux-amd64.tar.gz
tar -xvf prometheus-3.10.0.linux-amd64.tar.gz
cd prometheus-3.10.0.linux-amd64
cp prometheus /usr/local/bin/
cp promtool /usr/local/bin/
restorecon -v /usr/local/bin/prometheus
restorecon -v /usr/local/bin/promtool
cp prometheus.yml /etc/prometheus/prometheus.yml
```

> Note: Prometheus uses tarball installation because no official DNF repository exists for RHEL. Grafana uses DNF because an official repository exists for it.

Create service at `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
firewall-cmd --permanent --add-port=9090/tcp --zone=public
firewall-cmd --reload
```

Access Prometheus at: `http://192.168.253.136:9090`

---

## Phase 6 — Node Exporter on All VMs (VM1, VM2, VM3, VM4)

Node Exporter collects system-level metrics from each machine — CPU, RAM, disk, network. It exposes these as an HTTP endpoint that Prometheus scrapes every 15 seconds. Run these steps on each VM separately.

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
cd node_exporter-1.10.2.linux-amd64
cp node_exporter /usr/local/bin/
restorecon -v /usr/local/bin/node_exporter
useradd --no-create-home --shell /usr/sbin/nologin node_exporter
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Create service at `/usr/lib/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --collector.systemd

[Install]
WantedBy=multi-user.target
```

> `--collector.systemd` flag enables collection of systemd service states. This is what allows Prometheus to know if Nginx service is active or stopped — critical for the alert rule to work.

```bash
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
firewall-cmd --permanent --add-port=9100/tcp --zone=public
firewall-cmd --reload
```

Verify metrics endpoint on each VM — a long list of metrics should appear:

```
http://192.168.253.128:9100/metrics
http://192.168.253.129:9100/metrics
http://192.168.253.130:9100/metrics
http://192.168.253.136:9100/metrics
```

Add all Node Exporters as scrape targets in Prometheus config on VM4:

```bash
vim /etc/prometheus/prometheus.yml
```

Add under `scrape_configs`:

```yaml
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - '192.168.253.128:9100'
          - '192.168.253.129:9100'
          - '192.168.253.130:9100'
          - '192.168.253.136:9100'
```

```bash
systemctl restart prometheus
```

Verify at `http://192.168.253.136:9090` → Status → Targets. All four Node Exporter targets should show state UP.

---

## Phase 7 — Grafana on VM4

```bash
vim /etc/yum.repos.d/grafana.repo
```

```ini
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

```bash
dnf makecache -y
dnf install grafana -y
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --reload
```

Access Grafana at: `http://192.168.253.136:3000`
Default login: `admin / admin` — change password on first login.

**Add Prometheus data source:**
Left sidebar → Connections → Data Sources → Add data source → Prometheus → URL: `http://192.168.253.136:9090` → Save & Test

**Add Loki data source:**
Add data source → Loki → URL: `http://192.168.253.136:3100` → Save & Test

**Dashboard A — System Health:**
Import community dashboard ID `1860` for Node Exporter metrics. If direct import fails because Grafana cannot reach grafana.com, download JSON on Windows from:
`https://grafana.com/api/dashboards/1860/revisions/latest/download`

Upload manually: Dashboards → Import → Upload JSON file → Select Prometheus → Import

**Dashboard B — Nginx Logs:**
Dashboards → New Dashboard → Add visualization → Select Loki → Switch to Code mode

Query: `{job="nginx"}` → Change visualization type to Logs.

**Useful Loki Queries:**

| Query | What it shows |
|---|---|
| `{job="nginx"}` | All Nginx logs |
| `{host="lbserver"}` | Logs from VM2 only |
| `{filename="/var/log/nginx/access.log"}` | Access logs only |
| `{job="nginx"} \|= "error"` | Error logs only |
| `{job="nginx"} \|= "502"` | 502 errors only — backend down |
| `{job="nginx"} \|= "404"` | 404 errors only |
| `{job="nginx"} != "200"` | All logs except successful requests |

---

## Phase 8 — Alertmanager on VM4

Alertmanager handles alerts sent by Prometheus. It deduplicates, groups, and routes alerts to receivers like Slack, email, or PagerDuty.

| Without Alertmanager | With Alertmanager |
|---|---|
| Prometheus stores metrics, no notification | Prometheus pushes alert to Alertmanager |
| You find out when users complain | Slack message sent within 30 seconds |

**Alertmanager components:**

| Component | What it does |
|---|---|
| Receiver | Where alert goes (Slack, Email, PagerDuty) |
| Route | Which alert goes to which receiver |
| Group | Multiple alerts combined into one notification |
| Inhibition | If critical alert fires, suppress related smaller alerts |
| Silences | Temporarily mute alerts during planned maintenance |

```bash
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar -xvf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64
mv alertmanager /usr/local/bin/
mv amtool /usr/local/bin/
chmod +x /usr/local/bin/alertmanager
chmod +x /usr/local/bin/amtool
restorecon -v /usr/local/bin/alertmanager
useradd --system --no-create-home --shell /usr/sbin/nologin alertmanager
mkdir -p /etc/alertmanager
mkdir -p /var/lib/alertmanager
chown -R alertmanager:alertmanager /etc/alertmanager
chown -R alertmanager:alertmanager /var/lib/alertmanager
chown alertmanager:alertmanager /usr/local/bin/alertmanager
chown alertmanager:alertmanager /usr/local/bin/amtool
```

Create config at `/etc/alertmanager/alertmanager.yml`:

```yaml
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'slack'
receivers:
- name: 'slack'
  slack_configs:
  - api_url: 'YOUR_SLACK_WEBHOOK_URL'
    channel: '#alerts'
    send_resolved: true
    title: '{{ .Status | toUpper }} - {{ .CommonLabels.alertname }}'
    text: |-
      {{ range .Alerts }}
      *Status:* {{ .Status }}
      *Instance:* {{ .Labels.instance }}
      *Summary:* {{ .Annotations.summary }}
      *Description:* {{ .Annotations.description }}
      {{ end }}
```

Create service at `/etc/systemd/system/alertmanager.service`:

```ini
[Unit]
Description=Alertmanager
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --storage.path=/var/lib/alertmanager
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable alertmanager
systemctl start alertmanager
systemctl status alertmanager
firewall-cmd --permanent --add-port=9093/tcp
firewall-cmd --reload
```

Verify:

```bash
curl http://localhost:9093/-/healthy
```

Expected: `OK`

> **Note:** Alertmanager is NOT a Prometheus scrape target. Prometheus does not pull metrics from Alertmanager — it pushes alerts to Alertmanager. Alertmanager will never appear on the Prometheus Targets page. This is expected and correct behavior.

Verify Alertmanager is connected to Prometheus:
`http://192.168.253.136:9090/api/v1/alertmanagers`

---

## Phase 9 — Alert Rules and Slack

### Slack Setup

1. Go to your Slack workspace and create a channel named `#alerts`
2. Go to `api.slack.com` → Your Apps → Create New App
3. Enable Incoming Webhooks
4. Add webhook to the `#alerts` channel
5. Copy the webhook URL (starts with `https://hooks.slack.com/services/`)
6. Add this URL to `/etc/alertmanager/alertmanager.yml` in the `api_url` field
7. Restart Alertmanager: `systemctl restart alertmanager`

### Create Prometheus Alert Rule

Create alert rules file at `/etc/prometheus/alert_rules.yml`:

```yaml
groups:
- name: nginx_alerts
  rules:
  - alert: NginxServiceDown
    expr: node_systemd_unit_state{name="nginx.service", state="active"} == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Nginx service is down"
      description: "Nginx service is down on {{ $labels.instance }}"
```

Update Prometheus config — edit `/etc/prometheus/prometheus.yml` and add at the end:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
rule_files:
  - "alert_rules.yml"
```

```bash
systemctl restart prometheus
```

Verify alert rule is loaded: `http://192.168.253.136:9090/alerts`

`NginxServiceDown` should appear with state `Inactive` — because Nginx is currently running.

---

## Phase 10 — Demo and Failure Simulation

### Demo Flow

1. Open Grafana Dashboard A — all VMs show green metrics
2. Open Grafana Dashboard B — live Nginx log stream showing 200 OK responses
3. Open Prometheus Alerts page — `NginxServiceDown` is Inactive
4. Stop Nginx on VM1:

```bash
systemctl stop nginx
```

**Wait and observe:**

| Time | What happens |
|---|---|
| First 15 seconds | Prometheus detects Nginx is down. Alert status = **Pending** |
| Next 15 seconds | Alert remains Pending (waiting for the 30s `for:` threshold) |
| After 30 seconds | Alert status changes to **Firing** |
| Immediately after | Alertmanager sends message to Slack `#alerts` |

5. Check Slack — firing notification received
6. Open Grafana Loki dashboard — Nginx logs still show 200 OK responses from VM3 (load balancer ensures zero downtime despite VM1 being down)
7. Start Nginx on VM1:

```bash
systemctl start nginx
```

8. Wait — Prometheus detects Nginx is running. Alert changes from Firing to Inactive
9. Check Slack — resolved notification received automatically

This complete cycle demonstrates end-to-end observability from failure detection to notification to recovery confirmation, with the load balancer ensuring zero downtime throughout.

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| Nginx 502 Bad Gateway | Backend server down or unreachable | `systemctl status nginx` on VM1 and VM3. Check `firewall-cmd --list-all` on backends |
| Loki service 203/EXEC error | Wrong SELinux context after moving binary from /tmp | `restorecon -v /usr/local/bin/loki` |
| Promtail permission denied on Nginx logs | promtail user not in nginx group | `usermod -a -G nginx promtail` and `chmod 755 /var/log/nginx/` |
| Grafana cannot reach Prometheus or Loki | Wrong IP in data source config | Use VM4 IP `192.168.253.136`, not localhost. Verify firewall port is open |
| Prometheus target showing DOWN | Firewall blocking port 9100 | `firewall-cmd --permanent --add-port=9100/tcp --zone=public && firewall-cmd --reload` |
| Alert not firing in Slack | Wrong webhook URL or Alertmanager not restarted | Verify `api_url` in alertmanager.yml. Run `systemctl restart alertmanager` |
| Alert not showing in Prometheus | Rule file not loaded or wrong path | Check `rule_files` path in prometheus.yml. Run `promtool check config /etc/prometheus/prometheus.yml` |
| nginx -t fails after config edit | Syntax error in nginx.conf | Check exact line number in error output. Restore backup: `cp /etc/nginx/nginx.conf.bkp /etc/nginx/nginx.conf` |

---

## Key Ports Reference

| VM | Ports |
|---|---|
| VM1 (nginxwebserver) | 80, 443, 9100 |
| VM2 (lbserver) | 80, 443, 9080, 9100 |
| VM3 (backendserver2) | 80, 443, 9100 |
| VM4 (monitoring) | 3000, 3100, 9090, 9093, 9100 |

---

## What's Next

This setup can be further extended with:

- **Blackbox Exporter** — Monitor website URL availability externally. Alert when website returns non-200 status code
- **Email alerts** — Add email receiver in Alertmanager alongside Slack
- **Log retention** — Configure Loki to auto-delete logs older than 30 days
- **Ansible automation** — Deploy Node Exporter across all VMs using one Ansible playbook instead of repeating manual steps on each VM
- **Multi-backend scaling** — Add VM5 as third backend, update upstream block in VM2, observe Round Robin across three servers
- **Grafana alerting** — Create Grafana alerts directly on dashboards for disk space thresholds

---

*Document prepared as part of DevOps Home Lab*
