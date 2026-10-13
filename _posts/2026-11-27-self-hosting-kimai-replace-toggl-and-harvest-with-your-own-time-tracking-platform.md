---
layout: post
title: "Self-Hosting Kimai: Replace Toggl and Harvest with Your Own Time Tracking Platform"
date: 2026-11-27
author: "ARDOT Consulting"
tags: [kimai, time-tracking, self-hosting, docker, open-source, invoicing]
excerpt: "Track billable hours, generate invoices, and run reports on your own server with Kimai — the open source time tracking platform that replaces Toggl, Harvest, and Clockify."
---

If your business bills by the hour, time tracking isn't optional — it's how you get paid. But most teams end up paying $10–20 per user per month for tools like Toggl, Harvest, or Clockify, and the costs creep up as you grow. Worse, your time data — which projects are profitable, which clients are expensive, how your team actually spends their day — sits on someone else's servers.

[Kimai](https://www.kimai.org/) is an open source time tracking platform licensed under AGPL-3.0. You self-host it, own your data, and pay nothing per user. It handles time tracking, reporting, invoicing, and team management — the core features you'd expect from a commercial tool, minus the monthly bill and the vendor lock-in.

In this guide, we'll walk through what Kimai does, how it compares to the commercial alternatives, and how to get it running on your own server with Docker.

## What Kimai Actually Does

Kimai is built for project-driven teams — agencies, consultants, freelancers, and any business that tracks time against clients and projects. Here's what it handles out of the box:

### Time Tracking

The core feature: a start/stop timer that logs time against projects and activities. You can also enter time manually after the fact if you forgot to start the timer (which, let's be honest, happens a lot). Each time entry includes the project, activity, description, tags, and billable/non-billable status.

### Project and Customer Management

Kimai organizes time into a hierarchy: **Customers → Projects → Activities**. A customer (e.g., "Acme Corp") has one or more projects (e.g., "Website Redesign," "Monthly Maintenance"), and each project has activities (e.g., "Design," "Development," "Client Meetings"). This structure makes reporting and invoicing straightforward.

### Reporting and Analytics

You can filter time entries by user, customer, project, activity, tag, or date range, then export the results as PDF, CSV, or HTML. This is where self-hosting pays off — you can build custom reports and queries without hitting API rate limits or paying for a higher-tier plan.

### Invoicing

Kimai generates invoices directly from tracked time. You configure invoice templates, set your hourly rates per project or activity, and Kimai groups billable entries into an invoice with a unique invoice number. Templates support PDF and DOCX formats. You can also mark entries as "invoiced" so you don't double-bill.

### Team Management

Unlimited users with role-based permissions. Roles include User, Teamlead, and Admin, with granular permissions controlling who can see what data. A team lead can see all time entries for their team; a regular user only sees their own.

### Authentication and Security

Kimai supports LDAP and SAML for single sign-on, so you can integrate it with your existing identity provider (Authentik, Keycloak, Azure AD). Two-factor authentication via TOTP is built in — no plugin required.

### API

A full JSON API lets you read and write data programmatically. This means you can connect Kimai to n8n for automated workflows — like sending a Slack notification when someone logs overtime, or syncing time entries into Odoo for project accounting.

## Kimai vs. Commercial Time Trackers

| Feature | Kimai (Self-Hosted) | Toggl Track | Harvest | Clockify |
|---------|---------------------|-------------|---------|----------|
| Pricing | Free (self-hosted) | $10–18/user/mo | $11–14/user/mo | $3.99–14.99/user/mo |
| User limit | Unlimited | Plan-based | Plan-based | Unlimited (free tier) |
| Data ownership | Full — your server | Vendor-hosted | Vendor-hosted | Vendor-hosted |
| Invoicing | Built in | Add-on (extra cost) | Built in | Premium only |
| SSO (SAML/LDAP) | Built in | Enterprise plan | Enterprise plan | Enterprise plan |
| 2FA (TOTP) | Built in | Premium plan | Premium plan | Premium plan |
| API access | Full, no limits | Rate-limited | Rate-limited | Rate-limited |
| Self-hosting | Yes | No | No | No |
| License | AGPL-3.0 (open source) | Proprietary | Proprietary | Proprietary |

The cost difference is stark. A 15-person team on Harvest's $11/user/month plan pays $1,980/year. On Toggl Track's premium tier at $18/user/month, that's $3,240/year. Kimai's only cost is the server — a $5/month VPS can comfortably handle a team of that size.

## What You'll Need

- A Linux server (VPS or dedicated) with Docker and Docker Compose installed
- A domain name (e.g., `time.yourcompany.com`) with DNS pointing to your server
- 1 GB RAM minimum (2 GB recommended for teams over 10)
- About 30 minutes

## Step 1: Create the Docker Compose File

Create a directory for your Kimai deployment and add a `docker-compose.yml` file:

```bash
mkdir -p ~/kimai && cd ~/kimai
```

Create `docker-compose.yml`:

```yaml
services:

  sqldb:
    image: mysql:8.3
    volumes:
      - mysql:/var/lib/mysql
    environment:
      - MYSQL_DATABASE=kimai
      - MYSQL_USER=kimaiuser
      - MYSQL_PASSWORD=change_this_to_a_strong_password
      - MYSQL_ROOT_PASSWORD=change_this_to_a_different_strong_password
    command: --default-storage-engine innodb
    restart: unless-stopped
    healthcheck:
      test: mysqladmin -uroot -p$$MYSQL_ROOT_PASSWORD ping -h 127.0.0.1 --silent
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

  kimai:
    image: kimai/kimai2:stable
    depends_on:
      sqldb:
        condition: service_healthy
        restart: true
    volumes:
      - data:/opt/kimai/var/data
      - plugins:/opt/kimai/var/plugins
    ports:
      - 8001:8001
    environment:
      - APP_SECRET=change_this_to_a_long_random_string_at_least_32_chars
      - "TRUSTED_HOSTS=^time\\.yourcompany\\.com$"
      - ADMINMAIL=admin@yourcompany.com
      - ADMINPASS=change_this_to_a_secure_password
      - "DATABASE_URL=mysql://kimaiuser:change_this_to_a_strong_password@sqldb/kimai?charset=utf8mb4&serverVersion=8.3.0"
    restart: unless-stopped

volumes:
  data:
  mysql:
  plugins:
```

**Important:** Replace every `change_this_*` placeholder with real, strong values. The `APP_SECRET` should be a random string of at least 32 characters. The `TRUSTED_HOSTS` value should match your actual domain. Generate a secure secret with:

```bash
openssl rand -hex 32
```

## Step 2: Start Kimai

```bash
cd ~/kimai
docker compose up -d
```

The first startup takes 30–60 seconds — MySQL needs to initialize, and Kimai runs its database migrations. Check the logs to see when it's ready:

```bash
docker compose logs -f kimai
```

When you see lines about the database being ready and the admin user being created, Kimai is up. Open `http://your-server-ip:8001` in a browser and log in with the admin email and password you set.

## Step 3: Add a Reverse Proxy with HTTPS

Running Kimai on port 8001 is fine for testing, but for production you want HTTPS. Caddy is the simplest option — it handles Let's Encrypt certificates automatically.

Add Caddy to your `docker-compose.yml`:

```yaml
  caddy:
    image: caddy:2
    ports:
      - 80:80
      - 443:443
    volumes:
      - caddy_data:/data
      - caddy_config:/config
      - ./Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - kimai
    restart: unless-stopped
```

Create a `Caddyfile` in the same directory:

```
time.yourcompany.com {
    reverse_proxy kimai:8001
}
```

Add the Caddy volumes at the bottom:

```yaml
volumes:
  data:
  mysql:
  plugins:
  caddy_data:
  caddy_config:
```

Now remove the `ports: - 8001:8001` from the kimai service (Caddy will handle external access) and restart:

```bash
docker compose up -d
```

Caddy will automatically provision an SSL certificate from Let's Encrypt. Within a minute or two, `https://time.yourcompany.com` will be live with a valid certificate.

## Step 4: Configure Kimai

After logging in as admin, here's what to set up first:

1. **Create your customers** — Add the clients you bill. Each customer gets a name, address, currency, and optional VAT rate.
2. **Create projects** — Under each customer, add projects. Set the project budget (hours or money) if you want Kimai to warn you when you're approaching the limit.
3. **Create activities** — Common activities like "Development," "Design," "Meeting," "Research." You can also create project-specific activities.
4. **Add team members** — Create user accounts, assign roles, and add them to teams. Each user can be assigned to specific projects.
5. **Configure invoice templates** — Kimai includes default templates, but you'll want to customize the design with your logo, address, and payment terms.
6. **Set hourly rates** — Rates can be set per user, per project, or per activity. Kimai uses these when generating invoices.

## Step 5: Back Up Your Data

Kimai stores data in two places: the MySQL database and the `data` volume (which contains file uploads and configuration). Back up both:

```bash
#!/bin/bash
# backup-kimai.sh — run daily via cron
BACKUP_DIR=/backups/kimai
DATE=$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

# Back up the database
docker exec kimai-sqldb-1 mysqldump -ukimaiuser -p"YOUR_DB_PASSWORD" kimai > $BACKUP_DIR/kimai-db-$DATE.sql

# Back up the data volume
docker run --rm -v kimai_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/kimai-data-$DATE.tar.gz /data

# Keep 30 days of backups
find $BACKUP_DIR -name "kimai-*" -mtime +30 -delete
```

Add this to cron for automatic daily backups:

```bash
crontab -e
# Add:
0 2 * * * /path/to/backup-kimai.sh
```

For off-site backup, sync the backup directory to your preferred storage — an S3-compatible bucket (MinIO, Backblaze B2), or even a simple rsync to another server.

## Updating Kimai

Updates are straightforward — pull the latest image and restart:

```bash
cd ~/kimai
docker compose pull
docker compose down
docker compose up -d
```

Kimai automatically runs database migrations on startup. The `:stable` tag stays within the current major version, so you won't get surprise breaking changes. For more control, pin a specific version like `kimai/kimai2:2.20`.

Always back up before updating. Always.

## Connecting Kimai to n8n for Automation

Kimai's JSON API opens up automation possibilities. With n8n, you can build workflows that react to time tracking events. Here are a few practical examples:

**Daily time summary in team chat:** A scheduled n8n workflow queries the Kimai API for all time entries logged yesterday, formats them into a summary, and posts to a Mattermost or Rocket.Chat channel. Team leads see who logged what without opening Kimai.

**Auto-create invoices:** A monthly workflow gathers all uninvoiced, billable time entries for a customer, creates an invoice in Kimai via the API, and emails the PDF to the client. No more forgetting to bill at month-end.

**Sync to accounting:** Export approved time entries from Kimai and import them into Odoo or Invoice Ninja for revenue recognition and project profitability tracking.

Here's a simple n8n HTTP Request node configuration to fetch time entries:

```json
{
  "method": "GET",
  "url": "https://time.yourcompany.com/api/timesheets",
  "authentication": "headerAuth",
  "headers": {
    "X-AUTH-USER": "your-api-user",
    "X-AUTH-TOKEN": "your-api-token"
  },
  "queryStringParameters": {
    "size": 100
  }
}
```

You'll need to create an API user in Kimai (Users → your user → API settings → generate API token) and use those credentials in n8n.

## Who Should Self-Host Kimai?

**Good fit if:**
- You have 5+ team members tracking time — the savings over commercial tools add up quickly
- You already self-host other tools (n8n, Odoo, Mattermost) — adding Kimai to the stack is trivial
- You need full data ownership for compliance, privacy, or client contract reasons
- You want invoicing and reporting without paying for premium tiers
- Your team needs SSO/LDAP without enterprise pricing

**Maybe not if:**
- You have 1–2 people and just need a simple timer — the overhead of hosting isn't worth it
- You need mobile apps with offline sync — Kimai's mobile experience is browser-based, not a native app
- You don't have anyone comfortable with Docker and basic server administration

## The Bottom Line

Kimai gives you the core features of Toggl, Harvest, and Clockify — time tracking, reporting, invoicing, team management — without per-user pricing or vendor lock-in. If you're already self-hosting your business tools, it's a natural addition. If you're paying $100+ per month for time tracking across your team, self-hosting Kimai on a $5/month VPS pays for itself in the first week.

The setup takes about 30 minutes. The Docker Compose file above is a production-ready starting point. Add Caddy for HTTPS, set up backups, and you're running a full time tracking platform on your own terms.

---

*Want help setting up Kimai or connecting it to your existing automation stack? ARDOT Consulting specializes in open source infrastructure and AI automation for small businesses. [Get in touch through our contact form](/) — we'll help you get up and running fast.*