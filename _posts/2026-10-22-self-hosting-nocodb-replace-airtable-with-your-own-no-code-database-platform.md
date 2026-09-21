---
layout: post
title: "Self-Hosting NocoDB: Replace Airtable with Your Own No-Code Database Platform"
date: 2026-10-22
author: "ARDOT Consulting"
tags: [nocodb, airtable-alternative, no-code, database, self-hosting, open-source, docker]
excerpt: "Airtable costs $20+ per seat per month and locks your data in a proprietary cloud. Here's how to replace it with NocoDB — an open-source, self-hosted no-code database that gives you the same spreadsheet interface, Kanban boards, automations, and API access, all on your own server."
---

# Self-Hosting NocoDB: Replace Airtable with Your Own No-Code Database Platform

If your team uses Airtable, you already know the value of a no-code database. Spreadsheets are easy to understand, but they buckle under real work — no relationships, no proper field types, no API access. Airtable fixed that by wrapping a real database in a friendly spreadsheet UI. But it came with a catch: your data lives on their servers, and the price keeps climbing.

Airtable's Team plan runs $20 per seat per month. For a 15-person team, that's $3,600 a year. The Business plan is $45 per seat — $8,100 a year for the same team. And every plan has limits: record caps, attachment storage caps, automation run caps. Hit a limit, and you're funneled into a more expensive tier.

NocoDB is the open-source alternative. It does the same thing — wraps a real database in a spreadsheet interface — but it's free, self-hosted, and your data never leaves your server. 65,000 stars on GitHub, active development, and a feature list that covers most of what Airtable offers.

In this guide, we'll walk through what NocoDB does, how it compares to Airtable, and how to deploy it on your own server with Docker.

## What NocoDB Actually Does

NocoDB is a no-code database platform. You create databases (called "bases"), add tables, define fields with proper types, and work with your data in a spreadsheet-like grid. But unlike a plain spreadsheet, you get:

- **Relationships between tables** — link records, create lookups and rollups
- **Multiple view types** — Grid, Form, Gallery, Kanban, Calendar, Timeline, Gantt, List, and Map
- **Rich field types** — text, number, date/time, single/multi select, attachments, formulas, links, lookups, rollups
- **Access control** — fine-grained permissions at the workspace, base, table, field, and record level
- **Automations** — a visual workflow builder with triggers, conditions, actions, and loops
- **Dashboards** — bar, line, pie, and donut charts, plus numbers, text, and embedded iframes
- **API access** — REST and GraphQL APIs for every table, plus an MCP server for AI agent integration
- **Webhooks** — event-driven notifications when records are created, updated, or deleted

The key insight is that NocoDB sits on top of a real database — PostgreSQL, MySQL, or SQLite. You're not using a toy. You're using a production-grade database with a friendly interface on top. If you outgrow the interface, you can connect directly to the underlying database with any tool you want.

## NocoDB vs Airtable: Feature Comparison

| Feature | Airtable | NocoDB (Self-Hosted) |
|---------|----------|---------------------|
| Monthly cost (per seat) | $20–$45 | $0 |
| Data residency | Airtable's cloud | Your server |
| Record limit (Team plan) | 50,000 per base | Unlimited (your hardware) |
| Attachment storage | 5 GB per base | Unlimited (your disk) |
| Automation runs | 25,000/month (Team) | Unlimited |
| View types | Grid, Kanban, Calendar, Gallery, Timeline, Gantt, Form | Grid, Kanban, Calendar, Gallery, Timeline, Gantt, Form, List, Map |
| Field types | 20+ types | 20+ types (equivalent) |
| API access | REST API | REST + GraphQL + MCP Server |
| Access control | Workspace, base, view | Workspace, base, table, field, record |
| Automations | Visual builder + scripts | Visual builder + JavaScript scripts |
| Dashboards | Built-in (Business plan) | Built-in (free) |
| AI features | AI fields ($$) | NocoAI (base/table creation assist) |
| Underlying database | Proprietary | PostgreSQL, MySQL, or SQLite |
| Data export | CSV, limited | Direct database access — full export anytime |
| License | Proprietary SaaS | AGPL-3.0 (open source) |

The trade-off is simple: Airtable is easier to start with (managed hosting, polished UI, mobile app) but expensive at scale and opaque about your data. NocoDB requires you to run a server but costs nothing per seat, has no artificial limits, and gives you direct access to the underlying database.

## When NocoDB Makes Sense

NocoDB is the right choice when:

1. **You're hitting Airtable's record or automation limits** and the next tier is a 2x price jump
2. **Your data has compliance requirements** — healthcare (HIPAA), legal, finance — and needs to stay on infrastructure you control
3. **You want API access for automation** without paying for higher tiers
4. **Your team is larger than 5–10 people** and per-seat pricing is adding up
5. **You already self-host other tools** (n8n, Odoo, Mattermost) and adding one more is trivial

It's probably not the right choice if you need Airtable's polished mobile app, their pre-built integrations marketplace, or if your team has no one who can manage a Docker container.

## Deploying NocoDB with Docker

The fastest way to get NocoDB running is a single Docker command with SQLite as the backend. This is fine for testing or small teams (under 10 people, a few thousand records per table).

### Quick Start: SQLite Backend

```bash
docker run -d \
  --name nocodb \
  -v "$(pwd)"/nocodb:/usr/app/data/ \
  -p 8080:8080 \
  nocodb/nocodb:latest
```

That's it. Open `http://your-server-ip:8080` in a browser, create an admin account, and start building. The data is persisted in the `./nocodb` directory on your host.

### Production Setup: PostgreSQL Backend with Docker Compose

For anything beyond testing, use PostgreSQL as the metadata database. Here's a production-ready `docker-compose.yml`:

```yaml
version: "3.8"

services:
  nocodb:
    image: nocodb/nocodb:latest
    container_name: nocodb
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      NC_DB: "pg://nocodb-db:5432?u=nocodb&p=your_secure_password&d=nocodb"
      NC_AUTH_JWT_SECRET: "your_random_jwt_secret_string_here"
      NC_PUBLIC_API_TOKEN: "your_api_token_here"
    volumes:
      - nocodb-data:/usr/app/data/
    depends_on:
      nocodb-db:
        condition: service_healthy

  nocodb-db:
    image: postgres:16-alpine
    container_name: nocodb-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: nocodb
      POSTGRES_USER: nocodb
      POSTGRES_PASSWORD: your_secure_password
    volumes:
      - nocodb-pg:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nocodb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  nocodb-data:
  nocodb-pg:
```

Deploy it:

```bash
# Create the file
nano docker-compose.yml

# Start the stack
docker compose up -d

# Check that both containers are running
docker compose ps
```

### Securing with a Reverse Proxy

Don't expose port 8080 directly to the internet. Put NocoDB behind a reverse proxy with TLS. Here's a minimal Caddy configuration that gives you automatic HTTPS:

```caddyfile
db.yourcompany.com {
    reverse_proxy localhost:8080
}
```

Caddy handles Let's Encrypt certificates automatically. Install Caddy, drop that config in, and your NocoDB instance is accessible over HTTPS at `https://db.yourcompany.com`.

If you already use Nginx, the equivalent is:

```nginx
server {
    listen 80;
    server_name db.yourcompany.com;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Add Certbot for the TLS certificate and you're done.

## What You Can Build with NocoDB

Once NocoDB is running, here are practical use cases that replace common Airtable workflows:

### 1. CRM and Lead Tracking

Create a base with tables for **Companies**, **Contacts**, and **Deals**. Link contacts to companies, deals to contacts. Use a Kanban view on the Deals table to track pipeline stages. Set up an automation that sends a webhook to n8n when a deal moves to "Closed Won" — n8n can then create an invoice in Odoo and send a welcome email.

### 2. Project Management

Tables for **Projects**, **Tasks**, and **Team Members**. Use the Gantt view for project timelines, Calendar view for deadlines, and Grid view for task lists. Create a Form view for task submission — team members submit tasks without seeing the full board.

### 3. Inventory Management

Tables for **Products**, **Suppliers**, and **Stock Movements**. Use rollup fields to calculate current stock levels from the movements table. Set up an automation that fires a webhook when stock drops below a threshold — connect it to n8n to send a reorder notification.

### 4. Content Calendar

Tables for **Content Ideas**, **Drafts**, and **Published Posts**. Use the Calendar view to visualize your publishing schedule. Use a Gallery view with attachment fields to preview featured images. Set up a webhook that triggers when a post status changes to "Published" — n8n can cross-post to social media.

### 5. Customer Support Ticket System

Tables for **Tickets**, **Customers**, and **Responses**. Use a Kanban view for ticket status (Open, In Progress, Resolved, Closed). Use the Map view if you need geographic visualization of customer locations. Automations can escalate tickets that have been open for more than 48 hours.

## Connecting N8n to NocoDB

One of NocoDB's strongest features is its API. Every table automatically gets REST and GraphQL endpoints. This makes it trivially easy to connect n8n workflows to your NocoDB data.

Here's how to connect them:

1. In NocoDB, go to **Settings → API Tokens** and create a token with read/write access to your base
2. In n8n, use the **HTTP Request** node to interact with NocoDB's REST API
3. Base URL: `http://your-nocodb-server:8080/api/v2/tables/{tableId}/records`
4. Headers: `xc-token: your_api_token`

Example n8n workflow — when a new row is added to a "Leads" table in NocoDB, n8n picks it up via a webhook, enriches it with company data using Ollama, and writes the enriched data back to a second table:

```
NocoDB Webhook (new lead) 
  → HTTP Request: GET lead details from NocoDB API
  → Ollama Node: summarize company website
  → HTTP Request: POST enriched data back to NocoDB
  → Mattermost Node: notify sales team channel
```

This is the kind of integration that would require Airtable's Automations Pro add-on (at additional cost) or external tools like Zapier (also per-task pricing). With NocoDB + n8n, it's all free and self-hosted.

## NocoDB's Built-in Automation

If you don't want to use n8n for simple automations, NocoDB has its own visual workflow builder. You can:

- **Trigger** on record creation, update, or deletion — or on a schedule
- **Add conditions** — if field X equals Y, continue
- **Perform actions** — update records, send webhooks, run JavaScript scripts
- **Loop** over arrays of records

For example, you can create a workflow that triggers when a deal's status changes to "Won," calculates a commission value using a formula, updates a commission field on the record, and fires a webhook to your accounting system.

The JavaScript scripting capability is particularly powerful — you get full API access to bases, tables, fields, and records from within the script. This means you can implement business logic that would normally require an external service:

```javascript
// Example NocoDB automation script
// Triggered when a new support ticket is created

const tickets = $api.tables.list('Tickets');
const newTicket = $trigger.record;

// Check if customer has open tickets
const openTickets = await $api.records.list('Tickets', {
  filter: { customer_id: newTicket.customer_id, status: 'Open' }
});

if (openTickets.length > 3) {
  // Escalate: update priority to High
  await $api.records.update('Tickets', newTicket.id, {
    priority: 'High',
    escalated: true
  });
  
  // Fire webhook to alert support team
  await $api.utils.fetch('https://n8n.yourcompany.com/webhook/escalation', {
    method: 'POST',
    body: JSON.stringify({ ticket: newTicket, openCount: openTickets.length })
  });
}
```

## Backing Up Your NocoDB Instance

Since NocoDB runs on your server, backups are your responsibility. But this is actually an advantage — you have full control.

For the Docker Compose setup with PostgreSQL:

```bash
#!/bin/bash
# backup-nocodb.sh — run daily via cron

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backups/nocodb"

# Back up the PostgreSQL metadata database
docker exec nocodb-db pg_dump -U nocodb nocodb | gzip > "$BACKUP_DIR/nocodb-db-$DATE.sql.gz"

# If using external data sources, back those up too
# docker exec your-data-db pg_dump -U user your_data_db | gzip > "$BACKUP_DIR/data-db-$DATE.sql.gz"

# Keep 30 days of backups
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +30 -delete

echo "NocoDB backup complete: $DATE"
```

Add this to your crontab:

```bash
0 2 * * * /opt/scripts/backup-nocodb.sh >> /var/log/nocodb-backup.log 2>&1
```

Compare this to Airtable, where you can export to CSV but can't do a full database backup. If Airtable has an outage or your account is suspended, you're stuck. With NocoDB, you can restore your entire instance in minutes.

## Migrating from Airtable

If you're already on Airtable and want to switch, here's the migration path:

1. **Export each table from Airtable as CSV** — Airtable supports this natively
2. **Create corresponding tables in NocoDB** — define the same field types (text, number, date, select, attachment)
3. **Import the CSV files** — NocoDB's CSV importer maps columns to fields automatically
4. **Recreate relationships** — after all tables are imported, add link fields between them
5. **Rebuild views** — recreate your Kanban boards, calendars, and forms
6. **Recreate automations** — port your Airtable automations to NocoDB's workflow builder or n8n

The attachment migration is the one tricky part. Airtable stores attachments on their CDN, so exported CSVs contain URLs, not files. You'll need to download the attachments separately and re-upload them to NocoDB. A simple n8n workflow can automate this — read the CSV, download each attachment URL, upload to NocoDB via API.

## The Total Cost Comparison

Let's put real numbers on this for a 15-person team over three years:

| Item | Airtable (Team plan) | NocoDB (Self-Hosted) |
|------|---------------------|---------------------|
| Software license | $20/seat × 15 × 36 months = $10,800 | $0 |
| Server (VPS with 4GB RAM) | — | ~$20/month × 36 = $720 |
| Domain + TLS | — | ~$15/year × 3 = $45 |
| Backup storage | — | ~$5/month × 36 = $180 |
| Setup time (one-time) | ~2 hours | ~4 hours |
| **3-year total** | **$10,800** | **$945** |

That's a saving of nearly $10,000 over three years. And the gap widens if you're on the Business plan ($45/seat), where the Airtable cost would be $24,300.

Even if you factor in occasional maintenance — updating Docker images, checking disk space, rotating backups — at maybe 2 hours per month of admin time, the self-hosted path is dramatically cheaper.

## Limitations to Be Aware Of

NocoDB isn't a perfect drop-in replacement for everything Airtable does. Here's what you give up:

- **No native mobile app** — NocoDB's web interface is responsive, but there's no dedicated mobile app. For field teams that rely on mobile data entry, this is a real gap.
- **Smaller integration ecosystem** — Airtable has hundreds of pre-built integrations. NocoDB has fewer, though the REST API and webhooks let you connect to anything.
- **No built-in email/send features** — Airtable can send emails natively. With NocoDB, you'll use webhooks + n8n + an SMTP node for email automation.
- **Community support, not enterprise support** — If something breaks, you're relying on GitHub issues and community forums, not a dedicated support team. For critical business processes, budget time for troubleshooting.
- **AGPL-3.0 license** — NocoDB is open source under AGPL, which means if you modify and distribute the software, you must release your modifications under the same license. For internal use, this doesn't matter. But if you're building a product on top of NocoDB, consult your legal team.

## Conclusion

Airtable is a great product. But its pricing model — per seat, with escalating tiers for features that should be standard — makes it expensive for growing teams. And the fact that your data sits on someone else's servers, accessible only through their API and export tools, is a risk.

NocoDB gives you the same no-code database experience: spreadsheet interface, multiple views, relationships, automations, dashboards, and APIs. But it runs on your server, costs nothing per seat, has no artificial limits, and gives you direct access to the underlying PostgreSQL or MySQL database. You own your data, your automations, and your infrastructure.

If you're already self-hosting tools like n8n, Odoo, or Mattermost, adding NocoDB to the stack is a natural fit. And if you're tired of Airtable's pricing increases, it's the most direct replacement available.

---

**Want help setting up NocoDB or migrating from Airtable?** ARDOT Consulting specializes in open-source automation and self-hosting for small businesses. [Contact us](https://www.ardotconsulting.com/#contact) to talk about your specific needs — we'll help you deploy, configure, and connect NocoDB to your existing tools.