---
layout: post
title: "Self-Hosting Mautic: Replace HubSpot with Your Own Marketing Automation Platform"
date: 2026-11-23
author: "ARDOT Consulting"
tags: [mautic, marketing-automation, self-hosting, open-source, hubspot, crm, email]
excerpt: "HubSpot's marketing hub costs thousands per month. Mautic gives you the same capabilities — email campaigns, lead scoring, landing pages, and contact management — in a self-hosted platform you fully control."
---

## The Marketing Automation Tax

If you've ever looked at HubSpot's pricing page, you know the sticker shock. The Marketing Hub Professional tier starts at $890/month — and that's for a single brand, 2,000 contacts, and basic automation. Need more contacts? The price scales linearly. Need the Enterprise tier for A/B testing, custom reporting, or multi-touch attribution? You're looking at $3,200/month or more.

For a small business with 10,000 contacts, you could easily spend $15,000–$25,000 per year on marketing automation alone. That's before you factor in the training costs, the consultant fees to set up your workflows, and the fact that your data lives on someone else's servers.

There's a better way.

[Mautic](https://www.mautic.org/) is a fully open source marketing automation platform — released under the GPL v3 license — that gives you the same core capabilities as HubSpot's Marketing Hub: email campaigns, lead scoring, landing page builder, contact segmentation, form builder, and marketing analytics. The difference? You self-host it, so there are no per-contact fees, no monthly subscriptions, and no vendor lock-in.

In this guide, I'll walk you through what Mautic can do, how to deploy it with Docker, and how it compares to HubSpot across the features that matter most to small businesses.

## What Mautic Actually Does

Mautic is built around the concept of **contacts** — individual people whose interactions with your business you track and act on. Here's what that looks like in practice:

### Email Marketing and Campaigns

Mautic's campaign builder is a visual drag-and-drop editor. You define a sequence of actions and decisions:

- **Action:** Send an email
- **Decision:** Did the contact open it? → Yes: wait 2 days, send follow-up → No: wait 5 days, send a different follow-up
- **Action:** Add to a segment, notify a team member, update a contact field

This is the same workflow logic HubSpot charges premium pricing for. Mautic includes it in the core package.

### Lead Scoring

Every interaction a contact has — opening an email, visiting a page, filling out a form, downloading a resource — can add or subtract points from their score. You define the rules:

- Visited pricing page: +15 points
- Downloaded whitepaper: +20 points
- Opened 3+ emails in 30 days: +10 points
- Unsubscribed: -100 points

When a contact crosses a threshold (say, 50 points), Mautic can automatically notify your sales team, add them to a "hot leads" segment, or trigger a campaign. This is lead scoring the way HubSpot, Pardot, and Marketo do it — but without the per-seat license.

### Landing Pages and Forms

Mautic includes a landing page builder with pre-built templates and a form builder that integrates directly with your campaigns. When someone fills out a form on a Mautic landing page, they're instantly added as a contact, scored, segmented, and dropped into any campaign you've configured.

### Contact Segmentation

Segments are dynamic — contacts are automatically added or removed based on rules you define. "All contacts who opened an email in the last 30 days but haven't visited the pricing page" is a segment. "All contacts with a lead score above 50 in the healthcare industry" is a segment. You can use segments to target email sends, campaign entry, and reporting.

### Tracking and Analytics

Mautic drops a tracking pixel on your website (similar to Plausible or Google Analytics) that monitors visitor behavior. When an anonymous visitor fills out a form and becomes a known contact, Mautic retroactively associates their previous page visits with their profile — giving you a full interaction history.

## Deploying Mautic with Docker

Mautic runs as a standard PHP application with a MySQL/MariaDB database. The easiest way to deploy it is with Docker Compose. Here's a production-ready configuration:

```yaml
# docker-compose.yml
version: "3.8"

services:
  mautic_db:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: mautic
      MYSQL_USER: mautic
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mautic_db_data:/var/lib/mysql
    restart: unless-stopped

  mautic:
    image: mautic/mautic:v5
    depends_on:
      - mautic_db
    environment:
      MAUTIC_DB_HOST: mautic_db
      MAUTIC_DB_NAME: mautic
      MAUTIC_DB_USER: mautic
      MAUTIC_DB_PASSWORD: ${MYSQL_PASSWORD}
      MAUTIC_URL: https://marketing.yourdomain.com
      PHP_MEMORY_LIMIT: 512M
      PHP_MAX_EXECUTION_TIME: 300
    ports:
      - "8080:80"
    volumes:
      - mautic_data:/var/www/html
    restart: unless-stopped

volumes:
  mautic_db_data:
  mautic_data:
```

Create a `.env` file with your passwords:

```bash
MYSQL_ROOT_PASSWORD=use_a_strong_random_password_here
MYSQL_PASSWORD=another_strong_random_password_here
```

Then start it up:

```bash
docker compose up -d
```

Point your browser to `http://your-server-ip:8080` and you'll see the Mautic installer. Follow the setup wizard — create your admin account, configure your site URL, and you're in.

### Putting It Behind a Reverse Proxy

For production, you'll want HTTPS. If you're already using Caddy (the easiest reverse proxy for self-hosted setups), add this to your Caddyfile:

```
marketing.yourdomain.com {
    reverse_proxy localhost:8080
}
```

Caddy automatically provisions and renews Let's Encrypt TLS certificates. Reload Caddy and your Mautic instance is live at `https://marketing.yourdomain.com`.

If you're using Nginx instead, a basic server block looks like:

```nginx
server {
    listen 80;
    server_name marketing.yourdomain.com;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Pair it with Certbot for free SSL certificates.

## Mautic vs HubSpot: Feature Comparison

| Feature | Mautic (Self-Hosted) | HubSpot Marketing Hub Pro |
|---------|---------------------|--------------------------|
| Monthly cost | $0 (you pay for hosting) | $890+ / month |
| Contact limit | Unlimited | 2,000 (then $0.10/contact) |
| Email campaigns | ✅ Visual builder | ✅ Visual builder |
| Lead scoring | ✅ Custom rules | ✅ Custom rules |
| Landing pages | ✅ Built-in builder | ✅ Built-in builder |
| Forms | ✅ Built-in | ✅ Built-in |
| Segmentation | ✅ Dynamic segments | ✅ Dynamic segments |
| A/B testing | ✅ Subject lines, content | ✅ (Enterprise tier: $3,200/mo) |
| SMS marketing | ✅ Via Twilio integration | ✅ (Add-on cost) |
| Web tracking | ✅ Tracking pixel | ✅ Tracking pixel |
| CRM integration | ✅ Built-in CRM + API | ✅ Built-in CRM |
| Data ownership | ✅ Your server, your data | ❌ HubSpot's servers |
| Customization | ✅ Full source code access | ❌ Limited to API + marketplace |
| Support | Community + paid options | ✅ Phone & email support |
| Setup time | 2–4 hours (Docker) | 30 minutes (cloud) |

The trade-off is clear: HubSpot is easier to set up and comes with phone support. Mautic costs nothing per month, has no contact limits, and gives you complete control over your data. For a business that already self-hosts other tools (like n8n, Nextcloud, or Metabase), adding Mautic to the stack is a natural extension.

## Integrating Mautic with Your Existing Stack

Mautic doesn't exist in isolation. Here's how it connects to the other open source tools you might already be running:

### Mautic + n8n

[n8n](https://n8n.io/) is an open source automation platform that connects to over 400 apps. You can use n8n to push contacts into Mautic from external sources, or to trigger actions in other systems when Mautic fires an event.

Example workflow: When a contact completes a Mautic campaign (high lead score), n8n creates a task in your project management tool (like OpenProject or Plane), sends a notification to your Mattermost channel, and creates a record in Odoo CRM — all automatically.

### Mautic + Odoo

If you're using [Odoo](https://www.odoo.com/) as your CRM or ERP, Mautic can hand off qualified leads directly. Use Mautic's webhook feature to push a contact to an n8n workflow that creates an Odoo lead record with all the interaction history attached. Your sales team picks it up in Odoo with full context — what emails the lead opened, what pages they visited, what they downloaded.

### Mautic + Ollama

[Ollama](https://ollama.ai/) lets you run large language models locally. You can use it to enrich Mautic contacts automatically: feed a contact's company website and interaction history to a local LLM, and have it generate a personalized email draft or suggest the best next action. Because Ollama runs on your own hardware, your contact data never leaves your server.

Here's a simple n8n workflow that does this:

1. **Trigger:** Mautic webhook fires when a contact's score crosses 50
2. **Action:** n8n calls the Ollama API with the contact's interaction history
3. **Prompt:** "Summarize this contact's interests based on their page visits and email opens. Suggest a personalized follow-up email."
4. **Action:** n8n sends the LLM's suggestion to your sales team's Mattermost channel

The contact data stays entirely on your infrastructure — Mautic's database, Ollama's local inference, n8n's self-hosted instance. No data sent to third-party AI APIs.

### Mautic + Cal.com

When a Mautic campaign reaches a "book a demo" stage, you can embed a [Cal.com](https://cal.com/) scheduling widget directly in a Mautic landing page or email. When the prospect books a meeting, Cal.com's webhook can fire back to Mautic to update the contact's stage and score — closing the loop between marketing automation and sales scheduling.

## What About Email Deliverability?

The most common concern with self-hosted email marketing is deliverability. If you send marketing emails from your own server, will they land in spam?

The answer: it depends on your setup, but it's entirely manageable with proper configuration. Here's what you need:

1. **Use a transactional email service, not your own mail server.** Mautic supports SMTP relay through services like Amazon SES, Postmark, Mailgun, or self-hosted options like [Postal](https://github.com/postalserver/postal). Amazon SES is the cheapest option — $0.10 per 1,000 emails — and has excellent deliverability.

2. **Set up SPF, DKIM, and DMARC records.** These DNS records prove to receiving servers that your email is legitimate. Your email service provider will give you the exact records to add.

3. **Warm up your sending domain.** Start with small sends (500 emails/day) and gradually increase. Mautic's cron jobs handle throttling automatically.

4. **Monitor bounces and complaints.** Mautic tracks bounce rates and unsubscribe rates. Keep your bounce rate below 2% and complaint rate below 0.1% and you'll maintain good sender reputation.

Here's a Mautic configuration for Amazon SES:

```php
// In Mautic settings → Configuration → Email Settings
Mailer transport: SMTP
SMTP host: email-smtp.us-east-1.amazonaws.com
SMTP port: 587
Encryption: TLS
SMTP username: [your SES IAM key]
SMTP password: [your SES IAM secret]
```

## Realistic Costs: Self-Hosted Mautic vs HubSpot

Let's do the math for a small business with 10,000 contacts sending 4 emails per month:

| Cost Item | HubSpot Marketing Pro | Self-Hosted Mautic |
|-----------|----------------------|-------------------|
| Platform license | $890/mo ($10,680/yr) | $0 |
| Contact overage (10K - 2K = 8K × $0.10/mo) | $800/mo ($9,600/yr) | $0 |
| Email sending (40K emails/mo via SES) | Included | $4/mo ($48/yr) |
| VPS hosting (4GB RAM) | Included | $20/mo ($240/yr) |
| Domain + SSL | Included | $10/yr |
| **Annual total** | **$20,280** | **$298** |

That's a **$19,982 annual savings** — for the same core functionality. Even if you hire a consultant for a one-time $2,000 setup fee, you're still saving $17,982 in year one.

The savings scale with your contact list. At 50,000 contacts, HubSpot costs $42,000+/year. Mautic still costs $298/year.

## When to Choose HubSpot Instead

I'm not going to pretend self-hosting is always the right answer. HubSpot is the better choice if:

- **You don't have anyone technical on your team.** Mautic requires Docker, DNS configuration, and periodic maintenance. If you don't have a developer or a managed hosting provider, HubSpot's managed platform is worth the premium.
- **You need built-in sales and service hubs.** HubSpot's CRM, sales pipeline, and help desk are tightly integrated. Mautic is marketing automation — it handles the top of the funnel. For the middle and bottom, you'll need separate tools.
- **You want phone support.** Mautic has a community forum and paid support options through [Mautic Cloud](https://www.mautic.com/), but self-hosted Mautic is community-supported. HubSpot gives you a dedicated rep.
- **You need advanced attribution reporting.** Mautic's reporting is functional but basic. HubSpot's multi-touch revenue attribution (Enterprise tier) is genuinely more sophisticated.

For most small businesses — especially those already running n8n, Odoo, or other self-hosted tools — Mautic covers 90% of what HubSpot does at 1% of the cost. The remaining 10% can be filled with tools you may already have.

## Getting Started Checklist

1. **Provision a VPS** — 4GB RAM minimum (8GB recommended for 10K+ contacts). Hetzner, OVH, or DigitalOcean all work.
2. **Deploy Mautic with Docker Compose** — use the configuration above.
3. **Set up a reverse proxy** with Caddy or Nginx for HTTPS.
4. **Configure SMTP relay** through Amazon SES or Postal for email deliverability.
5. **Add the Mautic tracking pixel** to your website.
6. **Import your existing contacts** — Mautic supports CSV import and API-based migration from HubSpot, Mailchimp, and others.
7. **Create your first campaign** — start simple: a welcome series for new form submissions.
8. **Set up cron jobs** — Mautic needs scheduled tasks for campaign processing, email sending, and segment updates. The Docker image includes these by default.
9. **Connect Mautic to n8n** for cross-platform automation.
10. **Back up your Mautic database** — add it to your existing backup strategy (see our [disaster recovery guide](/blog/2026/11/17/backing-up-your-self-hosted-stack-a-practical-disaster-recovery-guide/)).

## Wrapping Up

Marketing automation shouldn't cost more than your marketing budget. Mautic gives you enterprise-grade email campaigns, lead scoring, segmentation, and contact management — all self-hosted, all open source, all yours.

The $20,000/year you'd save by switching from HubSpot to Mautic could fund an entire marketing hire. Or a year of content production. Or a complete self-hosted infrastructure stack with n8n, Odoo, Ollama, and a dozen other tools that replace your SaaS subscriptions.

The question isn't whether you can afford to self-host Mautic. It's whether you can afford not to.

---

*Want help setting up Mautic or migrating from HubSpot? ARDOT Consulting specializes in open source automation for small businesses. [Get in touch](/) — we'll help you deploy, configure, and integrate Mautic with your existing tools.*