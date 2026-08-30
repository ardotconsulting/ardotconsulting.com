---
layout: post
title: "Setting Up Odoo Community Edition: CRM Automation for Small Business"
date: 2026-09-09
author: "ARDOT Consulting"
tags: [odoo, crm, automation, self-hosting, docker]
excerpt: "A practical guide to installing Odoo Community Edition with Docker, configuring CRM automation, and extending it with n8n for lead capture, follow-ups, and pipeline management."
---

# Setting Up Odoo Community Edition: CRM Automation for Small Business

If you run a small business, you've probably felt the friction of managing customer relationships across spreadsheets, email threads, and sticky notes. A CRM (Customer Relationship Management) system fixes that — but most commercial CRMs lock you into per-seat pricing, vendor roadmaps, and data silos you don't control.

Enter **Odoo Community Edition**: a fully open source ERP and CRM platform that you can self-host, customize, and integrate with your existing tools. No per-seat fees. No vendor lock-in. Your data stays on your infrastructure.

In this guide, we'll walk through installing Odoo Community Edition with Docker, configuring it for CRM automation, and connecting it to n8n for extended workflow automation. By the end, you'll have a working CRM that captures leads, sends automated follow-ups, and manages your sales pipeline — all on hardware you control.

## Why Odoo Community Edition?

Odoo is a modular business application suite. The Community Edition is the open source version (LGPL-licensed), and it includes a surprisingly capable CRM module out of the box. Here's how it compares to other options:

| Feature | Odoo Community | SuiteCRM | Mautic | EspoCRM |
|---------|---------------|----------|--------|---------|
| License | LGPLv3 | GPLv3 | GPLv3 | AGPLv3 |
| CRM Module | ✅ (built-in) | ✅ (core) | ❌ (marketing focus) | ✅ (core) |
| ERP Modules | ✅ (30+ apps) | ❌ | ❌ | ❌ |
| Self-hosted | ✅ | ✅ | ✅ | ✅ |
| Docker support | ✅ (official) | ⚠️ (community) | ✅ | ⚠️ (community) |
| n8n integration | ✅ (REST API) | ✅ (REST API) | ✅ (REST API) | ✅ (REST API) |
| Active community | ✅ (large) | ✅ | ✅ | ⚠️ (smaller) |

Odoo stands out because CRM is just the beginning. Once you have it running, you can add inventory management, invoicing, project management, helpdesk, and more — all from the same interface, all sharing the same database. For a small business that wants to grow beyond a standalone CRM, that's a significant advantage.

The Enterprise Edition adds features like advanced reporting, mobile app improvements, and studio customization — but Community Edition covers everything a small business needs to get started with CRM automation.

## Prerequisites

Before we start, you'll need:

- A server (VPS or on-premises) with at least **2 GB RAM** and **20 GB disk space**
- **Docker** and **Docker Compose** installed
- A domain name (optional but recommended for SSL)
- Basic comfort with the command line

If you're new to Docker, we covered the basics in our [self-hosting Plausible Analytics guide](/blog/2026/08/26/self-hosting-plausible-analytics-a-step-by-step-docker-guide/) — the principles are the same.

## Step 1: Install Odoo with Docker Compose

Odoo needs a PostgreSQL database and the Odoo application server itself. We'll use Docker Compose to manage both.

Create a project directory and a `docker-compose.yml` file:

```bash
mkdir -p ~/odoo && cd ~/odoo
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=odoo
      - POSTGRES_PASSWORD=change_this_strong_password
    volumes:
      - odoo-db:/var/lib/postgresql/data
    restart: unless-stopped

  odoo:
    image: odoo:17.0
    depends_on:
      - db
    ports:
      - "8069:8069"
    volumes:
      - odoo-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
    restart: unless-stopped

volumes:
  odoo-db:
  odoo-data:
```

Create the Odoo configuration file:

```bash
mkdir -p config addons
```

```ini
# config/odoo.conf
[options]
admin_passwd = change_this_to_a_strong_master_password
db_host = db
db_user = odoo
db_password = change_this_strong_password
addons_path = /usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons
limit_time_cpu = 600
limit_time_real = 120
```

> **Security note:** Replace both passwords with strong, unique values. The `admin_passwd` is the Odoo master password — it protects database creation and deletion. Never use the defaults in production.

Start the stack:

```bash
docker compose up -d
```

Wait about 30 seconds for the database to initialize, then open your browser to `http://YOUR_SERVER_IP:8069`. You'll see the Odoo database creation screen.

## Step 2: Initialize Your Database

On the first visit, Odoo asks you to create a database:

1. Enter your **Master Password** (the `admin_passwd` from your config file)
2. Set a **Database Name** (e.g., `ardot_crm`)
3. Set an **Email** and **Password** for the admin account
4. Select your **Language** and **Country**
5. Leave the demo data checkbox **checked** for now — it populates sample leads and opportunities so you can explore
6. Click **Create Database**

Initialization takes 1–3 minutes. Once complete, you'll be logged into the Odoo dashboard.

### Install the CRM Module

From the main dashboard:

1. Click **Apps** in the top menu
2. Search for **CRM**
3. Click **Install** on the CRM app

The CRM module installs in under a minute. You'll now have a dedicated CRM dashboard with a sales pipeline view.

## Step 3: Configure Your Sales Pipeline

Odoo's CRM organizes opportunities into **stages** — steps in your sales process. The default stages are:

1. **New** — A lead just entered the pipeline
2. **Qualified** — You've confirmed they're a real prospect
3. **Proposition** — You've sent a quote or proposal
4. **Won** — Deal closed

Customize these for your business:

1. Go to **CRM → Configuration → Stages**
2. Click **New** to add a stage, or edit existing ones
3. Reorder stages by dragging

For example, a consulting firm might use:

```
New → Discovery Call → Proposal Sent → Negotiation → Won → Onboarding
```

Each stage can have **automatic actions** — like sending an email or creating a task when an opportunity enters that stage. We'll configure those next.

## Step 4: Automate Lead Capture and Follow-Ups

This is where Odoo goes from a static contact database to an active CRM. Let's set up two automations: one for new leads and one for follow-up reminders.

### Automation 1: Welcome Email for New Leads

1. Go to **Settings → Technical → Automation Rules** (you may need to enable Developer Mode first — click your profile icon → **Preferences → Developer Mode**)
2. Click **New**
3. Configure:
   - **Rule Name:** Welcome new lead
   - **Model:** Lead/Opportunity
   - **Trigger:** On Creation
   - **Action:** Send Email
   - **Email Template:** Create a new template with your welcome message

Your welcome email template might look like:

```
Subject: Thanks for reaching out, {{ object.contact_name }}!

Hi {{ object.contact_name }},

Thanks for your interest in {{ object.company_name }}. I wanted to personally 
reach out and let you know we've received your inquiry.

I'll review the details and get back to you within one business day. In the 
meantime, feel free to check out our resources at https://www.ardotconsulting.com.

Best regards,
The ARDOT Consulting Team
```

### Automation 2: Follow-Up Reminder After 3 Days

1. Create another automation rule
2. Configure:
   - **Rule Name:** Follow-up reminder
   - **Model:** Lead/Opportunity
   - **Trigger:** Timed Condition
   - **Condition:** Stage = New AND Last activity date = 3 days ago
   - **Action:** Create Activity (Call the lead)

This creates a phone call activity assigned to the sales rep when a lead sits in the "New" stage for 3 days without any activity. No lead falls through the cracks.

## Step 5: Connect Odoo to n8n for Extended Automation

Odoo's built-in automation is powerful, but sometimes you need to connect external systems. That's where **n8n** comes in. n8n is an open source workflow automation tool that can call Odoo's REST API and connect it to virtually any other service.

### Enable Odoo's REST API

Odoo's API is available by default. You'll need:

- Your **database name**
- A **user ID** and **password** (preferably a dedicated API user)
- The **server URL** (e.g., `http://YOUR_SERVER_IP:8069`)

### Create an API User

1. Go to **Settings → Manage Users**
2. Click **New**
3. Create a user named `n8n-integration` with **CRM** access rights
4. Note the username (email) and password

### Build a Workflow: Web Form Lead → Odoo

Here's a practical n8n workflow that captures leads from a web form and creates them in Odoo:

```json
{
  "nodes": [
    {
      "type": "n8n-nodes-base.webhook",
      "name": "Web Form Submit",
      "parameters": {
        "path": "lead-capture",
        "method": "POST"
      }
    },
    {
      "type": "n8n-nodes-base.httpRequest",
      "name": "Create Lead in Odoo",
      "parameters": {
        "method": "POST",
        "url": "http://YOUR_SERVER_IP:8069/xmlrpc/2/object",
        "bodyContentType": "json",
        "bodyParameters": {
          "model": "crm.lead",
          "method": "create",
          "args": [
            {
              "name": "={{$json.body.company}}",
              "contact_name": "={{$json.body.name}}",
              "email_from": "={{$json.body.email}}",
              "phone": "={{$json.body.phone}}",
              "description": "={{$json.body.message}}"
            }
          ]
        }
      }
    },
    {
      "type": "n8n-nodes-base.emailSend",
      "name": "Notify Sales Team",
      "parameters": {
        "to": "sales@yourcompany.com",
        "subject": "New lead from website: {{$json.body.company}}",
        "text": "A new lead was just created in Odoo. Check the CRM pipeline."
      }
    }
  ],
  "connections": {
    "Web Form Submit": {
      "main": [
        [
          { "node": "Create Lead in Odoo", "type": "main", "index": 0 }
        ]
      ]
    },
    "Create Lead in Odoo": {
      "main": [
        [
          { "node": "Notify Sales Team", "type": "main", "index": 0 }
        ]
      ]
    }
  }
}
```

This workflow:
1. **Receives** a form submission from your website (any form tool that supports webhooks)
2. **Creates** a lead in Odoo's CRM via the XML-RPC API
3. **Notifies** your sales team by email

The lead automatically appears in Odoo's pipeline in the "New" stage, triggering the welcome email automation we set up earlier. The entire flow — form submission to lead creation to welcome email to sales notification — happens in seconds, with zero manual data entry.

### More n8n + Odoo Integration Ideas

Once the connection is established, you can build workflows for:

- **Invoice generation:** When an opportunity moves to "Won," n8n creates an invoice in Odoo's accounting module and emails it to the client
- **Calendar sync:** New meetings scheduled in Cal.com automatically create activities in the matching Odoo opportunity
- **Analytics sync:** Weekly pipeline summary data from Odoo pushes into a Metabase dashboard for team-wide visibility
- **Document generation:** When a proposal is sent, n8n generates a PDF from a template, attaches it to the Odoo opportunity, and logs the activity

## Step 6: Back Up Your Odoo Instance

Your CRM data is critical. Set up automated backups with a simple cron job:

```bash
#!/bin/bash
# /opt/odoo-backup.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/backups/odoo

mkdir -p $BACKUP_DIR

# Back up the database
docker exec odoo-db-1 pg_dump -U odoo postgres > $BACKUP_DIR/odoo_db_$DATE.sql

# Back up the filestore
docker exec odoo-odoo-1 tar czf - /var/lib/odoo > $BACKUP_DIR/odoo_files_$DATE.tar.gz

# Keep only the last 30 days
find $BACKUP_DIR -type f -mtime +30 -delete
```

Add it to your crontab:

```bash
crontab -e
# Add this line — runs at 2 AM daily:
0 2 * * * /opt/odoo-backup.sh
```

For off-site redundancy, sync backups to a separate location using **rclone** (which supports S3, Backblaze B2, and other storage providers — all open source).

## Cost Breakdown

Here's what it actually costs to run Odoo Community Edition:

| Item | Cost |
|------|------|
| Odoo Community license | $0 (open source) |
| VPS (2 GB RAM, 2 vCPU) | $5–12/month |
| Domain name | $10–15/year |
| SSL certificate | $0 (Let's Encrypt) |
| Backups (B2 storage) | $0.50–2/month |
| **Total** | **~$7–15/month** |

Compare that to commercial CRM pricing at $20–75 per user per month. For a 10-person team, that's $2,400–9,000/year saved — and you own the data.

## What Odoo Community Edition Doesn't Do

Being honest about limitations matters. The Community Edition lacks:

- **Advanced reporting dashboards** — You get basic reporting, but the polished dashboard builder is Enterprise-only. For advanced analytics, export data to **Metabase** (open source) instead.
- **Mobile app polish** — The web interface works on mobile browsers, but the native mobile app requires Enterprise. Most teams find the mobile web version sufficient.
- **Studio customization** — The visual app builder is Enterprise-only. Community Edition customization requires writing Python modules or using the developer mode for field/form changes.
- **Predictive lead scoring** — Enterprise includes AI-based lead scoring. In Community, you'll score leads manually or build a scoring workflow in n8n using an Ollama-powered LLM.

None of these are dealbreakers for a small business getting started with CRM automation. You can always upgrade to Enterprise later if you need these features — or fill the gaps with other open source tools.

## Getting Started Checklist

To make this actionable, here's your step-by-step checklist:

- [ ] Provision a VPS with 2+ GB RAM
- [ ] Install Docker and Docker Compose
- [ ] Create the `docker-compose.yml` and `odoo.conf` files
- [ ] Run `docker compose up -d` and create your database
- [ ] Install the CRM module
- [ ] Customize your pipeline stages
- [ ] Set up the "Welcome new lead" automation
- [ ] Set up the "Follow-up reminder" automation
- [ ] Create an API user for n8n integration
- [ ] Build the lead capture workflow in n8n
- [ ] Set up automated daily backups
- [ ] Configure SSL with Let's Encrypt (Caddy or Traefik as a reverse proxy)
- [ ] Import existing contacts and leads

You can realistically complete this in one weekend. The result: a CRM that works for you instead of the other way around.

## Wrapping Up

Odoo Community Edition gives small businesses a professional CRM without per-seat licensing, vendor lock-in, or data sovereignty concerns. Paired with n8n for workflow automation and Ollama for AI-powered lead scoring, it becomes a foundation you can build an entire business operations stack on — all open source, all self-hosted, all yours.

The best CRM isn't the one with the most features. It's the one your team actually uses, that fits your process instead of forcing you to fit theirs, and that doesn't penalize you for growing. That's what self-hosting Odoo delivers.

**Ready to automate your CRM and customer pipeline?** [Get in touch with ARDOT Consulting](/) — we help small businesses set up self-hosted CRM, automation workflows, and AI integrations that fit your process, not the other way around.