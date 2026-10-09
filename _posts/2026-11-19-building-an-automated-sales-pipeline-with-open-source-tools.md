---
layout: post
title: "Building an Automated Sales Pipeline with Open Source Tools"
date: 2026-11-19
author: "ARDOT Consulting"
tags: [sales-pipeline, automation, n8n, odoo, ollama, cal-com, crm, open-source]
excerpt: "From lead capture to signed contract, here's how to build a complete automated sales pipeline using Odoo, n8n, Ollama, Cal.com, and Documenso — all open source, all self-hosted, no per-seat SaaS fees."
---

# Building an Automated Sales Pipeline with Open Source Tools

Most sales pipelines are held together with spreadsheets, sticky notes, and the memory of whoever has been at the company longest. Leads come in through a web form, someone manually enters them into a CRM, a sales rep emails them a few days later, a meeting gets scheduled through a back-and-forth email chain, and if the deal closes, a contract gets sent through whatever e-signature tool the company tried last quarter.

It works — sort of. But it leaks leads, creates delays, and means your sales team spends more time on admin work than on actually selling.

The fix isn't another SaaS subscription. It's connecting the open source tools you're already running (or could be running in an afternoon) into a single, automated pipeline that moves leads from first contact to signed deal with minimal manual intervention.

Here's how to build one.

## The Pipeline We're Building

Before we get into tools and configuration, let's map out what an automated sales pipeline actually does. We'll follow a lead from the moment they first interact with your business to the moment they sign a contract.

**Stage 1 — Capture.** A prospect visits your website and fills out a contact form. The form submission triggers an automated workflow.

**Stage 2 — Enrich.** The workflow sends the submission to a local AI model that scores the lead, categorizes it (e.g., "small business inquiry," "enterprise procurement," "spam"), and drafts a personalized response.

**Stage 3 — Route.** The enriched lead lands in your CRM with all the context attached. If it scores above a threshold, the sales team gets an instant notification.

**Stage 4 — Schedule.** The prospect receives an automated email with a booking link. They pick a time that works for them — no back-and-forth.

**Stage 5 — Follow Up.** If the prospect doesn't book within 48 hours, the system sends a gentle reminder. If they book but don't show up, it reschedules automatically.

**Stage 6 — Close.** After the sales call, the system generates a contract from a template, sends it for e-signature, and creates a project record when it's signed.

Six stages, zero spreadsheets. Let's build it.

## The Tool Stack

| Stage | Tool | Role | License |
|-------|------|------|---------|
| Capture | **OpnForm** | Open source form builder | AGPL-3.0 |
| Enrich | **Ollama** | Local LLM for lead scoring & drafting | MIT |
| Route | **Odoo Community** | CRM with pipeline management | LGPL-3.0 |
| Orchestrate | **n8n** | Workflow engine connecting everything | Sustainable Use License* |
| Schedule | **Cal.com** | Self-hosted appointment scheduling | AGPL-3.0 |
| Notify | **Mattermost** | Team chat for sales alerts | MIT |
| Close | **Documenso** | Open source e-signature | AGPL-3.0 |

\* n8n uses the Sustainable Use License — it's source-available but not OSI-certified open source. You can self-host it freely for internal use; you just can't resell it as a hosted product. For our purposes, it works exactly like open source.

If you've been following this blog, you've seen individual posts about most of these tools. This is the post that connects them all.

## Stage 1: Capture Leads with OpnForm

[OpnForm](https://opnform.com) is an open source form builder you can self-host. It's the replacement for Typeform or Google Forms that keeps submissions on your own server.

Deploy it with Docker:

```yaml
# docker-compose.yml (OpnForm)
version: "3.8"
services:
  opnform:
    image: opnform/api:latest
    ports:
      - "8000:80"
    environment:
      - APP_URL=https://forms.yourcompany.com
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16
    environment:
      - POSTGRES_DB=opnform
      - POSTGRES_PASSWORD=strongpassword
    volumes:
      - opnform-db:/var/lib/postgresql/data

  redis:
    image: redis:7

volumes:
  opnform-db:
```

Once deployed, create a form with the fields that matter for lead qualification:

- **Name** (text)
- **Email** (email)
- **Company** (text)
- **What can we help with?** (select: new project, ongoing support, general inquiry)
- **Budget range** (select: under $5k, $5k–$25k, $25k–$100k, $100k+)
- **Tell us more** (long text)

OpnForm generates a webhook URL for each form. Every submission fires a POST request to that URL with the form data as JSON. That's what triggers the rest of the pipeline.

## Stage 2: Enrich Leads with Ollama

This is where AI earns its keep. Instead of a human reading every form submission to figure out if it's worth pursuing, we send it to a local LLM running through Ollama.

[Ollama](https://ollama.com) runs large language models on your own hardware. No data leaves your server — which matters when prospects are sharing project details and budget information.

Install Ollama and pull a model suitable for business tasks:

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.1
```

In n8n, we'll create an HTTP Request node that sends the form submission to Ollama's API with a prompt like this:

```
You are a sales assistant. Analyze the following lead submission and return a JSON object with:
- score: 1-100 (likelihood this is a qualified lead)
- category: one of "qualified", "nurture", "unqualified", "spam"
- summary: one-sentence summary of what the prospect needs
- suggested_response: a 2-3 sentence personalized email reply
- priority: "high", "medium", or "low"

Lead data:
Name: {{$json.name}}
Email: {{$json.email}}
Company: {{$json.company}}
Inquiry type: {{$json.inquiry_type}}
Budget: {{$json.budget}}
Details: {{$json.details}}
```

The n8n node:

```
[HTTP Request]
URL: http://ollama:11434/api/generate
Method: POST
Body:
{
  "model": "llama3.1",
  "prompt": "<the prompt above>",
  "format": "json",
  "stream": false
}
```

Ollama returns structured JSON. n8n parses it and passes it to the next stage. The whole thing takes 3–5 seconds on modest hardware.

**Why this matters:** Every lead gets an instant, intelligent assessment. A high-budget project inquiry from a real company scores 85+ and triggers an immediate alert. A generic "I want to learn more about AI" with no company name scores 20 and goes into a nurture sequence instead of wasting a sales rep's time.

## Stage 3: Route to Odoo CRM

[Odoo Community Edition](https://www.odoo.com) is our CRM. It's open source, self-hosted, and has a proper sales pipeline with stages, probabilities, and activity tracking.

Odoo exposes an XML-RPC API that n8n can call to create lead records. Here's the n8n workflow node:

```
[HTTP Request — Create Lead in Odoo]
URL: https://odoo.yourcompany.com/xmlrpc/2/object
Method: POST
Body:
{
  "jsonrpc": "2.0",
  "method": "execute_kw",
  "params": [
    "your_database",
    2,           // user ID
    "your_api_key",
    "crm.lead",
    "create",
    [{
      "name": "{{$json.name}} — {{$json.company}}",
      "contact_name": "{{$json.name}}",
      "email_from": "{{$json.email}}",
      "description": "AI Summary: {{$json.summary}}\n\nOriginal inquiry: {{$json.details}}",
      "priority": "{{$json.priority === 'high' ? '2' : '1'}}",
      "tag_ids": [[6, 0, [tag_id_for_inquiry_type]]],
      "stage_id": stage_id_for_new
    }]
  ]
}
```

Now every form submission creates a properly categorized, AI-enriched lead in Odoo — with the score, summary, and suggested response stored in the description field.

**Simultaneous alert.** If the lead scores above 70, n8n fires a message to Mattermost:

```
[HTTP Request — Mattermost Alert]
URL: https://chat.yourcompany.com/hooks/your-webhook-id
Method: POST
Body:
{
  "text": "🔥 New qualified lead: {{$json.name}} from {{$json.company}}\nScore: {{$json.score}}/100\nNeed: {{$json.summary}}\nView in Odoo: https://odoo.yourcompany.com/odoo/crm"
}
```

Your sales team sees the alert before they've even finished their coffee.

## Stage 4: Automated Scheduling with Cal.com

Nobody likes scheduling emails. "Are you free Tuesday at 2?" "No, how about Wednesday morning?" "I have a conflict, does Thursday work?" — this exchange wastes more sales time than any other single activity.

[Cal.com](https://cal.com) is an open source scheduling tool (AGPL-3.0) that you self-host. It generates a booking link that shows your real-time availability and lets prospects book themselves.

Deploy with Docker:

```yaml
# docker-compose.yml (Cal.com)
version: "3.8"
services:
  calcom:
    image: calcom/cal.com:latest
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_WEBAPP_URL=https://book.yourcompany.com
      - NEXTAUTH_SECRET=your-secret
      - CALENDSO_ENCRYPTION_KEY=your-encryption-key
      - POSTGRES_USER=calcom
      - POSTGRES_PASSWORD=strongpassword
      - POSTGRES_DB=calcom
      - POSTGRES_HOST=postgres
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    environment:
      - POSTGRES_DB=calcom
      - POSTGRES_USER=calcom
      - POSTGRES_PASSWORD=strongpassword
    volumes:
      - calcom-db:/var/lib/postgresql/data

volumes:
  calcom-db:
```

Create an event type called "Initial Consultation" — 30 minutes, and set your available hours. Cal.com generates a link like `https://book.yourcompany.com/yourname/30min`.

Now, back in n8n, after the lead is created in Odoo, the workflow sends an automated email to the prospect:

```
[Send Email]
Subject: Thanks for reaching out, {{$json.name}} — let's talk
Body:
Hi {{$json.name}},

Thanks for contacting us! Based on what you shared, it sounds like you're looking for help with {{$json.summary}}.

I'd love to learn more about your project. Grab a time that works for you:
https://book.yourcompany.com/sales/30min

Looking forward to it,
The ARDOT Team
```

That email goes out within seconds of the form submission. The prospect can book a meeting immediately — no waiting for a human to respond, no scheduling tennis.

**The AI-suggested response is stored in the Odoo lead record**, so when the sales rep does get on the call, they have context: the original inquiry, the AI's summary, the suggested talking points, and the lead score. They're walking into the conversation prepared.

## Stage 5: Follow-Up Automation

Leads go cold when nobody follows up. This is where automation really pays for itself.

**Scenario A: Prospect doesn't book within 48 hours.**

n8n runs a scheduled workflow every hour that checks Odoo for leads in the "New" stage that are older than 48 hours and have no meeting booked:

```
[Cron Trigger: Every hour]
  → [HTTP Request: Query Odoo for stale leads]
  → [Filter: meeting_booked == false AND age_hours > 48]
  → [Send Email: Gentle reminder with booking link]
  → [HTTP Request: Move lead to "Nurture" stage in Odoo]
```

The reminder email:

> Hi {{$json.name}}, just wanted to make sure my last email didn't get buried. If you're still interested in exploring how we can help with {{$json.summary}}, here's my calendar link: https://book.yourcompany.com/sales/30min. No pressure — happy to answer questions by email too.

**Scenario B: Prospect books but doesn't show up.**

Cal.com fires a webhook when a meeting is marked as "no-show" (available in Cal.com's event settings). n8n catches it:

```
[Webhook: Cal.com no-show event]
  → [Wait 1 hour]
  → [Send Email: Reschedule with apology]
  → [HTTP Request: Add note to Odoo lead]
  → [HTTP Request: Mattermost notification]
```

The reschedule email:

> Hi {{$json.name}}, looks like we missed each other today. No worries — things come up. Here's my booking link to find a new time that works better: https://book.yourcompany.com/sales/30min

**Scenario C: Meeting happens, deal moves forward.**

Cal.com fires a webhook when a meeting is completed. n8n:

1. Updates the Odoo lead stage to "Qualified"
2. Sends a follow-up email with a summary of the meeting (rep can fill this in manually or use an Ollama-generated draft from meeting notes)
3. Notifies Mattermost that the call happened

## Stage 6: Closing the Deal with Documenso

When a deal reaches the "Proposal Sent" stage in Odoo and the prospect says yes, the final step is getting a contract signed.

[Documenso](https://documenso.com) is an open source e-signature platform (AGPL-3.0) that replaces DocuSign. We covered it in detail in a [previous post](/blog/2026/11/11/self-hosting-documenso-replace-docusign-with-your-own-document-signing-platform/), so here we'll focus on how it fits into the pipeline.

The n8n workflow for the final stage:

```
[Manual Trigger: Rep marks deal as "Won" in Odoo]
  → [HTTP Request: Fetch lead details from Odoo]
  → [HTTP Request: Create document in Documenso from template]
      - Template: Master Services Agreement
      - Recipient: {{$json.email}}
      - Variables: name, company, project_scope, start_date, rate
  → [Send Email: "Here's your contract to review and sign"]
  → [Webhook: Documenso "document signed" event]
  → [HTTP Request: Update Odoo lead to "Won"]
  → [HTTP Request: Create project in Odoo]
  → [HTTP Request: Mattermost notification: "🎉 Deal closed!"]
```

The contract is generated from a template, sent automatically, and when the prospect signs, the system closes the loop — updating the CRM, creating a project record, and celebrating in team chat.

## The Complete Pipeline (Diagram)

Here's the full flow, start to finish:

```
[Website Visitor]
       │
       ▼
[OpnForm: Contact Form]
       │ (webhook)
       ▼
[n8n: Workflow Engine]
       │
       ├──→ [Ollama: Score, categorize, draft response]
       │         │
       │         ▼ (enriched data)
       ├──→ [Odoo: Create lead in CRM]
       │         │
       │         ├──→ [Mattermost: Alert if score > 70]
       │         │
       │         └──→ [Email: Auto-reply with Cal.com link]
       │
       ├──→ [Cal.com: Prospect books meeting]
       │         │
       │         ├──→ (booked) → [Odoo: Update stage to "Meeting Scheduled"]
       │         │
       │         └── (no-show) → [Email: Reschedule] → [Odoo: Add note]
       │
       ├──→ [Cron: 48hr follow-up for unbooked leads]
       │         │
       │         └──→ [Email: Reminder with booking link]
       │
       └──→ [Documenso: Contract sent when deal is won]
                 │
                 ├──→ (signed) → [Odoo: Mark won, create project]
                 │
                 └──→ [Mattermost: Celebration message]
```

## What This Costs

Let's talk about the elephant in the room: cost. Here's the comparison.

| Component | Open Source (Self-Hosted) | SaaS Equivalent | Monthly SaaS Cost |
|-----------|--------------------------|-----------------|-------------------|
| Form builder | OpnForm | Typeform | $25–$70/mo |
| AI lead scoring | Ollama (local) | OpenAI API | $50–$200/mo |
| CRM | Odoo Community | HubSpot Starter | $20–$90/seat/mo |
| Workflow automation | n8n | Zapier | $20–$100/mo |
| Scheduling | Cal.com | Calendly | $10–$16/seat/mo |
| E-signature | Documenso | DocuSign | $10–$45/seat/mo |
| Team notifications | Mattermost | Slack | $7–$12/seat/mo |

For a 5-person sales team, the SaaS stack runs **$400–$800/month** depending on tiers and add-ons. The self-hosted stack runs on a single VPS that costs **$20–$40/month**.

The trade-off is setup time. You can sign up for HubSpot in 10 minutes. Building this pipeline takes a few days of focused work — or a call to someone who's done it before (that's us).

## What This Looks Like in Practice

Let's walk through a realistic scenario.

**Monday, 9:07 AM.** Sarah, operations manager at a 40-person logistics company, visits your website and fills out the contact form. She writes: "We're spending 15 hours/week manually matching drivers to shipments. Looking for help automating this."

**9:07 AM + 4 seconds.** n8n receives the webhook, sends the data to Ollama.

**9:07 AM + 8 seconds.** Ollama returns: Score 82/100, Category: "qualified," Summary: "Mid-size logistics company needs driver-shipment matching automation," Suggested response: "Hi Sarah, thanks for reaching out. Automating driver-shipment matching is exactly what we do — we'd love to understand your current workflow and show you what's possible."

**9:07 AM + 10 seconds.** Odoo lead created with all enrichment data. Mattermost fires: "🔥 New qualified lead: Sarah from [logistics company]. Score: 82/100."

**9:07 AM + 12 seconds.** Sarah receives an email with the suggested response and a Cal.com booking link.

**9:15 AM.** Sarah books a 30-minute consultation for Wednesday at 10 AM. Cal.com sends a webhook to n8n, which updates the Odoo lead stage and sends Sarah a confirmation email.

**Wednesday, 10:00 AM.** Your sales rep opens the Odoo lead record. They see Sarah's original inquiry, the AI summary, the lead score, the meeting details. They walk into the call prepared.

**Wednesday, 10:30 AM.** Great call. Sarah wants to move forward. The rep marks the deal as "Proposal Sent" in Odoo.

**Friday.** Sarah approves the proposal. The rep marks the deal as "Won" in Odoo. n8n generates the contract from a Documenso template and emails it to Sarah.

**Friday, 2:00 PM.** Sarah signs. n8n updates Odoo, creates a project record, and posts in Mattermost: "🎉 Deal closed! New project: Logistics automation for [company]."

Total manual work: one sales call and one proposal review. Everything else ran automatically.

## Getting Started: A Phased Approach

Don't try to build all six stages at once. Here's a phased rollout that delivers value at each step:

**Phase 1 (Week 1): Capture + Route.** Deploy OpnForm and Odoo. Wire the form webhook to n8n, which creates leads in Odoo. You now have every lead captured in a proper CRM with zero manual data entry. **Value: no more lost leads.**

**Phase 2 (Week 2): Enrich.** Add Ollama to the workflow for lead scoring and categorization. Add Mattermost alerts for high-score leads. **Value: sales team focuses on the right leads, faster.**

**Phase 3 (Week 3): Schedule.** Deploy Cal.com and add the booking link to automated response emails. Set up the 48-hour follow-up cron. **Value: meetings get booked without email tennis, no lead goes cold.**

**Phase 4 (Week 4): Close.** Deploy Documenso and wire the contract generation workflow. **Value: deals close faster, no manual contract preparation.**

Each phase takes a few hours to set up and delivers immediate, measurable improvement. By the end of week four, you have a fully automated sales pipeline running entirely on your own infrastructure.

## When This Might Not Be Right for You

Let's be honest about limitations:

- **You need this running by Friday.** This is a build, not a sign-up. If you need a CRM + scheduling + e-signature workflow live in 48 hours, use the SaaS equivalents and migrate later.
- **You don't have anyone to maintain it.** Self-hosted tools need updates, backups, and occasional troubleshooting. If you don't have a technical person (or a consulting partner), factor that in.
- **Your sales team is one person.** If you're a solo founder, a fully automated pipeline might be overkill. Odoo CRM + Cal.com without the AI enrichment and automation layer will get you most of the way there.
- **You need deep Salesforce integration.** If your enterprise customers require Salesforce-native processes, Odoo Community won't replace that. This stack is built for small and mid-sized businesses.

## Wrapping Up

The tools to build a complete, automated sales pipeline — form capture, AI enrichment, CRM, scheduling, follow-up, e-signature — are all available as open source. They're mature, well-documented, and actively maintained. The only question is whether you want to spend a few days setting them up or a few thousand dollars per year subscribing to their SaaS equivalents.

If you're tired of leads falling through the cracks and sales reps doing data entry instead of selling, this pipeline is the answer. And if you'd rather have someone build it for you — well, that's literally what we do.

**Ready to automate your sales pipeline?** [Get in touch](/) — we'll help you deploy, configure, and connect these tools on your own infrastructure. No per-seat fees, no vendor lock-in, no data leaving your server.