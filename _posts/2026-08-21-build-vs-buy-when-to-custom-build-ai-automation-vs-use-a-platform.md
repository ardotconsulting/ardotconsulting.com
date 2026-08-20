---
layout: post
title: "Build vs Buy: When to Custom-Build AI Automation vs Use a Platform"
date: 2026-08-21
description: "A practical decision framework for choosing between building custom AI automation with open source tools or buying a SaaS platform. Cost analysis, maintenance burden, data privacy, and a decision tree you can actually use."
tags: [build-vs-buy, strategy, decision-making, open-source, n8n]
author: "ARDOT Consulting"
excerpt: "Should you build your own AI automation or buy a platform? Here's an honest decision framework with real cost numbers, a comparison table, and a decision tree — no hype, no vendor bias."
---

# Build vs Buy: When to Custom-Build AI Automation vs Use a Platform

Every business owner who starts exploring AI automation hits the same fork in the road: do I build it myself, or do I buy a platform that does it for me?

It's a genuinely hard question, and most of the advice you'll find online is biased in one direction or the other. SaaS vendors will tell you that building is too complex and expensive. Consultants will tell you that buying is too rigid and you'll get locked in. The truth is that both are right sometimes, and neither is right all the time.

This post gives you a framework for making the decision for *your* business — not in the abstract, but with real cost numbers, a comparison table, and a decision tree you can actually follow.

## What "Build" and "Buy" Actually Mean

Before comparing, let's be clear about what we're talking about.

**Build** means self-hosting open source tools and configuring them to do what you need. In the AI automation world, this typically means running **n8n** (an open source workflow automation platform) on your own server, paired with **Ollama** (which runs AI models locally on your hardware). You own the infrastructure, you control the data, and you customize everything. You're not writing code from scratch — n8n is a visual workflow builder — but you are responsible for setup, maintenance, and updates.

**Buy** means subscribing to a SaaS platform that handles the automation for you. You log into a web dashboard, configure some settings, and the vendor takes care of the infrastructure, updates, and scaling. Examples include Zapier, Make.com, and various AI-powered SaaS products. You pay a monthly fee, and you work within the platform's constraints.

There's also a middle path — self-hosting an open source tool but paying for support or managed hosting — but let's keep the comparison clean for now.

## The Five Factors That Actually Matter

Forget feature checklists. When you're deciding between build and buy, five factors determine the right answer. Here they are, in order of how much they usually matter.

### 1. Data Privacy and Ownership

This is the factor most businesses underweight, and it's the one most likely to cause regret later.

When you buy a SaaS platform, your data flows through someone else's servers. Most vendors have privacy policies that say they won't use your data to train their models — but policies change, acquisitions happen, and you have no way to verify what actually happens to your data once it leaves your control.

When you build with self-hosted tools, your data never leaves your infrastructure. The AI models run on your hardware. The workflow logs stay on your server. You can audit everything.

**This matters most when:**
- You handle sensitive client data (legal, healthcare, finance)
- You're subject to regulations like HIPAA, GDPR, or attorney-client privilege
- Your data is a competitive advantage and you don't want it anywhere else
- You simply don't trust third parties with your business information

If any of those apply to you, the build path has a strong default advantage. The privacy gap between "your server" and "a vendor's cloud" is not cosmetic — it's structural.

### 2. Total Cost Over 2 Years

SaaS platforms advertise low monthly fees. A basic Zapier plan starts around $20/month. Make.com starts around $9/month. These numbers look small, and they are — until you scale.

The problem is that SaaS pricing is almost always usage-based. Every workflow run, every API call, every task executed costs money. A business that starts with 1,000 task executions per month can easily hit 50,000 within a year as they automate more processes. At that point, you're paying hundreds or thousands per month.

Here's a realistic 24-month cost comparison for a small business automating 5-10 workflows:

| Approach | Month 1 | Month 12 | Month 24 | 2-Year Total |
|----------|---------|----------|----------|-------------|
| SaaS (entry plan) | $20/mo | $50/mo | $100/mo | ~$1,020 |
| SaaS (scaling) | $50/mo | $150/mo | $300/mo | ~$2,700 |
| SaaS (heavy use) | $100/mo | $300/mo | $500/mo | ~$4,800 |
| Self-hosted (VPS) | $20/mo server | $20/mo | $20/mo | ~$480 + setup time |
| Self-hosted (own server) | $0/mo | $0/mo | $0/mo | ~$0 + hardware |

The self-hosted numbers assume you already have or can rent a modest server. A basic VPS from a provider like Hetzner runs about $5-15/month and handles n8n plus Ollama with a small model. If you have an existing office machine with a decent GPU, you can run everything locally for the cost of electricity.

The catch with self-hosting is that the cost shows up as **time** rather than money. Setup takes a day or two. Maintenance — updates, backups, troubleshooting — might be a few hours per month. If your time is worth $100/hour and you spend 4 hours/month on maintenance, that's $4,800/year in opportunity cost. But you're also building internal capability, which has its own value.

**The pattern:** SaaS is cheaper in month 1. Self-hosted is cheaper by month 6-12, and the gap widens over time. If you're planning to automate more than a couple of workflows and you expect usage to grow, build wins on cost.

### 3. Flexibility and Customization

SaaS platforms let you do what they've designed for. If your workflow fits their template, it's fast and easy. If it doesn't — if you need a custom data transformation, a specific API integration, or a particular output format — you hit a wall.

n8n, because it's open source and self-hosted, has no such wall. You can write custom JavaScript in any node. You can connect to any API that has an HTTP endpoint. You can run a local AI model with custom prompts. You can pipe data to a self-hosted database, a file system, or a webhook. If you can describe it, you can probably build it.

This matters most for businesses with unusual workflows. If your process is "standard" — email comes in, data gets extracted, data goes to a CRM — a SaaS platform will handle it fine. If your process is "we receive a PDF, extract three specific fields, cross-reference them against a local database, generate a custom report, and email it to a client only if certain conditions are met" — you're going to fight a SaaS platform every step of the way.

### 4. Maintenance and Support

This is where SaaS platforms have a genuine, honest advantage. When something breaks on a SaaS platform, you open a support ticket. When an API changes and breaks your Zapier integration, Zapier updates their connector. You don't think about servers, backups, SSL certificates, or security patches.

When you self-host, all of that is on you. n8n needs updates. Ollama needs updates. Your server needs security patches. If a workflow breaks at 2 AM, nobody gets paged unless you set up monitoring yourself.

The mitigation: this is more manageable than it sounds. Tools like Docker Compose make updates a single command. Services like Uptime Kuma (open source) handle monitoring. And if you're willing to learn a little Linux administration — or hire someone for a few hours a month — the maintenance burden drops significantly.

But if your business has no technical person, no IT support, and no interest in learning server management, the SaaS maintenance advantage is real. Don't discount it.

### 5. Lock-In and Exit Cost

When you build on an open source stack, you own the workflow. The n8n workflow definitions are JSON files you can export and version-control. The Ollama models are standard format. If you decide to switch tools, you can migrate.

When you buy a SaaS platform, you're building on someone else's land. If they raise prices, change their API, remove a feature, or shut down, you're stuck. We've all seen SaaS products get acquired and "sunsetted." Your workflows, your integrations, your historical data — all hostage to the vendor's business decisions.

The exit cost of migrating off a SaaS platform after 2 years of building workflows on it is significant. Not catastrophic, but significant. You'll rebuild most of what you built.

## The Decision Tree

Here's a practical decision tree. Walk through it in order:

1. **Do you handle sensitive or regulated data?**
   - **Yes** → Lean strongly toward **build**. Self-hosted keeps data on your infrastructure. This alone is often the deciding factor for legal, healthcare, and finance firms.

2. **Do you expect to automate more than 3-5 workflows over the next year?**
   - **Yes** → Lean toward **build**. SaaS usage pricing scales up fast, and the per-workflow cost of self-hosting approaches zero.
   - **No** (1-2 simple workflows) → Lean toward **buy**. The setup time for self-hosting isn't worth it for a single workflow.

3. **Are your workflows standard (email → CRM → notification) or custom?**
   - **Standard** → **Buy** works fine. SaaS platforms handle common patterns well.
   - **Custom** → Lean toward **build**. You'll fight SaaS constraints less.

4. **Do you have someone who can handle basic server maintenance?**
   - **Yes** (even if it's a few hours/month) → **Build** is viable.
   - **No, and you don't want to hire anyone** → **Buy**. The maintenance burden of self-hosting will frustrate you.

5. **Is avoiding vendor lock-in important to you?**
   - **Yes** → **Build**. You own everything.
   - **No, you just want it to work today** → **Buy** is fine.

## A Realistic Recommendation

For most small businesses we work with, the answer is **start with build, but start small.** Here's why:

The first workflow you automate is always the hardest, because you're setting up infrastructure. But once n8n is running and Ollama is installed, the second workflow takes half the time, the third takes less, and by the fifth you're moving fast. The setup cost is a one-time investment that pays off across every future automation.

If you're intimidated by the setup, that's fair — but it's a weekend project, not a multi-month engineering effort. Our [previous post on building your first n8n workflow](/blog/2026/08/17/how-to-build-your-first-ai-automation-workflow-with-n8n/) walks through the whole process step by step.

The exception: if you have one simple workflow, no sensitive data, no technical person, and no plans to expand, just buy a SaaS plan. That's a perfectly reasonable choice. Not every problem needs a self-hosted solution. The goal is automation, not ideology.

## What We'd Actually Do

If ARDOT were advising a client today, we'd ask three questions in the first 10 minutes:

1. "What data flows through this workflow, and who is allowed to see it?"
2. "How many workflows do you realistically want to automate in the next 12 months?"
3. "Who on your team can reboot a server if something goes down?"

The answers tell us everything. Sensitive data or many workflows or someone technical → build. None of the above → buy, and revisit in 6 months.

There's no wrong answer. There's only the answer that fits your business right now.

---

*Want help figuring out which path is right for your business? [Get in touch](/#contact) — we'll walk through your specific workflows and give you a straight recommendation, even if that recommendation is "just buy a SaaS plan." No hype, no upsell.*