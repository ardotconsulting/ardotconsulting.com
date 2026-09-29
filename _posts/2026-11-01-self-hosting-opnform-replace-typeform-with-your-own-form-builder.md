---
layout: post
title: "Self-Hosting OpnForm: Replace Typeform With Your Own Form Builder"
date: 2026-11-01
author: "ARDOT Consulting"
tags: [opnform, formbricks, forms, surveys, self-hosting, open-source, docker, typeform-alternative]
excerpt: "Typeform charges per response and locks your data behind a subscription. Here's how to self-host OpnForm and Formbricks — open source form and survey tools that cost nothing per submission."
---

Every form your business collects is a piece of customer data. Contact forms, lead qualification surveys, customer feedback, event registrations, intake questionnaires — each one captures information that should belong to you. Yet most businesses hand that data to a third-party platform and pay for the privilege.

Typeform starts at $25/month for 100 responses. Cross 1,000 responses and you're on the Business plan at $99/month. Need file uploads, logic jumps, or custom thank-you pages? Those are higher-tier features. Want to export your data? You can — but it lives on their servers until you do, and you're trusting their security, their uptime, and their pricing decisions.

There's a better approach. Two open source projects — **OpnForm** and **Formbricks** — let you run your own form builder on a $5/month VPS. Unlimited forms, unlimited responses, no per-submission pricing, and your data never leaves your server.

In this guide, we'll compare both tools, show you which one fits your use case, and walk through deploying OpnForm with Docker.

## OpnForm vs Formbricks: Which One Do You Need?

Both tools are open source, self-hostable, and actively maintained. But they're built for slightly different jobs.

**OpnForm** is a form builder. Think of it as a direct Typeform replacement. You create forms with a no-code drag-and-drop editor, embed them on your website, collect responses, and get email notifications. It supports file uploads, conditional logic, captcha, webhooks, and integrations with Slack and Discord. If you're collecting contact form submissions, job applications, event registrations, or customer inquiries, OpnForm is what you want.

**Formbricks** is a survey and experience management platform. It's positioned as an open source Qualtrics alternative. It does forms too, but its strength is in-app surveys, website intercept surveys, targeted user segmentation, and multi-channel feedback collection (link, email, in-app, website widget). If you're doing product research, customer satisfaction measurement, NPS tracking, or user journey analysis, Formbricks is the better fit.

| Feature | OpnForm | Formbricks | Typeform |
|---------|---------|------------|----------|
| Form types | Link + embed | Link, email, in-app, website widget | Link + embed |
| No-code editor | ✅ | ✅ | ✅ |
| Conditional logic | ✅ | ✅ | ✅ (higher tier) |
| File uploads | ✅ | ✅ | ✅ (higher tier) |
| Email notifications | ✅ | ✅ | ✅ |
| Webhooks | ✅ | ✅ | ✅ (higher tier) |
| Slack/Discord integration | ✅ | ✅ | Via Zapier ($$$) |
| In-app surveys | ❌ | ✅ | ❌ |
| User targeting/segmentation | ❌ | ✅ | ❌ |
| Captcha | ✅ | ✅ | ✅ |
| Analytics | Basic | Advanced | ✅ (higher tier) |
| GitHub stars | 3,755 | 13,039 | — |
| License | AGPL v3 | AGPL v3 | Proprietary |
| Cost (unlimited responses) | ~$5/mo VPS | ~$5/mo VPS | $25–$99+/mo |
| Data location | Your server | Your server | Vendor's cloud |

For most small businesses that just need forms on their website, OpnForm is the simpler choice. If you have a product or app and want to collect targeted feedback from specific user segments, Formbricks is worth the extra complexity.

We'll focus on OpnForm for the deployment walkthrough since it covers the most common use case: replacing Typeform for website forms.

## What OpnForm Gives You Out of the Box

OpnForm runs as a PHP/Laravel backend with a Nuxt.js frontend. The self-hosted version includes:

- **Unlimited forms and submissions** — no per-response pricing, no caps
- **Drag-and-drop form builder** — text, date, URL, file upload, dropdown, checkbox, radio, and more
- **Conditional logic** — show or hide questions based on previous answers
- **Custom branding** — match your brand colors, add your logo, customize the thank-you page
- **Embed anywhere** — JavaScript embed, iframe, or direct link
- **Email notifications** — get notified when someone submits a form
- **Webhooks** — send submission data to n8n, Discord, Slack, or any HTTP endpoint
- **Captcha protection** — spam prevention built in
- **Form analytics** — see views, completion rates, and drop-off
- **API access** — pull submissions programmatically for downstream automation

The webhooks and API access are where this gets interesting for automation. Every form submission can trigger an n8n workflow — add a row to NocoDB, create a contact in Odoo, send a Slack notification, or kick off any process you've built. You're not limited to the integrations a vendor decided to build.

## Deploying OpnForm with Docker

OpnForm provides an official Docker setup. You'll need a server with Docker and Docker Compose installed — a $5/month VPS with 1GB RAM is sufficient for small-to-medium form volume.

### Step 1: Create the Docker Compose File

Create a directory for your OpnForm deployment and add a `docker-compose.yml`:

```yaml
version: "3.8"

services:
  opnform-api:
    image: jhumanj/opnform-api:latest
    restart: unless-stopped
    env_file: .env
    depends_on:
      - postgres
      - redis
    ports:
      - "8080:80"

  opnform-client:
    image: jhumanj/opnform-client:latest
    restart: unless-stopped
    depends_on:
      - opnform-api
    ports:
      - "3000:3000"

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: opnform
      POSTGRES_USER: opnform
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Step 2: Configure the Environment File

Create a `.env` file in the same directory:

```bash
APP_NAME=OpnForm
APP_KEY=base64:GENERATE_A_32_CHAR_KEY_HERE
APP_URL=https://forms.yourdomain.com

DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=opnform
DB_USERNAME=opnform
DB_PASSWORD=your_secure_password_here

REDIS_HOST=redis
REDIS_PORT=6379

MAIL_HOST=smtp.yourprovider.com
MAIL_PORT=587
MAIL_USERNAME=you@yourdomain.com
MAIL_PASSWORD=your_smtp_password
MAIL_FROM_ADDRESS=you@yourdomain.com
MAIL_FROM_NAME="Your Business"
```

Generate an app key with `php artisan key:generate --show` or use any base64-encoded 32-character string. The mail settings are important — without them, you won't receive email notifications when forms are submitted.

### Step 3: Start the Stack

```bash
docker compose up -d
```

Wait a minute for the containers to initialize, then check that they're running:

```bash
docker compose ps
```

You should see four containers: `opnform-api`, `opnform-client`, `postgres`, and `redis` — all with status "Up".

### Step 4: Set Up a Reverse Proxy

You don't want to expose ports 8080 and 3000 directly. Put Caddy or Nginx in front to handle TLS and route traffic properly. With Caddy, a minimal config looks like:

```
forms.yourdomain.com {
    reverse_proxy localhost:3000
}
```

Caddy automatically provisions and renews Let's Encrypt certificates, so HTTPS is handled. Point your DNS `A` record for `forms.yourdomain.com` to your server's IP address and you're live.

### Step 5: Create Your First Form

Visit `https://forms.yourdomain.com`, register an account (the first user becomes the admin), and start building forms. The editor is self-explanatory — add fields, configure logic, set up notifications, and publish.

To embed a form on your website, OpnForm gives you a JavaScript snippet or an iframe URL. Drop it into any page and the form renders with your custom branding.

## Connecting OpnForm to Your Automation Stack

The real value of self-hosting forms isn't just saving money — it's the ability to pipe submissions directly into your workflows without a middleman.

OpnForm supports webhooks on form submission. In n8n, create a Webhook node and paste the URL into OpnForm's webhook settings. Now every submission triggers your workflow automatically.

A practical example: a real estate agency has a property inquiry form on their website. When someone submits it:

1. **OpnForm** captures the submission and fires a webhook
2. **n8n** receives the payload and routes it:
   - Creates a lead in **Odoo CRM** with the prospect's details
   - Sends a **Slack/Mattermost** notification to the relevant agent
   - Triggers an automatic email reply with property details
   - Adds the prospect to a **Listmonk** mailing list for that property type

No Zapier. No per-task pricing. No data passing through a third-party integration platform. Everything stays on your infrastructure.

## Backing Up Your Data

Your form submissions live in the PostgreSQL database. Back it up the same way you'd back up any Postgres instance:

```bash
docker exec opnform-postgres pg_dump -U opnform opnform > backup_$(date +%Y%m%d).sql
```

Schedule this as a daily cron job and copy the backups to off-site storage. If you're already running n8n, you can automate the backup-to-S3 step as a scheduled workflow — the same pattern we covered in the Listmonk backup guide.

## When to Stick With a Hosted Form Service

Self-hosting isn't always the right call. If you collect fewer than 50 responses a month and don't need custom integrations, a free Typeform or Tally account is simpler. You'll spend more time setting up Docker than you'll save in subscription costs.

Self-hosting OpnForm makes sense when:

- You collect more than 200 responses per month and per-response pricing stings
- You need form data to flow directly into n8n, Odoo, or other self-hosted tools
- You're already running a server for other self-hosted applications (Nextcloud, Listmonk, Mattermost) and adding one more service is trivial
- You want full control over form data for privacy or compliance reasons

If any of those apply, the setup pays for itself within the first month.

## The Bottom Line

Typeform makes beautiful forms. But you're renting access to your own data, and the rent goes up every time your forms succeed. OpnForm gives you the same core capabilities — no-code builder, conditional logic, file uploads, embeds, notifications, webhooks — with no per-response pricing and no data leaving your server.

If you're already on the self-hosting path with tools like Nextcloud, Listmonk, or Mattermost, OpnForm is a natural addition. It takes 30 minutes to set up and integrates cleanly with the rest of an open source automation stack.

For survey-heavy use cases — NPS, in-app feedback, user segmentation — pair it with Formbricks or use Formbricks on its own. Both tools are AGPL-licensed, actively maintained, and cost exactly what your VPS costs.

---

**Want help setting up OpnForm, Formbricks, or connecting your forms to an automation workflow?** [Get in touch](/#contact) — we help businesses build self-hosted form and automation stacks that replace expensive SaaS subscriptions without sacrificing capability.