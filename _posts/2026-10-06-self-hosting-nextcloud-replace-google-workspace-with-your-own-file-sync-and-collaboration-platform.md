---
layout: post
title: "Self-Hosting Nextcloud: Replace Google Workspace with Your Own File Sync, Calendar, and Collaboration Platform"
date: 2026-10-06
author: "ARDOT Consulting"
tags: [nextcloud, self-hosting, file-sync, collaboration, open-source, docker, google-workspace-replacement]
excerpt: "Google Workspace costs $7–$14 per user per month and locks your company's files in someone else's cloud. Here's how to replace it with Nextcloud — a self-hosted platform for file sync, calendar, contacts, and document collaboration that you fully control."
---

Your business runs on documents, calendars, and shared files. Right now, most of that lives in Google Workspace. You pay $7 to $14 per user per month for the privilege of storing your company's most sensitive information on servers you don't control, governed by terms of service you've never read, in a data center you can't visit. When Google decides to change a feature, deprecate a service, or adjust pricing — and they do, regularly — you have no say in the matter.

For a 15-person team on the Business Standard plan at $12/user/month, that's $2,160 per year. Not catastrophic, but not trivial either. The real cost isn't the subscription fee, though. It's the dependency. Your files, your calendar history, your contacts, your shared documents — all of it lives in a system you can't self-audit, can't back up to your own metal without proprietary tooling, and can't run offline if your internet drops or Google has an outage.

There's a better way. **Nextcloud** is a free, open-source platform that does most of what Google Workspace does — file sync, calendar, contacts, document editing, video chat, task management — and runs entirely on your own server. No per-user fees. No data leaving your network. No vendor making decisions about your workflow.

This guide walks through setting up Nextcloud with Docker, migrating your data off Google, and configuring the features your team actually uses day-to-day.

## What Nextcloud Replaces

Before we get into setup, let's be honest about what Nextcloud does well and where it falls short. No tool is perfect, and Nextcloud has trade-offs you should understand before committing.

| Google Workspace Feature | Nextcloud Equivalent | How Well It Works |
|--------------------------|----------------------|-------------------|
| Google Drive (file storage) | Nextcloud Files | Excellent — full sync, sharing, versioning |
| Google Calendar | Nextcloud Calendar | Excellent — CalDAV standard, works with all clients |
| Google Contacts | Nextcloud Contacts | Excellent — CardDAV standard, syncs everywhere |
| Google Docs (collaborative editing) | Nextcloud Text / Collabora | Good — real-time co-editing, slightly less polished |
| Google Sheets | Collabora Calc | Good — covers most spreadsheet needs |
| Google Meet | Nextcloud Talk | Adequate — works for internal calls, not great for large external meetings |
| Gmail | Nextcloud Mail (or self-hosted mail server) | Adequate for light use; most teams keep a dedicated mail provider |
| Google Chat | Nextcloud Talk (chat) | Adequate for internal team chat |

The file sync, calendar, and contacts replacements are genuinely excellent. Nextcloud uses industry standards (WebDAV, CalDAV, CardDAV) that work with every operating system and mobile app. You're not locked into Nextcloud's client — you can use any CalDAV-compatible calendar app.

Document collaboration is good but not as seamless as Google Docs. Nextcloud integrates with **Collabora** (a LibreOffice-based online editor) or **OnlyOffice** for real-time document editing. It works, but the interface is a bit more utilitarian. For most business documents — proposals, internal memos, meeting notes — it's more than sufficient.

Email is the one area where Nextcloud isn't a full replacement. While it has a mail app, running a full mail server (with spam filtering, DKIM, DMARC, deliverability management) is a specialized skill that most small businesses should outsource. We recommend keeping a dedicated email provider — we use and recommend **Mailcow** (open source, self-hosted) or a privacy-focused provider like **Posteo** or **Migadu** for email, and using Nextcloud for everything else.

## What You Need

- **A server:** A VPS with at least 2GB RAM and 40GB storage (Hetzner, OVH, or any provider you trust). For a team of 15 with moderate file storage, 4GB RAM and 100GB+ storage is more comfortable. You can also run it on a dedicated office machine.
- **Docker and Docker Compose:** Installed on your server.
- **A domain name:** Something like `files.yourcompany.com`. You can get one from any registrar — we use and recommend **Porkbun** or **Namecheap**.
- **30 minutes:** The actual setup is fast. Migration takes longer depending on how much data you have.

## Step 1: Set Up Nextcloud with Docker

Create a directory for your Nextcloud deployment and make a `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  db:
    image: postgres:16
    container_name: nextcloud_db
    restart: unless-stopped
    volumes:
      - ./db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=CHANGE_THIS_TO_A_STRONG_PASSWORD
    networks:
      - nextcloud_net

  nextcloud:
    image: nextcloud:apache
    container_name: nextcloud_app
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "8080:80"
    volumes:
      - ./nextcloud_data:/var/www/html
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=nextcloud
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=CHANGE_THIS_TO_A_STRONG_PASSWORD
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=CHANGE_THIS_TOO
      - NEXTCLOUD_TRUSTED_DOMAINS=files.yourcompany.com
    networks:
      - nextcloud_net

  cron:
    image: nextcloud:apache
    container_name: nextcloud_cron
    restart: unless-stopped
    depends_on:
      - nextcloud
    volumes:
      - ./nextcloud_data:/var/www/html
    entrypoint: |
      /bin/sh -c "echo '*/5 * * * * php /var/www/html/occ cron:run' > /var/spool/cron/crontabs/www-data && crond -f"
    networks:
      - nextcloud_net

networks:
  nextcloud_net:
    driver: bridge
```

A few things to note about this setup:

- **PostgreSQL** is used instead of the default SQLite. For any team larger than 2-3 people, you want a real database. SQLite works for testing but will slow down under concurrent use.
- **The cron container** runs Nextcloud's background jobs every 5 minutes. This handles things like file scanning, notification delivery, and calendar reminders. Without it, some features silently stop working.
- **Trusted domains** are set to your actual domain. Nextcloud blocks requests from unknown domains as a security measure.

Change the passwords, then start it up:

```bash
docker compose up -d
```

Wait about 30 seconds for the containers to initialize, then check that it's running:

```bash
docker compose ps
```

You should see all three containers (db, app, cron) running. Nextcloud is now accessible on port 8080 of your server.

## Step 2: Put It Behind a Reverse Proxy with HTTPS

Running Nextcloud on port 8080 without encryption is fine for testing, but you need HTTPS before anyone actually uses it. We'll use **Caddy** as a reverse proxy because it automatically handles Let's Encrypt certificates — no manual certbot renewals, no certificate expiry surprises.

Add Caddy to your docker-compose setup:

```yaml
  caddy:
    image: caddy:2
    container_name: nextcloud_caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - nextcloud
    networks:
      - nextcloud_net

volumes:
  caddy_data:
  caddy_config:
```

Create a `Caddyfile` in the same directory:

```
files.yourcompany.com {
    reverse_proxy nextcloud:80
    
    header {
        Strict-Transport-Security "max-age=31536000; includeSubdomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        Referrer-Policy "no-referrer"
    }
}
```

Point your domain's DNS A record to your server's IP address, restart the stack, and Caddy will automatically obtain and install a Let's Encrypt TLS certificate:

```bash
docker compose up -d
```

Visit `https://files.yourcompany.com` and you'll see the Nextcloud login screen with a valid HTTPS certificate. No manual cert management required.

## Step 3: Configure the Apps Your Team Actually Uses

Nextcloud starts with just file storage enabled. The real value comes from installing the apps that replace specific Google Workspace features. After logging in as admin, go to **Apps** in the top-right menu and install these:

### Calendar

Install the **Calendar** app from the Nextcloud App Store (built-in, one click). Each user gets a personal calendar, and you can create shared calendars for the team — "Company Events," "Project Deadlines," "Meeting Room Booking."

Because Nextcloud Calendar uses CalDAV, it works with every calendar app out there:
- **macOS Calendar:** Add a CalDAV account with the URL `https://files.yourcompany.com/remote.php/dav`
- **Thunderbird:** Built-in CalDAV support, works natively
- **Mobile:** Use **DAVx⁵** (Android) or the built-in CalDAV support on iOS

No proprietary sync protocols. No "Google Calendar Sync" tool. Just the standard that every calendar app already understands.

### Contacts

Install the **Contacts** app. Same story as calendar — it uses CardDAV, so it syncs with macOS Contacts, Thunderbird, iOS, and Android (via DAVx⁵). Your team's contact list lives on your server, not in Google's address book.

### Collaborative Document Editing

Install **Nextcloud Office** (which integrates Collabora) or the **OnlyOffice** app. Both give you real-time collaborative editing of text documents and spreadsheets directly in the browser. Multiple people can edit the same document simultaneously, see each other's cursors, and leave comments.

Collabora is the more popular choice and integrates more tightly with Nextcloud. It's based on LibreOffice, so the interface will feel familiar to anyone who's used LibreOffice or OpenOffice. It's not as polished as Google Docs — the formatting tools are a bit clunkier, and some advanced features (like pivot tables in spreadsheets) are less capable. But for 90% of business documents, it does the job.

If your team creates a lot of documents with complex formatting, you can also use **Nextcloud Text** — a lightweight Markdown editor built into Nextcloud that's excellent for meeting notes, internal documentation, and quick drafts. It's faster and simpler than a full word processor.

### File Sharing

File sharing is Nextcloud's strongest feature. You can:
- **Share files with internal users** — with read-only or edit permissions, optional expiration dates
- **Share via public link** — password-protected, expiration-dated, with optional upload-only mode (great for collecting files from clients)
- **Share entire folders** — with fine-grained permissions per user

This replaces Google Drive's sharing model completely. The advantage is that shared links point to your domain (`files.yourcompany.com/s/abc123`) rather than Google's, which looks more professional in client communications.

### Nextcloud Talk (Chat and Video)

Install **Nextcloud Talk** for internal chat and video calls. It's not a Slack replacement — it's more like a simplified version of Google Meet plus basic chat. For a small team that needs occasional video calls and a persistent chat channel, it works. For larger teams or external collaboration, you'd want a dedicated tool (we like **Mattermost** for self-hosted team chat).

## Step 4: Migrate Your Data Off Google

This is the part that intimidates people, but it's more tedious than difficult. Here's the migration path for each type of data:

### Files (Google Drive)

1. **Export from Google:** Use Google Takeout (takeout.google.com) to download all your Drive files as a zip archive. Select Drive, choose your export format, and wait for Google to prepare the download.
2. **Upload to Nextcloud:** Unzip the archive, then either drag-and-drop files through the Nextcloud web interface (good for small amounts) or use the **Nextcloud Desktop Client** (better for large migrations). The sync client is available for Windows, macOS, and Linux.
3. **Reorganize:** Take this opportunity to clean up your folder structure. You're moving to a new system — might as well organize it properly.

### Calendar (Google Calendar)

1. **Export from Google Calendar:** Go to calendar settings → Import & Export → Export. You'll get an `.ics` file per calendar.
2. **Import to Nextcloud:** In Nextcloud Calendar, click "Import calendar" and upload the `.ics` files.

### Contacts (Google Contacts)

1. **Export from Google Contacts:** Go to contacts.google.com → Export → choose vCard format.
2. **Import to Nextcloud:** In Nextcloud Contacts, click "Import" and upload the vCard file.

### Documents (Google Docs/Sheets)

This is the trickiest part. Google Docs documents aren't real files — they're web app documents stored in Google's proprietary format. To migrate:

1. **Export each document:** In Google Docs, File → Download → choose Microsoft Word (.docx) format. For spreadsheets, choose Excel (.xlsx). You can also export multiple documents at once through Google Drive (select all → right-click → Download).
2. **Upload to Nextcloud:** Upload the .docx/.xlsx files to Nextcloud. If you've installed Collabora or OnlyOffice, you can open and edit them directly in the browser with full collaborative editing.

This is manual but only needs to happen once. For a team with 50-100 active documents, it's a few hours of work. Spread across the team, each person handles their own documents.

## Step 5: Set Up Backups

This is the most important section in this guide. When your data was on Google's servers, Google handled backups. Now that it's on your server, **you** handle backups. This is the trade-off of self-hosting: you gain control, but you also gain responsibility.

At minimum, set up automated backups of two things:

1. **The database:** `docker exec nextcloud_db pg_dump -U nextcloud nextcloud > backup_$(date +%Y%m%d).sql`
2. **The file storage:** The `./nextcloud_data` directory

Here's a simple backup script that runs daily, keeps 7 days of backups, and syncs them to a remote location:

```bash
#!/bin/bash
# Nextcloud daily backup script

BACKUP_DIR="/backups/nextcloud"
NEXTCLOUD_DIR="/opt/nextcloud"
DATE=$(date +%Y%m%d)
RETENTION_DAYS=7

mkdir -p $BACKUP_DIR

# Backup database
docker exec nextcloud_db pg_dump -U nextcloud nextcloud > "$BACKUP_DIR/db_$DATE.sql"

# Backup files (exclude cache and temp)
tar czf "$BACKUP_DIR/files_$DATE.tar.gz" \
  --exclude="$NEXTCLOUD_DIR/nextcloud_data/data/*/cache" \
  --exclude="$NEXTCLOUD_DIR/nextcloud_data/data/*/uploads" \
  "$NEXTCLOUD_DIR/nextcloud_data"

# Clean old backups
find $BACKUP_DIR -name "*.sql" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Sync to remote storage (optional — use rclone with any S3-compatible provider)
# rclone sync $BACKUP_DIR remote:nextcloud-backups/
```

Set this up as a daily cron job on your server. If you want off-site backups (and you should), install **rclone** and sync the backup directory to an S3-compatible storage provider — we recommend **Backblaze B2** or **Wasabi** for affordable object storage. Both work with rclone out of the box.

Test your backups by restoring them at least once. An untested backup is not a backup — it's a hope.

## The Real Cost Comparison

Let's talk numbers for a 15-person team.

| Cost Category | Google Workspace (Business Standard) | Nextcloud (Self-Hosted) |
|---------------|--------------------------------------|-------------------------|
| Monthly subscription | $180/mo ($12/user × 15) | $0 |
| Server (4GB VPS) | Included | ~$8/mo |
| Backup storage (50GB) | Included | ~$2/mo |
| Domain name | ~$1/mo | ~$1/mo |
| IT setup time (one-time) | 2-3 hours | 4-6 hours |
| Maintenance (monthly) | ~0 | ~1 hour |
| **Year 1 total** | **~$2,160** | **~$132 + setup time** |
| **Year 2+ total** | **~$2,160/yr** | **~$132/yr** |

The savings are real: roughly $2,000 per year for a 15-person team, scaling linearly as you add people. The trade-off is that someone needs to maintain the server — updates, backups, the occasional troubleshooting. If you have a part-time IT person or work with a consultant (we know one), this is maybe an hour per month.

The maintenance burden is genuinely low once it's set up. Nextcloud updates are a single command (`docker compose pull && docker compose up -d`), and the platform is mature enough that major issues are rare. The biggest risk is neglecting backups — which is why Step 5 exists.

## When Nextcloud Is NOT the Right Choice

Being honest: Nextcloud isn't for everyone. Here's when you should stick with a hosted solution:

- **You have no one to maintain a server.** If your team is entirely non-technical and you don't have an IT consultant, the self-hosting model adds risk. A managed Nextcloud provider (like Nextcloud's official hosting partners) is a middle ground — you get the open source platform without the server management.
- **You rely heavily on Google Docs' advanced features.** If your team builds complex spreadsheets with pivot tables, apps script, and heavy formatting, Collabora/OnlyOffice will feel limiting. For most business documents it's fine, but power users will notice the difference.
- **You need deep integration with third-party apps that assume Google Workspace.** Some tools only offer "Sign in with Google" and Google Drive integration. If your workflow depends on these, switching requires finding alternatives or accepting manual workarounds.

For most small businesses — especially those whose file and calendar needs are straightforward — Nextcloud is a genuinely good replacement that saves money and gives you ownership of your data.

## Making the Switch Gradually

You don't have to migrate everything in one weekend. A phased approach works well:

1. **Week 1:** Set up Nextcloud, configure HTTPS, create user accounts. Migrate calendar and contacts (fast, low-risk).
2. **Week 2-3:** Migrate files. Start with the current active project files, then archive older files in batches.
3. **Week 4:** Set up Collabora for document editing. Migrate active documents.
4. **Week 5+:** Stop creating new content in Google Workspace. Keep Google Workspace active for a month as a safety net, then cancel.

This gives your team time to adjust and ensures nothing critical gets lost in the transition. By the end of the second month, you're fully on Nextcloud and saving ~$180/month.

## Wrapping Up

Google Workspace is convenient, but convenience isn't the same as control. Every file you store in Google Drive is a file you're renting access to rather than owning. Every calendar event in Google Calendar is data that lives on Google's terms, not yours. Nextcloud gives you the same core capabilities — file sync, calendar, contacts, collaborative editing — on infrastructure you control, for a fraction of the cost.

The setup takes an afternoon. The migration takes a few weeks of gradual transition. The payoff is permanent: your company's knowledge stays on your server, under your rules, with no per-user tax and no vendor deciding to change the deal next quarter.

If you want help setting up Nextcloud for your business — from server configuration to data migration to team training — [get in touch](/#contact). We specialize in helping small businesses move to self-hosted open source tools without the headaches.