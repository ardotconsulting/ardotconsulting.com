---
layout: post
title: "Automating Marketing Without Marketing SaaS: Open Source Tools That Replace Your Subscription Stack"
date: 2026-10-12
author: "ARDOT Consulting"
tags: [marketing-automation, open-source, listmonk, n8n, ollama, email-campaigns, lead-scoring]
excerpt: "Your marketing stack doesn't need five SaaS subscriptions costing $400 a month. Here's how to run email campaigns, draft content, and score leads with open source tools you control — and when the SaaS route still makes sense."
---

If you run a small business, you've probably seen the pitch: sign up for this email tool ($50/month), that social media scheduler ($30/month), this landing page builder ($99/month), a CRM with marketing automation ($149/month), and an AI copywriting assistant ($39/month). Before you know it, you're paying $367 a month for five tools that mostly talk to each other through fragile integrations and all want your customer data.

It works. Plenty of businesses run exactly this way. But it's not the only option, and for a lot of small teams, it's more expensive and more fragile than it needs to be.

Here's the thing: every one of those marketing functions — email campaigns, content drafting, lead scoring, social scheduling — can be done with open source tools you host yourself. Not someday. Not experimentally. Today, with mature software that real businesses use in production. And when you host them yourself, you own your data, you stop paying per-seat pricing that scales with your team size, and you can wire them together however you want.

Let me walk through the four biggest pieces of a marketing stack and show you the open source alternative for each — including where self-hosting wins, where it doesn't, and how to put it all together.

## 1. Email Marketing: Listmonk Instead of Mailchimp

Email is the backbone of small business marketing. It has the highest ROI of any channel — roughly $36 returned for every $1 spent, consistently, across industry surveys. The tools are mature, the formats are standard, and the open source options are genuinely good.

**Listmonk** is the one to know about. It's a self-hosted newsletter and email marketing tool that does the core things you actually pay Mailchimp or Brevo for:

- Build subscriber lists with double opt-in
- Design email campaigns with a drag-and-drop editor
- Schedule sends
- Track opens and clicks
- Manage unsubscribes automatically (legally required in most jurisdictions)
- Send transactional emails

It's a single binary or Docker container. You deploy it, point a domain at it, configure your SMTP credentials (you can use a service like Postmark, Amazon SES, or even self-host with Postal if you want to go all-in), and you're running. The interface is clean. The campaign analytics are built in.

Here's where it saves you money: Listmonk has no per-subscriber pricing. Mailchimp's standard plan charges you based on how many contacts you store — 10,000 contacts costs around $100/month, 50,000 costs $350/month. With Listmonk, you pay for your email sending infrastructure (Amazon SES charges $0.10 per 1,000 emails) and that's it. Your cost stays flat whether you have 500 subscribers or 50,000.

**Where self-hosting wins:** Cost at scale, data ownership, no contact limits, no surprise pricing tier jumps.

**Where SaaS still wins:** If you send less than 500 emails a month and don't want to manage infrastructure, Mailchimp's free tier is genuinely free. There's no setup, no server to maintain, no DNS records to configure. For a brand-new business just starting to build a list, that's fine. Move to Listmonk when your list grows past the free tier or when you start caring about where your subscriber data lives.

## 2. Content Drafting: Ollama Instead of an AI Copywriting SaaS

Every marketing team now uses AI to draft content — blog posts, social media updates, email newsletters, ad copy. The question isn't whether to use AI for drafting. It's whether you need to send your brand voice, product descriptions, and customer pain points to a third-party API every time you want a draft.

**Ollama** lets you run language models on your own hardware. We've covered this in detail before, but here's the marketing-specific use case: you install Ollama on a machine in your office (a desktop with 16GB RAM runs a capable 8B model fine), and you use it to draft marketing content without sending anything to the cloud.

The workflow looks like this:

1. You write a prompt with your brand guidelines, target audience, and topic
2. Ollama generates a first draft
3. You edit it (the editing part matters — AI drafts are starting points, not finished copy)
4. You publish

For marketing specifically, local models are good at:
- Repurposing one piece of content into five formats (turn a blog post into a newsletter intro, three social posts, and a LinkedIn article)
- Generating subject line variations for A/B testing
- Drafting product descriptions from a spec sheet
- Summarizing customer feedback into themes

They're not as polished as the latest cloud models for long-form content. An 8B parameter model won't write a 2,000-word thought leadership piece that reads like a human wrote it. But for the repetitive drafting work that takes up 60% of a marketing person's day — variations, summaries, rewrites — a local model is more than enough.

**Where self-hosting wins:** Privacy (your brand strategy and customer data never leaves your server), no per-token API costs, no rate limits.

**Where SaaS still wins:** If you need the absolute best output quality for customer-facing content and you don't want to edit heavily, a cloud model will produce more polished first drafts. The gap is closing, but it's still real. Use cloud APIs for final-polish content, use local models for internal drafts and variations.

## 3. Lead Scoring and Routing: n8n Instead of a Marketing Automation Platform

This is where most of the "marketing automation" money goes. HubSpot Marketing Hub ($890/month for the Professional tier), ActiveCampaign ($79/month and up), Pardot ($1,250/month) — these tools all do variations of the same thing: they watch for signals (form fills, email opens, page visits), apply rules to score those signals, and trigger actions when a lead crosses a threshold.

**n8n** does this. Self-hosted, open source, no per-contact pricing.

Here's a concrete lead scoring workflow you can build in n8n:

1. **Trigger:** A form submission comes in from your website (n8n has a built-in webhook node that catches any HTTP POST)
2. **Enrich:** n8n looks up the company domain via an open source data source or your own CRM, pulls employee count and industry
3. **Score:** A function node applies your scoring rules — +10 points for target industry, +5 for company size >50, +3 for specific pages visited, +2 for email opens in the last 7 days
4. **Route:** If score > 25, create a task in your project management tool and send a Slack-compatible notification (we've written about Mattermost for this). If score < 25, add to a nurture email sequence in Listmonk.

The whole workflow is visual. You drag nodes onto a canvas, connect them, configure each one, and save. No code required, though you can write JavaScript in any node if you want custom logic.

The advantage over a SaaS marketing platform isn't just cost. It's flexibility. When your scoring rules change — and they will, because your first scoring model is always wrong — you open n8n, change a number in a node, and save. In a SaaS platform, you're clicking through their UI, hoping their rule engine supports what you want, and paying more when you hit workflow limits.

**Where self-hosting wins:** Cost (n8n is free if you self-host), flexibility (no vendor lock-in on your workflow logic), integration with your other self-hosted tools.

**Where SaaS still wins:** Setup time. Building a lead scoring workflow in n8n takes a few hours the first time. HubSpot has it pre-built with templates. If you need lead scoring running tomorrow and you've never touched a workflow builder, a SaaS platform will get you there faster. Build your own when you want to own the logic and stop paying per contact.

## 4. Analytics and Attribution: Plausible Instead of Google Analytics

We've covered self-hosting Plausible Analytics in detail before, so I'll keep this short. For marketing specifically, the relevant point is attribution: knowing which campaign sent which visitor who eventually became a customer.

Plausible does this out of the box. You set up custom goals (e.g., "completed contact form"), tag your campaign links with UTM parameters, and Plausible shows you which campaigns drove goal completions. No event setup gymnastics, no 24-hour data processing delay, no sampling on high-traffic sites.

Google Analytics 4 can do this too, but it requires significant configuration, the interface is confusing for non-analysts, and — the part that matters for this article — you're handing your visitor data to Google. With self-hosted Plausible, the analytics data sits in a PostgreSQL database on your server. You own it. No one is using it to build an ad profile of your visitors.

**Where self-hosting wins:** Privacy, simplicity, data ownership, no cookie consent banner drama (Plausible is GDPR-compliant without cookies).

**Where SaaS still wins:** Plausible Cloud ($9/month for small sites) is cheap and removes the hosting burden. If you don't want to manage a Docker container for analytics, the hosted version is a reasonable middle ground — still open source, still privacy-first, just hosted by the people who wrote it.

## Putting It All Together: The Architecture

Here's what a fully self-hosted marketing stack looks like connected together:

| Function | SaaS Alternative | Open Source Tool | Monthly Cost |
|----------|-----------------|-----------------|--------------|
| Email marketing | Mailchimp ($100+/mo) | Listmonk + Amazon SES | ~$5 (email volume) |
| Content drafting | Copy.ai ($49/mo) | Ollama (local) | $0 (uses existing hardware) |
| Lead scoring & routing | HubSpot ($890/mo) | n8n (self-hosted) | $0 (server cost shared) |
| Analytics & attribution | Google Analytics (free but data trade-off) | Plausible (self-hosted) | $0 (server cost shared) |
| Team chat / notifications | Slack ($7+/user/mo) | Mattermost (self-hosted) | $0 (server cost shared) |

Total software cost for the self-hosted stack: roughly $50-80/month in server hosting (a single $20-40/month VPS can run all of these for a small team) plus email sending costs at $0.10 per 1,000 emails. For a 10-person team, the SaaS equivalent would cost $1,200-1,500/month.

That gap is the entire point. You're trading money for time spent on setup and maintenance. If you have more time than budget — which describes most small businesses in their first few years — self-hosting is a clear win. If you have more budget than time, the SaaS stack is faster to get running and requires less ongoing attention.

## The Honest Caveats

I'd be lying if I said self-hosting has no downsides. Here are the real ones:

**You are now your own IT department.** When Listmonk has an update, you apply it. When your VPS goes down at 2 AM, you get the alert. When an email bounces because your SPF record is misconfigured, you debug the DNS. This is real work. It's not a lot once things are set up, but it's not zero.

**Open source tools have smaller communities.** If you get stuck in Mailchimp, you can chat with their support. If you get stuck in Listmonk, you're reading GitHub issues and forum posts. The community is helpful, but it's not a phone call away.

**The first setup is the hard part.** After that, maintenance is occasional. But the first weekend of getting Docker Compose files working, DNS records configured, and SSL certificates set up is a real time investment. Budget 8-16 hours for the initial setup of the full stack.

**Some tools are worth paying for.** If your email deliverability is critical (and for most businesses, it is), paying for a managed SMTP service like Amazon SES or Postmark is worth it. You self-host the marketing tool, but you use a managed service for the actual email delivery. That's a reasonable hybrid.

## When to Start

Don't migrate everything at once. Pick one piece — usually email marketing, because it has the clearest cost savings — and move that. Run it alongside your existing tools for a month. If it works, move the next piece. If it doesn't, you've lost a weekend, not a year.

The most common path we see: businesses start with Listmonk to escape per-subscriber pricing, then add n8n for workflow automation, then add Ollama for content drafting, then realize they've replaced their entire SaaS stack. It happens incrementally, not in a single migration project.

## Want Help Putting This Together?

If you're tired of paying five SaaS subscriptions and want to explore what a self-hosted marketing stack would look like for your specific business, we can help. We'll audit your current tools, map out the open source replacements, and build the integrations — so you end up with a marketing stack you own, not one you rent.

[Get in touch through our contact form](/#contact) and we'll set up a call.