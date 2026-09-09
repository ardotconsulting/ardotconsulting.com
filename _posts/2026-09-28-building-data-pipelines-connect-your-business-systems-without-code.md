---
layout: post
title: "Building Data Pipelines: Connect Your Business Systems Without Writing Code"
date: 2026-09-28
author: "ARDOT Consulting"
tags: [data-pipelines, integration, n8n, etl, open-source, automation, data-flow, tutorial]
excerpt: "Your business data lives in five different tools that don't talk to each other. Here's how to build automated data pipelines that move information between your systems — no code, no SaaS connectors, no manual exports."
---

Most small businesses run on a patchwork of disconnected tools. Your CRM holds customer contacts. Your invoicing system holds payment records. Your project management tool tracks deliverables. Your email platform stores communication history. Your website analytics tool tracks visitor behavior. Each tool is decent at its job, but none of them talk to each other.

The result is a daily ritual of manual data movement. You export a CSV from one system, clean it up in a spreadsheet, import it into another, and hope nothing changed in between. Your team spends hours every week acting as human data pipelines — copying, pasting, reformatting, and reconciling information between systems that should have been connected from the start.

This isn't a technology problem. It's a plumbing problem. And like most plumbing problems, the fix isn't glamorous, but it changes everything once it works. This post is about building data pipelines with open source tools that connect your business systems automatically — no code, no per-connector SaaS fees, no data flowing through third-party clouds you don't control.

## What a Data Pipeline Actually Does

A data pipeline is a series of automated steps that move data from one place to another, optionally transforming it along the way. That's it. The term sounds technical, but the concept is simple:

1. **Extract** — Pull data from a source (a CRM, a database, a spreadsheet, an API)
2. **Transform** — Change the data's format, filter it, enrich it, or combine it with other data
3. **Load** — Put the data somewhere useful (another database, a dashboard, a report, a notification)

You might see this called ETL (Extract, Transform, Load) in enterprise contexts. The business version is simpler: "when something happens in System A, make sure System B knows about it."

Here are some concrete examples of what this looks like in practice:

- **New customer in CRM → invoice created in accounting** — When a lead converts to a customer in your CRM, a draft invoice is automatically created in your accounting system with the correct billing details.
- **Order placed on website → inventory updated in ERP** — When an order comes through your e-commerce site, stock levels are decremented in your inventory management system and a reorder alert fires if stock drops below threshold.
- **Support ticket closed → metrics updated on dashboard** — When a support ticket is resolved, the resolution time is logged to a metrics database and a weekly performance dashboard updates automatically.
- **Form submission → lead scored and routed** — When someone fills out a contact form, their data is enriched with company information, scored against your ideal customer profile, and routed to the right salesperson.

Each of these pipelines replaces a manual process. Each one eliminates a class of errors — the wrong customer ID, the missed reorder, the forgotten follow-up. And each one can be built without writing a single line of code.

## The Tool Stack

For data pipelines in a small business context, you need three components:

| Component | What It Does | Open Source Option |
|-----------|-------------|-------------------|
| **Orchestrator** | Runs the pipeline on a schedule or trigger, handles retries and error logging | **n8n** |
| **Data Store** | Holds the data at rest between steps, serves as a source or destination | **PostgreSQL** or **Directus** |
| **Transformation** | Cleans, filters, enriches, or reformats data as it moves | **n8n nodes** (for simple) or **Ollama** (for AI-powered enrichment) |

n8n is the backbone here. We've covered it extensively in previous posts, but the short version: it's a self-hosted workflow automation tool with a visual builder. You drag nodes onto a canvas, connect them, and configure each one. It has built-in connectors for hundreds of tools — CRMs, accounting software, databases, email platforms, messaging apps — and supports custom HTTP requests for anything it doesn't cover natively.

The key advantage over SaaS alternatives like Zapier is that n8n runs on your own server. There are no per-task fees (Zapier charges $0.01–$0.06 per task on paid plans, which adds up fast for high-volume pipelines). There are no limits on the number of active workflows. And your data flows through infrastructure you control, not through a third-party cloud that could change its pricing, terms, or data handling policies at any time.

## A Real Pipeline: CRM to Accounting

Let's build a real pipeline step by step. This is one of the most common integration needs: syncing customer data from a CRM to an accounting system so that invoices are created with correct billing information. We'll use Odoo (self-hosted CRM) as the source and a generic accounting API as the destination, but the pattern works with any pair of systems.

### Step 1: Set Up the Trigger

In n8n, create a new workflow. The first node is the trigger — the event that starts the pipeline. For a CRM-to-accounting sync, you want the pipeline to fire whenever a new customer is created:

```
Trigger: Webhook from Odoo
- Listen for: contact.created event
- Authentication: API key (stored in n8n credentials)
- Output: JSON with customer name, email, address, tax ID
```

Odoo's webhook system sends a POST request to your n8n instance whenever a new contact is created. n8n receives the payload and passes it to the next node.

If your source system doesn't support webhooks, you can use a scheduled trigger instead — n8n checks for new records every 15 minutes and processes any that haven't been synced yet. The tradeoff: webhooks are real-time but require the source system to support them; polling is slower but works with any system that has an API.

### Step 2: Transform the Data

The customer record from Odoo probably doesn't match the format your accounting system expects. Field names differ, address components are structured differently, and some fields need to be calculated. This is the transform step.

In n8n, add a **Set** node (for field mapping) and a **Function** node (for any logic):

```
Set Node: Map Fields
- company_name → customer_name
- email → billing_email
- street + city + zip → billing_address (combined)
- vat_number → tax_id

Function Node: Validate
- Check that customer_name is not empty
- Check that billing_email is a valid format
- Check that tax_id matches expected pattern
- If any check fails, route to error handling node
```

This is where most pipelines spend their complexity. Data transformation is unglamorous but critical — garbage in, garbage out applies at every stage. The validation step is especially important: if a customer record is missing a required field, you want to catch it here and alert a human, not create a broken invoice downstream.

### Step 3: Check for Duplicates

Before creating a new record in the accounting system, check whether one already exists. This prevents duplicate customers and invoices — a common problem when pipelines run on schedules and process the same records multiple times:

```
HTTP Request Node: Search Existing Customer
- Method: GET
- URL: https://accounting.example.com/api/customers?email={billing_email}
- If response contains a customer → skip creation, optionally update
- If response is empty → proceed to creation
```

### Step 4: Load the Data

Now create the customer record in the accounting system:

```
HTTP Request Node: Create Customer
- Method: POST
- URL: https://accounting.example.com/api/customers
- Body: {
    "customer_name": "{{customer_name}}",
    "billing_email": "{{billing_email}}",
    "billing_address": "{{billing_address}}",
    "tax_id": "{{tax_id}}"
  }
- Headers: Authorization: Bearer {api_key}
```

### Step 5: Log and Notify

The final step is logging and notification. You want a record of what the pipeline did, and you want to know if something went wrong:

```
PostgreSQL Node: Log Sync
- INSERT INTO sync_log (source, destination, record_id, status, timestamp)
- VALUES ('odoo', 'accounting', '{{customer_id}}', 'success', NOW())

IF Node: Check for Errors
- If error → Mattermost notification to #ops channel
- If success → no notification (silent on success)
```

The full pipeline looks like this:

```
[Odoo Webhook] → [Map Fields] → [Validate] → [Search Existing] → [Create Customer] → [Log Sync]
                                                                         ↓
                                                                  [Error Handler] → [Mattermost Alert]
```

That's a complete data pipeline. It runs automatically, retries on failure, logs every action, and alerts a human only when something goes wrong. Once it's built, you never think about it again — which is the whole point.

## When to Add AI to the Pipeline

Not every pipeline needs AI. The CRM-to-accounting sync above is pure rule-based logic — no AI required. But some transformations benefit from intelligence:

- **Categorizing free-text data** — A support ticket's subject line needs to be mapped to a product category. Rules can't handle the variety; an LLM can.
- **Enriching incomplete records** — A customer gives you a company name but no industry code. An LLM can infer the industry from the name and public data.
- **Sentiment analysis** — You want to route customer feedback to different teams based on whether it's positive, negative, or neutral.
- **Entity extraction** — You need to pull names, dates, and amounts from unstructured email bodies and turn them into structured fields.

For these cases, add an **Ollama** node to the pipeline. Ollama runs a local LLM on your server, processes the data in memory, and returns structured output — without sending anything to an external API:

```
Ollama Node: Categorize Ticket
- Model: qwen2.5:7b
- Prompt: "Categorize this support ticket into one of: Billing, Technical, 
  General Inquiry, Complaint. Ticket subject: {{subject}}. 
  Respond with only the category name."
- Output: category field added to the record
```

The key principle: use AI where judgment is needed, use rules where logic is sufficient. Most pipelines are 90% rules and 10% AI. Don't reach for an LLM when a simple if-statement will do.

## Pipeline Patterns That Work

Over many implementations, a few patterns consistently deliver value. Here are the ones worth building first:

### The Sync Pipeline

Keep two systems in sync so that a change in one propagates to the other. This is the most common pattern. Example: customer contact details updated in CRM → updated in accounting → updated in mailing list. The pipeline runs on a webhook trigger (real-time) or a schedule (every 15 minutes).

### The Aggregation Pipeline

Pull data from multiple sources, combine it, and load it into a single dashboard or report. Example: daily sales from your e-commerce platform, daily expenses from your accounting system, and daily website traffic from your analytics tool → combined into a single daily business summary delivered to your inbox every morning at 7 AM.

### The Alerting Pipeline

Monitor a data source for specific conditions and notify a human when they're met. Example: inventory levels checked every hour → if any product drops below reorder threshold → notification sent to purchasing team with a draft purchase order. The pipeline doesn't replace human decision-making; it surfaces the information that needs a decision.

### The Onboarding Pipeline

When a new customer, employee, or project is created, trigger a series of steps across multiple systems. Example: new employee in HR system → account created in email platform → access granted to project management tool → welcome email sent → equipment request submitted to IT. One trigger, many downstream actions.

## Common Pitfalls

A few things tend to go wrong when businesses first start building data pipelines:

**Not handling failures gracefully.** A pipeline that fails silently is worse than no pipeline at all — you think the data is synced, but it isn't. Every pipeline needs error handling: retry logic, logging, and human notification on persistent failure. n8n handles retries natively, but you need to configure the notification step yourself.

**Over-transforming data.** It's tempting to clean, enrich, and restructure data at every step. But each transformation adds complexity and potential failure points. Transform only what's necessary to make the data usable at the destination. If the destination system can handle the raw data, let it.

**Ignoring data volume.** A pipeline that processes 10 records a day works fine with a simple sequential flow. A pipeline that processes 10,000 records a day needs batching, parallel processing, and rate limiting. Build for your current volume, but know where the breaking point is.

**No monitoring.** Pipelines degrade over time. APIs change, data formats drift, rate limits shift. Set up a simple dashboard that shows pipeline run counts, success rates, and processing times. If a pipeline that usually runs 50 times a day suddenly runs 3, you want to know immediately.

## The Cost Question

A self-hosted data pipeline stack costs roughly:

| Component | Monthly Cost |
|-----------|-------------|
| VPS (8GB RAM, runs n8n + Ollama + PostgreSQL) | $10–$20 |
| Odoo Community Edition (self-hosted) | $0 (open source) |
| n8n (self-hosted) | $0 (open source) |
| PostgreSQL (self-hosted) | $0 (open source) |
| **Total** | **$10–$20/month** |

The equivalent SaaS setup — Zapier for workflow automation, a cloud CRM, a cloud accounting tool with API access — typically runs $200–$500/month for a small team, with per-task fees that scale with usage. The open source stack costs a fraction of that, runs on infrastructure you control, and has no per-task limits.

## Where to Start

Don't try to connect everything at once. Pick the single most painful manual data transfer in your business — the one that someone does every day, takes 20+ minutes, and is error-prone. Build that pipeline first. Get it running reliably for two weeks. Then pick the next one.

The first pipeline is the hardest because you're learning the tools. The second is easier. By the fifth, you have patterns to copy and the whole process takes an afternoon instead of a week. This is how automation compounds — not in one dramatic transformation, but in a series of pipelines that each remove a small piece of manual work until one day you realize your team hasn't exported a CSV in months.

---

*Your business data shouldn't require human beings to copy and paste it between systems. ARDOT Consulting designs and builds self-hosted data pipelines using open-source tools — connecting your CRM, accounting, inventory, and analytics without per-task fees or vendor lock-in. [Get in touch](/#contact) and we'll map out the pipelines that would save your team the most time.*