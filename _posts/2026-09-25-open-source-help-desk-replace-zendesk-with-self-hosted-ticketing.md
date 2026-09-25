---
layout: post
title: "Open Source Help Desk: Replace Zendesk With Self-Hosted Ticketing"
date: 2026-09-25
author: "ARDOT Consulting"
tags: [help-desk, ticketing, freescout, zammad, open-source, self-hosting, customer-support]
excerpt: "Customer support software doesn't have to cost $89 per agent per month. Here's how to replace Zendesk with open source ticketing tools you host yourself."
---

If you're paying $89 per agent per month for Zendesk — or wrestling with Freshdesk's tier limits — you're not alone. Customer support software has become one of the most quietly expensive line items for small and mid-size businesses. A five-person support team on Zendesk's Team plan costs $2,670 per year. Scale to the Growth plan and you're at $5,340. And that's before you add the marketplace apps, telephony integrations, and surcharges for "premium" features that used to be standard.

The good news: open source help desk software has matured dramatically. Tools like FreeScout, Zammad, and osTicket give you the core ticketing workflow — email-to-ticket, assignment, SLA tracking, reporting, knowledge base — without the per-seat tax. You host them on a $6/month VPS and own the data outright.

This post walks through the landscape, compares the top three options, and shows you how to get FreeScout running with Docker in under an hour.

## Why Self-Host Your Help Desk?

Before we get to the tools, let's talk about why this matters beyond cost.

**Data sovereignty.** Every support ticket your team writes contains customer information — names, email addresses, order numbers, sometimes partial payment data or internal notes about accounts. When you use a cloud-hosted SaaS, all of that lives on someone else's servers, subject to their data retention policies and breach risks. Self-hosting puts that data on infrastructure you control.

**No per-seat lock-in.** Want to add a seasonal temp agent for the holidays? With Zendesk, that's another monthly license. With self-hosted tools, you just create a user account. No procurement approval, no contract amendment, no surprise billing.

**Customization without app marketplace fees.** Need to integrate with your internal systems? Zendesk charges for many marketplace apps and limits what you can customize on lower tiers. Open source tools let you modify the code directly or build integrations through their APIs at no additional cost.

**Predictable costs.** A VPS that runs FreeScout or Zammad for a small team costs $6–$20/month total — flat, regardless of how many agents you add. Compare that to the linear scaling of SaaS pricing.

## The Top Three Open Source Help Desk Tools

There are dozens of open source ticketing systems, but three stand out for small-to-mid-size businesses based on features, community activity, and ease of deployment.

### FreeScout

FreeScout is the spiritual successor to Help Scout's self-hosted model. It's a PHP application that handles email-to-ticket conversion, shared inboxes, assignments, tags, saved replies, and a customer-facing portal. It's lightweight, runs on minimal hardware, and has a clean, minimalist interface.

**Best for:** Small teams (1–10 agents) who want a shared mailbox approach rather than a complex ticketing hierarchy. If you've used Help Scout and liked the conversation-centric workflow, FreeScout will feel familiar.

**License:** AGPL-3.0 (fully open source, no "community edition" feature gating)

**Key features:**
- Email-to-ticket conversion with full email threading
- Shared inboxes with private notes and internal replies
- Tags, folders, and custom fields
- SLA reminders and due dates
- Customer profiles with conversation history
- Mobile-responsive web interface
- API for integrations
- Multi-language support

### Zammad

Zammad is a more feature-rich help desk platform built in Ruby on Rails. It offers a more traditional ticketing workflow with views, triggers, schedulers, and a built-in knowledge base. It's heavier than FreeScout but scales well to larger teams.

**Best for:** Growing teams (5–50 agents) who need automation rules, reporting, a knowledge base, and more structured ticket routing.

**License:** AGPL-3.0 (open source community edition; commercial enterprise version adds SSO and clustering)

**Key features:**
- Email, chat, phone, and social media channel integration
- Trigger-based automation (e.g., auto-assign based on subject keywords)
- Built-in knowledge base for self-service
- Full-text search across all tickets
- Detailed reporting and statistics
- Core API and extensive integration options
- Multi-tenant support

### osTicket

osTicket is one of the oldest open source ticketing systems, written in PHP. It's battle-tested and widely deployed, though its interface feels dated compared to FreeScout and Zammad. It's a solid choice if you need a straightforward ticket pipeline without modern UI expectations.

**Best for:** Budget-conscious teams who need basic ticket routing and don't mind a utilitarian interface.

**License:** GPL-2.0

**Key features:**
- Email-to-ticket and web form ticket creation
- Department-based ticket routing
- SLA plans with escalation rules
- Custom forms and fields
- Agent collision avoidance
- Dashboard reports

## Comparison at a Glance

| Feature | FreeScout | Zammad | osTicket |
|---------|-----------|--------|----------|
| **License** | AGPL-3.0 | AGPL-3.0 | GPL-2.0 |
| **Language** | PHP | Ruby/JS | PHP |
| **Min. RAM** | 512MB | 2GB | 512MB |
| **Shared inbox model** | ✅ | ❌ (ticket model) | ❌ (ticket model) |
| **Knowledge base** | Add-on module | ✅ Built-in | ❌ |
| **Automation triggers** | Basic | Advanced | Moderate |
| **Chat/Social channels** | ❌ | ✅ | ❌ |
| **Reporting** | Basic | Detailed | Basic |
| **API** | ✅ | ✅ | ✅ |
| **Mobile UI** | Responsive web | Responsive web | Limited |
| **Active community** | ✅ | ✅ | Moderate |
| **Best for** | Small teams, shared inbox | Mid-size, multi-channel | Basic ticket routing |

## Deploying FreeScout with Docker

For this walkthrough, we'll set up FreeScout since it's the lightest option and works well for small teams. You'll need a VPS (any Linux server with Docker installed) and a domain name pointed at it.

### Prerequisites

- A VPS running Ubuntu 22.04+ (a $6/month instance from Hetzner, OVH, or DigitalOcean is plenty)
- Docker and Docker Compose installed
- A domain or subdomain (e.g., `support.yourcompany.com`) with an A record pointing to your server

### Step 1: Create the Docker Compose File

Create a directory for your FreeScout deployment and add a `docker-compose.yml`:

```yaml
version: "3.8"

services:
  freescout-db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: freescout
      MYSQL_USER: freescout
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql

  freescout-app:
    image: freescout/freescout:latest
    restart: unless-stopped
    environment:
      DB_HOST: freescout-db
      DB_NAME: freescout
      DB_USER: freescout
      DB_PASSWORD: ${DB_PASSWORD}
      SITE_URL: https://support.yourcompany.com
      ADMIN_EMAIL: admin@yourcompany.com
      ADMIN_PASS: ${ADMIN_PASSWORD}
    volumes:
      - app-data:/var/www/html
    ports:
      - "8080:80"
    depends_on:
      - freescout-db

volumes:
  db-data:
  app-data:
```

### Step 2: Create the Environment File

In the same directory, create a `.env` file with your passwords (don't commit this to version control):

```bash
DB_ROOT_PASSWORD=change_this_to_a_long_random_string
DB_PASSWORD=another_long_random_string
ADMIN_PASSWORD=your_secure_admin_password
```

### Step 3: Start the Stack

```bash
docker compose up -d
```

FreeScout will be available on port 8080. For production, put a reverse proxy like Caddy or Nginx in front for HTTPS:

### Step 4: Add HTTPS with Caddy

Create a `Caddyfile` alongside your docker-compose:

```caddyfile
support.yourcompany.com {
    reverse_proxy freescout-app:80
}
```

Add Caddy to your `docker-compose.yml`:

```yaml
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
    depends_on:
      - freescout-app
```

Add `caddy-data` to your volumes section, then restart:

```bash
docker compose up -d
```

Caddy automatically provisions and renews Let's Encrypt TLS certificates. Your FreeScout instance is now accessible at `https://support.yourcompany.com`.

### Step 5: Configure Email Forwarding

The last step is connecting your support email. FreeScout works by polling an email inbox (IMAP) or receiving forwarded emails. The simplest setup:

1. Create a dedicated mailbox (e.g., `support@yourcompany.com`) with your email provider
2. In FreeScout's admin panel, go to **Mailboxes → Settings**
3. Enter your IMAP credentials and SMTP settings
4. Set the polling frequency (every 5 minutes is typical)

Every email sent to `support@yourcompany.com` becomes a ticket in FreeScout. Replies from agents go out through SMTP, and the customer sees a normal email conversation — no portal required.

## Integrating with Your Existing Stack

FreeScout's API makes it straightforward to connect with other open source tools you may already be running:

**n8n automation.** Use n8n to trigger workflows when tickets are created. For example, when a ticket is tagged "billing," n8n can look up the customer in your Odoo CRM, attach the account details as a private note on the ticket, and post a notification in Mattermost.

**Ollama for draft replies.** Route incoming tickets through a local LLM (via Ollama) that suggests a draft response based on your knowledge base articles and past resolved tickets. The agent reviews, edits, and sends — cutting response time significantly for common questions.

**Mattermost notifications.** Post new critical-priority tickets to a dedicated Mattermost channel so the team sees them immediately, even if they're not logged into FreeScout.

Here's a simple n8n workflow that posts new FreeScout tickets to Mattermost:

```
[FreeScout Webhook] → [Filter: priority = urgent] → [Mattermost: Post Message]
```

The webhook fires when a ticket is created in FreeScout, the filter node checks if it's urgent, and the Mattermost node posts a formatted message to your `#support` channel.

## Cost Breakdown: Self-Hosted vs SaaS

| Expense | Zendesk (5 agents, Team plan) | FreeScout (5 agents, self-hosted) |
|---------|-------------------------------|-----------------------------------|
| Software license | $89/agent/month = $445/month | $0 (open source) |
| Hosting (VPS) | — | $6–$20/month |
| Domain (subdomain) | — | ~$1/month (amortized) |
| TLS certificates | — | $0 (Let's Encrypt) |
| Email hosting | — | ~$5/month (shared mailbox) |
| **Monthly total** | **$445** | **$12–$26** |
| **Annual total** | **$5,340** | **$144–$312** |

Even accounting for the one-time setup effort (2–4 hours), the savings are substantial. A five-agent team saves over $5,000 per year. A 20-agent team saves over $21,000 per year — and the self-hosted cost stays roughly the same, since the VPS doesn't care how many agents you add.

## What About Migration?

If you're moving from Zendesk or Freshdesk, both platforms allow you to export your ticket history as CSV. FreeScout has a community-maintained import script for Zendesk exports, and Zammad includes a built-in migration wizard that can pull data from OTRS, Zendesk, and Kayako via API.

For historical tickets, consider importing only the last 6–12 months. Older tickets can be archived as a searchable PDF bundle rather than imported into the new system — they're rarely referenced and importing them bloats your database.

## When Self-Hosted Might Not Be the Right Call

Being honest about limitations: self-hosting means you own the maintenance. If your server goes down at 2 AM, it's your responsibility (or your monitoring alert's responsibility). If you don't have anyone on your team comfortable with basic Docker and server administration, the operational overhead may outweigh the savings.

You should also consider managed hosting. FreeScout offers an official hosted plan starting at €12.99/month — which is still a fraction of Zendesk's cost and eliminates the server management burden. Zammad offers hosted plans as well. These give you the open source tool's feature set without the infrastructure responsibility.

## Getting Started Checklist

1. **Audit your current support volume** — how many tickets/month, how many agents, what channels (email, chat, phone)?
2. **Pick the right tool** — FreeScout for shared inbox simplicity, Zammad for multi-channel + automation, osTicket for bare-bones ticket routing
3. **Provision a VPS** — 1 vCPU, 1GB RAM minimum for FreeScout; 2 vCPU, 4GB RAM for Zammad
4. **Deploy with Docker** — use the compose files above as a starting point
5. **Connect your support email** — set up IMAP polling or email forwarding
6. **Add monitoring** — use Uptime Kuma to monitor your help desk URL and get alerts if it goes down
7. **Import recent tickets** — migrate the last 6–12 months from your previous platform
8. **Train your team** — FreeScout and Zammad are intuitive, but a 30-minute walkthrough saves confusion later

## Wrapping Up

Replacing Zendesk with an open source help desk is one of the highest-impact, lowest-risk migrations a small business can make. The tools are mature, the savings are real, and the data sovereignty benefit compounds over time. You're not just cutting a SaaS bill — you're investing in infrastructure you own and control.

If you want help evaluating which tool fits your team, or need someone to handle the migration end-to-end, [reach out through our contact form](/#contact). We specialize in setting up open source automation stacks for small businesses — no hype, no vendor lock-in, just tools that work.