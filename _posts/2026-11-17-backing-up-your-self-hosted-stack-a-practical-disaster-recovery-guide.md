---
layout: post
title: "Backing Up Your Self-Hosted Stack: A Practical Disaster Recovery Guide"
date: 2026-11-17
author: "ARDOT Consulting"
tags: [backups, disaster-recovery, self-hosting, docker, open-source, restic, borgbackup, strategy]
excerpt: "You self-host n8n, Ollama, Nextcloud, and a dozen other tools. But when was the last time you tested a restore? Here's a practical, no-panic guide to backing up your self-hosted stack — and making sure you can actually recover when something breaks."
---

# Backing Up Your Self-Hosted Stack: A Practical Disaster Recovery Guide

If you've been following this blog, you've probably self-hosted a few things by now. Maybe n8n for your automation workflows. Ollama for local AI. Nextcloud for file sync. Plausible for analytics. Perhaps NocoDB or Ghost or Vaultwarden. Each one running in its own Docker container, quietly doing its job on a server you control.

Good. That's the whole point of self-hosting — you own your data, you control your tools, and you're not paying a monthly subscription for the privilege of accessing your own information.

But here's the question that almost nobody asks until it's too late: **if that server disappeared right now, how much would you lose?**

Not "could you lose." How much *would* you lose. Because there's a difference between having a backup strategy on paper and having one that actually works when the server catches fire, the disk fails, or you accidentally run `docker compose down -v` and delete every volume along with it.

This guide is about building a backup and recovery process that works in reality, not in theory. No enterprise disaster recovery jargon. No "RTO and RPO" consultantspeak. Just what to back up, how to back it up, where to store it, and — the part everyone skips — how to verify you can actually restore it.

---

## Why Self-Hosting Changes the Backup Equation

When you used SaaS tools, backups were someone else's problem. If Mailchimp lost your subscriber list, that was Mailchimp's crisis. If Salesforce had an outage, Salesforce had the pager. You paid for that reliability as part of your subscription.

When you self-host, the pager is yours. If your n8n server's disk dies, your automation workflows are gone. If your Nextcloud volume corrupts, your team's files are gone. If your Vaultwarden database disappears, your passwords are gone.

This isn't a reason to avoid self-hosting. It's a reason to take backups seriously — the same way you take insurance seriously when you own a building instead of renting office space. The trade-off of ownership is that you're responsible for protecting what you own.

The good news: backing up self-hosted tools is straightforward, costs almost nothing, and once set up, runs on autopilot. The bad news: most people set up the backup part and never test the restore part. We'll cover both.

---

## What You Actually Need to Back Up

Not everything on your server matters equally. Let's break your self-hosted stack into categories so you can prioritize.

### Tier 1: Irreplaceable Data (Back Up Immediately)

This is data that exists only on your server and can't be recreated. If it's gone, it's gone.

| Tool | What to Back Up | Why It's Irreplaceable |
|------|----------------|----------------------|
| n8n | Database (SQLite or Postgres) + workflow JSON exports | Your automation logic — hours of workflow building |
| Nextcloud | Data directory + database | Your team's files, shared calendars, contacts |
| Vaultwarden | Database (SQLite) + attachments | Your passwords — literally everything |
| Ghost | Content directory + database | Your blog posts and subscriber list |
| Paperless-ngx | Media directory + database | Your scanned documents |
| BookStack | Database + file uploads | Your internal knowledge base |
| Odoo | Database + filestore | Your CRM data, contacts, sales pipeline |

### Tier 2: Configurable Data (Back Up Regularly)

This is data you *could* recreate, but it would be painful and time-consuming.

| Tool | What to Back Up | Pain Level If Lost |
|------|----------------|-------------------|
| Plausible | Database | Moderate — you lose analytics history |
| Metabase | Database | Moderate — you lose saved dashboards and queries |
| Uptime Kuma | Database | Low — you lose monitoring history but can reconfigure |
| Immich | Database + media | High — photos are irreplaceable (move to Tier 1) |

### Tier 3: Config Files (Back Up to Git)

Your `docker-compose.yml` files, `.env` files (without secrets), Nginx configs, and tool-specific configuration. These should live in a private Git repository. If your server dies, you can rebuild the entire stack from a `git clone` and your backups.

**A practical rule:** if you'd be upset to lose it, it's Tier 1. If you'd be annoyed to lose it, it's Tier 2. If it's a config file, it goes in Git.

---

## The 3-2-1 Rule (And Why It Still Matters)

The backup industry has a principle called 3-2-1 that's survived decades because it works:

- **3** copies of your data
- **2** different storage types
- **1** copy off-site

For a self-hosted stack, this translates to:

1. **The live data** on your server (copy 1, storage type 1)
2. **A local backup** on a separate disk or NAS (copy 2, storage type 2)
3. **An off-site backup** in cloud storage or a remote server (copy 3, off-site)

The off-site copy is the one that saves you when the building floods, the server gets stolen, or ransomware encrypts everything on the local network including your backup drive. It happens. Not often. But when it does, the off-site copy is the difference between a bad week and a closed business.

---

## The Tools: Open Source Backup Software

You don't need expensive enterprise backup software. Two open source tools cover 90% of self-hosted backup needs.

### Restic

[Restic](https://restic.net/) is a fast, secure, modern backup tool that handles encryption, deduplication, and incremental backups automatically. It backs up to local directories, S3-compatible storage, SFTP servers, and several cloud backends.

**Why we recommend it:** Restic encrypts everything client-side, so even if your backup storage is compromised, your data is unreadable without the password. It deduplicates across backups, so storing 30 days of daily backups doesn't take 30× the space. And it's a single binary — no server component, no database, no web UI to maintain.

**Best for:** Automated, encrypted, off-site backups to cloud storage or a remote server.

### BorgBackup

[BorgBackup](https://www.borgbackup.org/) (often just called "borg") is another excellent deduplicating backup tool with strong compression and encryption. It's been around longer than Restic and has a mature ecosystem, including [Borgmatic](https://torsion.org/borgmatic/) for simpler configuration.

**Why we recommend it:** Borg's deduplication is extremely efficient — backing up a 50GB Nextcloud directory daily for a month might only use 60GB total if files change slowly. It also supports client-side encryption and mounts backups as filesystems for easy browsing.

**Best for:** Local backups to a NAS or external drive, and situations where storage efficiency matters most.

### Which One Should You Pick?

Either one works well. If we had to choose: use **Restic for off-site backups** (it has better cloud storage support) and **Borg for local backups** (it has slightly better deduplication). But honestly, picking one and using it consistently is far more important than which one you pick.

---

## A Practical Backup Setup: Step by Step

Let's walk through a real backup setup for a typical self-hosted stack. Assume you're running n8n, Nextcloud, and Vaultwarden in Docker containers on a single Linux server.

### Step 1: Identify Your Docker Volumes

First, find out where your data lives:

```bash
docker volume ls
```

This lists all named volumes. For our stack, the important ones might be:
- `n8n_data` — n8n's database and workflow files
- `nextcloud_data` — uploaded files
- `nextcloud_db` — the MariaDB/Postgres database
- `vaultwarden_data` — the SQLite database and attachments

### Step 2: Create a Backup Script

Here's a practical backup script using Restic. It stops containers gracefully, backs up the volumes, and restarts everything.

```bash
#!/bin/bash
# backup.sh — Daily backup of self-hosted stack
set -euo pipefail

# Configuration
RESTIC_REPOSITORY="s3:s3.amazonaws.com/my-company-backups/self-hosted"
RESTIC_PASSWORD_FILE="/root/.restic-password"
BACKUP_PATHS=(
  "/var/lib/docker/volumes/n8n_data"
  "/var/lib/docker/volumes/nextcloud_data"
  "/var/lib/docker/volumes/nextcloud_db"
  "/var/lib/docker/volumes/vaultwarden_data"
  "/opt/docker/compose"  # your docker-compose.yml files
)
RETENTION_DAYS=30
RETENTION_WEEKS=8
RETENTION_MONTHS=12

# Export database from running containers (cleaner than copying live DB files)
echo "Exporting Nextcloud database..."
docker exec nextcloud-db mysqldump -u root -p"$DB_PASSWORD" --single-transaction nextcloud > /tmp/nextcloud.sql

echo "Exporting n8n database..."
docker exec n8n n8n export:workflow --all --output=/tmp/n8n-workflows.json

# Run the backup
echo "Running Restic backup..."
restic backup \
  "${BACKUP_PATHS[@]}" \
  /tmp/nextcloud.sql \
  /tmp/n8n-workflows.json \
  --tag daily

# Apply retention policy (keep daily for 30 days, weekly for 8 weeks, monthly for 12 months)
echo "Pruning old backups..."
restic forget \
  --keep-daily "$RETENTION_DAYS" \
  --keep-weekly "$RETENTION_WEEKS" \
  --keep-monthly "$RETENTION_MONTHS" \
  --prune

# Clean up temporary files
rm -f /tmp/nextcloud.sql /tmp/n8n-workflows.json

echo "Backup complete: $(date)"
```

A few things worth noting about this script:

1. **It exports databases properly.** Copying a live SQLite or Postgres file can give you a corrupted backup if the database is mid-write. Using `mysqldump` or the tool's built-in export command gives you a consistent snapshot. For SQLite databases (like Vaultwarden's), either stop the container first or use `sqlite3 backup` to create a safe copy.

2. **It exports n8n workflows as JSON.** Even if the database backup fails, you have a portable JSON export of every workflow that you can import into a fresh n8n instance.

3. **It uses a retention policy.** You don't keep every backup forever — that gets expensive. The policy keeps 30 daily, 8 weekly, and 12 monthly snapshots, which gives you granular recovery for recent issues and long-term recovery for disasters.

4. **It uses `set -euo pipefail`.** If any step fails, the script stops immediately and doesn't silently skip a broken backup.

### Step 3: Schedule It With Cron

Run the backup daily. A simple cron entry:

```bash
# Backup at 2 AM every day
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

**Important:** Monitor the log file. A backup script that silently fails for three months is worse than no backup at all, because it gives you false confidence. We'll cover monitoring below.

### Step 4: Store the Off-Site Backup Somewhere Safe

For the off-site copy, you have several open source-friendly options:

| Storage Option | Cost | Notes |
|---------------|------|-------|
| S3-compatible (Backblaze B2, Wasabi, MinIO) | $5-6/TB/month | Restic supports S3 natively; Backblaze B2 is affordable and popular |
| SFTP to a remote server | VPS cost only | If you have a second server (even a cheap $5 VPS), use it as an SFTP backup target |
| Self-hosted MinIO on a second server | Server cost only | Full control, but you're maintaining another service |
| Rsync to a NAS at a different location | One-time hardware cost | Good for offices with multiple locations |

**What about Backblaze B2 — isn't that a SaaS?** It is cloud storage, but it's not a proprietary lock-in tool — it speaks the standard S3 API, works with open source backup software, and your data is portable. We recommend it as storage infrastructure, not as a platform. The key distinction: your backup software (Restic/Borg) is open source and under your control. The storage is just a bucket.

---

## The Part Everyone Skips: Testing Restores

A backup you've never restored is a hope, not a backup. We've seen too many businesses discover — in the middle of a crisis — that their backups were corrupted, incomplete, or missing the one file they desperately needed.

Here's a restore testing process you can run once a month. It takes about 30 minutes and it's the most valuable 30 minutes you'll spend on infrastructure.

### Monthly Restore Test

1. **Spin up a test server.** A cheap VPS, a local VM, or even a Docker container on your laptop. Anything that isn't your production server.

2. **Restore one backup.** Pick a recent Restic snapshot and restore it to the test server:

```bash
# List available snapshots
restic snapshots

# Restore the latest snapshot
restic restore latest --target /tmp/restore-test
```

3. **Start the services.** Copy the restored data to the right Docker volumes, bring up the containers with `docker compose up`, and verify they start cleanly.

4. **Check the data.** Open n8n — are your workflows there? Open Nextcloud — can you browse files? Open Vaultwarden — can you log in and see your vault? This is where you'll discover if a database export was incomplete or a volume path was wrong.

5. **Document the result.** Keep a simple log: "November restore test — n8n workflows restored successfully, Nextcloud database had a minor schema warning but mounted correctly, Vaultwarden vault intact." If something failed, fix the backup script and retest.

6. **Tear down the test server.** Don't leave it running — it's a security surface with your real data on it.

### What If the Restore Fails?

If your restore test fails, that's actually good — you found the problem during a test, not during a crisis. Common failures and fixes:

- **Database won't start after restore.** You probably copied a live database file instead of doing a proper export. Fix: use `mysqldump`/`pg_dump`/`sqlite3 backup` in your backup script.
- **Missing files.** A Docker volume path changed or a new container was added but never added to the backup script. Fix: update `BACKUP_PATHS` and re-run.
- **Wrong encryption password.** The Restic repository password was changed or the password file was lost. Fix: store the password in a password manager (like Vaultwarden — yes, back up the password manager that backs up your password manager) and in a sealed envelope in a physical location.
- **Corrupted backup.** Disk errors on the backup target. Fix: enable Restic's integrity check (`restic check`) in your backup script and monitor the results.

---

## Adding Backup Monitoring

A backup that fails silently is dangerous. Here's how to know when things go wrong.

### Option 1: Healthchecks.io (Open Source)

[Healthchecks.io](https://healthchecks.io) is an open source cron monitoring tool you can self-host (it's available as a Docker container). The idea is simple: your backup script "checks in" after a successful run. If it doesn't check in within the expected window, you get an alert.

Add this to the end of your backup script:

```bash
# Ping Healthchecks after successful backup
curl -fsS --retry 3 https://healthchecks.example.com/ping/<your-uuid> > /dev/null
```

And add this to handle failures (using a trap at the top of the script):

```bash
# Alert on failure
trap 'curl -fsS --retry 3 https://healthchecks.example.com/ping/<your-uuid>/fail > /dev/null' ERR
```

Now you'll get an alert if the backup doesn't run *or* if it runs but fails.

### Option 2: Uptime Kuma (Which You May Already Have)

If you followed our [monitoring stack guide](/blog/2026/10/24/self-hosted-monitoring-stack-uptime-kuma-grafana-replace-statuspage-and-datadog/), you already have Uptime Kuma running. You can add a "Push" monitor type that works the same way — your backup script pings a Uptime Kuma URL after each run, and Uptime Kuma alerts you (via Mattermost, email, or whatever you've configured) if the ping doesn't arrive on schedule.

### Option 3: Restic Check

Add a weekly integrity check to catch corruption in your backup repository:

```bash
# Weekly: verify backup repository integrity
restic check --read-data-subset=5%
```

The `--read-data-subset` flag checks 5% of the repository each run, so over 20 weeks you've verified the entire repository without doing a full scan every time. Schedule this as a separate weekly cron job.

---

## Disaster Recovery: When It Actually Hits

Let's walk through a real scenario. Your server died. The disk is gone. You need to rebuild. Here's what the process looks like if you've been doing everything above.

### The 2-Hour Recovery

**Minute 0-15: Provision a new server.** Spin up a VPS with Docker installed. Clone your config repository (the one with all your `docker-compose.yml` files).

**Minute 15-30: Restore data.** Install Restic, configure the repository and password, and restore the latest snapshot:

```bash
restic snapshots  # find the latest good backup
restic restore <snapshot-id> --target /tmp/restore
```

**Minute 30-60: Rebuild volumes.** Copy the restored data into Docker volumes, import the database dumps, and place the n8n workflow JSON exports.

**Minute 60-90: Start services.** Run `docker compose up -d` for each service. Verify each container starts cleanly and the data is accessible.

**Minute 90-120: Verify and switch DNS.** Open each service in a browser, confirm data is intact, update DNS to point at the new server's IP, and you're back online.

Two hours. From a dead server to fully operational. That's what a working backup strategy buys you.

Compare that to the alternative: no tested backups, no config repository, no documentation. You'd spend days reconstructing workflows from memory, re-uploading files you can find, and rebuilding configs you barely remember. Some data would be gone permanently. Your team would be idle while you scramble. That's the cost of skipping backups.

---

## A Backup Checklist

If you read nothing else in this post, read this:

- [ ] **Every Docker volume with irreplaceable data is in your backup script**
- [ ] **Databases are exported properly (not live-file-copied)**
- [ ] **Backups run automatically via cron, daily**
- [ ] **Backups are encrypted (Restic and Borg do this by default)**
- [ ] **At least one backup copy is off-site**
- [ ] **Config files (docker-compose.yml, .env templates) are in a private Git repo**
- [ ] **Retention policy is set (don't keep everything forever, don't delete too aggressively)**
- [ ] **Backup success/failure is monitored (Healthchecks or Uptime Kuma)**
- [ ] **You've done at least one restore test in the last 30 days**
- [ ] **The Restic/Borg password is stored somewhere safe and accessible**

If you can check all ten of those boxes, you're in better shape than 90% of self-hosted setups. If you can't, pick the cheapest one to fix and start there.

---

## What This Costs

Let's talk money, because business owners ask.

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| Restic or Borg | $0 | Open source, free |
| Backup storage (Backblaze B2, 100GB) | ~$0.50 | Most small stacks are under 100GB |
| Backup storage (Wasabi, 100GB) | ~$0.67 | Slightly more expensive but no egress fees |
| Healthchecks (self-hosted) | $0 | Runs in Docker alongside your other tools |
| Test server (monthly restore test) | $5-10 | A temporary VPS you spin up and destroy |
| **Total** | **~$6/month** | For a backup system that can save your business |

Six dollars a month. That's less than a single SaaS subscription you replaced by self-hosting.

---

## The Honest Truth

Backups are unglamorous. Nobody gets excited about writing a shell script that runs at 2 AM and copies files to a bucket. There's no demo you can show your team, no workflow diagram that impresses clients, no automation ROI calculation that makes the spreadsheet look good.

But backups are the foundation that makes everything else on this blog possible. Every self-hosted tool we've covered — n8n, Ollama, Nextcloud, Vaultwarden, Ghost, every single one — is only as valuable as your ability to recover it when something goes wrong. The automation workflows you spent weeks building, the knowledge base your team relies on, the password vault that guards your entire digital identity — they all depend on a backup system that works and that you've actually tested.

So if you've been putting this off (and you probably have — most people do), make it the next thing you do. Not the next thing after the cool new tool. The next thing. Thirty minutes to set up Restic, ten minutes to schedule the cron job, thirty minutes for the first restore test. You'll never regret the time spent. You'll only regret not spending it sooner.

---

## Ready to Build Your Backup Strategy?

If you're running a self-hosted stack and haven't tested a restore — or if you're planning to self-host and want to build backups in from day one — we can help. ARDOT Consulting designs and implements practical, open source backup and disaster recovery systems for small businesses. No enterprise contracts, no proprietary lock-in, just a backup system that works and that you control.

[Get in touch](/#contact) and we'll help you build a backup strategy you can actually trust.