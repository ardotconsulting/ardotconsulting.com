---
layout: post
title: "Self-Hosting Mattermost: Replace Slack with Your Own Team Chat Platform"
date: 2026-10-08
author: "ARDOT Consulting"
tags: [mattermost, self-hosting, team-chat, open-source, slack-alternative, docker]
excerpt: "Slack's per-seat pricing adds up fast. Here's how to self-host Mattermost — a fully featured open source team chat platform — with Docker, keep your data under your own roof, and cut your messaging bill to near zero."
---

Every time someone joins your team, Slack sends you a bill. At $8–$15 per user per month, a 25-person company pays $2,400–$4,500 a year just for chat. Add in the fact that your conversations — including sensitive business discussions, file shares, and integration webhooks — live on someone else's servers, and you start to wonder whether there's a better way.

There is. [Mattermost](https://mattermost.com/) is an open source team messaging platform that does most of what Slack does, runs on your own infrastructure, and costs nothing in licensing for teams using the free Team Edition. You own the data, you control the access, and you never get a per-seat invoice.

In this guide, we'll walk through what Mattermost actually offers, how it compares to Slack, and how to get it running with Docker in under an hour.

## What Mattermost Actually Does

Mattermost is a real-time messaging platform designed for teams. If you've used Slack, the interface will feel immediately familiar — channels on the left, messages in the center, threads, direct messages, file attachments, emoji reactions. But under the hood, it's a fundamentally different model.

**Core features:**

- **Channels** — Public, private, and direct message channels organized by team
- **Threads** — Reply in threads to keep conversations organized
- **File sharing** — Drag-and-drop file uploads with preview support
- **Search** — Full-text search across all messages and files you have access to
- **Integrations** — Incoming webhooks, outgoing webhooks, slash commands, and bot accounts
- **Mobile apps** — Native iOS and Android apps that connect to your self-hosted server
- **Desktop apps** — Native apps for Windows, Mac, and Linux
- **Message history** — No limit on message history (Slack's free plan caps you at 10,000 messages)

The key difference from Slack isn't the feature list — it's the architecture. Mattermost runs on a server you control. Your messages, files, and metadata never touch a third-party SaaS provider.

## Mattermost vs Slack: The Honest Comparison

Let's not pretend Mattermost is a drop-in replacement for every Slack feature. It's not. But for most small-to-mid-size businesses, the trade-offs favor self-hosting. Here's an honest comparison:

| Feature | Slack (Pro) | Mattermost (Team Edition) |
|---------|-----------|--------------------------|
| Monthly cost per user | $8–$15 | $0 |
| Message history limit | 90 days | Unlimited |
| File storage limit | 10–20 GB per user | Whatever your server has |
| Data location | Slack's cloud | Your server |
| Data retention control | Limited | Full control |
| SSO/SAML | Enterprise plan only | Built in (LDAP/AD) |
| Custom integrations | App Marketplace + API | Webhooks, API, plugins |
| Guest accounts | Paid add-on | Free, unlimited |
| Self-hosting option | No | Yes |
| Compliance (HIPAA, FedRAMP) | Enterprise only | Possible with self-hosting |
| Setup effort | None (SaaS) | 1–2 hours initial setup |
| Maintenance effort | None | Patching, backups, monitoring |

The trade-off is clear: Slack is zero-effort but you pay per seat and hand over your data. Mattermost requires some upfront setup and ongoing maintenance, but you keep your data and eliminate per-seat costs.

**When Slack makes more sense:** You have fewer than 10 people, no compliance requirements, and you'd rather not manage infrastructure. The convenience is worth the cost.

**When Mattermost makes more sense:** You have 15+ people, you care about data sovereignty, you have compliance requirements, or you already run your own servers for other things.

## What You'll Need

Before we start, here's the infrastructure you'll need:

- **A server** — A VPS with 2 GB RAM and 20 GB disk (a $5–$10/month VPS from Hetzner, OVH, or DigitalOcean works fine). If you already have a server running other services, Mattermost can share it.
- **Docker and Docker Compose** — We'll use Docker to keep the installation clean and portable.
- **A domain name** — Something like `chat.yourcompany.com` with an SSL certificate (we'll use Let's Encrypt, which is free).
- **Basic comfort with the command line** — You don't need to be a sysadmin, but you should be comfortable running commands and editing configuration files.

## Step 1: Set Up the Directory Structure

On your server, create a directory for the Mattermost deployment:

```bash
mkdir -p /opt/mattermost/{config,data,logs,plugins}
cd /opt/mattermost
```

The `data` directory is where uploaded files and the database will live, so make sure the disk it's on has enough space. For a 25-person team, 20–50 GB is plenty to start.

## Step 2: Create the Docker Compose File

Create a `docker-compose.yml` file in `/opt/mattermost/`:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    container_name: mattermost-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: mattermost
      POSTGRES_PASSWORD: CHANGE_THIS_TO_A_STRONG_PASSWORD
      POSTGRES_DB: mattermost
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    networks:
      - mattermost-net

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    container_name: mattermost
    restart: unless-stopped
    depends_on:
      - postgres
    ports:
      - "127.0.0.1:8065:8065"
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: "postgres://mattermost:CHANGE_THIS_TO_A_STRONG_PASSWORD@postgres:5432/mattermost?sslmode=disable&connect_timeout=10"
      MM_SERVICESETTINGS_SITEURL: "https://chat.yourcompany.com"
      MM_SERVICESETTINGS_ENABLELOCALMODE: "true"
      MM_FILESETTINGS_DIRECTORY: "/mattermost/data"
      MM_LOGSETTINGS_CONSOLELEVEL: INFO
    volumes:
      - ./config:/mattermost/config
      - ./data:/mattermost/data
      - ./logs:/mattermost/logs
      - ./plugins:/mattermost/plugins
    networks:
      - mattermost-net

networks:
  mattermost-net:
    driver: bridge
```

A few things to note:

- **Change the password** — Replace `CHANGE_THIS_TO_A_STRONG_PASSWORD` with a real, strong password. Use the same one in both places.
- **The database password** — Use the same value in the `POSTGRES_PASSWORD` and the `DATASOURCE` connection string.
- **The SiteURL** — Change `chat.yourcompany.com` to your actual domain.
- **Port binding** — We bind to `127.0.0.1:8065` so Mattermost is only accessible locally. We'll put a reverse proxy in front for SSL.

## Step 3: Start the Services

```bash
cd /opt/mattermost
docker compose up -d
```

This downloads the images and starts both PostgreSQL and Mattermost. The first startup takes 1–2 minutes. Check that both containers are running:

```bash
docker compose ps
```

You should see both services with a status of "running." If Mattermost keeps restarting, check the logs:

```bash
docker compose logs mattermost
```

The most common issue is a mismatch between the database password in the `POSTGRES_PASSWORD` environment variable and the `DATASOURCE` connection string. Double-check they're identical.

## Step 4: Set Up the Reverse Proxy with SSL

Mattermost listens on port 8065 locally, but you need HTTPS for browser security and for the Mattermost desktop and mobile apps to connect properly. We'll use Caddy as a reverse proxy because it automatically handles Let's Encrypt certificates — no manual cert management.

Create a `Caddyfile`:

```
chat.yourcompany.com {
    reverse_proxy localhost:8065
}
```

Install and start Caddy (on Ubuntu/Debian):

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Caddy will start automatically, read the Caddyfile, obtain a Let's Encrypt certificate, and proxy traffic to Mattermost. Within a minute, `https://chat.yourcompany.com` should show the Mattermost setup screen.

If you prefer nginx, you can use `nginx` with `certbot` — the setup is a bit more involved but equally valid. Caddy is simpler because the SSL certificate management is fully automatic.

## Step 5: Create Your Admin Account

Visit `https://chat.yourcompany.com` in your browser. You'll see the initial setup screen:

1. **Create the first user** — Enter your email, username, and password. This becomes the admin account.
2. **Create your first team** — Give it a name (e.g., "YourCompany" or "Engineering"). This becomes the primary workspace.
3. **Create your first channels** — Mattermost creates default channels like "Town Square" (the general channel) and "Off-Topic." Add channels for your teams or projects.

That's it. Your Mattermost instance is live and accessible to your team. Share the URL with your team members and they can create accounts from the sign-up page.

## Step 6: Configure Key Settings

Once you're logged in as admin, go to **System Console** (the gear icon → System Console) and review these important settings:

### Email Notifications

Without email notifications, your team won't know when they have messages. Go to **Environment → SMTP** and configure your SMTP server. If you don't have an SMTP server, you can use a service like [Postmark](https://postmark.com/) or run your own with [Postal](https://docs.postalserver.io/).

Test the connection from the System Console before moving on.

### File Upload Size

By default, Mattermost allows file uploads up to 100 MB. Adjust this in **Site Configuration → File Storage** if you need more or less. Keep in mind that files are stored on your server's disk, so set a reasonable limit based on available space.

### Data Retention

If your industry requires data retention policies (or the opposite — auto-deletion of old messages), configure this in **Data Retention Policy**. You can set rules to keep messages indefinitely, delete after N days, or delete based on channel type.

### Session Length

In **Security → Sessions**, you can control how long users stay logged in. For sensitive environments, set shorter session lengths and require re-authentication.

## Step 7: Invite Your Team and Create Channels

With the server configured, set up your team structure:

**Channel suggestions for a small business:**

| Channel | Purpose | Visibility |
|---------|---------|------------|
| `#town-square` | Company-wide announcements | Public |
| `#general` | Day-to-day discussion | Public |
| `#engineering` | Dev team discussions | Public |
| `#sales` | Sales pipeline, lead updates | Private |
| `#hr` | HR discussions, hiring | Private |
| `#incidents` | Production incidents, on-call | Public |
| `#random` | Non-work chatter | Public |

To invite people, go to the team menu → **Get Team Invite Link**. Share the link with your team. They'll create their own accounts and join automatically.

For larger organizations, you can connect Mattermost to LDAP or Active Directory for centralized authentication. This is available in the free Team Edition — no enterprise license needed.

## Step 8: Set Up Integrations

Mattermost supports the same integration patterns as Slack: incoming webhooks, outgoing webhooks, slash commands, and bot accounts. Here's a practical example.

### Example: Post a Notification from n8n

If you use n8n for automation (see our [n8n guide](/blog/2026/08/17/how-to-build-your-first-ai-automation-workflow-with-n8n/)), you can send notifications to a Mattermost channel when a workflow completes.

1. In Mattermost, go to the channel where you want notifications → **Channel Settings → Integrations → Incoming Webhook**.
2. Create a webhook and copy the URL.
3. In n8n, add an HTTP Request node:

```json
{
  "method": "POST",
  "url": "https://chat.yourcompany.com/hooks/XXXXXXXXXX",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "text": "✅ Workflow completed: {{ $json.workflowName }} at {{ $now }}"
  }
}
```

Now every time your n8n workflow runs, you get a notification in Mattermost. This works for any system that can make an HTTP POST — Odoo, Metabase, your CI/CD pipeline, custom scripts, anything.

### Slash Commands

You can create custom slash commands that trigger external actions. For example, `/lookup ACME Corp` could query your CRM and return the company's details in the channel. This is configured in **Integrations → Slash Commands** with a callback URL that receives the command and returns a Mattermost-formatted response.

## Step 9: Back Up Your Data

This is the most important step. Since you're self-hosting, you're responsible for backups. Here's a simple backup strategy:

**What to back up:**
- The PostgreSQL database (contains all messages, users, and settings)
- The `data` directory (contains uploaded files)

**Simple backup script:**

```bash
#!/bin/bash
BACKUP_DIR="/backups/mattermost"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Back up the database
docker exec mattermost-postgres pg_dump -U mattermost mattermost | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Back up uploaded files
tar czf $BACKUP_DIR/files_$DATE.tar.gz -C /opt/mattermost data/

# Keep only the last 7 days of backups
find $BACKUP_DIR -mtime +7 -delete
```

Schedule this with a cron job:

```bash
# Run backup daily at 2 AM
0 2 * * * /opt/mattermost/backup.sh
```

For offsite backups, sync the backup directory to an S3-compatible storage service like MinIO (which you can also self-host) or a remote server using `rsync`.

## Step 10: Keep It Updated

Mattermost releases updates regularly. To update:

```bash
cd /opt/mattermost
docker compose pull
docker compose up -d
```

This pulls the latest image and restarts the containers. The database schema is migrated automatically. The whole process takes less than a minute.

**Update best practices:**
- Check the [release notes](https://docs.mattermost.com/administration/upgrade.html) before upgrading to a major version
- Back up your database before upgrading (run the backup script above)
- Test on a staging instance if you have critical workflows

## The Real Cost Comparison

Let's put real numbers on this. Assume a 25-person team:

| Cost Item | Slack (Pro) | Mattermost (Self-Hosted) |
|-----------|------------|--------------------------|
| Per-seat licensing | $2,400/year ($8/user × 25 × 12) | $0 |
| Server (VPS) | — | $60–120/year |
| Domain + SSL | — | $10/year (SSL is free via Let's Encrypt) |
| Backup storage | — | $0–50/year |
| **Year 1 total** | **$2,400** | **$70–180** |
| **Year 2 total** | **$4,800** | **$140–360** |
| **3-year total** | **$7,200** | **$210–540** |

For a 25-person team, Mattermost saves roughly **$2,200–$2,300 per year** even after accounting for server costs. For a 50-person team, the savings double. The larger your team, the more dramatic the savings.

The hidden cost is your time. Budget 2–4 hours for initial setup and 1–2 hours per quarter for updates and maintenance. If you value your time at $100/hour, that's $200–$400/year in labor — still a fraction of the Slack licensing cost.

## What About the Desktop and Mobile Apps?

Mattermost provides native apps for all platforms:

- **Desktop:** Windows, macOS, Linux — available from [mattermost.com/apps](https://mattermost.com/apps/) or your package manager
- **Mobile:** iOS (App Store) and Android (Play Store/F-Droid)

The apps connect to your self-hosted server by entering the server URL. They support push notifications (Mattermost runs its own push notification service — the HPNS — which is free for the Team Edition).

If you've used the Slack desktop app, the Mattermost desktop app feels nearly identical: multiple teams in a sidebar, unread badges, notification preferences, dark mode.

## When Self-Hosting Chat Makes Sense

Self-hosting your team chat isn't for everyone. But it makes particular sense if:

1. **You're already self-hosting other tools** — If you run Odoo, n8n, or Plausible on your own server, adding Mattermost is a natural extension. You already have the skills and infrastructure.

2. **You have compliance requirements** — HIPAA, GDPR, or industry-specific data handling rules are much easier to satisfy when your data stays on servers you control.

3. **You have a growing team** — The per-seat cost of Slack scales linearly with headcount. Mattermost doesn't. At 30+ people, the savings become significant.

4. **You want unlimited message history** — Slack's free and low-tier plans cap your message history. Mattermost keeps everything, forever, at no extra cost.

5. **You want deeper integrations** — Self-hosted Mattermost gives you full access to the database and API. You can build custom integrations, run analytics on message patterns, or connect it to your data warehouse.

## Common Gotchas

A few things catch people off guard when setting up Mattermost for the first time:

**Email notifications don't work out of the box.** Mattermost needs an SMTP server to send emails. If you skip this, your team won't get email notifications for mentions or direct messages. Configure SMTP in the System Console before inviting your team.

**Push notifications need the internet.** Even though Mattermost is self-hosted, mobile push notifications go through Mattermost's notification relay service (HPNS). This is free but does require outbound internet access from your server. If you're on a fully air-gapped network, push notifications won't work without additional configuration.

**File uploads can fill your disk.** Unlike Slack, where file storage is someone else's problem, Mattermost stores files on your server. Monitor disk usage and set upload size limits. A 25-person team uploading documents and screenshots will generate 5–20 GB of files per year.

**Upgrades occasionally require database migration.** Major version upgrades may run database migrations that can take several minutes. Schedule upgrades during off-hours and always back up first.

## Next Steps

Mattermost is just one piece of a self-hosted communication stack. Once your team chat is running, consider:

- **Connect it to your automation** — Use n8n to send automated alerts to Mattermost channels when things happen (new leads, completed tasks, system alerts)
- **Add Mattermost to your backup strategy** — Include the database and file storage in your regular backup routine
- **Set up monitoring** — Use a tool like Uptime Kuma to alert you if your Mattermost instance goes down
- **Create a guest access policy** — If you work with external contractors, configure guest accounts with limited channel access

Self-hosting your team chat is one of the highest-impact, lowest-cost moves a small business can make. You get enterprise-grade messaging, unlimited history, full data control, and per-seat costs that stay at zero no matter how much your team grows.

---

*Want help setting up Mattermost or migrating from Slack? [Get in touch](/#contact) — ARDOT Consulting specializes in open source infrastructure for small businesses. We'll handle the setup, migration, and training so your team can start chatting on day one.*