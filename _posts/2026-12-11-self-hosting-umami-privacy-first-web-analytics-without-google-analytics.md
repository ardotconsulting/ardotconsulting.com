---
layout: post
title: "Self-Hosting Umami: Privacy-First Web Analytics Without Google Analytics"
date: 2026-12-11
author: "ARDOT Consulting"
tags: [umami, analytics, self-hosting, privacy, open-source, docker, web-analytics, google-analytics-alternative]
excerpt: "Umami is a lightweight, open-source web analytics platform that gives you the numbers you actually need — page views, visitors, sources, goals — without cookies, without tracking, and without feeding your visitors' data to an advertising company. Here's how to self-host it with Docker."
---

# Self-Hosting Umami: Privacy-First Web Analytics Without Google Analytics

You need to know how many people visit your website, where they come from, and which pages they actually read. That's not a complicated ask. But the default answer — install Google Analytics — comes with baggage most business owners don't realize they're carrying.

Google Analytics is built by an advertising company. The data it collects from your website doesn't just sit in a dashboard for you to look at. It feeds into a broader ecosystem designed to track people across the web, build profiles, and sell ads. You get "free" analytics. Google gets behavioral data on every person who visits your site. That's the actual trade.

There's a better option. **Umami** is an open-source web analytics platform that gives you the numbers you need — page views, unique visitors, traffic sources, bounce rate, conversion goals — without cookies, without cross-site tracking, and without sending your visitors' data to anyone. You run it on your own server. You own the data. The script is under 2 kilobytes, so it doesn't slow your site down.

We've written about Plausible Analytics before, and Plausible is excellent. Umami is the other strong option in this space — slightly different philosophy, different trade-offs, and worth understanding if you're choosing between the two. This guide covers what Umami does well, where it's different from Plausible, and a complete Docker setup you can run today.

## Why Umami (And Why Self-Host It)

### Umami vs Google Analytics

Google Analytics 4 is the tool most businesses install by default. It's also massively over-engineered for what a typical small business actually needs. You don't need 50 dashboards, sampled data, and a learning curve measured in weeks to know whether your last blog post brought in traffic.

Here's the honest comparison:

| Factor | Google Analytics 4 | Umami |
|--------|-------------------|-------|
| Cookies | Yes (requires consent banner, GDPR compliance) | No cookies, no banner needed |
| Data ownership | Google owns and uses your data | You own all data, on your server |
| Privacy compliance | Complex (consent management required) | Simple (no personal data collected) |
| Dashboard complexity | High (weeks to learn) | Low (minutes to understand) |
| Script size | ~200 KB | < 2 KB |
| Cross-site tracking | Yes (part of Google's ad network) | None |
| Cost | "Free" (you pay with data) | Self-hosted: free / Cloud: from $10/mo |

The script size point matters more than people realize. Google Analytics adds a heavy JavaScript payload to every page load. Umami's tracking script is under 2 kilobytes — smaller than most images on your site. Your pages load faster, which helps both user experience and search rankings.

### Umami vs Plausible

We've recommended Plausible in a previous post, and we still recommend it. But Umami has a few differences worth knowing:

- **Umami is MIT-licensed.** Plausible uses AGPL-3.0. For most businesses this doesn't matter, but if your legal team has opinions about license terms, MIT is the more permissive option.
- **Umami uses a single database (PostgreSQL or MySQL).** Plausible uses PostgreSQL plus ClickHouse. Fewer moving parts means simpler backups and lower memory usage. Umami runs comfortably on a 1 GB RAM server.
- **Umami's dashboard is more customizable.** You can build custom reports and filter by any event property. Plausible's dashboard is simpler and more opinionated — some people prefer that.
- **Umami supports multi-site analytics natively** with a clean site-switcher. If you manage several websites (client sites, multiple brands, subdomains), Umami handles this elegantly in one installation.
- **Plausible has better built-in email reports and a more polished mobile experience.** Umami is more of a "sit down at your laptop and look" tool.

Both are good. If you want the absolute simplest setup and the smallest footprint, Umami wins. If you want email reports and a slightly more polished dashboard out of the box, Plausible wins. Either way, you're making a dramatically better choice than Google Analytics.

### Why self-host instead of Umami Cloud?

Umami offers a hosted cloud plan starting at $10/month. It's a good option if you don't want to manage a server. But self-hosting makes sense when:

- **You want zero ongoing cost.** Self-hosted Umami is free. You pay only for the server, which you may already have.
- **You want full data ownership.** Even Umami Cloud stores data on their servers. Self-hosting puts it on yours.
- **You're already self-hosting other tools.** If you have a server running n8n, Ollama, or Odoo, adding Umami costs almost nothing in additional resources.
- **You manage multiple sites.** Self-hosting lets you track as many sites as you want without per-site pricing tiers.

The trade-off is maintenance — you handle updates and backups. For most small businesses with basic Linux comfort, that's a 20-minute initial setup and occasional updates.

## What You Need

- **A Linux server** with Docker and Docker Compose installed. A $5/month VPS (1 GB RAM, 1 CPU) is more than enough — Umami is very lightweight.
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
mkdir -p ~/umami
cd ~/umami
```

### Step 3: Create the Docker Compose file

Create `docker-compose.yml` in your `~/umami` directory:

```yaml
services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    restart: always
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      DATABASE_URL: postgresql://umami:umami@umami-db:5432/umami
      DATABASE_TYPE: postgresql
      APP_SECRET: replace-me-with-a-random-string
    depends_on:
      umami-db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  umami-db:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umami
    volumes:
      - umami-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U umami"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  umami-db-data:
```

**Important: replace `replace-me-with-a-random-string` with a real random string.** Generate one with:

```bash
openssl rand -hex 32
```

The `APP_SECRET` is used to sign session cookies. Use a strong random value — don't leave the placeholder.

A few things worth noting in this file:

- **The port binding `127.0.0.1:3000:3000`** means Umami only listens on localhost. This is intentional — we'll put a reverse proxy in front of it for HTTPS. Never expose Umami directly to the internet without TLS.
- **PostgreSQL** stores everything: account credentials, site configuration, and analytics events. One database, one backup target. Simpler than Plausible's two-database setup.
- **The volume `umami-db-data`** persists your data across container restarts. Without it, you'd lose all data on every update.

### Step 4: Start the stack

```bash
docker compose up -d
```

This pulls the images and starts both containers in the background. The first startup takes about a minute as the database initializes and Umami runs its migrations. Check that everything is running:

```bash
docker compose ps
```

You should see two services with status `Up`. Test that Umami is responding:

```bash
curl http://localhost:3000/api/health
# Should return: {"status": "ok"}
```

### Step 5: Set up the reverse proxy with Caddy

Umami is running, but it's only accessible on localhost. To serve it over HTTPS on your domain, you need a reverse proxy. We recommend [Caddy](https://caddyserver.com) — it's the simplest way to get automatic HTTPS with Let's Encrypt certificates. No manual cert management.

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
    reverse_proxy localhost:3000
}
```

Replace `stats.yourdomain.com` with your domain. Restart Caddy:

```bash
sudo systemctl restart caddy
```

That's it. Caddy automatically provisions an SSL certificate from Let's Encrypt and renews it before expiry. Visit `https://stats.yourdomain.com` in your browser — you should see the Umami login screen.

### Step 6: Log in and add your first site

1. Visit `https://stats.yourdomain.com`
2. The default login is username `admin` and password `umami`. **Change this immediately** after logging in — go to Settings → Profile and set a new password.
3. Go to Settings → Websites and click **Add Website**.
4. Enter your website name and domain (e.g., `yourdomain.com`).
5. Umami gives you a tracking snippet to add to your site:

```html
<script defer src="https://stats.yourdomain.com/script.js" data-website-id="YOUR-WEBSITE-ID"></script>
```

Add this single line to your website's `<head>` section. That's the entire tracking setup. No cookie banner required — Umami doesn't use cookies or collect personal data.

### Step 7: Verify it's working

Visit your website, then check back at your Umami dashboard within a minute. You should see your visit appear in real-time on the dashboard. If nothing shows up, check:

- The script URL is correct and loads without errors (check your browser's developer tools → Network tab)
- The `data-website-id` matches the ID shown in Umami's website settings
- Your server firewall allows traffic on ports 80 and 443 (for Caddy)

## Tracking More Than Page Views: Events and Goals

Umami isn't limited to page views. You can track custom events — button clicks, form submissions, downloads, video plays — anything that matters to your business. Add a line of JavaScript wherever the action happens:

```javascript
umami.track('newsletter_signup', { source: 'footer' });
```

That's it. The event shows up in your dashboard under the Events tab, and you can set it as a **conversion goal** in Settings → Goals. Goals let you see conversion rates by traffic source — so you know which referrer or campaign actually drives signups, not just clicks.

This is the kind of thing Google Analytics makes unnecessarily complicated. In Umami, it's one line of code and one click in the dashboard.

## Tracking Multiple Sites

If you manage more than one website — a main site, a documentation subdomain, a client's site — Umami handles this cleanly. Each site gets its own website ID and tracking snippet, but they all live in one installation. Switch between them from a dropdown in the dashboard. No per-site pricing, no separate logins.

This is one area where Umami genuinely outshines Plausible's self-hosted version, which charges per site on its cloud plan and can feel cluttered when you're tracking many properties on a self-hosted instance.

## Maintenance: Backups and Updates

### Backups

All your data lives in one Docker volume: `umami-db-data` (PostgreSQL). Back it up regularly:

```bash
cd ~/umami
docker compose exec umami-db pg_dump -U umami umami > backup_$(date +%Y%m%d).sql
```

Store the backup somewhere off the server — an S3-compatible bucket, a NAS, or another server. A simple cron job that runs the backup command and syncs the file to offsite storage is all you need. Because Umami uses a single database, your backup is one file. That's simpler than Plausible, which requires backing up both PostgreSQL and ClickHouse separately.

### Updates

Updating Umami is a one-liner:

```bash
cd ~/umami
docker compose pull          # Pull the latest images
docker compose up -d         # Restart with new images
```

Umami runs database migrations automatically on startup. Check the [release notes](https://github.com/umami-software/umami/releases) before updating — occasionally a major version has specific upgrade instructions.

### Resource usage

On a typical small business website (1,000–50,000 monthly visitors), self-hosted Umami uses:

- **RAM:** ~150–300 MB total across both containers
- **Disk:** ~200 MB per 100,000 page views (PostgreSQL is efficient for this workload)
- **CPU:** Negligible (spikes briefly during dashboard queries)

You can comfortably run it alongside n8n, Ollama, or Odoo on a single 2 GB RAM server. Umami is one of the lightest self-hosted services you can run.

## Cost Breakdown: Self-Hosted vs Cloud

| Factor | Umami Cloud | Self-Hosted |
|--------|------------|-------------|
| Monthly cost | $10–49 (based on page views and sites) | $5 (VPS, if you don't have one) |
| Setup time | 5 minutes | 20 minutes |
| Maintenance | Zero | ~1 hour/year (updates, backups) |
| Data location | Umami's servers | Your server, wherever you choose |
| Number of sites | Tied to pricing tier | Unlimited |
| Custom domain | Included | Full control |

If you're already self-hosting other tools, the math is straightforward — adding Umami to an existing server costs you nothing per month. If you'd need to rent a server just for analytics, the cloud plan is worth considering until you have other reasons to self-host.

## Is Self-Hosting Analytics Worth It?

For most businesses reading this post: yes, especially if you're already on the self-hosting path. Here's the decision framework:

**Self-host if:**
- You already have a server for other self-hosted tools (n8n, Ollama, Odoo, Nextcloud)
- You want to eliminate recurring SaaS costs
- You're comfortable with basic Docker and Linux commands
- You manage multiple websites and don't want per-site pricing
- You value having all your business data on infrastructure you control

**Use Umami Cloud if:**
- You have no server and no desire to manage one
- Your time is worth more than $10/month to you
- You want zero-touch maintenance and automatic updates

Either way, you're making a dramatically better choice than Google Analytics. The privacy improvement, the simpler dashboard, the faster page loads, and the data ownership are benefits that apply to both Umami Cloud and self-hosted. The only question is whether you want to manage the infrastructure.

If you're already self-hosting tools like n8n and Ollama, adding Umami is a natural next step. It's one of the lowest-maintenance services you can run, it gives you honest data about your website, and it doesn't require your visitors to surrender their privacy in exchange for you knowing whether your marketing is working.

---

*Want help setting up privacy-first analytics or a full self-hosted automation stack? ARDOT Consulting designs and deploys open source infrastructure tailored to your business — analytics, automation, CRM, and AI, all under your control. [Get in touch through our contact form](/#contact) and we'll help you go from scattered SaaS subscriptions to a unified, self-owned system.*