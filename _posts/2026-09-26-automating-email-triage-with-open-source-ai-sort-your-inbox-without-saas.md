---
layout: post
title: "Automating Email Triage with Open-Source AI: Sort Your Inbox Without a SaaS"
date: 2026-09-26
author: "ARDOT Consulting"
tags: [email, automation, ollama, n8n, open-source, inbox-zero, tutorial, self-hosting]
excerpt: "How to build a self-hosted email triage system that categorizes, prioritizes, and drafts replies to your inbox using Ollama and n8n — no per-seat SaaS fees, no third-party access to your messages."
---

If you run a small business, your inbox is probably the single biggest drain on your day. Not because the emails are hard — most of them aren't — but because there are so many of them, and sorting the important ones from the noise takes real mental effort. A client question buried under twelve promotional newsletters. A supplier invoice sitting in the same folder as a meeting confirmation. A support request that should have been answered an hour ago but got pushed below the fold.

The standard advice is "use a smart inbox" — Superhuman, SaneBox, one of the dozens of AI email tools that charge $15–$30 per user per month and, critically, require you to pipe your entire email history through their servers. For a solo operator that's an annoying cost. For a law firm, a financial advisory, a healthcare practice, or any business that handles sensitive client communications, it's a non-starter. Your email contains contracts, client data, financial documents. Sending it to a third-party SaaS to be categorized is a privacy decision dressed up as a productivity one.

This guide walks through a different approach: a self-hosted email triage system built with **Ollama** (for local AI) and **n8n** (for workflow automation) that reads your inbox, categorizes each message, prioritizes what matters, and drafts replies — all on your own server. No API fees. No data leaving your infrastructure. No per-seat pricing. You can run it on a $40/month VPS for the whole company.

## What We're Building

The goal isn't to replace your email client. It's to add a layer on top of it that does the triage work for you. Specifically, the system will:

1. **Check your inbox on a schedule** (every 10–15 minutes via IMAP)
2. **Read each new message** and classify it into a category — Support Request, Client Communication, Invoice/Payment, Internal, Newsletter/Promotional, or Urgent
3. **Assign a priority** (High, Medium, Low) based on the sender, content, and urgency signals
4. **Move or tag the email** in your mailbox so your inbox only shows what actually needs your attention
5. **Draft a suggested reply** for messages that need a response, stored alongside the email for your review

The AI processing happens locally via Ollama. The workflow orchestration happens in n8n. Your email stays on your mail server and your VPS — it never touches a third-party API.

## Prerequisites

You'll need:

- **A VPS or local server** with at least 8GB RAM (16GB recommended if you're running a larger model). A basic Hetzner or OVH cloud server works fine. No GPU required — we'll use a smaller, efficient model.
- **Docker and Docker Compose** installed
- **An email account with IMAP access** — this works with self-hosted mail servers (Mailcow, Poste.io, Postal) as well as standard IMAP providers
- **About an hour** to set everything up

## Step 1: Set Up Ollama

If you followed our [Ollama guide](/blog/2026/08/20/running-local-llms-with-ollama-a-practical-guide-for-businesses/), you already have this. If not, the quick version:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model suitable for classification and short text tasks
ollama pull llama3.2:3b

# Verify it's running
ollama run llama3.2:3b "Say hello in one sentence."
```

We're using Llama 3.2 3B here because email triage is a classification task, not a creative writing task. You don't need a 70-billion-parameter model to figure out whether an email is an invoice or a newsletter. The 3B model runs comfortably on CPU, responds in under 2 seconds per email, and costs nothing per inference. If you have a GPU or more RAM, `qwen2.5:7b` will give you slightly better accuracy on ambiguous messages.

## Step 2: Set Up n8n

Again, if you already have n8n running, skip ahead. For a fresh install via Docker Compose:

```yaml
# docker-compose.yml
version: "3.8"
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=your-strong-password
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n_data:
```

```bash
docker compose up -d
```

Navigate to `https://n8n.yourdomain.com/` (behind a reverse proxy with TLS — we recommend Caddy for automatic HTTPS) and log in.

## Step 3: Build the Triage Workflow

This is the core of the system. In n8n, create a new workflow with these nodes:

### Node 1: IMAP Trigger (Schedule)

Use the **Email Read IMAP** node (or the IMAP trigger in newer n8n versions):

- **Host:** `imap.yourmailserver.com`
- **Port:** 993
- **SSL:** enabled
- **Username / Password:** your email credentials
- **Mailbox:** `INBOX`
- **Post Process:** `Mark as read` — but **don't** move or delete yet
- **Schedule:** every 15 minutes

This node polls your inbox at the interval you set and outputs one item per new unread message. Each item contains the subject, sender, body text, and date.

### Node 2: HTTP Request → Ollama

Add an **HTTP Request** node that sends each email to your local Ollama instance for classification. Ollama exposes a simple REST API at `http://localhost:11434/api/generate`.

Configure it as a POST request:

- **URL:** `http://host.docker.internal:11434/api/generate` (or your Ollama server's IP)
- **Method:** POST
- **Body (JSON):**

```json
{
  "model": "llama3.2:3b",
  "prompt": "You are an email triage assistant. Read the email below and respond with ONLY a JSON object, no markdown, no explanation.\n\nEmail Subject: {{ $json.subject }}\nEmail Sender: {{ $json.from.value[0].address }}\nEmail Body: {{ $json.textPlain.substring(0, 2000) }}\n\nRespond in this exact format:\n{\"category\": \"<one of: Support Request, Client Communication, Invoice/Payment, Internal, Newsletter/Promotional, Urgent>\", \"priority\": \"<one of: High, Medium, Low>\", \"needs_reply\": <true or false>, \"suggested_reply\": \"<a brief professional reply if needs_reply is true, otherwise empty string>\", \"summary\": \"<one sentence summary>\"}",
  "stream": false,
  "options": {
    "temperature": 0.3
  }
}
```

A few notes on the prompt:

- **Temperature 0.3** keeps the model's output consistent. You don't want creative variation in a classification task — you want the same email to get the same category every time.
- **Truncating the body to 2000 characters** keeps inference fast and avoids context limits. For 95% of emails, the first paragraph plus the subject is enough to classify correctly.
- **The strict JSON-only instruction** is important. Llama 3.2 will sometimes wrap output in markdown code fences. Adding "no markdown, no explanation" and parsing defensively (stripping ```json fences if present) handles the edge cases.

### Node 3: Parse the JSON Response

Add a **Code** node (JavaScript) to parse Ollama's response and handle any formatting quirks:

```javascript
let rawResponse = $input.first().json.response;
let parsed;

try {
  // Strip markdown code fences if the model added them
  let cleaned = rawResponse.replace(/```json\n?/g, '').replace(/```/g, '').trim();
  parsed = JSON.parse(cleaned);
} catch (e) {
  // Fallback: treat as uncategorized
  parsed = {
    category: 'Uncategorized',
    priority: 'Medium',
    needs_reply: false,
    suggested_reply: '',
    summary: 'Could not parse AI response'
  };
}

return {
  json: {
    ...parsed,
    original_subject: $items('Email Read IMAP')[0].json.subject,
    original_from: $items('Email Read IMAP')[0].json.from.value[0].address,
    message_id: $items('Email Read IMAP')[0].json.messageId
  }
};
```

### Node 4: Route Based on Category

Add an **IF** node (or Switch node) to route each email based on its category. Here's where you decide what actually happens:

| Category | Action |
|----------|--------|
| **Urgent** | Move to `Urgent` folder, send a Slack/Matrix notification immediately |
| **Support Request** | Move to `Support` folder, draft reply, notify support channel |
| **Client Communication** | Move to `Clients` folder, draft reply for review |
| **Invoice/Payment** | Move to `Finance` folder, notify accounting |
| **Internal** | Leave in inbox or move to `Internal` |
| **Newsletter/Promotional** | Move to `Newsletters` folder, no notification |

The "move" action uses the **Email Move** node (or IMAP Move) in n8n. The notifications use the **Slack** or **Matrix** node — we recommend [Matrix](https://matrix.org/) if you want to keep your internal messaging self-hosted too.

### Node 5: Save Draft Replies (Optional)

For emails where `needs_reply` is true, you can store the suggested reply rather than sending it automatically. We recommend **never auto-sending** — the AI draft should go into a review queue. You can:

- Save the draft to a **Directus** collection (a self-hosted headless CMS) for a simple review dashboard
- Append it to a shared **Nextcloud Notes** or Markdown file
- Send it to a dedicated `Drafts` mailbox folder for review in your email client

The point is that the AI does the first draft — the slow part — and you do the final review and send. This is the right division of labor. The AI saves you 80% of the effort; you keep 100% of the accountability.

## Step 4: Test and Tune

Before turning this loose on your real inbox, test it. Create a test mailbox, forward a sampling of real emails into it, and run the workflow manually. Check:

- **Classification accuracy:** Are invoices going to Finance? Are newsletters getting filtered out? Watch for false positives — a client email that gets categorized as "Newsletter" because it contains the word "update."
- **Reply quality:** Read the suggested replies. If they're too generic or too confident, adjust the prompt. Adding "If unsure whether a reply is needed, set needs_reply to false" reduces over-eager drafting.
- **Speed:** With Llama 3.2 3B on CPU, each email takes 1–3 seconds to process. If you get 50 emails overnight, that's under 2 minutes of processing — well within a 15-minute polling interval.

A common tuning issue: the model may misclassify transactional emails (order confirmations, shipping notifications) as "Support Request." Fix this by adding to the prompt: `"Transaction confirmations and shipping notifications are NOT support requests — categorize them as Newsletter/Promotional unless there's a problem indicated."` Prompt refinement is an iterative process. Budget an hour for it.

## What This Costs

Let's be concrete:

| Component | Cost |
|-----------|------|
| VPS (8GB RAM, 2 vCPU) | ~$6–$12/month |
| Ollama | Free (open source) |
| n8n (self-hosted) | Free (open source) |
| Domain + TLS | ~$10/year |
| **Total** | **~$8–$15/month flat, regardless of team size** |

Compare that to a SaaS email tool at $20/user/month. For a 10-person team, that's $200/month versus $15. And the SaaS version reads every email you send it. The self-hosted version never lets a message leave your network.

## What This System Does — and Doesn't — Do

**It does:**
- Sort your inbox automatically so you only see what needs a human
- Flag urgent messages so they don't get lost
- Draft first-pass replies to save you the blank-page problem
- Keep all processing on infrastructure you control

**It doesn't:**
- Send replies on your behalf (and it shouldn't — that's a trust boundary you don't want to cross)
- Handle complex multi-thread negotiations (those still need a human reading the full context)
- Replace your email client (it works alongside it, organizing what's already there)
- Work without a mail server that supports IMAP (most do, but check yours)

The system is a triage layer, not an autoresponder. Think of it as a really fast assistant who pre-sorts your mail and writes draft responses on a notepad for you to review. The assistant never sends anything. You stay in control.

## When to Use Cloud AI Instead

There's one honest caveat: if you're processing thousands of emails per day (a large customer service operation, for example), local inference on CPU may be too slow. At that volume, a GPU server or a cloud inference provider makes economic sense. But for a small business handling 50–200 emails a day — which is most of you reading this — a $10/month VPS with Ollama handles it comfortably.

The other scenario where cloud APIs make sense is if you need a very large model for nuanced classification (legal correspondence, for instance, where the difference between "this is a standard update" and "this is a deadline notice" matters a lot). In that case, running a 70B model locally requires serious hardware. But for standard business email — invoices, support, newsletters, client check-ins — a 3B model is more than enough.

## A Note on Privacy

This is the part that matters most and gets talked about least. Every email you send to a SaaS email tool is stored on their servers, often used to train or improve their models, and covered by their privacy policy — which can change. You have no audit trail. You can't inspect what they do with your data.

When you self-host with Ollama, your emails are processed in memory on a server you control and then discarded. There's no training data retention. There's no third party with a copy. For businesses in regulated industries — legal, healthcare, finance — this isn't a nice-to-have. It's the difference between a defensible compliance posture and a liability.

## Getting Started

If you already have n8n and Ollama running from our previous guides, this workflow takes about 45 minutes to build. If you're starting from scratch, budget an afternoon for the full stack setup. Either way, the payoff is immediate: the first morning after you turn it on, you'll open your inbox to find it already sorted, with draft replies waiting for the messages that need them.

Email triage is one of the highest-ROI automations a small business can build. It touches every employee, every day, and the manual version of it is pure overhead — work that has to happen but creates no value. Automating it doesn't just save time. It removes a daily source of friction that makes people dread opening their inbox.

---

*Want help setting up an email triage system tailored to your business? ARDOT Consulting builds self-hosted automation workflows using open-source tools — no vendor lock-in, no per-seat fees, your data stays on your infrastructure. [Get in touch](/#contact) and we'll map out what's possible for your team.*