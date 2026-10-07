---
layout: post
title: "Self-Hosting Ghost: Replace Medium and Substack With Your Own Publishing Platform"
date: 2026-11-15
author: "ARDOT Consulting"
tags: [ghost, self-hosting, publishing, newsletter, cms, open-source]
excerpt: "Ghost is a full publishing platform — blog, newsletter, and membership — that you control end to end. Here's how to self-host it and stop paying publishing platforms for your own content."
---

If you publish a company blog, a newsletter, or both, you're probably paying a platform to host your own words. Medium charges readers to access your posts. Substack takes 10% of every paid subscription. Both control your subscriber list, your design, and your content's presentation.

[Ghost](https://ghost.org/) is an open source publishing platform that handles blogs, newsletters, and paid memberships in one application. You self-host it, own your subscriber data, and keep 100% of subscription revenue. No platform fees, no algorithm deciding who sees your work, no surprise pricing changes.

In this guide, we'll walk through what Ghost does, when it makes sense to self-host, and how to deploy it with Docker Compose.

---

## What Ghost Replaces

Ghost combines three tools into one:

| Capability | Proprietary Alternative | Ghost Feature |
|---|---|---|
| Blog publishing | Medium, WordPress.com | Full CMS with Markdown editor, SEO tools, themes |
| Email newsletter | Substack, Mailchimp, ConvertKit | Built-in newsletter sending via Mailgun/SES |
| Paid memberships | Patreon, Substack Plus | Native membership tiers, Stripe integration |
| Subscriber management | Spread across multiple tools | Single unified member database |

The key difference: Ghost is a **content business platform**, not just a blog. If you're building an audience and eventually want to charge for premium content, Ghost handles the entire pipeline — from publishing free posts to collecting Stripe payments for subscriber-only material.

---

## When Self-Hosting Ghost Makes Sense

Self-hosting isn't the right call for everyone. Here's a quick decision guide:

**Self-host if:**
- You publish regularly and want full control over design and data
- You plan to offer paid memberships (the savings vs. Substack's 10% fee add up fast)
- You already self-host other tools (n8n, Plausible, Metabase) and have infrastructure in place
- You want to integrate your publishing platform with automation workflows

**Use Ghost(Pro) hosted if:**
- You don't want to manage server updates or backups
- You have a small team with no DevOps capacity
- You just want to write and not think about infrastructure

Ghost's official hosting (Ghost Pro) is a managed service starting around $9/month. It's reasonably priced and supports the open source project. But if you're already running a self-hosted stack — or if you publish enough that Substack's 10% cut stings — self-hosting is the better economics.

---

## What You'll Need

- A Linux server (VPS with 2GB RAM minimum — Ghost is Node.js-based and memory-hungry)
- Docker and Docker Compose installed
- A domain name (e.g., `blog.yourcompany.com`)
- A Mailgun or Amazon SES account for newsletter delivery (the free tier of either works to start)

---

## Step 1: Create the Docker Compose Setup

Create a directory for your Ghost installation:

```bash
mkdir -p ~/ghost-stack && cd ~/ghost-stack
```

Create a `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  ghost:
    image: ghost:latest
    container_name: ghost
    restart: unless-stopped
    ports:
      - "2368:2368"
    environment:
      # The URL your blog will be accessed at
      url: https://blog.yourcompany.com
      # Use MySQL for production (SQLite works for testing but isn't recommended at scale)
      database__client: mysql
      database__connection__host: ghost-db
      database__connection__user: ghost
      database__connection__password: ${DB_PASSWORD}
      database__connection__database: ghost
      # Mail configuration for newsletter delivery
      mail__transport: SMTP
      mail__options__host: smtp.mailgun.org
      mail__options__port: 587
      mail__options__auth__user: postmaster@yourcompany.com
      mail__options__auth__pass: ${MAIL_PASSWORD}
      mail__from: '"Your Company" <newsletter@yourcompany.com>'
    volumes:
      - ghost-content:/var/lib/ghost/content
    depends_on:
      - ghost-db

  ghost-db:
    image: mysql:8
    container_name: ghost-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ghost
      MYSQL_USER: ghost
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ghost-db:/var/lib/mysql

volumes:
  ghost-content:
  ghost-db:
```

Create a `.env` file in the same directory:

```bash
# Generate strong passwords — don't copy these
DB_ROOT_PASSWORD=change-this-to-a-strong-password
DB_PASSWORD=change-this-to-a-different-strong-password
MAIL_PASSWORD=your-mailgun-smtp-password
```

---

## Step 2: Start Ghost

```bash
docker compose up -d
```

Give it about 30 seconds to initialize the database. Then check the logs:

```bash
docker compose logs -f ghost
```

You should see Ghost start up and report that it's listening on port 2368. Hit `Ctrl+C` to exit the log stream (Ghost keeps running in the background).

---

## Step 3: Set Up Reverse Proxy with SSL

Ghost needs HTTPS for Stripe checkout, newsletter delivery, and SEO. Use Caddy or Nginx as a reverse proxy. Caddy is the simplest — it handles SSL certificates automatically via Let's Encrypt.

Create a `Caddyfile` alongside your docker-compose:

```caddyfile
blog.yourcompany.com {
    reverse_proxy ghost:2368
}
```

Or, if you already have Nginx on your server:

```nginx
server {
    listen 80;
    server_name blog.yourcompany.com;

    location / {
        proxy_pass http://localhost:2368;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Run `certbot --nginx -d blog.yourcompany.com` to get a free SSL certificate from Let's Encrypt.

---

## Step 4: Complete the Setup Wizard

Open `https://blog.yourcompany.com/ghost` in your browser. The first-time setup wizard will:

1. **Create your admin account** — set your name, email, and password
2. **Name your publication** — this appears in your newsletter and site title
3. **Choose a theme** — Ghost ships with a clean default (Caspar). You can install free or premium themes from the Ghost Marketplace
4. **Connect Stripe** (optional) — if you want to offer paid memberships, connect your Stripe account. Ghost handles subscription billing, tiers, and member access automatically
5. **Configure newsletter settings** — set your "from" address and test delivery

Once complete, you'll see the Ghost admin dashboard — a clean, distraction-free editor that supports Markdown and rich media.

---

## Step 5: Write and Publish Your First Post

Ghost's editor is one of its best features. It's a block-based editor — type Markdown for speed, or use the `/` command to insert images, galleries, bookmarks, code blocks, buttons, and more.

**To publish a free blog post:**
1. Click "Posts" → "New post"
2. Write your content
3. Click "Publish" → choose "Publish and send email" if you also want to email it to free members
4. Click "Continue" → "Confirm publish"

**To create a paid members-only post:**
1. Write your post
2. In the right sidebar, under "Access", select "Paid-members only"
3. Publish — free members see a preview with a paywall CTA prompting them to upgrade

This is where Ghost shines. The same post can serve as:
- A public blog post (for SEO and discovery)
- A free newsletter (sent to all subscribers)
- A paid membership exclusive (generating revenue)

No separate tools. No data sync. One publish action.

---

## Step 6: Configure Memberships and Pricing

Navigate to **Settings → Membership** to configure your tiers:

| Tier | Use Case | Example Pricing |
|---|---|---|
| Free | Casual readers, newsletter subscribers | $0 |
| Monthly | Premium content access | $5–$15/month |
| Yearly | Committed subscribers (discount incentive) | $50–$150/year |

Ghost integrates directly with Stripe — you don't need a separate checkout system. Members can subscribe, upgrade, downgrade, or cancel from a self-service portal. All member data lives in your Ghost database, not on a third-party platform.

**The economics matter.** On Substack, a $10/month subscription costs you $1/month in platform fees (10%). At 500 subscribers, that's $500/month — $6,000/year — going to Substack. Self-hosted Ghost with a $6/month VPS and Mailgun's pay-as-you-go email costs you under $15/month total, regardless of subscriber count.

---

## Integrating Ghost with Your Automation Stack

Ghost has a robust [Admin API](https://ghost.org/docs/admin-api/) and [Content API](https://ghost.org/docs/content-api/) that work well with automation tools like n8n.

### Example: Auto-share new posts to social media

In n8n, create a workflow that:
1. Polls the Ghost Content API for new posts every 30 minutes
2. Posts a summary to your team's Mattermost channel
3. Queues a social media post via your scheduling tool

```javascript
// n8n HTTP Request node — fetch recent posts from Ghost
// Method: GET
// URL: https://blog.yourcompany.com/ghost/api/content/posts/
// Headers:
//   Accept: application/json
// Query params:
//   key: YOUR_CONTENT_API_KEY
//   limit: 5
//   filter: published_at:>'2026-11-14'
```

### Example: Sync members with your CRM

If you use Odoo or SuiteCRM for customer management, an n8n workflow can sync new Ghost members to your CRM contacts list — tagging them as "blog subscriber" so your sales team knows who's engaged with your content.

---

## Backing Up Ghost

Your Ghost instance contains your content, subscriber data, and theme customizations. Back it up regularly:

```bash
#!/bin/bash
# Simple Ghost backup script — run daily via cron

BACKUP_DIR=/opt/backups/ghost
DATE=$(date +%Y%m%d)

mkdir -p $BACKUP_DIR

# Dump the MySQL database
docker exec ghost-db mysqldump -u ghost -p"$DB_PASSWORD" ghost > $BACKUP_DIR/ghost-db-$DATE.sql

# Copy the content directory (images, themes, files)
docker cp ghost:/var/lib/ghost/content $BACKUP_DIR/ghost-content-$DATE

# Keep only the last 7 days of backups
find $BACKUP_DIR -maxdepth 1 -mtime +7 -exec rm -rf {} \;
```

Add this to crontab: `0 3 * * * /opt/scripts/ghost-backup.sh`

For offsite backups, sync the backup directory to an S3-compatible storage service (like MinIO, which you can also self-host) using `rclone`.

---

## Keeping Ghost Updated

Ghost releases updates frequently — usually monthly. To update:

```bash
cd ~/ghost-stack
docker compose pull ghost
docker compose up -d
```

Ghost runs automatic database migrations on startup, so you don't need to run migration scripts manually. The whole process takes about 30 seconds of downtime.

Always check the [Ghost release notes](https://github.com/TryGhost/Ghost/releases) before upgrading, and take a backup first.

---

## Theme Customization

Ghost themes use Handlebars templating. The default Caspar theme is clean and professional, but you'll likely want to customize it.

**Free themes:** Browse the [Ghost Marketplace](https://ghost.org/themes/) for community themes. Most are MIT-licensed and can be modified freely.

**Custom theme development:**
```bash
# Install the Ghost CLI theme tools
npm install -g gscan

# Validate your theme before uploading
gscan /path/to/your/theme
```

Upload themes via **Settings → Design → Upload theme**. Ghost supports hot-swapping themes without downtime.

---

## Ghost vs. Other Publishing Tools

| Feature | Ghost (Self-Hosted) | Substack | Medium | WordPress |
|---|---|---|---|---|
| Platform fees | $0 | 10% of revenue | $5/mo reader paywall | $0 (self-hosted) |
| Newsletter built-in | Yes | Yes | No | Via plugins |
| Paid memberships | Yes (Stripe) | Yes | Yes (limited) | Via plugins |
| Subscriber data ownership | Full | Partial | No | Full |
| Design control | Full themes | No | Minimal | Full themes |
| Open source | Yes (MIT) | No | No | Yes (GPL) |
| Maintenance burden | Medium | None | None | Medium |

Ghost occupies a sweet spot: more control than Substack or Medium, less complexity than WordPress with its constellation of newsletter and membership plugins.

---

## Common Gotchas

**1. Ghost is memory-hungry.** A 1GB VPS will struggle. Use at least 2GB RAM, or configure Node.js memory limits in your Docker environment:

```yaml
environment:
  NODE_OPTIONS: --max-old-space-size=512
```

**2. Email delivery is critical.** Ghost's newsletter feature depends on your SMTP provider. Test delivery before announcing your newsletter to subscribers. Mailgun's free tier allows 5,000 emails/month for the first 3 months, then 1,000/month — sufficient for a small list. Amazon SES is cheaper at scale ($0.10 per 1,000 emails) but requires more setup.

**3. Don't use SQLite for production.** Ghost defaults to SQLite, which works for testing but can corrupt under concurrent load. Always use the MySQL configuration shown above for real deployments.

**4. Backup before upgrading.** Ghost's migrations are generally smooth, but a failed migration on an unbacked-up database is a bad day. Always run the backup script before `docker compose pull`.

**5. Set `url` correctly.** The `url` environment variable must match your actual domain with the correct protocol (`https://`). If Ghost thinks it's at `http://localhost:2368` but visitors access it via `https://blog.yourcompany.com`, newsletter links and RSS feeds will break.

---

## Cost Breakout

Here's what self-hosted Ghost actually costs per month:

| Item | Provider | Monthly Cost |
|---|---|---|
| VPS (2GB RAM) | Hetzner, Contabo, or similar | $4–$6 |
| Domain | Any registrar | ~$1 (annual amortized) |
| Email delivery | Mailgun free tier or Amazon SES | $0–$2 |
| SSL certificate | Let's Encrypt | $0 |
| **Total** | | **$5–$9/month** |

Compare that to Substack's 10% cut: at 200 paid subscribers paying $10/month ($2,000/month in revenue), Substack takes $200/month. Ghost's $6/month VPS saves you $194/month — and the savings scale linearly as your audience grows.

---

## Who This Is For

Self-hosting Ghost makes the most sense if you:

- **Publish regularly** — at least weekly — and want a professional publishing platform
- **Plan to monetize** through paid memberships or newsletters
- **Already self-host other tools** and are comfortable with Docker Compose
- **Want to own your subscriber list** — the most valuable asset any publisher has

If you're just starting out and have 20 readers, Medium or Ghost(Pro) hosted is fine. But once you're serious about building an audience and a content business, the economics and control of self-hosting Ghost are hard to beat.

---

## Wrapping Up

Ghost gives you a professional publishing platform — blog, newsletter, and memberships — with no platform fees and full data ownership. It's one of the best open source tools for any business that takes content seriously.

The setup takes about 30 minutes if you're already familiar with Docker Compose. The ongoing maintenance is minimal: pull updates once a month, run a daily backup script, and monitor your email delivery.

If you want help setting up Ghost or integrating it with your existing automation stack — n8n workflows, CRM sync, social media cross-posting — [get in touch with us](https://www.ardotconsulting.com/#contact). We specialize in open source infrastructure for small businesses, and we'd be happy to help you migrate off publishing platforms that tax your own content.