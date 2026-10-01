---
layout: post
title: "Self-Hosting Directus: Build Custom Business Apps Without Writing Code"
date: 2026-10-01
author: "ARDOT Consulting"
tags: [directus, headless-cms, self-hosting, no-code, database, api, automation]
excerpt: "Directus wraps any SQL database with instant APIs, a no-code admin panel, and automation — perfect for businesses that need custom data tools without a developer."
---

## The Spreadsheet Problem Every Business Hits

You start with a spreadsheet. It works fine for ten rows. Then a hundred. Then someone adds a second sheet with a VLOOKUP to the first. Before long, you've got a fragile web of interconnected spreadsheets that nobody fully understands, no one wants to touch, and that breaks when someone sorts a column wrong.

This is the universal small-business data story. You need something more structured than a spreadsheet but less expensive than hiring a developer to build a custom application. That gap — between "spreadsheet" and "custom software" — is exactly where **Directus** fits.

Directus is an open-source data platform that connects to any standard SQL database and instantly gives you three things: a REST and GraphQL API, a visual admin interface (called the Data Studio), and a built-in automation engine called Flows. You define your data model through a no-code interface, and Directus handles the rest — APIs, file management, user roles, access control, and dashboards.

No per-seat SaaS pricing. No vendor lock-in. You own the database, the data, and the server.

## What Directus Actually Does

Let's break this down concretely. Imagine you run a small distribution company and you want to manage products, customers, and orders — without buying an ERP, without hiring a developer, and without living in spreadsheets forever.

Here's what happens when you point Directus at a fresh database:

### 1. You Define Your Data Model — No Code

Through the Data Studio's visual interface, you create collections (tables) and fields. You define relationships — a customer has many orders, an order has many line items. You set field types: text, numbers, dates, dropdowns, files, JSON. You add validation rules: this field is required, that one must be unique, this one can't be negative.

No SQL. No migration scripts. No ORM configuration. You click through a form, and Directus writes the schema to your database behind the scenes.

### 2. You Get APIs Automatically

The moment your collections exist, Directus generates REST and GraphQL endpoints for them. Want to fetch all published products? 

```bash
curl https://directus.yourcompany.com/items/products?filter[status]=published
```

Want to create a new order via GraphQL?

```graphql
mutation {
  create_orders_item(data: {
    customer_id: 42
    total: 1299.99
    status: "pending"
  }) {
    id
    status
  }
}
```

This is the core value proposition: **your database becomes an API without you writing a single line of backend code.** If you later want to build a mobile app, a web storefront, or connect n8n workflows to your data, the API is already there.

### 3. You Get a Visual Admin Interface

The Data Studio is a full-featured admin panel that adapts to your data model automatically. Your team sees forms, tables, calendars, and maps — all generated from the fields you defined. Non-technical staff can browse records, search, filter, edit, and upload files without ever touching a database or an API.

This is the interface your operations team actually uses day to day. It replaces the spreadsheet.

### 4. You Get Role-Based Access Control

Directus has granular, policy-based access control. You define roles (admin, manager, staff, read-only) and assign permissions per collection: who can read, create, update, or delete. You can even filter by field — a staff user might see order totals but not customer email addresses.

This matters for businesses handling sensitive data. You don't want everyone in the company to see everything.

### 5. You Get Automation with Flows

Flows is Directus's built-in automation engine — think of it as a lightweight n8n inside your data platform. You define triggers (a record is created, updated, or deleted; a webhook fires; a schedule runs) and then chain operations: send an email, call an external API, run custom JavaScript, create a log entry, trigger another flow.

For example, you could set up a flow that fires when an order status changes to "shipped" — it sends a notification to the customer, updates an inventory count, and logs the shipment to an external logistics API.

### 6. You Get File Management

Directus includes a digital asset manager. Upload images, PDFs, documents — they're stored alongside your structured data. Directus can generate thumbnails on demand, serve files through a CDN, and even let you request images at specific dimensions without pre-generating variants.

## How Directus Compares to Alternatives

| Feature | Directus | Strapi | NocoDB | Airtable |
|---|---|---|---|---|
| License | MSCL (free core) | MIT | AGPL | Proprietary |
| Self-hosted | Yes | Yes | Yes | No |
| Database | Any SQL (Postgres, MySQL, etc.) | Any SQL | Any SQL | Proprietary |
| REST API | Yes | Yes | Yes | Limited |
| GraphQL API | Yes | Yes | No | No |
| No-code admin UI | Full Data Studio | Content-type Builder | Spreadsheet-like | Native |
| File management | Built-in DAM | Basic | Basic | Built-in |
| Automation | Flows (built-in) | Webhooks only | Webhooks only | Automations |
| Role-based access | Granular policies | Basic roles | Basic | Basic |
| Realtime | Yes | No | No | No |
| Pricing | Free self-hosted | Free self-hosted | Free self-hosted | $20+/user/month |

The key differences: **Strapi** is more developer-oriented and focused on content management (it's a headless CMS first). **NocoDB** turns any database into a smart spreadsheet — simpler, less feature-rich. **Airtable** is the closed-source SaaS most people are trying to escape. Directus sits in the middle: powerful enough for real applications, approachable enough for non-developers, and fully open source.

## Self-Hosting Directus with Docker

The fastest way to get started is Docker Compose. Here's a working setup that runs Directus with a PostgreSQL database:

```yaml
# docker-compose.yml
version: "3.8"

services:
  database:
    image: postgis/postgis:16-3.4
    volumes:
      - directus_db:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: directus
      POSTGRES_PASSWORD: change_this_strong_password
      POSTGRES_DB: directus
    restart: unless-stopped

  directus:
    image: directus/directus:latest
    ports:
      - "8055:8055"
    volumes:
      - directus_uploads:/directus/uploads
      - directus_extensions:/directus/extensions
    environment:
      SECRET: "generate_a_random_32_char_string"
      ADMIN_EMAIL: "admin@yourcompany.com"
      ADMIN_PASSWORD: "change_this_strong_password"
      DB_CLIENT: "pg"
      DB_HOST: "database"
      DB_PORT: "5432"
      DB_DATABASE: "directus"
      DB_USER: "directus"
      DB_PASSWORD: "change_this_strong_password"
      PUBLIC_URL: "https://directus.yourcompany.com"
      # Optional: enable caching with Redis
      # REDIS: "redis://redis:6379"
    depends_on:
      - database
    restart: unless-stopped

volumes:
  directus_db:
  directus_uploads:
  directus_extensions:
```

Spin it up:

```bash
docker compose up -d
```

Navigate to `http://your-server-ip:8055` and log in with the admin email and password you set. That's it — you have a running Directus instance.

### Putting It Behind a Reverse Proxy

For production, you'll want HTTPS. Here's a minimal Nginx config:

```nginx
server {
    server_name directus.yourcompany.com;

    location / {
        proxy_pass http://127.0.0.1:8055;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/directus.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/directus.yourcompany.com/privkey.pem;
}
```

Use Certbot (Let's Encrypt) for free SSL certificates:

```bash
sudo certbot --nginx -d directus.yourcompany.com
```

## A Practical Example: Building a Product Catalog

Let's walk through building something real — a product catalog that your sales team can browse and your website can query via API.

**Step 1: Create a "Products" collection.**

In the Data Studio, go to Data Model → Create Collection. Name it `products`. Add these fields:

| Field Name | Type | Notes |
|---|---|---|
| name | String | Required |
| sku | String | Unique |
| description | Text | — |
| price | Float | Min: 0 |
| stock_quantity | Integer | Default: 0 |
| category | String (dropdown) | Options: Electronics, Home, Office, Other |
| status | String (dropdown) | Options: draft, published, archived |
| image | File (single) | — |
| created_at | Timestamp | Auto-generated |

**Step 2: Add a "Categories" collection (optional).**

Instead of a dropdown, you could create a separate `categories` collection with fields for name and description, then create an M2O (many-to-one) relationship from products to categories. Directus handles the foreign key automatically.

**Step 3: Set up access.**

Create a role called "Sales Team" with read access to products where `status` equals `published`. Create an admin role with full access to all collections.

**Step 4: Query via API.**

Your website can now fetch published products:

```bash
curl "https://directus.yourcompany.com/items/products?filter[status]=published&fields=name,price,image&sort=price"
```

The response is clean JSON:

```json
{
  "data": [
    {
      "name": "Wireless Headphones",
      "price": 89.99,
      "image": "https://directus.yourcompany.com/assets/abc123"
    },
    {
      "name": "USB-C Cable",
      "price": 12.50,
      "image": "https://directus.yourcompany.com/assets/def456"
    }
  ]
}
```

**Step 5: Add automation.**

Create a Flow that triggers when `stock_quantity` drops below 5. The flow sends an email to your purchasing manager with the product name and current stock level. Now you have automated low-stock alerts without any external tools.

## When Directus Is the Right Choice

Directus shines in these scenarios:

- **You've outgrown spreadsheets** but can't justify a custom-built application
- **You need an API** for a mobile app, website, or integration, but don't want to build and maintain a backend
- **You need a shared admin interface** that non-technical staff can use without training
- **You want role-based access control** so different team members see different data
- **You're already using a SQL database** (PostgreSQL, MySQL, SQLite, Oracle, MariaDB, MS SQL) and want a front-end for it

It's less ideal if:

- You need a public-facing website with a full templating engine (use a static site generator or Jekyll for that — Directus provides the data, not the presentation)
- You need complex offline sync or mobile-first data collection (something like Odoo or a purpose-built app may serve better)
- You want a pure content/blog CMS (Ghost or Jekyll are simpler for that narrow use case)

## Connecting Directus to n8n for Extended Automation

Directus's built-in Flows handle simple automations well. But when you need to connect to dozens of external services — email providers, CRMs, messaging platforms — you can pair Directus with n8n for a powerful combination.

The pattern: **Directus manages your data and APIs, n8n orchestrates complex multi-step workflows that pull from and write to Directus.**

Here's an example n8n workflow:

1. **Webhook trigger** — Directus Flow fires a webhook when a new order is created
2. **HTTP Request node** — n8n fetches the full order details from the Directus REST API
3. **Ollama node** — a local LLM generates a personalized thank-you message based on the customer's order history
4. **Email node** — n8n sends the order confirmation with the AI-generated message
5. **HTTP Request node** — n8n updates the order status in Directus to "confirmed"

This gives you the best of both worlds: Directus as your structured data layer, n8n as your integration orchestrator, and Ollama for AI-powered personalization — all self-hosted, all open source.

## Licensing: What You Need to Know

Directus uses the **Monospace Sustainable Core License (MSCL)**. The free core tier covers most small-business use cases — unlimited collections, unlimited records, full API access, the Data Studio, Flows, and file management. Paid licenses unlock higher limits and enterprise features.

For a small business self-hosting Directus on a single server with a modest user base, the free tier is more than sufficient. And because your data lives in a standard SQL database, you're never locked in — if you ever decide to move away from Directus, your data stays in PostgreSQL, fully accessible.

## Hardware Requirements

Directus is a Node.js application, so it's lightweight compared to many enterprise platforms. For a small team (under 25 users):

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 1 core | 2 cores |
| RAM | 1 GB | 2 GB |
| Storage | 10 GB | 50 GB (for file uploads) |
| Database | SQLite (testing) | PostgreSQL 14+ (production) |

A $6/month VPS from Hetzner, OVH, or DigitalOcean can comfortably run Directus with PostgreSQL for a small business. Compare that to $20+ per user per month for most SaaS alternatives.

## The Bottom Line

Directus fills a gap that most businesses encounter eventually: the space between "we'll just use a spreadsheet" and "we need to hire a developer." By wrapping any SQL database with instant APIs, a visual admin interface, and built-in automation, it lets you build real data-driven tools without writing backend code.

Pair it with n8n for complex integrations and Ollama for AI features, and you have a complete open-source data platform that replaces several SaaS subscriptions at once — for the cost of a single small VPS.

---

*Want help setting up Directus or building custom business automation workflows? [Get in touch with ARDOT Consulting](/) — we design and implement open-source automation systems tailored to your business.*