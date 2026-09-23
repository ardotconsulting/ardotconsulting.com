---
layout: post
title: "Building a Self-Hosted Monitoring Stack with Uptime Kuma and Grafana"
date: 2026-10-24
author: "ARDOT Consulting"
tags: [monitoring, uptime-kuma, grafana, self-hosting, open-source, docker, observability]
excerpt: "If a service goes down at 2 AM, how long before you know? Statuspage charges $29+ per month just to tell your customers. Datadog bills by the host and spirals fast. Here's how to build a complete monitoring stack with Uptime Kuma and Grafana — open source, self-hosted, and sending you alerts the moment something breaks."
---

# Building a Self-Hosted Monitoring Stack with Uptime Kuma and Grafana

Here's a question that catches a lot of business owners off guard: when your website, your CRM, or your order system goes down at 2 AM, how long does it take you to find out?

If the answer is "a customer tells us in the morning," you have a monitoring gap. A gap that costs you sales, trust, and sleep. Most businesses solve this with a SaaS monitoring tool — Pingdom, Statuspage, Datadog, Better Stack — and those tools work fine. But they also bill you every month, per check or per host, and the price climbs as you add more things to watch.

A small business monitoring a website, an API, a mail server, and a self-hosted tool like n8n or Odoo can easily spend $50–$150 a month on monitoring SaaS. Statuspage alone starts at $29/month just to show a public status page. Datadog's infrastructure monitoring runs $15 per host per month — and "host" means every server, every container, every cloud instance. Four servers and you're at $60/month before you've configured a single alert.

There's a self-hosted alternative that does the same job for the cost of a single VPS. In this guide, we'll build a monitoring stack with two open-source tools: **Uptime Kuma** for uptime checks and alerting, and **Grafana** with **Prometheus** for metrics, dashboards, and historical data. By the end, you'll have alerts that wake you up when things break, dashboards that show you what's happening, and a status page your customers can check — all on your own server, with no per-check or per-host fees.

## Why Two Tools Instead of One?

Uptime Kuma and Grafana do different things, and they're better together than either is alone.

**Uptime Kuma** is an uptime monitor. It pings your services at regular intervals — every 30 seconds, every minute, whatever you set — and if a service stops responding, it sends you an alert. It supports HTTP, HTTPS, TCP, DNS, ping, database queries, and even Docker container health checks. It also includes a built-in status page that you can make public or keep private. Think of it as a self-hosted replacement for Pingdom, UptimeRobot, or Better Stack.

**Grafana** is a visualization and dashboarding tool. It pulls metrics from data sources — Prometheus is the most common — and renders them as graphs, gauges, heat maps, and alerts. It doesn't check whether your services are up (that's Uptime Kuma's job), but it shows you *how* your services are performing: CPU usage, memory, disk space, request latency, error rates over time. Think of it as a self-hosted replacement for the dashboards you'd get from Datadog or New Relic.

Together, they cover the two questions you need answered:

1. **Is it up?** → Uptime Kuma
2. **How's it performing?** → Grafana

You can run either one alone. But if you're already setting up a monitoring server, installing both gives you the full picture for almost no extra effort.

## What You'll Need

- A VPS with at least 2 GB of RAM (4 GB is comfortable). A $10–$20/month server from any provider will handle this easily.
- Docker and Docker Compose installed
- A domain name or subdomain for your monitoring dashboards (e.g., `status.yourcompany.com`)
- About an hour of setup time

You should already be using Docker for your other self-hosted tools. If you've followed our previous guides on self-hosting n8n, Odoo, or Mattermost, you know the drill.

## Part 1: Uptime Kuma — Uptime Checks and Alerts

Uptime Kuma is the simplest monitoring tool you'll ever set up. One container, a web UI, and you're adding monitors within minutes.

### Step 1: Create the Docker Compose File

Create a directory for your monitoring stack:

```bash
mkdir -p /opt/monitoring
cd /opt/monitoring
```

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./uptime-kuma-data:/app/data
```

### Step 2: Start It Up

```bash
docker compose up -d
```

Open `http://your-server-ip:3001` in your browser. You'll be prompted to create an admin account. That's it — no database to configure, no config files to edit. Uptime Kuma uses SQLite under the hood and stores everything in the volume you mounted.

### Step 3: Add Your First Monitor

Click "Add New Monitor" and you'll see a form. Here's how to configure the most common monitor types:

**Website (HTTP/HTTPS):**
- Monitor Type: HTTP(s)
- Friendly Name: "Company Website"
- URL: `https://www.yourcompany.com`
- Heartbeat Interval: 60 seconds
- Retries: 2 (wait for 2 failed checks before alerting — avoids false positives from brief network blips)

**API Endpoint:**
- Monitor Type: HTTP(s) - Keyword
- URL: `https://api.yourcompany.com/health`
- Keyword: `"status":"ok"` (Kuma checks that this string appears in the response body)
- Heartbeat Interval: 30 seconds

This keyword monitor is important. A simple HTTP 200 response doesn't tell you the app is healthy — it might be returning an error page with a 200 status. Checking for a specific string in the response (like a health endpoint that returns `{"status":"ok"}`) gives you a real signal.

**Database (PostgreSQL):**
- Monitor Type: PostgreSQL
- Connection String: `postgres://user:password@db-host:5432/yourdb`
- Heartbeat Interval: 60 seconds

This is useful if you're self-hosting NocoDB, Odoo, or anything else with a PostgreSQL backend. Uptime Kuma will run a simple `SELECT 1` query and confirm the database responds.

**Docker Container:**
- Monitor Type: Docker Container
- Container ID/Name: `n8n` (or whatever your container is called)
- Heartbeat Interval: 30 seconds

This checks whether a specific Docker container is running. If n8n crashes and the container stops, Uptime Kuma alerts you — even if the host server is fine.

### Step 4: Configure Alert Notifications

Uptime Kuma supports a long list of notification channels. The ones most useful for a small business:

- **Email (SMTP)** — Set up with your existing mail server or any SMTP relay. Use a dedicated monitoring email address so alerts don't get buried.
- **Telegram** — Create a bot via BotFather, get the bot token and a chat ID, and Uptime Kuma sends alerts to a Telegram channel or group. This is the fastest way to get alerts on your phone without an app.
- **Webhook** — Send the alert to n8n, which can then route it anywhere: a Mattermost channel, an SMS gateway, a phone call via a VoIP API.
- **Signal, Slack, Discord, Pushbullet, Pushover** — All supported natively.

Here's the practical setup: configure **both** email and Telegram. Email gives you a searchable history. Telegram gives you an immediate push notification on your phone. If a service goes down at 3 AM, the Telegram notification wakes you up. If you need to look back at when it happened and how long it lasted, the email thread has the full timeline.

### Step 5: Set Up a Status Page

Uptime Kuma includes a built-in status page that you can publish at a public URL. This is your self-hosted replacement for Statuspage.io ($29–$99/month).

Go to Status Pages → Add Status Page. Give it a name, select which monitors to display, and choose a slug (e.g., `status`). The page will be available at `http://your-server-ip:3001/status/status` or, once you set up a reverse proxy, at `https://status.yourcompany.com`.

The status page shows:
- Each monitored service with its current status (up/down/maintenance)
- Uptime percentage over the last 30 days
- Response time graph
- Incident history

You can customize the logo, colors, and footer text. For a small business, this gives you a professional-looking status page that reassures customers and partners — without paying a monthly fee for the privilege.

## Part 2: Grafana + Prometheus — Metrics and Dashboards

Uptime Kuma tells you *whether* your services are up. Grafana tells you *how they're doing* — CPU load, memory usage, disk space, network traffic, and any custom metric your applications expose.

To feed Grafana, you need a metrics collector. **Prometheus** is the standard choice. It scrapes metrics from your servers and applications at regular intervals, stores them in a time-series database, and Grafana queries that database to render dashboards.

### Step 1: Add Prometheus and Grafana to Your Compose File

Extend your `docker-compose.yml`:

```yaml
version: "3.8"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./uptime-kuma-data:/app/data

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus-data:/prometheus

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=your-strong-password-here
      - GF_USERS_ALLOW_SIGN_UP=false

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
```

### Step 2: Configure Prometheus

Create the Prometheus config file:

```bash
mkdir -p /opt/monitoring/prometheus
```

Create `/opt/monitoring/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

This tells Prometheus to scrape metrics from `node-exporter` (which monitors the host server's CPU, memory, disk, and network) and from itself. If you run node-exporter on other servers, add them as additional targets.

**Node-exporter** is the key piece for server monitoring. It runs on each server you want to monitor and exposes hardware and OS metrics in a format Prometheus can scrape. In the compose file above, it's running on the monitoring server itself. To monitor additional servers, install node-exporter on each one and add their IP addresses to the Prometheus config.

### Step 3: Start the Stack

```bash
docker compose up -d
```

You now have four containers running:
- Uptime Kuma on port 3001
- Prometheus on port 9090
- Grafana on port 3000
- Node-exporter on port 9100

### Step 4: Connect Grafana to Prometheus

Open `http://your-server-ip:3000`. Log in with `admin` and the password you set in the compose file.

1. Go to **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. URL: `http://prometheus:9090` (Docker networking — Grafana talks to Prometheus by container name)
4. Click **Save & Test** — you should see a green "Data source is working" message

### Step 5: Import a Dashboard

You don't need to build dashboards from scratch. Grafana has a library of community-built dashboards you can import with a few clicks.

1. Go to **Dashboards → Import**
2. Enter dashboard ID **1860** — this is the "Node Exporter Full" dashboard, one of the most popular. It shows CPU, memory, disk, network, and system metrics in a clean layout.
3. Select your Prometheus data source
4. Click **Import**

You now have a full server monitoring dashboard. It shows:

- CPU usage by core and by mode (user, system, I/O wait)
- Memory usage (used, available, cached, buffers)
- Disk space and disk I/O per mount point
- Network traffic per interface
- System load average
- Uptime

This is the same information you'd get from a Datadog host dashboard — but without the per-host billing.

### Step 6: Set Up Grafana Alerts

Grafana can also send alerts, complementing Uptime Kuma. Where Uptime Kuma alerts on "is the service responding," Grafana alerts on "is a metric crossing a threshold."

Common alert rules:

- **Disk space below 15%** — `node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.15`
- **CPU usage above 90% for 5 minutes** — `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90`
- **Memory usage above 90%** — `(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.90`

Configure the notification channel (email, Telegram, webhook — same options as Uptime Kuma, plus Grafana's own alerting system with escalation rules). The disk space alert is the most important one — a full disk will take down every service on the server, and it's the most common cause of self-hosted service outages.

## Part 3: Nginx Reverse Proxy and TLS

You don't want to access your monitoring tools by IP and port number. Set up Nginx with Let's Encrypt certificates to give each tool a proper domain:

```nginx
# /etc/nginx/sites-available/status.yourcompany.com
server {
    server_name status.yourcompany.com;
    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# /etc/nginx/sites-available/monitor.yourcompany.com
server {
    server_name monitor.yourcompany.com;
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the sites and get TLS certificates:

```bash
ln -s /etc/nginx/sites-available/status.yourcompany.com /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/monitor.yourcompany.com /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d status.yourcompany.com -d monitor.yourcompany.com
```

Now your status page is at `https://status.yourcompany.com` and your Grafana dashboards at `https://monitor.yourcompany.com`.

**Security note:** Protect Grafana with something stronger than the default admin login. Enable HTTP basic auth at the Nginx level as a second layer, or restrict access to a VPN / WireGuard network. Grafana has powerful access to your infrastructure metrics — don't leave it open to the internet.

## The Cost Comparison

Let's put real numbers on this. Here's what a small business would pay for equivalent SaaS monitoring vs. self-hosting:

| Capability | SaaS Tool | Monthly Cost | Self-Hosted Equivalent | Monthly Cost |
|------------|-----------|-------------|------------------------|-------------|
| Uptime monitoring (10 checks) | Better Stack ($20) or UptimeRobot ($14) | $14–$20 | Uptime Kuma | $0 |
| Public status page | Statuspage ($29) | $29 | Uptime Kuma status page | $0 |
| Infrastructure dashboards (4 hosts) | Datadog ($15/host) | $60 | Grafana + Prometheus | $0 |
| Alerting | Included in above | — | Uptime Kuma + Grafana | $0 |
| **Total** | | **$103/month** | VPS ($10–$20) | **$10–$20/month** |

Over three years, that's $3,708 for SaaS vs. $360–$720 for self-hosting. The gap gets wider as you add more monitors and more hosts — SaaS pricing scales linearly, while your self-hosted stack handles more on the same server.

## Monitoring Your Other Self-Hosted Tools

If you've been following our self-hosting series, you already have several services running. Here's how to monitor each one:

| Tool | Uptime Kuma Monitor | Grafana Metric |
|------|-------------------|----------------|
| n8n | HTTP keyword check on `/healthz` | Prometheus metrics endpoint |
| Odoo | HTTP check on `/web/login` | PostgreSQL monitor + node-exporter on host |
| Mattermost | HTTP check on `/api/v4/system/ping` | Built-in Prometheus metrics |
| NocoDB | HTTP check on `/api/v1/health` | PostgreSQL monitor |
| Nextcloud | HTTP check on `/status.php` | Server metrics via node-exporter |
| Vaultwarden | HTTP check on `/alive` | Server metrics via node-exporter |
| Jitsi Meet | HTTP check on root URL | Built-in stats endpoint |
| Plausible Analytics | HTTP check on root URL | Server metrics via node-exporter |

The pattern is the same for every tool: Uptime Kuma does an HTTP check against a health endpoint, and node-exporter (or the tool's built-in metrics) feeds Grafana. Once you've set up the pattern for one tool, adding the next takes five minutes.

## What This Stack Doesn't Do

Being honest about limitations:

- **No log aggregation.** This stack monitors uptime and metrics, not logs. If you need centralized log collection (searching across all your services' log files), look at Loki — it integrates with Grafana and is built by the same team. We'll cover it in a future post.
- **No distributed tracing.** If you have a microservices architecture and need to trace requests across services, you'll want Jaeger or Tempo. For most small businesses with a handful of services, this is overkill.
- **No APM (application performance monitoring).** Tools like Datadog's APM track individual requests through your application code. This stack gives you infrastructure-level metrics, not code-level traces. For most small business use cases, that's the right level — but if you're running a high-traffic application, you may need more.
- **You're responsible for maintenance.** Updates, disk space, backup of the monitoring data itself. Budget 1–2 hours per month. And make sure your monitoring server isn't hosted on the same machine as the services it monitors — that defeats the purpose.

## Backing Up Your Monitoring Stack

Your monitoring data has value — uptime history, incident timelines, dashboard configurations. Back it up the same way you'd back up any other service:

```bash
#!/bin/bash
# /opt/scripts/backup-monitoring.sh
DATE=$(date +%Y%m%d)
BACKUP_DIR=/opt/backups/monitoring

mkdir -p "$BACKUP_DIR"

# Uptime Kuma data (SQLite database + settings)
tar czf "$BACKUP_DIR/uptime-kuma-$DATE.tar.gz" -C /opt/monitoring uptime-kuma-data

# Grafana data (dashboards, data sources, users)
tar czf "$BACKUP_DIR/grafana-$DATE.tar.gz" -C /opt/monitoring grafana-data

# Prometheus data
tar czf "$BACKUP_DIR/prometheus-$DATE.tar.gz" -C /opt/monitoring prometheus-data

# Keep 14 days (monitoring data loses value quickly)
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +14 -delete

echo "Monitoring backup complete: $DATE"
```

Add to crontab:

```bash
0 3 * * * /opt/scripts/backup-monitoring.sh >> /var/log/monitoring-backup.log 2>&1
```

Keep monitoring data for 14 days — it's useful for post-incident analysis, but you don't need years of historical CPU graphs.

## Getting Started

You can set up Uptime Kuma alone in 15 minutes and immediately know when your services go down. That alone is worth doing today, even if you're not ready for the full Grafana stack. Start with Uptime Kuma, add the monitors from the table above for every service you run, and configure Telegram or email alerts. Then, when you have an afternoon, add Grafana and Prometheus for the deeper visibility.

The tools are free. The server costs less than a single SaaS subscription. And the first time Uptime Kuma alerts you about a problem before a customer notices, the stack pays for itself.

If you want help setting up monitoring for your self-hosted infrastructure — or if you're not self-hosting yet and want to start — [get in touch](/#contact). ARDOT Consulting builds and maintains open-source monitoring and automation stacks for small businesses. We'll get you alerted, dash-boarded, and sleeping through the night.