---
layout: post
title: "Self-Hosting Plausible Analytics: A Step-by-Step Docker Guide"
date: 2026-08-26
author: "ARDOT Consulting"
tags: [plausible, analytics, self-hosting, docker, privacy, open-source]
excerpt: "Stop feeding your visitors' data to advertising giants. Self-host Plausible Analytics with Docker — lightweight, privacy-first, cookie-free, and entirely under your control. Here's the complete setup guide."
---

Every website needs analytics. You need to know how many people visit, where they come from, and what pages they read. But the way most businesses get those answers is by installing Google Analytics — a tool built by an advertising company that uses your visitors' data to build profiles, serve ads, and track people across the web.

There's a better way. [Plausible Analytics](https://plausible.io) is an open source, privacy-first analytics platform that gives you the numbers you actually need — page views, visitors, sources, bounce rate — without cookies, without tracking, and without selling your visitors' data to anyone. And because it's open source, you can self-host it on your own server for free.

This guide walks through the entire setup: why Plausible over other options, what you need, and a complete Docker Compose configuration you can copy and run.

## Why Plausible (And Why Self-Host It)

Before the technical setup, let's address the two questions every business owner asks: why Plausible instead of the alternatives, and why self-host instead of using Plausible's hosted cloud plan?

### Plausible vs Google Analytics

Google Analytics is free, ubiquitous, and powerful. It's also massively over-engineered for most small businesses. You don't need 50 dashboards and real-time event streams to know if your marketing campaign is working. Here's the honest comparison:

| Factor | Google Analytics 4 | Plausible |
|--------|-------------------|-----------|
| Cookies | Yes (requires cookie banner, GDPR consent) | No cookies, no banner needed |
| Data ownership | Google owns and uses your data | You own all data |
| Privacy compliance | Complex (requires consent management) | Simple (no personal data collected) |
| Dashboard complexity | High (weeks to learn) | Low (minutes to understand) |
| Script size | ~200 KB | < 1 KB |
| Accuracy | Sampled, estimated | Non-sampled, exact counts |
| Cost | Free (you pay with data) | Self-hosted: free / Cloud: $9–69/mo |

The script size alone matters. Google Analytics adds a heavy JavaScript payload to every page load. Plausible's tracking script is under 1 kilobyte — smaller than most favicons. Your site loads faster, which helps both user experience and search rankings.

### Plausible vs other open source analytics

Plausible isn't the only open source option. Two alternatives worth knowing:

- **Matomo** — Feature-rich, closer to a Google Analytics replacement. Heavier, more complex to configure, still uses cookies by default (though it can run cookieless). Good if you need deep e-commerce tracking.
- **Umami** — Lightweight and modern, similar philosophy to Plausible. Simpler feature set. Good choice if you want minimalism.

Plausible hits the sweet spot for most businesses: simple enough to understand at a glance, privacy-first by design, and actively maintained. This guide focuses on Plausible, but the self-hosting approach applies to all three.

### Why self-host instead of Plausible Cloud?

Plausible offers a hosted cloud plan starting at $9/month. It's good — zero maintenance, automatic updates, EU-hosted. But self-hosting makes sense when:

- **You want zero ongoing cost.** Self-hosted Plausible is free. You pay only for the server (which you may already have).
- **You want full data ownership.** Even Plausible Cloud stores data on their servers. Self-hosting puts it on yours.
- **You're already self-hosting other tools.** If you have a server running n8n, Ollama, or Odoo, adding Plausible costs almost nothing in additional resources.
- **You want custom domains and full control.** Self-hosting lets you configure everything exactly how you want.

The trade-off is maintenance. You're responsible for updates, backups, and uptime. For most small businesses with basic Linux comfort, this is a 30-minute initial setup and occasional updates.

## What You Need

- **A Linux server** with Docker and Docker Compose installed. A $5/month VPS (1 GB RAM, 1 CPU) is more than enough — Plausible is lightweight.
- **A domain name** for the analytics dashboard. Something like `stats.yourdomain.com`.
- **DNS access** to point a subdomain to your server's IP address.
- **15 minutes** of terminal time.

If you don't have a server yet, any budget VPS provider works — Hetzner, OVH, DigitalOcean, or a small cloud instance. Avoid providers that lock you into proprietary services. The whole point is owning your infrastructure.

## The Setup: Step by Step

### Step 1: Prepare your server

Make sure Docker and Docker Compose are installed:

```bash
docker --version
# Docker version 24.x or newer

docker compose version
# Docker Compose version v2.x or newer
```

If you need to install Docker, the official install script works on most Linux distributions:

```bash
curl -fsSL https://get.docker.com | sh
```

### Step 2: Create the project directory

```bash
mkdir -p ~/plausible
cd ~/plausible
```

### Step 3: Generate a secret key

Plausible needs a secret key base for encrypting sessions. Generate one:

```bash
openssl rand -base64 48
# Copy the output — you'll paste it into the config file below
```

### Step 4: Create the Docker Compose file

Create `docker-compose.yml` in your `~/plausible` directory:

```yaml
services:
  plausible-db:
    image: postgres:16-alpine
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=plausible_db
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  plausible-events-db:
    image: clickhouse/clickhouse-server:24.3.2-alpine
    restart: always
    volumes:
      - events-data:/var/lib/clickhouse
      - ./clickhouse-config.xml:/etc/clickhouse-server/config.d/logging.xml:ro
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

  plausible:
    image: ghcr.io/plausible/analytics:latest
    restart: always
    command: sh -c "/entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"
    depends_on:
      plausible-db:
        condition: service_healthy
      plausible-events-db:
        condition: service_started
    ports:
      - "127.0.0.1:8000:8000"
    environment:
      - BASE_URL=https://stats.yourdomain.com
      - SECRET_KEY_BASE=YOUR_SECRET_KEY_HERE
      - DATABASE_URL=postgres://postgres:postgres@plausible-db:5432/plausible_db
      - CLICKHOUSE_DATABASE_URL=http://plausible-events-db:8123/plausible_events_db
    volumes:
      - ./clickhouse-config.xml:/etc/clickhouse-server/config.d/logging.xml:ro

volumes:
  db-data:
  events-data:
```

**Important: replace `stats.yourdomain.com` with your actual analytics domain, and `YOUR_SECRET_KEY_HERE` with the key you generated in Step 3.**

A few things worth noting in this file:

- **The port binding `127.0.0.1:8000:8000`** means Plausible only listens on localhost. This is intentional — we'll put a reverse proxy in front of it for HTTPS. Never expose Plausible directly to the internet without TLS.
- **PostgreSQL** stores user accounts and site configuration.
- **ClickHouse** stores the analytics events. It's a column-oriented database designed for fast analytics queries — this is what makes Plausible fast even with millions of page views.
- **Volumes** persist your data across container restarts. Without them, you'd lose all data on every update.

### Step 5: Create the ClickHouse config

Plausible's ClickHouse setup needs a small config tweak to reduce excessive logging. Create `clickhouse-config.xml` in the same directory:

```xml
<clickhouse>
    <logger>
        <level>warning</level>
        <console>1</console>
    </logger>
</clickhouse>
```

### Step 6: Start the stack

```bash
docker compose up -d
```

This pulls the images and starts all three containers in the background. The first startup takes 1–2 minutes as it runs database migrations. Check that everything is running:

```bash
docker compose ps
```

You should see three services with status `Up`. Test that Plausible is responding:

```bash
curl http://localhost:8000
# Should return HTML — the Plausible login page
```

### Step 7: Set up the reverse proxy with Caddy

Plausible is running, but it's only accessible on localhost. To serve it over HTTPS on your domain, you need a reverse proxy. We recommend [Caddy](https://caddyserver.com) — it's the simplest way to get automatic HTTPS with Let's Encrypt certificates. No manual cert management.

Install Caddy on your server (Ubuntu/Debian):

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Edit the Caddyfile (usually at `/etc/caddy/Caddyfile`):

```
stats.yourdomain.com {
    reverse_proxy localhost:8000
}
```

Replace `stats.yourdomain.com` with your domain. Restart Caddy:

```bash
sudo systemctl restart caddy
```

That's it. Caddy automatically provisions an SSL certificate from Let's Encrypt and renews it before expiry. Visit `https://stats.yourdomain.com` in your browser — you should see the Plausible interface.

### Step 8: Create your account and add your first site

1. Visit `https://stats.yourdomain.com`
2. The first screen prompts you to create an admin account. Use a real email and a strong password.
3. Click **Add Site** and enter your website domain (e.g., `yourdomain.com`).
4. Plausible gives you a snippet of JavaScript to add to your site:

```html
<script defer data-domain="yourdomain.com" src="https://stats.yourdomain.com/js/script.js"></script>
```

Add this single line to your website's `<head>` section. That's the entire tracking setup. No cookie banner required — Plausible doesn't use cookies or collect personal data.

### Step 9: Verify it's working

Visit your website, then check back at your Plausible dashboard within a minute. You should see your visit appear in real-time. If nothing shows up, check:

- The script URL is correct and loads without errors (check your browser's developer tools → Network tab)
- Your domain in the `data-domain` attribute matches what you entered in Plausible
- Your server firewall allows traffic on ports 80 and 443 (for Caddy)

## Maintenance: Backups and Updates

### Backups

Your analytics data lives in two Docker volumes: `db-data` (PostgreSQL) and `events-data` (ClickHouse). Back them up regularly:

```bash
cd ~/plausible

# Backup PostgreSQL (accounts, settings)
docker compose exec plausible-db pg_dump -U postgres plausible_db > backup_$(date +%Y%m%d).sql

# Backup ClickHouse (analytics events)
docker compose exec plausible-events-db clickhouse-client \
  --query "BACKUP TABLE plausible_events_db.events TO Disk('backups', 'events.zip')"
```

Store these backups somewhere off the server — an S3-compatible bucket, a NAS, or even another server. A simple cron job that runs the backup commands and syncs the files to offsite storage is all you need.

### Updates

Updating Plausible is a one-liner:

```bash
cd ~/plausible
docker compose pull          # Pull the latest images
docker compose up -d         # Restart with new images
```

Plausible runs database migrations automatically on startup, so there's no manual migration step. Check the [release notes](https://github.com/plausible/analytics/releases) before updating — occasionally a major version requires a specific upgrade path.

### Resource usage

On a typical small business website (1,000–50,000 monthly visitors), self-hosted Plausible uses:

- **RAM:** ~200–400 MB total across all three containers
- **Disk:** ~1 GB per 100,000 page views (mostly ClickHouse)
- **CPU:** Negligible (spikes briefly during dashboard queries)

You can comfortably run it alongside n8n, Ollama, or Odoo on a single 2 GB RAM server.

## Migrating from Plausible Cloud

If you're currently on Plausible Cloud and want to move to self-hosted, Plausible doesn't offer an automated migration tool, but the process is straightforward:

1. **Export your data from Plausible Cloud** — contact their support or use the stats API to pull historical data.
2. **Set up self-hosted Plausible** following the steps above.
3. **Import historical data** — for most businesses, starting fresh on the self-hosted instance is simpler than migrating. You keep your Cloud account active for historical reference and start collecting new data on the self-hosted version.

Honestly, for most small businesses, the migration is more trouble than it's worth. If you're already on Plausible Cloud and it's working, stay there. Self-hosting is best for new setups or businesses that want to eliminate the monthly cost.

## Cost Breakdown: Self-Hosted vs Cloud

| Factor | Plausible Cloud | Self-Hosted |
|--------|----------------|-------------|
| Monthly cost | $9–69 (based on page views) | $5 (VPS, if you don't have one) |
| Setup time | 5 minutes | 30 minutes |
| Maintenance | Zero | ~1 hour/year (updates, backups) |
| Data location | Plausible's EU servers | Your server, wherever you choose |
| Custom domain | Included | Full control |
| Email reports | Included | Requires additional SMTP setup |

The self-hosted SMTP setup is the one piece that requires extra work. Plausible can send weekly email reports, but it needs an SMTP server to do so. You can use any transactional email service (Postmark, Resend, or a self-hosted Postfix setup). Add these environment variables to the Plausible service in your `docker-compose.yml`:

```yaml
- SMTP_HOST_ADDR=smtp.yourprovider.com
- SMTP_HOST_PORT=587
- SMTP_USER_NAME=your_username
- SMTP_USER_PASSWORD=your_password
- SMTP_HOST_SSL_ENABLED=true
- MAILER_EMAIL=analytics@yourdomain.com
```

Email reports are a nice-to-have, not essential. The dashboard works perfectly without them.

## Is Self-Hosting Analytics Worth It?

For most businesses reading this post: yes. Here's the decision framework:

**Self-host if:**
- You already have a server for other self-hosted tools (n8n, Ollama, Odoo)
- You want to eliminate recurring SaaS costs
- You're comfortable with basic Docker and Linux commands
- You value having all your business data on infrastructure you control

**Use Plausible Cloud if:**
- You have no server and no desire to manage one
- Your time is worth more than $9/month to you
- You want zero-touch maintenance and automatic updates

Either way, you're making a better choice than Google Analytics. The privacy improvement, the simpler dashboard, the faster page loads, and the data ownership are benefits that apply to both Plausible Cloud and self-hosted. The only question is whether you want to manage the infrastructure.

If you're already on the self-hosting path with tools like n8n and Ollama, adding Plausible is a natural next step. It's one of the lowest-maintenance services you can self-host, and it gives you clear, honest data about your website without compromising your visitors' privacy.

---

*Want help setting up privacy-first analytics or a full self-hosted automation stack? ARDOT Consulting designs and deploys open source infrastructure tailored to your business — analytics, automation, CRM, and AI, all under your control. [Get in touch through our contact form](/#contact) and we'll help you go from scattered SaaS subscriptions to a unified, self-owned system.*