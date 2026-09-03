---
layout: post
title: "Cal.com: Open-Source Scheduling That Keeps Your Data Yours"
date: 2026-09-20
author: "ARDOT Consulting"
tags: [cal-com, scheduling, open-source, self-hosting, automation, privacy]
excerpt: "A practical guide to self-hosting Cal.com for appointment scheduling — with Docker setup, feature comparison against Calendly, and integration tips for n8n and Ollama."
---

Every business that takes client meetings knows the scheduling dance. Three emails to find a time that works. A fourth to confirm. A fifth when someone needs to reschedule. It's not hard work, but it's relentless—and it's exactly the kind of friction that makes potential clients disengage before you ever get to speak with them.

Scheduling tools like Calendly solved this problem. You send a link, the client picks a slot, it's on everyone's calendar. But there's a catch: every meeting you book, every attendee email, every note about your availability lives on someone else's servers. For solo consultants that's a reasonable trade-off. For businesses handling sensitive client data—law firms, financial advisors, healthcare practices—it's worth asking whether that data should be sitting in a third-party SaaS database at all.

Enter [Cal.com](https://github.com/calcom/cal.com). It's the open-source scheduling platform that gives you everything Calendly does, with one critical difference: you can run it on your own infrastructure. Your scheduling data stays on your server, under your control, behind your firewall.

This guide walks through what Cal.com does, how it compares to Calendly, and how to self-host it with Docker. We'll also cover how to connect it to n8n for automation and why that combination is more powerful than any standalone scheduling tool.

## What Cal.com Actually Does

Cal.com is scheduling infrastructure. At its core, it lets you create event types—30-minute consultations, 60-minute discovery calls, group workshops—and share booking links with clients. The client sees your real-time availability, picks a time, and the event lands on both calendars automatically.

But that's table stakes. What makes Cal.com interesting for businesses with automation goals is what happens around the booking:

- **Routing forms**: Instead of a flat list of time slots, you can present a form that asks qualifying questions first. "What's your company size?" "Are you a new or existing client?" Based on the answers, Cal.com routes to different team members or event types. This is workflow logic built into the scheduling layer.

- **Workflow automations**: Cal.com can trigger actions when bookings are created, rescheduled, or cancelled. Send a confirmation email, fire a Slack notification, generate a video meeting link. These built-in automations cover the common cases without needing an external tool.

- **Apps and integrations**: Cal.com connects to calendar services (via CalDAV, Microsoft Graph, or Google Calendar API), video conferencing tools, payment processors (Stripe), and CRM systems. The app ecosystem is built into the platform, so you configure integrations through the UI rather than coding them.

- **API and webhooks**: Every booking event fires a webhook. This is where the real automation potential lives—you can point those webhooks at n8n and build custom workflows that respond to scheduling events in real time.

- **Team scheduling**: Round-robin assignment, shared event types, collective availability across team members. If you have three consultants who can all handle intake calls, Cal.com distributes bookings evenly instead of letting one person's calendar fill up while others sit empty.

## Cal.com vs Calendly: An Honest Comparison

Let's be direct about the trade-offs. Calendly is a polished, hosted product. It works out of the box, requires no infrastructure, and has a generous free tier. For many small businesses, it's the right choice. Cal.com isn't automatically better just because it's open source—it's better *for specific situations*.

| Feature | Calendly (SaaS) | Cal.com (Self-Hosted) |
|---------|-----------------|----------------------|
| Data location | Calendly's servers | Your server |
| Cost (entry) | Free tier, $10–24/user/mo for paid | Free (self-hosted), infra costs only |
| Setup time | 5 minutes | 30–60 minutes (Docker deploy) |
| Maintenance | None (vendor handles) | You handle updates, backups |
| Customization | Limited to provided features | Full source code access |
| Privacy/control | Vendor has your booking data | You own all data |
| API access | Paid plans only | Free, full API |
| White-label | Enterprise only | Built-in, free |
| License | Proprietary | MIT (cal.com) + AGPL (platform) |

**When to choose Calendly**: You have fewer than 5 users, no sensitive data concerns, no in-house technical capacity, and you just want scheduling to work with zero maintenance.

**When to choose Cal.com**: You have client data you'd rather not hand to a third party, you want full API access without paying for a higher tier, you need white-label scheduling on your own domain, or you're already running other self-hosted infrastructure (n8n, Odoo, Ollama) and want scheduling in the same ecosystem.

The privacy argument matters more than it might seem. When a client books a meeting, they're providing their name, email, phone number, company, and sometimes details about their legal matter or financial situation. That data has real value—and real risk. Self-hosting means it stays in your database, protected by your security measures, not sitting in a SaaS vendor's system that could be breached, acquired, or change its data practices.

## Self-Hosting Cal.com with Docker

Cal.com provides an official Docker image and a `docker-compose` setup that handles the database, the web app, and the API. Here's how to get it running.

### Prerequisites

- A server with Docker and Docker Compose installed (2 GB RAM minimum, 4 GB recommended)
- A domain name with DNS pointing to your server (e.g., `book.yourcompany.com`)
- An SSL certificate (Let's Encrypt via Caddy or Traefik works well)

### Step 1: Clone and Configure

```bash
git clone https://github.com/calcom/cal.com.git
cd cal.com
```

Cal.com uses environment variables for configuration. Copy the example file and edit it:

```bash
cp .env.example .env
```

The key variables to set:

```env
# Database
DATABASE_URL=postgresql://cal:yourpassword@db:5432/cal

# Core
NEXT_PUBLIC_WEBAPP_URL=https://book.yourcompany.com
NEXTAUTH_SECRET=your-random-secret-string
CALENDAR_ENCRYPTION_KEY=your-32-byte-encryption-key

# Email (for booking notifications)
EMAIL_FROM=noreply@yourcompany.com
SMTP_HOST=mail.yourcompany.com
SMTP_PORT=587
SMTP_USER=your-smtp-user
SMTP_PASSWORD=your-smtp-password
```

The `CALENDAR_ENCRYPTION_KEY` is important—Cal.com encrypts calendar credentials at rest using this key. Generate one with:

```bash
openssl rand -hex 32
```

### Step 2: Docker Compose

Cal.com includes a Dockerfile. Here's a working `docker-compose.yml` you can adapt:

```yaml
version: "3.8"

services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: cal
      POSTGRES_PASSWORD: yourpassword
      POSTGRES_DB: cal
    volumes:
      - cal-db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U cal"]
      interval: 5s
      timeout: 5s
      retries: 5

  cal:
    image: calcom/cal.com:latest
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://cal:yourpassword@db:5432/cal
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      CALENDAR_ENCRYPTION_KEY: ${CALENDAR_ENCRYPTION_KEY}
      NEXT_PUBLIC_WEBAPP_URL: https://book.yourcompany.com
      EMAIL_FROM: noreply@yourcompany.com
      SMTP_HOST: mail.yourcompany.com
      SMTP_PORT: 587
    ports:
      - "3000:3000"

volumes:
  cal-db:
```

### Step 3: Start and Initialize

```bash
docker compose up -d
docker compose exec cal pnpm db-migrate
docker compose exec cal pnpm db-seed
```

The seed command creates an initial admin user. Navigate to `http://your-server:3000` and log in to complete setup.

### Step 4: Reverse Proxy with SSL

Put Caddy or Traefik in front of Cal.com for automatic HTTPS. A minimal Caddyfile:

```
book.yourcompany.com {
    reverse_proxy localhost:3000
}
```

Caddy handles Let's Encrypt certificates automatically. No manual certbot, no renewal scripts.

## Connecting Cal.com to n8n

This is where scheduling becomes automation. Cal.com fires webhooks for every booking event. You can point those webhooks at n8n and trigger complex workflows that go far beyond what any scheduling tool offers out of the box.

### Setting Up the Webhook

In Cal.com's admin panel:

1. Go to **Apps** → **Webhooks**
2. Add a new webhook pointing to your n8n webhook URL: `https://n8n.yourcompany.com/webhook/cal-booking`
3. Select the events you want: `BOOKING_CREATED`, `BOOKING_CANCELLED`, `BOOKING_RESCHEDULED`

### The n8n Workflow

In n8n, create a workflow that starts with a Webhook node set to POST. Here's what a typical booking workflow looks like:

```
[Webhook: Cal.com booking] 
    → [Function: Parse booking payload]
    → [Ollama: Generate personalized prep note]
    → [Slack/Mattermost: Notify assigned team member]
    → [Odoo: Create or update CRM lead]
    → [Email: Send confirmation to client]
```

The payload Cal.com sends includes the event type, attendee details, booking time, and any custom answers from routing forms. Here's a simplified example:

```json
{
  "triggerEvent": "BOOKING_CREATED",
  "payload": {
    "title": "30 Minute Consultation",
    "startTime": "2026-09-22T14:00:00Z",
    "endTime": "2026-09-22T14:30:00Z",
    "attendees": [{
      "email": "client@example.com",
      "name": "Jane Smith",
      "timeZone": "America/New_York"
    }],
    "responses": {
      "company": "Acme Corp",
      "companySize": "11-50"
    }
  }
}
```

### Using Ollama for Prep Notes

The most valuable automation here is the prep note. When a booking comes in, you can feed the client's name, company, and routing form answers to a local LLM running on Ollama to generate a brief preparation summary:

```python
# n8n Code node — builds the prompt for Ollama
const attendee = $json.payload.attendees[0];
const responses = $json.payload.responses;

const prompt = `You are preparing a consultant for a 30-minute client call. 
Based on the booking details, write a brief 3-bullet preparation note.

Client: ${attendee.name}
Company: ${responses.company}
Company size: ${responses.companySize}
Meeting type: ${$json.payload.title}

Focus on what the consultant should research, what questions to prepare, 
and what likely pain points a company of this size might have.`;

return { prompt };
```

Feed that into an Ollama HTTP Request node (`http://ollama:11434/api/generate`, model `llama3.1`) and you get a tailored prep note in your team chat before the meeting starts. No one had to write it. No data left your server.

## Practical Use Cases

### Professional Services Firms

A law firm uses Cal.com's routing forms to direct potential clients to the right attorney. "Is this about personal injury or corporate law?" routes to different team members. The webhook fires into n8n, which creates a matter in Odoo, generates a conflict check request, and sends the client an intake form via DocuSeal. The attorney gets a Mattermost notification with a summary generated by Ollama—all before they even open their calendar.

### Healthcare Practices

A small clinic runs Cal.com on an internal server. Patients book appointments through a link on the clinic's website. The booking data never leaves the clinic's network. n8n sends automated reminders 24 hours before the appointment, and cancellations automatically free up the slot and notify the waitlist. Because everything is self-hosted, there's no third-party vendor with access to patient scheduling data.

### Consulting Businesses

A 12-person consulting firm uses Cal.com's round-robin feature to distribute discovery calls evenly across the team. Each booking triggers a workflow that creates a project in Odoo, sets up a shared note in their knowledge base, and posts a brief in the relevant Mattermost channel. The consultant walking into the call already knows the client's company, industry, and likely concerns.

## What About Maintenance?

Self-hosting means you handle updates. Cal.com releases new versions regularly—the project has over 48,000 GitHub stars and an active contributor base, so releases are frequent and well-tested.

The update process is straightforward:

```bash
cd cal.com
git pull
docker compose pull cal
docker compose up -d
docker compose exec cal pnpm db-migrate
```

That's it. Docker handles the rest. The database volume persists across updates, so no data is lost.

For backups, the key component is the PostgreSQL database. A nightly `pg_dump` to an offsite location covers your scheduling data. Calendar credentials are encrypted with the `CALENDAR_ENCRYPTION_KEY`, so back that up somewhere safe—if you lose it, you'll need to re-authorize all calendar connections.

## The Bottom Line

Scheduling is one of those operational tasks that seems too simple to automate until you realize how much time it consumes across a team. Cal.com handles the scheduling part. The webhook integration with n8n handles everything that happens after the booking—CRM updates, team notifications, AI-generated prep notes, automated reminders.

The advantage of doing this with open source tools isn't just cost savings (though that's real—no per-user SaaS fees). It's data sovereignty. Every client name, every booking detail, every routing form answer lives in your database. You control who sees it, where it's stored, and how long it's kept. For businesses in regulated industries, that's not a nice-to-have. It's the requirement.

If you're already running n8n and Ollama, adding Cal.com to your self-hosted stack is a natural next step. If you're starting fresh, it's one of the highest-ROI tools you can deploy—scheduling friction disappears, and the automation possibilities compound from day one.

---

*Want help setting up Cal.com, n8n, and Ollama for your business? [Get in touch](/) — we design and deploy self-hosted automation systems that keep your data where it belongs.*