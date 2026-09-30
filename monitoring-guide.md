# 📊 Complete VPS Monitoring Setup Guide

**Prometheus + Grafana + Loki + Promtail + Node Exporter — Free Forever, Self-Hosted**

From a fresh Hostinger VPS to enterprise-grade observability with real-time dashboards and email alerts. Zero SaaS cost.

---

## 📖 Table of Contents

- [Overview](#overview)
- [Tools Explained](#tools-explained)
- [Prerequisites](#prerequisites)
- **Installation**
  - [Step 1: Install Docker](#step-1-install-docker)
  - [Step 2: Create Folder Structure](#step-2-create-folder-structure)
  - [Step 3: Configuration Files](#step-3-configuration-files)
  - [Step 4: Docker Compose File](#step-4-docker-compose-file)
  - [Step 5: First Start](#step-5-first-start)
- **Troubleshooting**
  - [Fix: Port Conflict](#fix-port-conflict)
  - [Fix: Loki Permission Denied](#fix-loki-permission-denied)
- **Grafana Setup**
  - [Step 6: Access Grafana](#step-6-access-grafana)
  - [Step 7: Import VPS Dashboard](#step-7-import-vps-dashboard)
  - [Step 8: Verify the Stack](#step-8-verify-the-stack)
- **Logs Pipeline**
  - [Step 9: Configure Promtail for PM2 Logs](#step-9-configure-promtail-for-pm2-logs)
  - [Step 10: Explore Logs in Grafana](#step-10-explore-logs-in-grafana)
- **Custom Dashboard**
  - [Step 11: Build a Production Logs Dashboard](#step-11-build-a-production-logs-dashboard)
- **Email Alerts**
  - [Step 12: Configure SMTP](#step-12-configure-smtp)
  - [Step 13: Contact Point + Route](#step-13-contact-point--route)
  - [Step 14: Alert Rules](#step-14-alert-rules)
- **Next Steps**
  - [Nginx Reverse Proxy + Domain](#nginx-reverse-proxy--domain)
  - [Expose Node.js App Metrics](#expose-nodejs-app-metrics)
- [Command Cheatsheet](#command-cheatsheet)

---

## Overview

### What you'll build

| Feature | Description |
|---|---|
| 📈 **VPS Metrics Dashboard** | CPU, RAM, disk, network, load — updated every 15 seconds |
| 📝 **Live Log Aggregation** | Search all Node.js/PM2 logs from one UI — no more `ssh + tail` |
| 🔍 **Custom Dashboards** | Errors per app, error rate over time, live feeds — with variables & filters |
| 🚨 **Email Alerts** | Automatic Gmail alerts on CPU / RAM / disk / errors / app-down |

**Tech stack:** Prometheus · Grafana · Loki · Promtail · Node Exporter · Docker Compose

---

## Tools Explained

| Tool | What it watches | Question it answers |
|---|---|---|
| **Prometheus** 📊 | Numbers / metrics | Is my app slow or overloaded? |
| **Grafana** 📈 | Visualizes everything | Show me graphs |
| **Loki** 📝 | Log text | What did my app print when it crashed? |
| **Promtail** 🚚 | Ships log files to Loki | Pipe my `.log` files into Loki |
| **Node Exporter** 🖥️ | VPS host metrics | CPU / RAM / disk of the VPS itself |
| **Sentry** 🐛 | Errors / exceptions | Paid SaaS — skipping (Loki covers this) |

> 💡 **Mental model:** Prometheus + Node Exporter collect metrics. Promtail ships logs to Loki. Grafana visualizes both. Docker Compose runs everything together.

---

## Prerequisites

- A Linux VPS (Ubuntu 22.04 / 24.04 recommended)
- Minimum **2 GB RAM** (4–8 GB comfortable)
- Root or sudo access
- Ports `3003`, `9090`, `3100` free (we'll adjust if not)

---

## Step 1: Install Docker

Skip this if Docker is already installed (check with `docker --version`).

```bash
# Update system
apt update && apt upgrade -y

# Install Docker (official one-liner)
curl -fsSL https://get.docker.com | sh

# Verify
docker --version
docker compose version
```

> ✅ You should see `Docker version 27.x` or newer, and `Docker Compose version v2.x`.

---

## Step 2: Create Folder Structure

```bash
mkdir -p /home/monitoring
cd /home/monitoring
mkdir -p prometheus loki promtail grafana/provisioning/datasources
```

Final structure:

```
/home/monitoring/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasources.yml
```

---

## Step 3: Configuration Files

### 3.1 `prometheus/prometheus.yml`

Tells Prometheus what to scrape.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Your Node.js API (add /metrics endpoint later)
  - job_name: 'my-api'
    static_configs:
      - targets: ['host.docker.internal:5000']
```

### 3.2 `loki/loki-config.yml`

Loki's storage + WAL config. The `wal.dir` is important — without it Loki crashes.

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  wal:
    dir: /loki/wal
    enabled: true
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
```

### 3.3 `promtail/promtail-config.yml`

Tails log files and ships to Loki. Extracts labels from PM2 filenames.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # PM2 logs from all users
  - job_name: pm2-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: pm2
          __path__: /logs/pm2/*/*.log
    pipeline_stages:
      - regex:
          source: filename
          expression: '/logs/pm2/(?P<pm2_user>[^/]+)/(?P<app>[^-]+(?:-[^-]+)*?)-(?P<stream>out|error)(?:-\d+)?\.log'
      - labels:
          pm2_user:
          app:
          stream:

  # App-specific logs
  - job_name: app-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: app
          __path__: /logs/apps/*/*.log
    pipeline_stages:
      - regex:
          source: filename
          expression: '/logs/apps/(?P<app>[^/]+)/(?P<logfile>[^/]+)\.log'
      - labels:
          app:
          logfile:
```

### 3.4 `grafana/provisioning/datasources/datasources.yml`

Auto-provisions Prometheus + Loki as datasources on first Grafana start.

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    url: http://loki:3100
```

---

## Step 4: Docker Compose File

Master file that runs all 5 services. Save as `/home/monitoring/docker-compose.yml`.

> ⚠️ **Before saving, edit these values:**
> - `GF_SECURITY_ADMIN_PASSWORD` — pick a strong password
> - The `promtail` volume mounts — match your VPS user directories (list them with `find /home /root -type d -name ".pm2" 2>/dev/null`)

```yaml
networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  loki_data:
  grafana_data:

services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
    networks:
      - monitoring

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: unless-stopped
    user: "10001:10001"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    ports:
      - "3100:3100"
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      # PM2 logs — add one line per user found on your VPS
      - /root/.pm2/logs:/logs/pm2/root:ro
      - /home/USER1/.pm2/logs:/logs/pm2/USER1:ro
      - /home/USER2/.pm2/logs:/logs/pm2/USER2:ro
      # App-specific logs (optional)
      - /home/USER1/htdocs/app.example.com/logs:/logs/apps/app-example:ro
    command: -config.file=/etc/promtail/promtail-config.yml
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=CHANGE_ME_STRONG_PASSWORD
      - GF_USERS_ALLOW_SIGN_UP=false
      # SMTP (fill later after Gmail app password)
      - GF_SMTP_ENABLED=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3003:3000"
    networks:
      - monitoring
```

> ℹ️ **Why port `3003`?** Because `3000` is very often used by other Node.js apps managed by PM2. If you know `3000` is free on your VPS, feel free to use it.

---

## Step 5: First Start

```bash
cd /home/monitoring
docker compose up -d
docker compose ps
```

You should see all 5 containers listed with status `Up`.

> ✅ **Checkpoint:** `docker compose ps` shows `grafana`, `loki`, `node-exporter`, `prometheus`, `promtail` all running.

---

## Fix: Port Conflict

If `docker compose up -d` fails with `address already in use`:

```bash
# Find what's using the port (e.g. 3000)
ss -tlnp | grep ':3000'
lsof -i :3000

# See all occupied ports
ss -tlnp | awk '{print $4}' | grep -oP ':\K\d+' | sort -n | uniq
```

Pick a free port, then edit `docker-compose.yml`:

```yaml
# Change Grafana port mapping
    ports:
      - "3003:3000"  # host:container — pick a free host port
```

```bash
docker compose down
docker compose up -d
```

---

## Fix: Loki Permission Denied

If `docker logs loki` shows `creating WAL folder at "/wal": mkdir wal: permission denied`:

```bash
docker compose down

# Fix volume ownership (Loki runs as UID 10001)
docker run --rm -v monitoring_loki_data:/loki busybox chown -R 10001:10001 /loki

docker compose up -d
sleep 5
docker logs loki --tail 20
```

Also verify these two settings are present:
- `docker-compose.yml` → `loki:` service → `user: "10001:10001"`
- `loki-config.yml` → under `ingester:` → `wal: { dir: /loki/wal, enabled: true }`

---

## Step 6: Access Grafana

Open in browser:

```
http://YOUR_VPS_IP:3003
```

Login with:
- Username: `admin`
- Password: whatever you set in `GF_SECURITY_ADMIN_PASSWORD`

---

## Step 7: Import VPS Dashboard

Grafana has a huge library of prebuilt dashboards. **Node Exporter Full (ID 1860)** is the gold standard for VPS metrics.

**Step 1 — Import dashboard**

`Grafana Sidebar` → **Dashboards** → **New** → **Import**

**Step 2 — Enter dashboard ID**

In the "Grafana.com dashboard URL or ID" input, type:

```
1860
```

Click the blue **Load** button next to the input.

**Step 3 — Configure & import**

- **Name:** Node Exporter Full (leave default)
- **Folder:** Dashboards
- **Prometheus dropdown:** select `Prometheus`

Click blue **Import**.

> 🎉 You'll instantly see CPU, RAM, disk, network, uptime, systemd, storage — all live and updating every 15s.

---

## Step 8: Verify the Stack

### Prometheus targets

```
http://YOUR_VPS_IP:9090/targets
```

Expected:
- `prometheus` → **UP** ✅
- `node-exporter` → **UP** ✅
- `my-api` → **DOWN** (expected — your app doesn't expose `/metrics` yet)

### Loki readiness

```
http://YOUR_VPS_IP:3100/ready
```

Should return `ready` after ~15–30 seconds.

---

## Step 9: Configure Promtail for PM2 Logs

### 9.1 Discover your log locations

```bash
# Find all PM2 directories across all users
find /home /root -type d -name ".pm2" 2>/dev/null

# Find all log directories
find /home /root -type d -name "logs" 2>/dev/null | head -30
```

### 9.2 Add volume mounts

Edit `docker-compose.yml` → `promtail` → `volumes:` and add one line per PM2 user found:

```yaml
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      # One line per user found by the find command above
      - /root/.pm2/logs:/logs/pm2/root:ro
      - /home/app1/.pm2/logs:/logs/pm2/app1:ro
      - /home/app2/.pm2/logs:/logs/pm2/app2:ro
      # ... add all users
      # Optional: app-specific log folders
      - /home/app1/htdocs/app1.com/logs:/logs/apps/app1:ro
```

### 9.3 Restart & verify

```bash
cd /home/monitoring
docker compose up -d promtail
docker logs promtail --tail 30
```

You should see multiple `tail routine: started` lines — one per `.log` file it's tailing.

---

## Step 10: Explore Logs in Grafana

**Step 1 — Open the Explore view**

`Grafana Sidebar` → **Explore** (compass icon)

**Step 2 — Select Loki data source**

Top dropdown → **Loki**

**Step 3 — Run useful queries**

Toggle to **Code** mode (top-right of query editor) and try:

| What to see | Query |
|---|---|
| All PM2 logs | `{job="pm2"}` |
| Only error streams | `{job="pm2", stream="error"}` |
| One specific app | `{job="pm2", app="my-api"}` |
| Search a phone / user ID | `{job="pm2"} \|= "917303988436"` |
| Errors (case-insensitive) | `{job="pm2"} \|~ "(?i)error\|failed\|exception"` |

> ⚡ **Live tail:** click the **Live** button (top-right) to see logs stream in real-time — no refresh needed. Perfect for watching a fix deploy.

---

## Step 11: Build a Production Logs Dashboard

**Step 0 — Create the dashboard**

`Sidebar` → **Dashboards** → **New** → **New dashboard** → **+ Add visualization** → pick **Loki**

Rename to **Production Logs — Overview** in the right sidebar.

### Panel 1 — Total Errors (Stat)

- **Visualization:** Stat
- **Data source:** Loki
- **Query:**

```
sum(count_over_time({job="pm2", stream="error"}[$__range]))
```

- **Standard options → Color scheme →** Single color → red
- **Title:** Total Errors — Last Hour

### Panel 2 — Errors per App (Bar chart or Table)

- **Visualization:** Bar chart *or* Table (Table is cleaner for 10+ apps)
- **Query:**

```
sum by (app) (count_over_time({job="pm2", stream="error"}[$__range]))
```

- **Query options → Type: Instant** (very important — otherwise the chart is broken)

### Panel 3 — Error Rate Over Time (Time series)

- **Visualization:** Time series (default)
- **Query:**

```
sum by (app) (rate({job="pm2", stream="error"}[5m]))
```

### Panel 4 — Live Error Feed (Logs)

- **Visualization:** Logs
- **Query:**

```
{job="pm2", stream="error"}
```

- Right panel → Logs section → toggle on: **Time**, **Unique labels**, **Wrap lines**, **Prettify JSON**
- **Order:** Descending (newest first)

### Panel 5 — Log Volume per App

```
sum by (app) (rate({job="pm2"}[5m]))
```

### Panel 6 — Top Errors by App (Table)

- **Visualization:** Table
- **Query (Instant):**

```
topk(10, sum by (app, pm2_user) (count_over_time({job="pm2", stream="error"}[$__range])))
```

### Bonus — Dashboard variable to filter by app

`Top-right ⚙️ Settings` → **Variables** → **+ Add variable**

- **Name:** `app`
- **Type:** Query
- **Data source:** Loki
- **Query type:** Label values
- **Label:** `app`
- **Include All option:** ✅

Then edit each panel and change queries to use `app=~"$app"`:

```
{job="pm2", app=~"$app", stream="error"}
```

> 💾 **Save the dashboard** (💾 top-right) → name it `Production Logs — Overview`. Set auto-refresh to **30s** (dropdown top-right).

---

## Step 12: Configure SMTP

### 12.1 Get a Gmail App Password

> ⚠️ Gmail does **not** allow apps to sign in with your regular password. You must generate a special 16-character app password.

1. Go to https://myaccount.google.com/apppasswords
2. If not available, enable **2-Step Verification** first at https://myaccount.google.com/security
3. App name: `Grafana Alerts` → **Create**
4. Copy the 16 characters (e.g. `abcd efgh ijkl mnop`)
5. Remove spaces when using it → `abcdefghijklmnop`

### 12.2 Add SMTP config to `docker-compose.yml`

Under `grafana:` → `environment:`, add these lines (replace your email + app password):

```yaml
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.gmail.com:587
      - GF_SMTP_USER=YOUR_EMAIL@gmail.com
      - GF_SMTP_PASSWORD=YOUR_16_CHAR_APP_PASSWORD
      - GF_SMTP_FROM_ADDRESS=YOUR_EMAIL@gmail.com
      - GF_SMTP_FROM_NAME=Grafana Alerts
      - GF_SMTP_STARTTLS_POLICY=MandatoryStartTLS
      - GF_SMTP_SKIP_VERIFY=false
```

### 12.3 Restart & verify

```bash
cd /home/monitoring
docker compose up -d grafana
sleep 5
docker logs grafana 2>&1 | grep -i smtp
```

Expected — you'll see 8 lines confirming SMTP env vars loaded.

---

## Step 13: Contact Point + Route

### 13.1 Create the contact point

`Grafana Sidebar` → **Alerting** → **Contact points** → **+ Add contact point**

- **Name:** `My Email`
- **Integration:** Email
- **Addresses:** your Gmail (can be a different receiving email)
- Click **Test** → **Predefined** → **Send test notification**

> 📧 Within 30 seconds you'll get a beautiful **[FIRING:1] TestAlert** email from "Grafana Alerts". Check spam if missing.

Click **Save contact point**.

### 13.2 Route all alerts to it

`Alerting` → **Notification policies** tab → **Default policy** → **Edit**

- **Default contact point:** `My Email`
- Click **Update default policy**

---

## Step 14: Alert Rules

Common pattern for every rule:

1. **Alerting → Alert rules → + New alert rule**
2. Fill **Name** + **Data source** + **Query** (Code mode)
3. **Alert condition:** set threshold (IS ABOVE / IS BELOW)
4. **Folder:** `Production Alerts` (create once, reuse)
5. **Evaluation group:** `every-1-min`, interval `1m` (create once, reuse)
6. **Pending period:** `5m` for slow metrics, `2m` for fast ones
7. **Contact point:** `My Email`
8. Add **Summary** + **Description**
9. **Save rule and exit**

### 🔴 Rule 1 — High CPU

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- **Data source:** Prometheus &nbsp; **Condition:** IS ABOVE **80**
- **Summary:** CPU usage above 80% for 5 minutes
- **Description:** `Current CPU: {{ $values.C.Value }}%. Check what's running: pm2 list`

### 🔴 Rule 2 — High RAM

```
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

- **Prometheus &nbsp; IS ABOVE 85**
- **Summary:** RAM usage above 85% for 5 minutes
- **Description:** `Current RAM: {{ $values.C.Value }}%. Restart heavy apps or upgrade.`

### 🔴 Rule 3 — Disk Almost Full

```
(1 - (node_filesystem_avail_bytes{mountpoint="/",fstype!="tmpfs"} / node_filesystem_size_bytes{mountpoint="/",fstype!="tmpfs"})) * 100
```

- **Prometheus &nbsp; IS ABOVE 85**
- **Summary:** Root disk is above 85% full

### 🔴 Rule 4 — Error Spike

```
sum by (app) (rate({job="pm2", stream="error"}[5m]))
```

- **Loki &nbsp; IS ABOVE 50** (errors/second per app — tune to your baseline)
- **Pending period:** `2m` (faster reaction)
- **Summary:** Error spike detected in an app

### 🔴 Rule 5 — App Silent (crashed?)

```
sum by (app) (count_over_time({job="pm2"}[10m]))
```

- **Loki &nbsp; IS BELOW 1** (zero logs in 10 min = likely down)
- **Pending period:** `5m`
- **Description:** `App {{ $labels.app }} produced 0 logs in 10 min. Run: pm2 list`

> ⏩ **Speed tip:** instead of building each rule from scratch, use the **⋯ → Duplicate** option on an existing rule and just change the name + query + threshold.

> ✅ **You now have full production observability:** real-time metrics, log search, custom dashboards, and email alerts on every failure mode — all self-hosted, all free.

---

## Nginx Reverse Proxy + Domain

Right now Grafana is exposed on `http://IP:3003`. To access it as `https://monitor.yourdomain.com`:

1. Point an A record: `monitor.yourdomain.com` → your VPS IP
2. Install Nginx + Certbot: `apt install nginx certbot python3-certbot-nginx -y`
3. Create `/etc/nginx/sites-available/monitor`:

```nginx
server {
    listen 80;
    server_name monitor.yourdomain.com;

    location / {
        proxy_pass http://localhost:3003;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # Grafana Live (WebSocket)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

```bash
ln -s /etc/nginx/sites-available/monitor /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d monitor.yourdomain.com
```

Then in `docker-compose.yml`, add these Grafana env vars so URLs generate correctly:

```yaml
      - GF_SERVER_ROOT_URL=https://monitor.yourdomain.com
      - GF_SERVER_DOMAIN=monitor.yourdomain.com
```

Optionally, add firewall rules so ports 3003 / 9090 / 3100 are only accessible locally:

```bash
ufw deny 3003
ufw deny 9090
ufw deny 3100
ufw allow 80
ufw allow 443
ufw allow 22
ufw enable
```

---

## Expose Node.js App Metrics

To fix the Prometheus "my-api DOWN" target and get requests/sec, latency, error rate per endpoint, add `prom-client` to your Node.js apps:

```bash
npm install prom-client
```

```javascript
// server.js
const express = require('express');
const client = require('prom-client');

const app = express();
const register = new client.Registry();
client.collectDefaultMetrics({ register });

// Custom counter
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
});
register.registerMetric(httpRequestsTotal);

// Middleware to count requests
app.use((req, res, next) => {
  res.on('finish', () => {
    httpRequestsTotal.labels(req.method, req.route?.path || req.path, res.statusCode).inc();
  });
  next();
});

// /metrics endpoint (scraped by Prometheus)
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(5000);
```

Update `prometheus.yml` to scrape your app's real address (Docker on Linux can't use `host.docker.internal` by default). Either use `--add-host` or your VPS's private/public IP:

```yaml
  - job_name: 'my-api'
    static_configs:
      - targets: ['172.17.0.1:5000']  # docker0 bridge IP on Linux
```

Restart Prometheus:

```bash
docker compose restart prometheus
```

Then import the dashboard **ID 11159** (Node.js Application Dashboard) in Grafana — you'll get requests/sec, response time percentiles, error rate, event loop lag, GC pauses, memory usage, and per-route breakdowns instantly.

---

## Command Cheatsheet

| Task | Command |
|---|---|
| Start everything | `cd /home/monitoring && docker compose up -d` |
| Stop everything | `docker compose down` |
| Restart one service | `docker compose restart grafana` |
| View a service's logs | `docker logs grafana --tail 50 -f` |
| Update all images | `docker compose pull && docker compose up -d` |
| Free disk from old images | `docker system prune -a --volumes` |
| Check what runs on a port | `ss -tlnp \| grep ':3003'` |
| Find all PM2 log dirs | `find /home /root -type d -name ".pm2" 2>/dev/null` |

---

## 🎉 You're Done

You now have a **production-grade observability stack** running on your own VPS:

- 📊 Real-time metrics dashboards
- 📝 Centralized log search across all apps
- 🚨 Automated email alerts on every failure mode
- 💰 Zero monthly cost

Bookmark this doc, share with your team, use it on every new VPS.

---

*📊 VPS Monitoring Guide — Prometheus + Grafana + Loki. Built from a real production setup. No SaaS lock-in, no monthly bills.*
