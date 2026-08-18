---
layout: post
title: "How to Build Your First AI Automation Workflow with n8n"
date: 2026-08-17
author: "ARDOT Consulting"
tags: [n8n, automation, tutorial, workflow, open-source, ollama]
excerpt: "A step-by-step guide to building a real AI automation workflow with n8n and a local LLM — no coding experience required, no cloud subscriptions."
---

# How to Build Your First AI Automation Workflow with n8n

You've read the articles. You've heard the pitch. AI automation is going to save your business time, reduce errors, and let your team focus on real work instead of busywork. But when you sit down to actually *do* it, the question is always the same: where do I start?

This post answers that. We're going to build a real, working automation workflow from scratch using **n8n** — an open source automation tool you can run on your own server — and **Ollama**, which lets you run AI models locally without sending your data to anyone.

By the end, you'll have a workflow that monitors an inbox, uses AI to figure out what each email is about, and posts a summary to a chat channel where your team can act on it. No cloud subscriptions. No per-task fees. No data leaving your infrastructure.

We'll go slow, explain every step, and be honest about what works and what doesn't.

---

## What You're Building

Before we touch any software, let's be clear about the goal. Here's the workflow in plain English:

1. **An email arrives** in a monitored inbox (e.g., `support@yourcompany.com`).
2. **n8n picks it up** automatically.
3. **An AI model reads the email** and extracts the key information: who sent it, what they want, how urgent it is.
4. **n8n posts a summary** to a Slack-compatible channel (we'll use [Mattermost](https://mattermost.com) as the open source option, but this works with Slack too if you already use it).

This is a real workflow we've deployed for clients. It takes email chaos and turns it into a searchable, prioritized chat feed. Total cost: whatever you pay for a small server (often under $20/month). Total time to set up: about an hour once you have the tools installed.

Why this specific workflow? Because it's useful on its own, and it teaches you the three building blocks of almost every AI automation: **a trigger** (something happens), **AI processing** (the model does something smart), and **an action** (send the result somewhere). Once you understand those three steps, you can build hundreds of variations.

---

## Step 1: Install n8n with Docker

n8n is a visual workflow builder. Think of it as a flowchart where each box does something — read an email, call an API, send a message. You drag boxes onto a canvas, connect them, and configure each one. No coding required for most workflows, though you *can* drop into code when you need to.

The cleanest way to run n8n is with Docker. Docker is a tool that packages software into isolated containers, so you don't have to worry about conflicting dependencies or messing up your system. If you've never used Docker, that's fine — you only need two commands.

First, create a directory for your n8n setup and a file called `docker-compose.yml` inside it:

```yaml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

This tells Docker to run n8n, keep it running, expose it on port 5678, and save your workflow data so it survives restarts. The `GENERIC_TIMEZONE` setting matters — n8n uses it for scheduled triggers. Set it to your actual timezone.

Now start it:

```bash
docker compose up -d
```

Open `http://localhost:5678` in your browser. You'll see n8n's setup screen. Create an owner account (this is just for your local instance — your password stays on your server). That's it. n8n is running.

**A note on security:** If you're running this on a public server instead of your local machine, put n8n behind a reverse proxy with HTTPS and a strong password. Never expose port 5678 directly to the internet. We use [Caddy](https://caddyserver.com) for this — it handles HTTPS automatically — but any reverse proxy works.

---

## Step 2: Install Ollama for Local AI

Now we need the AI part. Ollama is a tool that runs large language models on your own hardware. It's like having a private version of ChatGPT that lives on your server and never sends data anywhere.

Installing Ollama is a single command on Linux:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

On macOS or Windows, download the installer from [ollama.com](https://ollama.com). Once installed, pull a model:

```bash
ollama pull llama3.1
```

This downloads Meta's Llama 3.1 model — about 4.5 GB. It's a solid general-purpose model that runs on most modern hardware. If you have a machine with at least 16 GB of RAM, this will work. If you have a GPU, it'll be fast. If you don't, it'll be slower but still functional for this workflow.

Start the Ollama server (it usually starts automatically, but just in case):

```bash
ollama serve
```

Ollama now listens on `http://localhost:11434`. That's the address n8n will use to talk to it.

**Which model should you use?** For email classification and summarization, you don't need the biggest model available. Llama 3.1 (8B parameters) handles this well. If you want something lighter and faster, `qwen2.5:7b` or `mistral:7b` are good alternatives. All three are open source and run locally. Avoid the temptation to grab a 70B model unless you have serious hardware — it'll be too slow for a real-time workflow.

---

## Step 3: Build the Workflow in n8n

Now for the fun part. Open n8n in your browser and click **Add workflow**. You'll see an empty canvas. We're going to add four nodes:

1. **Email Trigger** — watches an inbox for new messages
2. **HTTP Request** — sends the email text to Ollama for analysis
3. **Code node** — formats the AI output
4. **HTTP Request** — posts the summary to chat

Let's go through each.

### Node 1: Email Trigger

n8n has a built-in **Email Trigger** node. Add it, and configure it with your inbox details:

- **Host:** your IMAP server (e.g., `imap.yourdomain.com`)
- **Port:** 993 (SSL)
- **Username:** `support@yourcompany.com`
- **Password:** your email password or app-specific password
- **Action:** "Mark as read" (so you don't process the same email twice)

Set the polling interval to whatever makes sense — every 5 minutes is a good start. n8n will check the inbox every 5 minutes and trigger the workflow when new mail arrives.

If your email provider uses OAuth (many modern providers do), n8n supports that too. Check n8n's credential setup for your specific provider.

### Node 2: Send Email to Ollama

Add an **HTTP Request** node after the email trigger. This is where we ask the local AI model to analyze the email.

Configure it like this:

- **Method:** POST
- **URL:** `http://localhost:11434/api/generate`
- **Body type:** JSON

For the body, we'll use n8n's expression syntax to pull in the email content and build a prompt:

```json
{
  "model": "llama3.1",
  "prompt": "You are an assistant that analyzes customer support emails. Read the email below and extract:\n1. Sender name\n2. What they need (one sentence)\n3. Urgency (low, medium, high)\n4. Suggested category (billing, technical, sales, general)\n\nFormat your response as plain text, one item per line. Do not add extra commentary.\n\nEmail:\n{{ $json.text }}",
  "stream": false
}
```

The `{{ $json.text }}` part is n8n's way of saying "insert the email body here." When the workflow runs, n8n replaces it with the actual email content before sending the request to Ollama.

This prompt is the heart of the workflow. It tells the model exactly what to do and how to format the answer. Good prompts are specific and structured. Bad prompts are vague ("summarize this") and give you unpredictable output.

### Node 3: Format the Output

The model returns a text response. We want to turn it into a clean message for chat. Add a **Code** node and use this JavaScript:

```javascript
const aiResponse = $input.item.json.response;
const emailSubject = $('Email Trigger').item.json.subject;
const sender = $('Email Trigger').item.json.from.value[0].address;

return {
  json: {
    message: `📧 *New support email*\n*From:* ${sender}\n*Subject:* ${emailSubject}\n\n*AI summary:*\n${aiResponse}`
  }
};
```

This pulls the original email's subject and sender, combines them with the AI summary, and formats it as a single message. The asterisks create bold text in Slack/Mattermost.

### Node 4: Post to Chat

Add another **HTTP Request** node. This one sends the formatted message to your chat platform.

If you're using Mattermost (open source, self-hosted), configure it with an incoming webhook:

- **Method:** POST
- **URL:** `http://your-mattermost-url/hooks/your-webhook-key`
- **Body type:** JSON
- **Body:**

```json
{
  "text": "{{ $json.message }}"
}
```

If you're using Slack, the setup is nearly identical — Slack also uses incoming webhooks with the same `text` field. The point is that n8n doesn't care which platform you use; the HTTP Request node works with anything that accepts a web request.

---

## Step 4: Test It

Before you activate the workflow, test it manually. n8n has a **Test workflow** button that runs the entire flow with sample data or, in the case of trigger nodes, waits for a real event.

Send a test email to your monitored inbox. Click **Test workflow**. Within a few seconds, you should see:

1. The Email Trigger node lights up green — it caught the email.
2. The first HTTP Request node lights up — Ollama analyzed it.
3. The Code node formats the output.
4. The final HTTP Request posts to your chat channel.

Check your chat channel. You should see something like:

```
📧 New support email
From: jane.customer@example.com
Subject: Invoice question

AI summary:
Sender name: Jane Customer
What they need: Wants to know why invoice #1043 is higher than expected
Urgency: medium
Suggested category: billing
```

That's a working AI automation. In about 60 seconds, an email went from an unopened inbox to a structured, categorized alert in your team chat — with zero manual effort.

---

## What This Costs You

Let's be transparent about the economics, because "open source" doesn't mean "free" — it means "no license fees." Here's the real cost breakdown:

| Item | Cost |
|------|------|
| n8n (self-hosted) | $0 software, your server |
| Ollama + Llama 3.1 | $0 software, your hardware |
| Server (VPS with 8GB RAM) | ~$10-20/month |
| Mattermost (self-hosted) | $0 software, same server or another |
| Email hosting | You probably already pay this |
| **Total new cost** | **~$10-20/month for the server** |

Compare that to a SaaS automation platform that charges per task, per user, or per AI call. If you process 1,000 emails a month, a per-task platform at $0.10/task would cost $100/month — and that's before the AI token charges. Your self-hosted setup processes 10 or 10,000 emails for the same flat server cost.

The trade-off is that *you* maintain the server. If n8n crashes at 2 AM, you're the one who notices and restarts it. For most small businesses, this trade-off is worth it. For others, a managed option makes sense. We're happy to help you figure out which side of that line you're on — [get in touch](/#contact).

---

## Common Pitfalls (and How to Avoid Them)

**The model gives weird or inconsistent output.** This is almost always a prompt problem. Your prompt needs to be specific about format and content. If the model rambling, add "Do not add extra commentary" or "Respond only with the requested information." If it's classifying things wrong, give it examples in the prompt: "For example, an email about a broken login should be categorized as 'technical'."

**Ollama is slow.** Local models on CPU-only hardware can take 10-30 seconds per email. That's fine for email triage (you're not waiting in real time), but not for a live chatbot. If speed matters, either use a smaller model (7B instead of 8B) or get a server with a GPU. You can also use a quantized model — Ollama handles this automatically when you pull a model.

**Emails aren't being picked up.** Check three things: the IMAP credentials are correct, the polling interval isn't too long, and n8n is actually running (Docker containers restart on reboot, but verify). Also make sure your email provider doesn't require app-specific passwords (Gmail does, for instance — though we'd recommend a privacy-focused provider like [ProtonMail](https://proton.me) or self-hosted [Mailcow](https://mailcow.email) instead).

**The workflow breaks after a few days.** This is usually a volume issue. If your inbox gets hundreds of emails a day, n8n processes them in batches and you might hit memory limits. Solution: add a filter node early in the workflow to skip newsletters, receipts, and other non-actionable mail before they reach the AI step.

---

## Where to Go Next

Once you have this workflow running, you've learned the pattern. Every AI automation is a variation of: **trigger → AI processing → action**. Here are some ways to extend what you just built:

- **Add auto-reply for simple cases.** If the AI classifies an email as "general" with low urgency, draft a reply and save it as a draft for human review.
- **Route to different channels.** Send billing emails to a `#billing` channel, technical issues to `#support`, sales inquiries to `#sales`.
- **Create tickets automatically.** Instead of (or in addition to) chat, create a ticket in an open source helpdesk like [FreeScout](https://freescout.net) or [Zammad](https://zammad.org).
- **Build a weekly digest.** Add a scheduled trigger that summarizes all emails from the week into a Monday morning report for your team.

The tools are the same. The pattern is the same. You're just changing the trigger, the prompt, or the destination.

---

## The Honest Caveat

This workflow works. We've built it for clients, and it saves real time. But it's not magic. The AI will occasionally misclassify an email. The summary will sometimes miss a nuance that a human would catch. Your server will need a restart now and then.

That's fine. The goal isn't perfection — it's *good enough to save your team hours of manual sorting*. You keep humans in the loop for anything that matters. The AI handles the 80% of email that's routine, so your people can spend their time on the 20% that actually requires judgment.

If that trade-off makes sense for your business, the best next step is to try it. Install n8n, install Ollama, and build this workflow. It'll take you an evening, and you'll come out the other side understanding exactly how AI automation works — not from a blog post, but from doing it.

---

## Want Help Setting This Up?

If you'd rather have someone walk you through it — or just build the whole thing for you — that's what we do. ARDOT Consulting designs and deploys open source AI automation for small businesses. No vendor lock-in, no per-task fees, no data leaving your control.

[Reach out through our contact form](/#contact) and tell us what you're trying to automate. We'll give you a straight answer about whether it's worth doing and what it'll take.