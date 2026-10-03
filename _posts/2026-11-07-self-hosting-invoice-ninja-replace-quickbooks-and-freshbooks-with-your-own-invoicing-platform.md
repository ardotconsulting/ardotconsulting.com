---
layout: post
title: "Self-Hosting Invoice Ninja: Replace QuickBooks and Freshbooks with Your Own Invoicing Platform"
date: 2026-11-07
author: "ARDOT Consulting"
tags: [invoice-ninja, invoicing, self-hosting, docker, open-source, finance]
excerpt: "Invoice Ninja gives you professional invoicing, expense tracking, and payment collection without SaaS subscription fees — here's how to self-host it with Docker."
---

If you run a small business or freelance, you probably pay somewhere between $15 and $50 per month for invoicing software. QuickBooks charges $30+/month. Freshbooks starts at $17/month and climbs fast. Over a year, that's $200–$600 — and the price goes up almost annually.

What if you could host your own invoicing platform with the same features — professional invoices, online payments, expense tracking, time-tracking, client portal — for the cost of a $5/month VPS?

That's what Invoice Ninja offers. It's a mature, feature-rich invoicing application built with Laravel (PHP) that you can self-host on your own server. In this guide, we'll walk through what it does, how it compares to the SaaS alternatives, and how to get it running with Docker.

## What Is Invoice Ninja?

Invoice Ninja is a invoicing, quoting, project management, and time-tracking application that has been in active development since 2014. It's used by over 200,000 businesses worldwide and has over 10,000 stars on GitHub.

A quick note on licensing: Invoice Ninja v5 is **source-available**, not OSI-certified open source. This means the source code is publicly available and you can self-host it freely, but the license has some restrictions compared to traditional open source licenses (similar to Directus). All Pro and Enterprise features from the hosted SaaS version are included in the self-hosted code at no cost. A $40/year white-label license removes the "Created by Invoice Ninja" branding from client-facing pages.

This is an important distinction — and a reasonable one. You get full access to the software, can modify it, and can self-host it. The license just prevents you from reselling it as a competing product. For 95% of businesses, this works exactly like open source.

### Key Features

Here's what you get when you self-host Invoice Ninja:

- **Invoicing**: Create and send professional invoices with 11 customizable templates
- **Online payments**: Accept payments via Stripe, PayPal, Square, GoCardless, and 50+ other gateways
- **Quotes and proposals**: Send branded quotes that clients can approve online
- **Recurring invoices**: Set up automatic billing for retainer clients
- **Expense tracking**: Log business expenses, attach receipts, link to vendors
- **Time-tracking**: Track billable hours and convert them directly into invoices
- **Project management**: Create projects, assign tasks, track progress
- **Client portal**: Clients can view invoices, pay, and download documents
- **Recurring billing**: Automated subscription billing
- **Partial payments and deposits**: Accept deposits on large projects
- **Multi-company support**: Manage multiple businesses from one installation
- **REST API**: Full API for integrations with other business tools
- **Mobile apps**: Native apps for iPhone and Android (also on F-Droid)
- **Desktop apps**: Available for macOS, Windows, and Linux (Snap and Flatpak)

That feature list rivals or exceeds what most SaaS invoicing tools offer — and you control all of it.

## Invoice Ninja vs. SaaS Invoicing Tools

| Feature | Invoice Ninja (Self-Hosted) | QuickBooks Online | Freshbooks |
|---------|---------------------------|-------------------|------------|
| Monthly cost | ~$5 (VPS) + $0 | $30–$200 | $17–$60 |
| White-label cost | $40/year (optional) | Not available | $25+/month (branded) |
| Invoice templates | 11, fully customizable | Limited | Limited |
| Payment gateways | 50+ | Intuit-only + Stripe | Stripe, PayPal, Square |
| Client portal | Yes | Yes | Yes |
| Time-tracking | Built-in | Add-on | Built-in |
| Expense tracking | Built-in | Built-in | Built-in |
| Multi-company | Yes | No (separate accounts) | No |
| REST API | Yes, full | Yes, limited | Yes, limited |
| Mobile app | Yes (iOS, Android, F-Droid) | Yes | Yes |
| Desktop app | Yes (macOS, Win, Linux) | No | No |
| Data ownership | 100% yours | Vendor-controlled | Vendor-controlled |
| Self-hosting | Yes | No | No |

The biggest advantages aren't just cost. Data ownership matters: your financial data — client lists, invoice histories, payment records — stays on your server. No vendor can lock you out, change their pricing model, or discontinue a feature you depend on.

## What You'll Need

Before we start, here's what you need:

- **A VPS or dedicated server** — 2GB RAM minimum, 4GB recommended. A $5–$10/month VPS from Hetzner, OVH, or DigitalOcean is plenty.
- **Docker and Docker Compose** — installed on your server
- **A domain name** — for accessing the app (e.g., `invoices.yourcompany.com`)
- **An SSL certificate** — Let's Encrypt is free and automated via Caddy or Traefik
- **Basic comfort with the command line** — you'll be editing a few config files

## Step-by-Step: Self-Hosting Invoice Ninja with Docker

### Step 1: Set Up Your Server

SSH into your server and make sure Docker and Docker Compose are installed:

```bash
# Install Docker (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh

# Verify installation
docker --version
docker compose version
```

### Step 2: Create the Project Directory

```bash
mkdir -p /opt/invoiceninja
cd /opt/invoiceninja
```

### Step 3: Create the Docker Compose File

Create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  app:
    image: invoiceninja/invoiceninja:5
    restart: always
    environment:
      - APP_URL=https://invoices.yourcompany.com
      - APP_KEY=base64:YOUR_GENERATED_APP_KEY
      - DB_HOST=db
      - DB_DATABASE=ninja
      - DB_USERNAME=ninja
      - DB_PASSWORD=your_secure_password
      - DB_PORT=3306
      - MAIL_HOST=smtp.yourprovider.com
      - MAIL_PORT=587
      - MAIL_USERNAME=your_email@yourcompany.com
      - MAIL_PASSWORD=your_email_password
      - MAIL_FROM_ADDRESS=billing@yourcompany.com
      - MAIL_FROM_NAME="Your Company"
    volumes:
      - ./storage:/var/www/app/storage
      - ./logo:/var/www/app/public/logo
      - ./config/.env:/var/www/app/.env
    depends_on:
      - db
    ports:
      - "8080:8000"

  db:
    image: mariadb:10.11
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=your_root_password
      - MYSQL_DATABASE=ninja
      - MYSQL_USER=ninja
      - MYSQL_PASSWORD=your_secure_password
    volumes:
      - ./db:/var/lib/mysql
```

### Step 4: Generate the Application Key

The `APP_KEY` is critical — it's used to encrypt sensitive data in the database. If you lose it, your installation is unrecoverable.

```bash
# Generate a base64-encoded key
docker run --rm invoiceninja/invoiceninja:5 php artisan key:generate --show
```

Copy the output (it looks like `base64:randomcharacters...`) and paste it into your `docker-compose.yml` where `YOUR_GENERATED_APP_KEY` is.

### Step 5: Create the .env File

```bash
mkdir -p config
```

Create `config/.env`:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://invoices.yourcompany.com
APP_KEY=base64:YOUR_GENERATED_APP_KEY

DB_CONNECTION=mysql
DB_HOST=db
DB_DATABASE=ninja
DB_USERNAME=ninja
DB_PASSWORD=your_secure_password
DB_PORT=3306

MAIL_MAILER=smtp
MAIL_HOST=smtp.yourprovider.com
MAIL_PORT=587
MAIL_USERNAME=your_email@yourcompany.com
MAIL_PASSWORD=your_email_password
MAIL_FROM_ADDRESS=billing@yourcompany.com
MAIL_FROM_NAME="Your Company"
MAIL_ENCRYPTION=tls

REQUIRE_HTTPS=true
```

### Step 6: Start the Application

```bash
docker compose up -d
```

Check that the containers are running:

```bash
docker compose ps
```

You should see both the `app` and `db` containers with status "Up."

### Step 7: Run the Setup Wizard

Open your browser and navigate to:

```
http://your-server-ip:8080/setup
```

The setup wizard will walk you through:
1. Creating your admin account
2. Setting up your company details (name, logo, address)
3. Configuring your invoice numbering scheme
4. Choosing your currency and language settings

### Step 8: Set Up HTTPS with a Reverse Proxy

You should never expose Invoice Ninja directly over HTTP. Use Caddy as a reverse proxy for automatic HTTPS:

```yaml
# Add to docker-compose.yml
  caddy:
    image: caddy:2
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - app

volumes:
  caddy_data:
  caddy_config:
```

Create a `Caddyfile`:

```
invoices.yourcompany.com {
    reverse_proxy app:8000
}
```

Then restart everything:

```bash
docker compose up -d
```

Caddy will automatically provision a free Let's Encrypt SSL certificate. Your Invoice Ninja installation is now accessible at `https://invoices.yourcompany.com`.

### Step 9: Configure Payment Gateways

Once logged in, go to **Settings → Payment Gateways** and connect your preferred payment processor. Invoice Ninja supports:

- **Stripe** — credit cards, ACH, and international payments
- **PayPal** — including Venmo
- **Square** — credit cards and in-person payments
- **GoCardless** — bank-to-bank transfers (great for recurring billing)
- **Mollie** — European payment methods
- **PayFast** — South African payments

You'll need API credentials from each gateway, which you can get from their respective dashboards.

### Step 10: Connect Your Bank for Expense Import

Invoice Ninja supports bank feed integration via Yodlee, allowing you to automatically import and categorize business expenses. Go to **Settings → Bank Integration** to connect your accounts.

If you prefer not to use third-party bank feeds (or your bank isn't supported), you can manually import expenses via CSV upload — a perfectly valid approach for small businesses with manageable transaction volumes.

## Automating Invoice Ninja with n8n

One of the biggest advantages of self-hosting is the ability to integrate Invoice Ninja with your other business tools. Since Invoice Ninja has a full REST API, you can connect it to n8n (an open-source workflow automation tool) to automate your billing workflows.

Here are three practical automation examples:

### Automation 1: Auto-Create Invoices from Project Management

If you use a tool like Odoo or Nextcloud Deck for project management, you can create an n8n workflow that:

1. Watches for completed project milestones
2. Creates a draft invoice in Invoice Ninja via API
3. Sends you a notification for review before sending

```json
{
  "method": "POST",
  "url": "https://invoices.yourcompany.com/api/v1/invoices",
  "headers": {
    "X-API-TOKEN": "your_api_token",
    "X-Requested-With": "XMLHttpRequest"
  },
  "body": {
    "client_id": "client_id_from_n8n",
    "line_items": [
      {
        "product_key": "Milestone 1",
        "notes": "Project milestone completed",
        "cost": 5000,
        "quantity": 1
      }
    ]
  }
}
```

### Automation 2: Payment Confirmation Notifications

Create a webhook in Invoice Ninja that fires when an invoice is paid, then use n8n to:
- Post a notification in your Mattermost or team chat channel
- Update a Google Sheets alternative (like NocoDB) with the payment record
- Send a thank-you email to the client

### Automation 3: Overdue Invoice Reminders

Set up an n8n workflow that runs daily:
1. Queries Invoice Ninja's API for overdue invoices
2. Sends a polite reminder email to clients with unpaid invoices
3. Escalates to a phone call task for invoices more than 30 days late

This kind of automated follow-up typically reduces overdue invoices by 30–40%.

## Comparison: Self-Hosted Invoice Ninja vs. Hosted Invoice Ninja

Invoice Ninja offers both a hosted SaaS plan and self-hosted option. Here's when to choose each:

| Consideration | Self-Hosted | Hosted SaaS |
|--------------|-------------|-------------|
| Cost | $5–10/month VPS + optional $40/year white-label | $0–$18/month |
| Setup time | 30–60 minutes | 5 minutes |
| Maintenance | You handle updates, backups | Automatic |
| Data location | Your server | Invoice Ninja's servers |
| Customization | Full source access | Limited to settings |
| API access | Full | Full |
| Best for | Businesses with a server, tech comfort, or data sovereignty needs | Businesses that want zero maintenance |

If you're already self-hosting other tools (n8n, Odoo, Plausible), adding Invoice Ninja to the same server makes perfect sense. The marginal cost is essentially zero since you're already paying for the VPS.

## Maintenance and Backups

Once Invoice Ninja is running, maintenance is straightforward:

**Updates** — Check the [Invoice Ninja GitHub releases](https://github.com/invoiceninja/invoiceninja/releases) periodically. To update:

```bash
cd /opt/invoiceninja
docker compose pull app
docker compose up -d
```

**Backups** — Set up automated daily backups of the database and storage:

```bash
#!/bin/bash
# Save as /opt/invoiceninja/backup.sh
DATE=$(date +%Y%m%d)
docker compose exec db mysqldump -u ninja -pyour_secure_password ninja > /backups/ninja_db_$DATE.sql
tar -czf /backups/ninja_storage_$DATE.tar.gz /opt/invoiceninja/storage
# Keep only last 30 days
find /backups -name "ninja_*" -mtime +30 -delete
```

Add this to your crontab:

```bash
0 2 * * * /opt/invoiceninja/backup.sh
```

**APP_KEY backup** — Store your `APP_KEY` somewhere safe (a password manager like Vaultwarden, which we covered in a previous post). If you lose this key, your installation is unrecoverable — all encrypted data becomes permanently unreadable.

## Real Cost Comparison

Let's break down the actual costs for a typical 5-person business:

| Item | SaaS (QuickBooks) | Self-Hosted Invoice Ninja |
|------|-------------------|--------------------------|
| Monthly subscription | $50/month ($600/year) | $0 |
| VPS hosting | — | $8/month ($96/year) |
| White-label license | Not available | $40/year (optional) |
| Domain + SSL | — | $12/year + free SSL |
| **Year 1 total** | **$600** | **$148** (with white-label) |
| **Year 2+ total** | **$600+** (price increases likely) | **$148** |

Even accounting for the time you spend on initial setup (about 1 hour) and occasional maintenance (maybe 2 hours/year), the savings are significant. And you own the data.

## Is It Right for Your Business?

**Self-host Invoice Ninja if:**
- You already self-host other business tools (or want to start)
- You're comfortable with Docker and basic server administration
- You want full control over your financial data
- You need multi-company support without paying for multiple subscriptions
- You want to integrate invoicing with n8n or other automation tools

**Use the hosted SaaS version if:**
- You have no server and don't want one
- Your invoicing volume is low and a free SaaS plan works fine
- You prefer zero maintenance

Either way, Invoice Ninja gives you a professional invoicing platform without the QuickBooks/Freshbooks tax.

## Getting Help

Invoice Ninja has a strong community:
- [Documentation](https://invoiceninja.github.io/) — official user and admin guides
- [Community Forum](https://forum.invoiceninja.com/) — active user discussions
- [Slack Community](https://join.slack.com/t/invoiceninja/shared_invite/) — real-time help
- [GitHub Issues](https://github.com/invoiceninja/invoiceninja/issues) — bug reports and feature requests

## Wrapping Up

Self-hosting your invoicing platform is one of those changes that pays for itself within the first month. You get professional invoices, online payments, expense tracking, and a client portal — all under your control, on your server, with your data staying yours.

If you're already running n8n, Odoo, or other self-hosted business tools, Invoice Ninja fits naturally into that stack. And if you're just getting started with self-hosting, it's a great first "real business application" to deploy — the setup is well-documented, the Docker images are reliable, and the community is helpful.

---

*Want help setting up Invoice Ninja or connecting it to your existing business tools? [Contact ARDOT Consulting](/) — we specialize in open-source automation and self-hosting for small businesses. We'll handle the setup, integration, and training so you can focus on running your business.*