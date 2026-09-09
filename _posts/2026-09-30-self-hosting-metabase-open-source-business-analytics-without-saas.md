---
layout: post
title: "Self-Hosting Metabase: Open-Source Business Analytics Without the SaaS Tax"
date: 2026-09-30
author: "ARDOT Consulting"
tags: [metabase, analytics, business-intelligence, self-hosting, docker, open-source]
excerpt: "Replace expensive BI tools with self-hosted Metabase — open-source dashboards and SQL queries that keep your data on your own server."
---

Every growing business eventually hits the same wall: you have data spread across a dozen tools — your CRM, your invoicing system, your web analytics, your inventory manager — but no single place to see it all. The obvious answer is a business intelligence (BI) tool. The obvious problem is the price tag.

Tableau starts at $75 per user per month. Looker requires a conversation with sales before you even see a number. Power BI's licensing tiers are a spreadsheet unto themselves. For a 10-person team that needs three dashboards, you're looking at $2,000+ per year before anyone has built a single chart.

Metabase is the open-source alternative. It gives you visual dashboards, a no-code query builder, and native SQL access — all self-hosted, all yours, with zero per-user licensing. In this guide, we'll walk through what Metabase does well, how to deploy it with Docker, and how to connect it to your existing data sources.

## What Metabase Actually Does

Metabase is a business intelligence tool. You connect it to a database (PostgreSQL, MySQL, MongoDB, SQLite, and others), and it gives you two ways to explore that data:

1. **A visual query builder** — filter, group, and aggregate data without writing SQL. You pick a table, choose columns, apply filters, and Metabase generates the chart. This is what non-technical team members use day-to-day.

2. **Native SQL editor** — write your own queries, save them as reusable models, and build dashboards from the results. This is what your technical folks use for complex analysis.

The key feature that separates Metabase from a simple charting library is **questions and dashboards**. A "question" is any saved query — visual or SQL. A "dashboard" is a collection of questions arranged on a grid with filters that affect all questions at once. You build once, and your team interacts without touching the underlying query.

### What It Does Not Do

Metabase is not an ETL tool. It reads from your database; it doesn't transform or move data between systems. If your data lives in five different databases, you'll need to consolidate it first (or connect Metabase to each one separately). For ETL, look at tools like n8n or Directus — we covered building data pipelines in a [previous post](/blog/2026/09/28/building-data-pipelines-connect-your-business-systems-without-code/).

Metabase also doesn't do real-time streaming analytics. It queries your database on demand or on a refresh schedule (every hour, daily, etc.). For most small-to-medium businesses, that's perfectly fine.

## Metabase vs Other BI Tools

| Feature | Metabase (OSS) | Tableau | Power BI | Grafana |
|---------|---------------|---------|----------|---------|
| License | AGPL (free, self-hosted) | Commercial | Commercial | AGPL (free, self-hosted) |
| Per-user cost | $0 | $75+/user/mo | $10-20/user/mo | $0 |
| No-code query builder | Yes | Yes | Yes | Limited |
| Native SQL | Yes | Yes | Yes | Yes |
| Self-hosted | Yes | No | No (Power BI Report Server limited) | Yes |
| Best for | Business analytics | Enterprise BI | Microsoft shops | Infrastructure/ops monitoring |
| Setup time | ~15 min with Docker | Days (procurement + setup) | Hours | ~15 min with Docker |

The Grafana comparison is worth clarifying. Grafana is excellent, but it's designed for time-series monitoring — server metrics, application performance, IoT sensors. Metabase is designed for business data — sales, inventory, customer behavior. They overlap in capability but serve different primary use cases. Many businesses run both.

## Deploying Metabase with Docker

The fastest way to get Metabase running is Docker. You'll need Docker and Docker Compose installed on a server (a $10/month VPS is plenty for a small team). Here's a complete `docker-compose.yml`:

```yaml
version: "3.8"

services:
  metabase:
    image: metabase/metabase:latest
    container_name: metabase
    hostname: metabase
    ports:
      - "3000:3000"
    environment:
      - MB_DB_TYPE=postgres
      - MB_DB_DBNAME=metabase
      - MB_DB_PORT=5432
      - MB_DB_USER=metabase
      - MB_DB_PASS=change_this_password
      - MB_DB_HOST=metabase-db
    depends_on:
      - metabase-db
    restart: unless-stopped

  metabase-db:
    image: postgres:15-alpine
    container_name: metabase-db
    environment:
      - POSTGRES_USER=metabase
      - POSTGRES_PASSWORD=change_this_password
      - POSTGRES_DB=metabase
    volumes:
      - metabase-db-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  metabase-db-data:
```

Save this as `docker-compose.yml` and run:

```bash
docker compose up -d
```

Metabase uses its own PostgreSQL database to store dashboards, saved questions, and user accounts — that's the `metabase-db` service. The app itself runs on port 3000. Open `http://your-server-ip:3000` in a browser and you'll see the setup wizard.

### Securing the Deployment

Running Metabase on port 3000 without TLS is fine for testing, but for production you should put it behind a reverse proxy with HTTPS. Here's a minimal Caddy configuration:

```caddyfile
analytics.yourcompany.com {
    reverse_proxy localhost:3000
}
```

Caddy automatically provisions and renews Let's Encrypt certificates. Install Caddy, drop this config in `/etc/caddy/Caddyfile`, and restart. Your Metabase instance is now accessible over HTTPS at `analytics.yourcompany.com` — no certificate management required.

### Authentication and Access Control

Metabase has built-in user management. During setup, you create an admin account. From the admin settings, you can:

- **Create user accounts** for each team member with email/password login
- **Define groups** (e.g., "Sales", "Engineering", "Leadership")
- **Set permissions** per database, per table, or per saved question — control who can view, who can edit, and who sees raw data vs aggregated results
- **Enable SSO** via LDAP or SAML if you have an identity provider

For a small team, the default email/password auth is fine. For larger organizations, connect Metabase to your existing SSO provider so access is managed centrally.

## Connecting Metabase to Your Data

Once Metabase is running, the real work begins: connecting it to your data sources. Here are the most common scenarios for small businesses:

### Scenario 1: You Already Have a PostgreSQL Database

If your application (Odoo, Directus, a custom app) already uses PostgreSQL, Metabase can connect directly. In the admin settings:

1. Click **Add Database**
2. Select **PostgreSQL**
3. Enter the host, port, database name, username, and password
4. Metabase scans the schema and makes all tables available

That's it. Your team can immediately start building charts from your production data.

### Scenario 2: Your Data Is in a CSV or Spreadsheet

Metabase doesn't connect directly to CSV files, but you can load them into a SQLite database in seconds:

```bash
# Install sqlite3 if needed
sudo apt install sqlite3

# Create a database from a CSV
sqlite3 mydata.db
.mode csv
.import sales_data.csv sales
.import customers.csv customers
.exit
```

Then run a temporary Metabase container with the SQLite file mounted:

```bash
docker run -d -p 3000:3000 \
  -v /path/to/mydata.db:/data/mydata.db \
  metabase/metabase:latest
```

In the Metabase admin settings, add a new database of type "SQLite" and point it to `/data/mydata.db`. Your spreadsheet data is now queryable with the full power of Metabase's visualization engine.

### Scenario 3: Multiple Data Sources

If your CRM is in PostgreSQL and your web analytics are in a separate database, connect Metabase to both. You'll have two databases listed in Metabase, and you can build dashboards that pull from either — though cross-database queries aren't supported natively. For unified reporting, consolidate your data into a single database using an ETL pipeline (n8n is excellent for this).

## Building Your First Dashboard

Let's walk through a practical example. Say you want a sales overview dashboard with three panels: revenue by month, top 10 customers, and order count by status.

### Step 1: Connect Your Database

In the admin panel, add your PostgreSQL database. Metabase auto-detects tables and columns.

### Step 2: Create a "Revenue by Month" Question

1. Click **+ New** → **Question**
2. Select your database and the `orders` table
3. Choose **Sum of** `total_amount`
4. Group by `created_at` → **Month**
5. Pick a bar chart visualization
6. Save the question as "Revenue by Month"

### Step 3: Create a "Top 10 Customers" Question

1. Click **+ New** → **Native Query**
2. Write:
```sql
SELECT c.name, SUM(o.total_amount) AS total_spent
FROM orders o
JOIN customers c ON o.customer_id = c.id
GROUP BY c.name
ORDER BY total_spent DESC
LIMIT 10
```
3. Choose a horizontal bar chart
4. Save as "Top 10 Customers"

### Step 4: Create an "Orders by Status" Question

1. Visual query builder → `orders` table
2. Count of rows
3. Group by `status`
4. Choose a pie or donut chart
5. Save as "Orders by Status"

### Step 5: Assemble the Dashboard

1. Click **+ New** → **Dashboard**
2. Name it "Sales Overview"
3. Add your three saved questions
4. Add a date filter that affects the Revenue question
5. Arrange the panels with drag-and-drop

The entire process takes about 15 minutes for a straightforward dashboard. Once saved, any team member with access can view it, and the data refreshes on every page load (or on the schedule you configure).

## Automating Metabase with n8n

Metabase has a REST API that lets you programmatically query saved questions, trigger dashboard refreshes, and export results. Combined with n8n, you can build automated workflows like:

- **Daily email report** — every morning, n8n calls the Metabase API to fetch yesterday's revenue numbers and emails a summary to the team
- **Slack/Mattermost alert** — when a saved question returns results below a threshold, n8n posts an alert to a channel
- **Scheduled export** — every Friday, n8n pulls a dashboard as CSV and uploads it to a shared drive

Here's an example n8n workflow that sends a weekly Metabase summary via Mattermost:

```json
{
  "trigger": "Every Monday at 8:00 AM",
  "steps": [
    {
      "action": "HTTP Request",
      "method": "GET",
      "url": "http://metabase:3000/api/card/42/query",
      "headers": { "X-Metabase-Session": "{{metabase_api_key}}" }
    },
    {
      "action": "Set",
      "fields": {
        "revenue": "{{ $json.data.rows[0][0] }}",
        "orders": "{{ $json.data.rows[0][1] }}"
      }
    },
    {
      "action": "Mattermost",
      "message": "📊 Weekly Summary: Revenue ${{ $json.revenue }} from {{ $json.orders }} orders."
    }
  ]
}
```

This pattern — Metabase for visualization, n8n for orchestration — gives you a BI platform that rivals commercial tools costing thousands per year, entirely on infrastructure you control.

## Realistic Costs

Self-hosting isn't free — you pay for the server. Here's a realistic cost breakdown for a 10-person team:

| Item | Monthly Cost | Notes |
|------|-------------|-------|
| VPS (2 vCPU, 4GB RAM) | $10-20 | Hetzner, DigitalOcean, or similar |
| Domain (analytics subdomain) | $1 | If you already own the domain |
| TLS certificate | $0 | Let's Encrypt via Caddy |
| Metabase license | $0 | AGPL open source |
| Per-user licensing | $0 | Unlimited users |
| **Total** | **$11-21/mo** | For the entire team |

Compare that to Tableau Creator at $75/user/month — for 10 users, that's $750/month, or $9,000/year. The self-hosted Metabase setup costs under $250/year for the same team.

The trade-off is maintenance. You're responsible for backups, updates, and security patches. This is real work, but it's a few hours per quarter, not a full-time job. If that still feels like too much, Metabase also offers a cloud-hosted version starting at $85/month for 5 users — still significantly cheaper than Tableau, and with a clear upgrade path to self-hosting when you're ready.

## When to Choose Metabase (and When Not To)

**Choose Metabase if:**

- You have a database and want visual dashboards without per-user licensing
- Your team needs both no-code and SQL access to data
- You want to self-host for data sovereignty or compliance reasons
- You're already using PostgreSQL, MySQL, or MongoDB

**Consider alternatives if:**

- You need real-time streaming analytics (look at Apache Superset with a columnar database)
- Your team has no one who can manage a Docker deployment (use Metabase Cloud or a managed alternative)
- You need pixel-perfect, branded report exports (Metabase's exports are functional but not design-grade)
- Your data is primarily in spreadsheets and you don't want to import it into a database (look at a tool like Grist, which works directly with spreadsheets)

## Getting Started Checklist

1. **[ ] Provision a VPS** with at least 2 vCPU and 4GB RAM
2. **[ ] Install Docker and Docker Compose** on the server
3. **[ ] Copy the docker-compose.yml** from above and run `docker compose up -d`
4. **[ ] Set up Caddy** (or nginx) as a reverse proxy with HTTPS
5. **[ ] Complete the Metabase setup wizard** — create your admin account
6. **[ ] Connect your first database** — start with whatever has the most interesting data
7. **[ ] Build your first dashboard** — pick a question your team asks every week
8. **[ ] Create user accounts** for your team and assign them to groups
9. **[ ] Set up a backup** — at minimum, `pg_dump` the Metabase database daily
10. **[ ] Schedule a review** — after 30 days, check what dashboards are actually being used

## Wrapping Up

Business intelligence doesn't have to mean a five-figure annual contract. Metabase gives you the core capabilities — visual queries, SQL access, shared dashboards, user permissions — in a package you can deploy in an afternoon and run for the cost of a cheap server.

The combination of Metabase for analytics, n8n for automation, and tools like Ollama for AI creates a complete open-source business stack that replaces commercial alternatives at a fraction of the cost. You keep your data, you control your costs, and you're never at the mercy of a pricing page change.

If you'd like help setting up Metabase or building a custom analytics dashboard for your business, [reach out through our contact form](/#contact). We specialize in open-source automation for small and medium businesses — no vendor lock-in, no surprise bills.

---

*Want to see Metabase in action with your own data? ARDOT Consulting can help you deploy, configure, and connect your business systems. [Contact us](/#contact) to schedule a free consultation.*