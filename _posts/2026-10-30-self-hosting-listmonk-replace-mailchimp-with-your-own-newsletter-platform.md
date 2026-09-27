---
layout: post
title: "Self-Hosting Listmonk: Replace Mailchimp With Your Own Newsletter Platform"
date: 2026-10-30
author: "ARDOT Consulting"
tags: [listmonk, email-marketing, newsletter, self-hosting, open-source, docker]
excerpt: "Mailchimp and ConvertKit get expensive fast. Here's how to self-host Listmonk — a powerful open source newsletter and mailing list manager — for a fraction of the cost."
---

If you've been sending newsletters for any length of time, you already know the pain: your list grows, your bill grows faster. Mailchimp's pricing scales with subscriber count, and once you cross 10,000 subscribers, you're paying hundreds of dollars per month. ConvertKit, Brevo, and most other hosted platforms follow the same pattern.

What if you could run the same kind of software — subscriber management, campaign scheduling, analytics, templates — on your own server for the cost of a $5/month VPS?

That's exactly what Listmonk does. It's a self-hosted newsletter and mailing list manager that's been battle-tested at scale — the project's own screenshots show a single instance sending 7+ million emails in one campaign while using a fraction of a CPU core and 57 MB of RAM. It has 23,000+ GitHub stars, an active maintainer, and a feature set that rivals commercial platforms.

In this guide, we'll walk through what Listmonk does, how it compares to Mailchimp, and how to get it running with Docker in under 30 minutes.

## What Listmonk Actually Does

Listmonk is a standalone application written in Go. It ships as a single binary and uses PostgreSQL as its database. That's it — no Node.js runtime, no Python dependencies, no cluster of microservices. You run one binary and connect one database.

Here's what you get out of the box:

| Feature | Listmonk | Mailchimp |
|---------|----------|-----------|
| Subscriber management | ✅ Unlimited | ✅ (priced per tier) |
| Single & double opt-in | ✅ | ✅ |
| Subscriber segmentation | ✅ SQL-based | ✅ Visual builder |
| Campaign scheduling | ✅ | ✅ |
| Drag-and-drop email builder | ✅ | ✅ |
| WYSIWYG & Markdown editors | ✅ | WYSIWYG only |
| Transactional emails | ✅ Via API | ✅ Add-on ($$$) |
| Analytics (opens, clicks, bounces) | ✅ | ✅ |
| SMS / WhatsApp messaging | ✅ Via webhooks | ❌ |
| SSO (OIDC) | ✅ | ✅ Enterprise |
| API access | ✅ Full REST API | ✅ |
| Subscriber limit | None | Tiers (10K = ~$100/mo) |
| Monthly cost (10K subscribers) | ~$5 (VPS) | ~$100+ |
| Data location | Your server | Vendor's cloud |

The key difference isn't just price. With Listmonk, your subscriber data lives on your server. You're not handing email addresses, open rates, and click data to a third party. For businesses that take data privacy seriously — or operate under GDPR, HIPAA, or similar regulations — that matters.

## Who Listmonk Is For

Listmonk fits well if you:

- **Send newsletters regularly** and your subscriber list is growing past the free tiers of hosted platforms
- **Want data sovereignty** — your subscriber list is an asset, and you want to control where it lives
- **Already self-host other tools** — if you're running Plausible, Metabase, or n8n, adding Listmonk is a natural fit
- **Need transactional emails** alongside newsletters — Listmonk's API handles both
- **Have technical capacity** — you or someone on your team can manage a Docker container and a DNS record

It's *not* the right fit if you need a fully managed, no-touch solution and have no interest in running infrastructure. In that case, a hosted service makes sense — you're paying for someone else to handle deliverability, updates, and server maintenance.

## What You'll Need

- A Linux server (VPS or dedicated) with Docker and Docker Compose installed
- A domain name or subdomain (e.g., `news.yourcompany.com`)
- An SMTP server for sending emails — this can be a self-hosted Postfix setup, a transactional email service like Amazon SES, or any SMTP relay
- About 30 minutes

The SMTP piece is the one most people ask about. Listmonk doesn't send email directly — it connects to an SMTP server, which handles delivery. You have several options:

1. **Amazon SES** — cheapest at $0.10 per 1,000 emails. Requires AWS account setup and domain verification.
2. **Self-hosted Postfix** — free, but you'll need to configure SPF, DKIM, and DMARC records properly for deliverability. More work, maximum control.
3. **Transactional relay services** (Mailgun, Postmark, SendGrid) — easier setup, paid, but handle deliverability for you.

For this guide, we'll assume you're using an external SMTP service. You can swap in self-hosted Postfix later.

## Step 1: Download the Docker Compose File

Listmonk provides a ready-to-use `docker-compose.yml` that includes both Listmonk and a PostgreSQL database. Start by downloading it:

```bash
# Create a directory for Listmonk
mkdir -p ~/listmonk && cd ~/listmonk

# Download the official compose file
curl -LO https://github.com/knadh/listmonk/raw/master/docker-compose.yml
```

The default compose file looks something like this:

```yaml
version: "3"

services:
  listmonk:
    image: listmonk/listmonk:latest
    container_name: listmonk
    restart: unless-stopped
    ports:
      - "9000:9000"
    environment:
      - TZ=Etc/UTC
    volumes:
      - ./config.toml:/listmonk/config.toml
    depends_on:
      - db

  db:
    image: postgres:15
    container_name: listmonk-db
    restart: unless-stopped
    environment:
      - POSTGRES_PASSWORD=listmonk
      - POSTGRES_USER=listmonk
      - POSTGRES_DB=listmonk
    volumes:
      - ./data:/var/lib/postgresql/data
```

This gives you two containers: Listmonk itself and a PostgreSQL database. The database data is persisted to `./data` on your host.

## Step 2: Generate the Configuration

Before starting the containers, generate a configuration file:

```bash
# Generate the default config
docker run --rm listmonk/listmonk:latest --new-config > config.toml
```

Now edit `config.toml`. The key settings to update:

```toml
[app]
  admin_username = "admin"
  admin_password = "CHANGE_THIS_TO_A_STRONG_PASSWORD"
  admin_emaillist = "you@yourcompany.com"

# Database connection (matches the compose file)
[db]
  host = "db"
  port = 5432
  user = "listmonk"
  password = "listmonk"
  database = "listmonk"

# SMTP settings — replace with your provider's details
[smtp]
  [smtp.myprovider]
    enabled = true
    host = "email-smtp.us-east-1.amazonaws.com"
    port = 587
    auth_protocol = "login"
    username = "YOUR_SES_USERNAME"
    password = "YOUR_SES_PASSWORD"
    max_conns = 10
    max_msg_retries = 2
    idle_timeout = "15s"
    wait_timeout = "5s"
```

Replace the SMTP section with your provider's credentials. If you're using Amazon SES, the host will be something like `email-smtp.us-east-1.amazonaws.com`. If you're using Postfix on the same server, use `host = "localhost"` and `port = 25`.

## Step 3: Initialize the Database and Start

```bash
# Initialize the database schema
docker compose run --rm listmonk ./listmonk --install

# Start both containers in the background
docker compose up -d
```

The `--install` flag creates all the necessary tables in PostgreSQL. You only run this once. For future upgrades, use `--upgrade` instead.

Visit `http://localhost:9000` (or your server's IP on port 9000) and you should see the Listmonk dashboard. Log in with the admin credentials you set in `config.toml`.

## Step 4: Set Up a Reverse Proxy with SSL

You don't want to expose port 9000 directly. Use Caddy or Nginx as a reverse proxy to handle HTTPS automatically.

Here's a simple Caddyfile:

```
news.yourcompany.com {
    reverse_proxy localhost:9000
}
```

Caddy automatically provisions and renews Let's Encrypt certificates. Install Caddy, drop this config in, and reload:

```bash
# Install Caddy (Debian/Ubuntu)
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy

# Copy your Caddyfile
sudo cp Caddyfile /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Point your DNS `news.yourcompany.com` A record to your server's IP, and Caddy will handle the rest.

## Step 5: Create Your First List and Campaign

Inside the Listmonk dashboard:

1. **Go to Lists → New List.** Give it a name (e.g., "Weekly Newsletter") and choose double opt-in if you want subscribers to confirm their email addresses.
2. **Go to Subscribers → Import** if you're migrating from another platform. Listmonk accepts CSV files. If you're coming from Mailchimp, export your list as CSV and import directly.
3. **Go to Campaigns → New Campaign.** Choose your list, write your subject line, and compose your email. Listmonk supports a drag-and-drop builder, raw HTML, Markdown, and plain text — all in the same editor.

The template editor uses Go's templating language, which lets you insert dynamic content:

```go
{{ .Subscriber.Name | default "there" }}
```

This inserts the subscriber's name, or falls back to "there" if the name field is empty. You can use conditional logic, loops, and over 100 built-in functions.

## Step 6: Embed a Subscription Form on Your Website

Listmonk generates embeddable HTML forms for each list. Go to Lists → your list → Subscription Form, and you'll get code like this:

```html
<form method="post" action="https://news.yourcompany.com/subscription/form">
  <input type="hidden" name="l" value="a1b2c3d4">
  <div>
    <label for="email">Email</label>
    <input type="email" name="email" id="email" required>
  </div>
  <div>
    <label for="name">Name</label>
    <input type="text" name="name" id="name">
  </div>
  <button type="submit">Subscribe</button>
</form>
```

Paste this into any page on your website. Submissions go directly to Listmonk — no JavaScript SDK, no third-party scripts, no cookie banners required.

## Automating With n8n

If you're already running n8n for automation, Listmonk's REST API opens up useful workflows:

- **New customer → auto-subscribe:** When a new customer is created in Odoo or your CRM, an n8n webhook calls Listmonk's API to add them to a welcome sequence list.
- **Blog published → newsletter campaign:** When a new post is published, n8n creates a draft campaign in Listmonk with the post content, ready for review.
- **Tag-based segmentation:** Sync tags from your CRM to Listmonk subscriber attributes, then use SQL-based segmentation to target campaigns.

Here's an example n8n HTTP Request node that subscribes a new contact:

```json
{
  "method": "POST",
  "url": "https://news.yourcompany.com/api/subscribers",
  "authentication": "header",
  "headers": {
    "Authorization": "Bearer YOUR_API_TOKEN"
  },
  "body": {
    "email": "{{$json.customer_email}}",
    "name": "{{$json.customer_name}}",
    "lists": [1],
    "status": "enabled"
  }
}
```

Generate an API token in Listmonk under Settings → API Tokens.

## Cost Comparison: One Year

Let's put real numbers on this. Say you have 15,000 subscribers and send two campaigns per month.

| Platform | Monthly Cost | Annual Cost |
|----------|-------------|-------------|
| Mailchimp (Standard, 15K) | ~$150 | ~$1,800 |
| Convertkit (Creator, 15K) | ~$130 | ~$1,560 |
| Brevo (15K emails/mo) | ~$65 | ~$780 |
| **Listmonk + VPS + SES** | **~$12** | **~$144** |

The Listmonk setup breaks down as:
- VPS (2GB RAM): $5–10/month
- Amazon SES (30K emails/mo at $0.10/1K): $3/month
- Domain: ~$10/year

That's a 92% saving over Mailchimp. Even if you factor in the time spent setting up and maintaining the server (a few hours per year for updates), the math is hard to argue with.

## Migration From Mailchimp

Moving from Mailchimp to Listmonk is straightforward:

1. **Export your subscribers** from Mailchimp as a CSV file (Audience → Manage Audience → Export Audience).
2. **Clean up the CSV** — Listmonk needs at minimum an `email` column. A `name` column is optional but recommended.
3. **Import into Listmonk** — Subscribers → Import → upload CSV. Map the columns and choose your target list.
4. **Recreate your email templates** — Listmonk's drag-and-drop builder is different from Mailchimp's, so you'll need to rebuild your template. If your template is HTML, you can paste it directly into the raw HTML editor.
5. **Update your subscription forms** — Replace Mailchimp embed codes on your website with Listmonk's form code.

The one thing you can't migrate is historical analytics. Mailchimp's open/click data stays in Mailchimp. Listmonk will start tracking from your first campaign onward.

## Deliverability: The Elephant in the Room

Self-hosting your newsletter software doesn't mean you should self-host your email delivery. Deliverability — making sure your emails actually reach the inbox instead of the spam folder — depends on your sender reputation, which depends on your SMTP infrastructure.

This is why most Listmonk users pair it with a transactional email service like Amazon SES. SES handles the hard parts: IP reputation, bounce processing, complaint feedback loops, and DKIM signing. You get the cost savings of self-hosted software with the deliverability of a managed SMTP service.

If you do want to self-host SMTP (with Postfix), make sure you set up:

- **SPF record** in DNS — identifies which servers can send from your domain
- **DKIM signing** — cryptographically signs your emails
- **DMARC record** — tells receiving servers what to do if SPF/DKIM fail
- **Warm up your IP** — start with small sends and ramp up gradually

## Keeping Listmonk Updated

Updating Listmonk is simple:

```bash
cd ~/listmonk

# Pull the latest image
docker compose pull

# Run the database upgrade
docker compose run --rm listmonk ./listmonk --upgrade

# Restart with the new version
docker compose up -d
```

The project releases new versions regularly — the current version as of this writing is v6.2.0. Updates typically include new features, bug fixes, and performance improvements. The maintainer (Kailash Nadh, who also built the popular Knacksteem and golifto projects) has been consistently active since the project's inception.

## Backing Up Your Data

Your subscriber list is valuable. Back it up regularly:

```bash
# Back up the PostgreSQL database
docker exec listmonk-db pg_dump -U listmonk listmonk > backup_$(date +%Y%m%d).sql

# Optional: back up the config and data directory
tar czf listmonk_files_$(date +%Y%m%d).tar.gz config.toml ./data
```

Set this up as a cron job to run daily, and copy the backups to off-site storage (another server, S3, or a B2 bucket). If you're using n8n, you can even automate the backup-to-S3 step as a scheduled workflow.

## Is It Worth It?

If you have fewer than 500 subscribers and send one email a month, a free Mailchimp account is fine. The math doesn't justify the setup time.

But if you're past 2,000 subscribers, sending regularly, and comfortable with basic Docker commands — Listmonk pays for itself within the first month. You get unlimited subscribers, unlimited campaigns, full data ownership, and an API that integrates with the rest of your automation stack. The features you'd pay extra for on hosted platforms (transactional emails, SSO, API access) are all included.

The tradeoff is maintenance. You're responsible for updates, backups, and server security. If that's a dealbreaker, stick with a hosted service. If you're already running other self-hosted tools, Listmonk fits naturally alongside them.

---

**Want help setting up Listmonk or integrating it with your existing automation?** [Get in touch](/contact/) — we help businesses build self-hosted automation stacks that replace expensive SaaS subscriptions without sacrificing capability.