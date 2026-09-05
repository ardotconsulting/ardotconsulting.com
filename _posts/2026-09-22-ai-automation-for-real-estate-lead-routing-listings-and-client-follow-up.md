---
layout: post
title: "AI Automation for Real Estate: Lead Routing, Listings, and Client Follow-Up"
date: 2026-09-22
author: "ARDOT Consulting"
tags: [real-estate, automation, lead-routing, listings, open-source, n8n, ollama]
excerpt: "How small real estate brokerages can automate lead routing, listing updates, and client follow-up using open-source tools like n8n, Ollama, and Odoo — without sending sensitive client data to third-party SaaS."
---

Real estate runs on speed and relationships. The agent who responds to a new lead first usually wins the listing. The brokerage that keeps sellers updated weekly keeps clients for life. But the work that makes those relationships happen — routing leads to the right agent, pulling comparables for a listing, sending follow-up emails to thirty prospects — is repetitive, time-sensitive, and exactly the kind of thing that breaks down when someone goes on vacation or gets busy with a closing.

Automation can handle a surprising amount of it. And unlike the proprietary CRM platforms that dominate real estate (and charge per seat, per lead, per listing), open-source tools let you build the same workflows on your own infrastructure, with your data staying on your server. This guide walks through three concrete automation use cases for a small real estate brokerage: lead routing, listing management, and client follow-up. We'll use n8n for workflow orchestration, Ollama for local AI processing, and Odoo for CRM — all open source, all self-hosted.

## Why Real Estate Needs Automation

Before we get into the how, let's be honest about where automation helps and where it doesn't. Real estate has a specific workflow shape:

- **High lead volume, low conversion.** A busy Zillow or Realtor.com feed sends dozens of leads per week. Most go nowhere. Sorting the serious from the curious is a filtering problem.
- **Time-sensitive first contact.** Studies consistently show that responding to a lead within the first five minutes dramatically increases conversion. Human agents can't always do that. Automation can.
- **Repetitive listing maintenance.** Every listing needs photos, descriptions, MLS updates, price adjustments, and status changes. Each one is small, but they add up.
- **Long client lifecycles.** A buyer might search for six months before making an offer. Keeping in touch over that period — without being annoying — is a nurturing workflow.

The tasks that benefit most from automation are the ones where speed matters, repetition is high, and the decision logic is predictable. Tasks that require physical presence (showings, inspections), negotiation skill, or deep local knowledge stay with the agent. Automation handles the plumbing; the agent handles the relationship.

## Tool Stack: Open Source, Self-Hosted

Here's what we're working with:

| Tool | Role | Why It Fits Real Estate |
|------|------|-------------------------|
| **n8n** | Workflow orchestration | Connects lead sources, CRM, email, and AI into automated pipelines |
| **Ollama** | Local AI inference | Summarizes leads, drafts listing descriptions, scores inquiries — without sending client data to OpenAI |
| **Odoo (Community)** | CRM | Tracks leads, contacts, properties, and interactions in one self-hosted system |
| **Postfix or Mailtrap** | Email delivery | Sends automated follow-ups from your own domain |

The critical point: none of this sends your client data to a third-party AI API. When a lead submits their phone number and budget, that information stays on your server. For an industry handling financial details, credit info, and personal addresses, that's not a niche concern — it's a fiduciary one.

## Use Case 1: Automated Lead Routing

Here's the scenario: a new lead comes in from your website's contact form, a property portal, or a paid ad. The clock starts ticking. In most brokerages, that lead sits in a general inbox until someone manually forwards it to an agent — if it gets forwarded at all.

### The Workflow

We can build an n8n workflow that handles this end to end:

1. **Trigger**: A webhook receives the lead data (name, email, phone, property interest, budget range, source).
2. **AI scoring**: Ollama analyzes the lead — is this a hot prospect (specific property, ready timeline, pre-approved) or a cold browser (vague interest, no timeline)? A simple prompt classifies the lead as "hot," "warm," or "cold."
3. **Routing logic**: Based on the score and the property type, n8n assigns the lead to the appropriate agent. Luxury listings go to your senior agent. Rentals go to the junior team. Commercial goes to the specialist.
4. **Instant notification**: The assigned agent gets a Slack or SMS notification within seconds.
5. **CRM entry**: The lead is automatically created in Odoo with all details, source tracking, and the AI score logged.
6. **Auto-response**: The lead receives an immediate acknowledgment email — not a generic "we'll get back to you," but a personalized message referencing the property they inquired about.

Here's what the n8n workflow looks like in structure:

```
[Webhook: New Lead]
    → [Ollama: Score Lead]
    → [Switch: Route by Score + Property Type]
        → [Hot + Residential → Agent A]
        → [Hot + Commercial → Agent B]
        → [Warm → Round Robin Pool]
        → [Cold → Nurture Campaign]
    → [Odoo: Create Lead Record]
    → [Email: Send Personalized Auto-Response]
    → [Slack: Notify Assigned Agent]
```

### The Ollama Lead Scoring Prompt

The AI scoring step is where local inference shines. You're sending the lead's inquiry text to Ollama running on your own server — not to an external API. Here's a working prompt:

```
You are a real estate lead scoring assistant. Analyze this inquiry
and classify it as HOT, WARM, or COLD based on:

- Timeline urgency (moving in 30 days = hot; browsing = cold)
- Financial readiness (pre-approved, specific budget = hot;
  "what's the price range" = warm; no budget mention = cold)
- Property specificity (named a specific listing = hot;
  general area interest = warm; "just looking" = cold)

Lead inquiry: {{ $json.message }}
Lead source: {{ $json.source }}
Property interest: {{ $json.property }}

Respond in JSON format:
{
  "score": "HOT|WARM|COLD",
  "reasoning": "one sentence explanation",
  "suggested_response_tone": "urgent|helpful|casual"
}
```

Running this through a model like Llama 3 (8B) on Ollama takes about 2–3 seconds on modest hardware and costs nothing per inference. Compare that to paying per-token API fees for hundreds of leads per week.

### Why This Beats Manual Routing

The math is straightforward. A brokerage receiving 50 leads per week, where each manual routing decision takes 5 minutes of admin time, spends over 4 hours per week just sorting leads. The automated workflow does it in seconds, routes more consistently (no bias toward the agent who happens to be sitting near the inbox), and responds to the lead before they've finished browsing competitors' websites.

The 5-minute response window isn't a theoretical advantage. It's the single biggest predictor of lead conversion in real estate, and it's almost impossible to hit consistently with manual processes.

## Use Case 2: Listing Management Automation

Every listing has a lifecycle: create, update, adjust price, change status, remove. Each transition involves multiple systems — your CRM, your website, the MLS, property portals. Doing this manually across 40 active listings is a full-time job.

### Automated Listing Descriptions

Writing listing descriptions is one of those tasks that's easy but tedious. You have the property specs (bedrooms, bathrooms, square footage, neighborhood) and you need 150–300 words of compelling copy. Ollama can draft this from structured data:

```
Write a real estate listing description for:

Property: {{ $json.address }}
Type: {{ $json.property_type }}
Bedrooms: {{ $json.bedrooms }}
Bathrooms: {{ $json.bathrooms }}
Square feet: {{ $json.sqft }}
Neighborhood: {{ $json.neighborhood }}
Key features: {{ $json.features }}

Style: professional, warm, no hype. 200 words max.
Mention the neighborhood by name. Do not use exclamation points.
```

The agent reviews the output, edits if needed, and publishes. This takes the drafting time from 15 minutes down to 2. The AI isn't replacing the agent's judgment about what makes the property special — it's handling the boilerplate structure so the agent only edits rather than writes from scratch.

### Price Adjustment Alerts

Here's an n8n workflow that monitors market data and flags listings that may need price adjustments:

1. **Scheduled trigger**: Runs weekly (every Monday at 8 AM).
2. **Data pull**: Fetches recent comparable sales from your MLS data feed or a property data API.
3. **Comparison**: For each active listing, compares the list price against the average price per square foot of recently sold comparables in the same neighborhood.
4. **AI analysis**: Ollama reviews the comparison and generates a recommendation: "Your listing at 123 Main St is priced 8% above comparable sales in the area over the past 30 days. Consider a price reduction of $15,000–$20,000 to align with market."
5. **Notification**: Sends a weekly summary email to each listing agent with their properties that may need attention.

This isn't automated decision-making — the agent still decides whether to adjust. But the agent now has market data surfaced automatically instead of having to pull comps manually for every listing every week.

### Status Sync Across Platforms

When a property goes under contract, the status needs to update everywhere simultaneously: your website, the MLS, Zillow, Realtor.com, your CRM. An n8n workflow can trigger on a status change in Odoo and fan out updates to all connected platforms via their APIs:

```
[Odoo: Listing Status Changed]
    → [Update: Company Website (via Directus API)]
    → [Update: MLS (via RETS/API)]
    → [Update: Portal Feeds (via syndication API)]
    → [Notify: Listing Agent + Seller]
    → [Log: Audit trail in Odoo]
```

One status change in the CRM propagates everywhere. No more "sorry, that property went under contract yesterday but I forgot to update the website."

## Use Case 3: Client Follow-Up and Nurturing

Real estate clients have long lifecycles. A buyer who first inquires in January might not close until August. During that period, they need periodic contact — not daily spam, but meaningful touchpoints that keep your brokerage top of mind.

### The Drip Campaign Problem

Most CRMs offer drip campaigns, but they're usually generic: "Hi [First Name], just checking in!" sent every two weeks. These get ignored. Effective follow-up is contextual — it references what the client is actually looking for and provides useful information.

### Contextual Follow-Up with AI

Here's an n8n workflow that sends genuinely useful follow-ups:

1. **Trigger**: Scheduled — checks Odoo daily for clients due for follow-up (based on last contact date and client status).
2. **Context gathering**: Pulls the client's search criteria, properties they've viewed, recent market activity in their target neighborhoods.
3. **AI personalization**: Ollama generates a personalized message:

```
You are a real estate assistant writing a follow-up email to a client.

Client name: {{ $json.client_name }}
Looking for: {{ $json.search_criteria }}
Target neighborhoods: {{ $json.neighborhoods }}
Properties viewed recently: {{ $json.recent_views }}
New listings matching criteria: {{ $json.new_matches }}
Last contact: {{ $json.last_contact_date }}

Write a short (100-150 word) email that:
- Mentions 1-2 new listings that match their criteria
- References something specific about their search
- Does NOT use pressure tactics or hype
- Sounds like it's from a real person, not a template
- Includes a soft invitation to schedule a viewing
```

4. **Draft review**: The message goes to the agent's draft folder, not directly to the client. The agent reviews, edits, and sends.
5. **Logging**: When sent, the interaction is logged in Odoo with the full message content.

The key design decision here: the AI drafts, the human sends. This keeps the agent in control of client communication while eliminating the blank-page problem that makes follow-up procrastination so common. The agent isn't writing from scratch — they're editing a draft that's already 90% right.

### Seller Updates

Sellers want to know what's happening with their listing. A weekly automated update builds trust and reduces "any news?" phone calls:

- Showings this week (from showing software API)
- Online views (from website analytics)
- Price comparison vs. neighborhood (from MLS data)
- Agent's notes (manual input from Odoo)

Ollama summarizes this into a readable weekly digest. The agent reviews it in 30 seconds and sends it. The seller feels informed. The agent spends 30 seconds instead of 15 minutes per seller per week.

## What Not to Automate in Real Estate

In the spirit of honesty, here's what should stay manual:

- **Pricing decisions.** AI can surface comparables and flag misalignment, but the final list price is a judgment call involving property condition, market timing, and seller motivation. Don't automate this.
- **Showing coordination.** The logistics of scheduling a showing involve buyer availability, seller availability, tenant access, and sometimes lockbox codes. The edge cases will eat your automation alive.
- **Negotiation.** Any communication during an active offer/counter-offer cycle should be human-to-human. This is where relationships and reading the room matter most.
- **Initial buyer consultations.** Understanding a client's real needs (not just their stated criteria) requires conversation. Automate the follow-up, not the discovery.

Automate the infrastructure. Keep the judgment calls human. That's the line.

## Cost Reality Check

Let's talk about what this actually costs to run, compared to a typical real estate SaaS stack:

| Component | Open-Source Self-Hosted | Typical SaaS Equivalent |
|-----------|------------------------|------------------------|
| CRM | Odoo Community (free) | Follow Up Boss ($69/mo per user) |
| Workflow automation | n8n self-hosted (free) | Zapier ($20-100/mo) |
| AI inference | Ollama (free, local) | OpenAI API ($50-200/mo) |
| Scheduling | Cal.com self-hosted (free) | Calendly ($10-24/user/mo) |
| Infrastructure | VPS ($20-40/mo) | Included in SaaS pricing |

For a 5-agent brokerage, the SaaS stack runs $400–700/month. The self-hosted stack runs $40/month in infrastructure plus your time for setup and maintenance. The trade-off is clear: SaaS is faster to set up and someone else handles updates. Self-hosted is cheaper, gives you data control, and has no per-seat scaling costs.

For a brokerage that's already paying for a website and has someone who can manage a Docker deployment, the open-source route pays for itself within the first month.

## Getting Started: A 7-Day Plan

If you want to try this without disrupting your current operations, here's a phased approach:

**Day 1–2**: Set up Odoo Community Edition via Docker. Import your existing contacts and active listings. Get comfortable with the CRM interface.

**Day 3–4**: Install n8n (Docker) and Ollama. Pull a mid-size model like Llama 3 (8B) — it runs on a server with 8GB RAM. Test the lead scoring prompt with sample inquiries.

**Day 5**: Build the lead routing workflow. Start with a webhook trigger from your website contact form. Route all leads to yourself first — don't auto-assign to agents until you've reviewed the AI scores for accuracy.

**Day 6**: Build the listing description generator. Run it on your existing listings and compare the AI output to your current descriptions. You'll quickly see where the model shines and where it needs prompting adjustments.

**Day 7**: Build one follow-up workflow — the seller weekly update is a good starting point because the data is structured and the output is easy to verify. Send it to yourself first.

After a week, you'll have a working automation stack handling three real workflows. Expand from there based on what's actually saving time.

## The Bottom Line

Real estate brokerages spend thousands per month on SaaS tools that do what open-source software can do for free — and do it with better data privacy. The automation use cases we covered (lead routing, listing management, client follow-up) aren't speculative. They're the exact workflows that eat agent time and lose deals when they're done manually.

The advantage of self-hosting isn't just cost. It's control. Your leads, your listings, your client communications — all on your server, under your security, with no vendor deciding to raise prices or change terms. For an industry built on trust and long-term relationships, owning your infrastructure is a strategic decision, not just a technical one.

If you want help setting up any of these workflows — lead routing, listing automation, or a self-hosted CRM — [get in touch](/#contact). We build real estate automation with open-source tools, and we'll help you get from manual to automated without the hype.