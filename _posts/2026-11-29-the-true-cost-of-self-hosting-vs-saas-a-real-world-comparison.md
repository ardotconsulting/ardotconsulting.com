---
layout: post
title: "The True Cost of Self-Hosting vs SaaS: A Real-World Comparison"
date: 2026-11-29
author: "ARDOT Consulting"
tags: [self-hosting, cost-comparison, saas, strategy, open-source, roi, total-cost-of-ownership]
excerpt: "Self-hosting saves you per-user subscription fees, but it's not free. We break down the real costs — server hardware, electricity, maintenance time, and opportunity cost — and compare them head-to-head with equivalent SaaS subscriptions so you can make an honest decision."
---

# The True Cost of Self-Hosting vs SaaS: A Real-World Comparison

If you've read a few of our self-hosting guides, you might be noticing a pattern. We talk a lot about replacing SaaS tools with open source alternatives you run yourself — Nextcloud instead of Google Workspace, Mautic instead of HubSpot, Kimai instead of Toggl, Mattermost instead of Slack. The pitch is consistent: stop paying per-user subscriptions, own your data, break free of vendor lock-in.

But here's the thing we want to be honest about: **self-hosting is not free.** It's cheaper, usually significantly cheaper, but it trades one kind of cost for another. You stop paying monthly subscription fees and start paying in server costs, setup time, maintenance hours, and a certain amount of mental overhead.

This post is for the business owner who's read about self-hosting and is thinking: "This sounds great, but what's the catch?" Let's break it down with real numbers.

## The SaaS Subscription Stack

Let's start by building a realistic SaaS stack for a small business — say, a 10-person team. Not a huge enterprise, not a solo freelancer. A small agency, consulting firm, or professional services business that needs the standard set of tools.

Here's what a typical 10-person team might be paying:

| Tool | Purpose | Cost (10 users) |
|------|---------|-----------------|
| Google Workspace (Business Standard) | Email, docs, file storage, calendar | $72/mo ($7.20/user) |
| Slack (Pro) | Team chat | $90/mo ($9/user) |
| HubSpot Marketing (Starter) | CRM + marketing automation | $45/mo (base, limited) |
| Toggl Track (Starter) | Time tracking | $100/mo ($10/user) |
| LastPass (Business) | Password manager | $40/mo ($4/user) |
| Zoom (Pro) | Video conferencing | $149/mo ($14.99/user) |
| Typeform (Essentials) | Forms and surveys | $25/mo (base) |
| Airtable (Team) | No-code database | $120/mo ($12/user) |
| Trello (Standard) | Project management | $50/mo ($5/user) |
| Calendly (Teams) | Scheduling | $160/mo ($16/user) |
| **Total** | | **~$851/mo ($10,212/yr)** |

And this is a conservative estimate. We picked mid-tier plans, not enterprise. We didn't include a help desk tool, analytics, document signing, or a newsletter platform — all of which would add another $200–500/month. If you went with HubSpot Marketing Professional instead of Starter, you'd add $890/month on its own.

The reality for most small businesses is a SaaS stack that costs **$800–1,500/month** for 10 people, growing linearly as you add users.

## The Self-Hosted Equivalent

Now let's build the equivalent stack using self-hosted open source tools. We've covered most of these in previous posts, so we won't re-explain each one — but here's the mapping:

| SaaS Tool | Self-Hosted Alternative | License |
|-----------|------------------------|---------|
| Google Workspace | Nextcloud | AGPL-3.0 |
| Slack | Mattermost | MIT/Enterprise |
| HubSpot (Starter) | Mautic | GPL-3.0 |
| Toggl Track | Kimai | AGPL-3.0 |
| LastPass | Vaultwarden | AGPL-3.0 |
| Zoom | Jitsi Meet | Apache-2.0 |
| Typeform | OpnForm | LGPL-3.0 |
| Airtable | NocoDB | AGPL-3.0 |
| Trello | OpenProject or Focalboard | GPL-3.0 |
| Calendly | Cal.com | AGPL-3.0 |

**Software licensing cost: $0.** That's the headline number, and it's accurate. The software itself is free. But you need somewhere to run it.

## The Real Costs of Self-Hosting

### Server Infrastructure

This is where the real money goes. You need a server (or servers) to run these applications. Here are the practical options:

**Option 1: A single VPS (Virtual Private Server)**

For a 10-person team running most of these tools, a mid-range VPS works. Something like a Hetzner Cloud CPX31 (8 vCPU, 16GB RAM, 240GB storage) runs about **$20/month**. You could also use OVH, Contabo, or DigitalOcean — pricing is similar, though we'd recommend avoiding the hyperscale clouds (AWS, Azure) for self-hosting because their egress fees and complexity will eat your savings fast.

If your team is light on concurrent usage, you might even start with a smaller instance — 4 vCPU, 8GB RAM for about **$10/month** — and scale up when needed.

**Option 2: A dedicated server or mini PC at home/office**

Some businesses prefer to keep hardware on-premises. A used mini PC (Dell OptiPlex, Lenovo ThinkCentre, or an Intel NUC) with 32GB RAM and a 1TB SSD costs **$300–600 one-time**. Add a UPS (uninterruptible power supply) for **$100–150**. This gives you more power per dollar than a VPS, but you're responsible for power, cooling, and physical security.

Electricity for a mini PC running 24/7 at ~15W idle, ~40W under load: roughly **$2–5/month** depending on your local rates.

**Option 3: A hybrid approach**

Run lightweight tools (Vaultwarden, OpnForm, Cal.com) on a small VPS for **$5–10/month**, and heavier tools (Nextcloud, Mautic, Jitsi) on a dedicated machine or larger VPS for **$20–30/month**. This is what most businesses settle on.

**Realistic server cost: $25–40/month.**

### Domain and DNS

You need a domain name and DNS hosting. A .com domain costs about **$12/year** ($1/month). DNS hosting is free with Cloudflare or your domain registrar. If you want a managed DNS service, it's still under $5/month.

**Domain and DNS cost: ~$2/month.**

### Backups

This is the one people forget, and it's the most important. You need off-site backups of your self-hosted data. Options:

- **Restic + Backblaze B2**: Restic is an open source backup tool. Backblaze B2 is object storage at $0.006/GB/month. For 50GB of data, that's **$0.30/month**. For 200GB, it's **$1.20/month**.
- **BorgBackup + Hetzner Storage Box**: 100GB storage box for **~$4/month**.
- **Duplicati + any S3-compatible storage**: Similar pricing.

**Backup cost: $1–5/month.** Not a significant expense, but you absolutely cannot skip it. We covered this in detail in our [disaster recovery guide](/blog/2026/11/17/backing-up-your-self-hosted-stack-a-practical-disaster-recovery-guide/).

### SSL Certificates

Free. Let's Encrypt provides unlimited SSL certificates, and Caddy or Traefik (reverse proxies) handle automatic renewal. **$0/month.**

### The Hidden Cost: Time

This is the real trade-off. Self-hosting requires setup and maintenance time. Let's be honest about how much:

**Initial setup**: Deploying each tool with Docker, configuring DNS and SSL, setting up user accounts, and testing takes 2–6 hours per tool depending on complexity. For 10 tools, that's **20–60 hours** of setup time. You can do this yourself, or hire someone (like us), or spread it out over weeks.

**Ongoing maintenance**: Updates, monitoring, occasional troubleshooting. Once everything is running, this averages **2–4 hours/month** for a 10-tool stack. Some months it's zero. Some months a container breaks after an update and you spend an afternoon fixing it. Over a year, plan for **24–48 hours of maintenance**.

If you value your time at $75/hour (a reasonable rate for a business owner or technical staff), that's:

- Setup: $1,500–4,500 (one-time)
- Maintenance: $1,800–3,600/year

But here's the nuance: setup is a one-time cost. You don't redo it every year. And maintenance time decreases as your setup stabilizes and you automate updates. After the first year, expect maintenance to drop to 1–2 hours/month.

### The Other Hidden Cost: Downtime Risk

When SaaS goes down, it's their problem. When your self-hosted stack goes down, it's yours. A misconfigured update, a full disk, a DNS issue — these can take a tool offline for hours if you're not monitoring.

You mitigate this with monitoring (Uptime Kuma, which we covered in our [monitoring stack guide](/blog/2026/10/24/self-hosted-monitoring-stack-uptime-kuma-grafana-replace-statuspage-and-datadog/)) and by keeping your setup simple. But there's an inherent risk: you are the IT department.

## The Honest Comparison

Let's put it all together. Here's a 3-year total cost of ownership for a 10-person team:

| Category | SaaS Stack | Self-Hosted Stack |
|----------|-----------|-------------------|
| Software licensing | $30,636 ($851/mo × 36) | $0 |
| Server infrastructure | $0 | $1,080 ($30/mo × 36) |
| Domain/DNS | Included | $72 ($2/mo × 36) |
| Backups | Included | $108 ($3/mo × 36) |
| Initial setup time | ~8 hours ($600) | ~40 hours ($3,000) |
| Ongoing maintenance | ~12 hours/year ($900/yr) | ~36 hours/year ($2,700/yr) |
| **3-year total** | **~$33,336** | **~$11,760** |
| **3-year savings** | — | **~$21,576** |

Even accounting for setup time and maintenance, self-hosting saves roughly **$21,500 over three years** for a 10-person team. That's a 65% reduction in total cost.

And the gap widens over time. The SaaS cost grows linearly with headcount — add a 10th user and every tool charges more. Add a 20th and it doubles. Self-hosted costs stay roughly flat: the same $30/month server handles 5 users or 50.

### Where the Numbers Shift

There are scenarios where self-hosting's advantage narrows:

**Very small teams (1–3 people):** If you only need 2–3 tools, the SaaS bill might be $50–100/month, and the setup time for self-hosting isn't worth it. A solo consultant who needs email and a password manager is better off paying $12/month for a managed solution.

**Teams that scale up and down frequently:** SaaS lets you add and remove users instantly. Self-hosting's user management is free, but if you have high turnover or seasonal staff, the admin overhead of managing accounts (and revoking access) can add up.

**Teams with zero technical capacity:** If nobody on your team can run a Docker command and you have no budget for outside help, self-hosting is a risk. You can hire a consultant for setup (we do this), but you need someone on-call for when things break.

**Mission-critical uptime requirements:** If your business loses $1,000/hour when a tool is down, the SaaS provider's 99.9% uptime SLA might be worth the premium. Your self-hosted setup can match that uptime, but only if you invest in redundancy, monitoring, and failover — which adds cost and complexity.

## When Self-Hosting Makes Sense

Based on the numbers and our experience, self-hosting is the clear winner when:

1. **You have 5+ users.** The per-user SaaS model starts hurting at scale. At 5 users, self-hosting usually breaks even within the first year. At 20+, the savings are dramatic.
2. **You're replacing 4+ tools.** The setup time is amortized across multiple tools. Replacing one $15/month tool with a self-hosted alternative isn't worth the effort. Replacing a $800/month stack is.
3. **You have basic technical comfort.** You don't need to be a developer. You need to be comfortable following instructions, using a terminal, and not panicking when a container needs restarting. Or you need to hire someone who is.
4. **Data ownership matters to you.** If you're in healthcare, legal, finance, or any industry where client data confidentiality is a selling point, self-hosting isn't just cheaper — it's a competitive advantage.
5. **You plan to stay in business for 2+ years.** The setup cost is one-time. The longer you run the stack, the better the ROI.

## When SaaS Makes Sense

And SaaS is the right call when:

1. **You're a solo operator or 2-person team.** The math doesn't work in your favor yet. Focus on your business, not your infrastructure.
2. **You need something live this week.** Self-hosting setup takes days to weeks. SaaS takes minutes. If you have a deadline, pay for speed.
3. **Your tool needs are minimal.** If you genuinely only need email and a calendar, a $6/month subscription is the right answer.
4. **Your team is entirely non-technical and you can't get support.** Self-hosting without anyone who can troubleshoot is a ticking time bomb. Either get a support arrangement or stick with SaaS.

## A Practical Recommendation

If you're unsure, start in the middle. Don't try to replace everything at once. Pick one or two tools where the SaaS cost is highest and the self-hosted alternative is most mature.

**Start with Nextcloud.** It replaces Google Workspace (email, files, calendar, contacts) — typically your biggest per-user line item. The setup is well-documented, the software is rock-solid, and the savings are immediate.

**Then add Mattermost.** It replaces Slack. Simple to deploy, and once your team is on it, the chat history and integrations are yours forever.

**Then add whichever tool is costing you the most.** For marketing-heavy businesses, that's Mautic (replaces HubSpot). For project-based teams, that's Kimai (replaces Toggl). For teams that schedule a lot of calls, that's Cal.com (replaces Calendly).

Each replacement pays for the server infrastructure many times over. And once you've done two or three, the pattern repeats — Docker, reverse proxy, backups, done.

## The Bottom Line

Self-hosting isn't free, and anyone who tells you it is is selling you something (usually their own hosting service). But when you account for all the real costs — servers, backups, setup time, maintenance — it's **roughly one-third the cost of equivalent SaaS** for a team of 10, and the advantage grows with size.

The trade-off is control and cost vs. convenience and someone-else's-problem. For businesses that plan to be around for years, have more than a few users, and care about where their data lives, the math points clearly toward self-hosting.

If you want help running the numbers for your specific stack, or you'd like someone to handle the setup so you can skip the learning curve, [get in touch](/#contact). We'll look at what you're paying for now and show you exactly what self-hosting would cost and save.