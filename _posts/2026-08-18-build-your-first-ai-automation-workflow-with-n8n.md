---
layout: post
title: "How to Build Your First AI Automation Workflow with n8n"
date: 2026-08-18
author: "ARDOT Consulting"
tags: [n8n, automation, tutorial, workflow, ollama, self-hosting]
excerpt: "A step-by-step guide to building a real automation workflow with n8n and a local LLM — no cloud APIs, no per-message fees, no vendor lock-in."
---

You've heard about AI automation. You've seen the demos. But when it comes to actually *building* something — something real, something that runs on your data, under your control — the gap between "cool demo" and "working workflow" feels wide.

This guide closes that gap. We'll build a complete automation workflow using **n8n** (an open-source workflow engine) and **Ollama** (a local LLM runtime). The workflow does something every business deals with: monitors an email inbox, uses AI to extract key information from incoming messages, and posts a structured summary to a team chat channel.

No cloud AI APIs. No per-message fees. No data leaving your network. Everything runs on a machine you control.

## What We're Building

Here's the scenario: your team receives customer inquiries by email. Today, someone manually reads each one, pulls out the important details (who sent it, what they're asking, how urgent it is), and shares that with the team in a chat channel. It's repetitive, it's slow, and it's exactly the kind of task AI handles well.

Our workflow will:

1. **Watch** a mailbox for new messages (via IMAP)
2. **Extract** structured data from each email using a local LLM
3. **Format** the output into a clean chat notification
4. **Post** the summary to a Mattermost or Slack-compatible channel

Here's what the architecture looks like:

```
┌──────────┐     ┌─────────┐     ┌──────────┐     ┌────────────┐
│  IMAP    │────▶│  n8n    │────▶│  Ollama  │────▶│  Mattermost │
│  Inbox   │     │  Engine │     │  (LLM)   │     │  / Slack    │
└──────────┘     └─────────┘     └──────────┘     └────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  n8n Format  │
                 │  + Route     │
                 └──────────────┘
```

## Prerequisites

You'll need:

- A machine (cloud VPS, home server, or even a beefy desktop) with **8 GB RAM minimum** — 16 GB recommended if you're also running Ollama on the same box
- **Docker** and **Docker Compose** installed
- An email account with IMAP access
- A Mattermost or Slack-compatible webhook URL (we'll cover both)

No programming experience required. If you can edit a YAML file and click through a web UI, you can do this.

## Step 1: Set Up Ollama (Your Local AI)

Ollama runs large language models locally. Think of it as a lightweight server that loads a model and exposes a simple HTTP API — similar to the OpenAI API, but running entirely on your hardware.

### Install Ollama

On Linux:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

On macOS, download the installer from [ollama.com](https://ollama.com). On Windows, use the Windows preview build or run it in WSL2.

### Pull a Model

For email summarization and information extraction, you want a model that's good at instruction-following but small enough to run on modest hardware. **Llama 3.2 (3B parameters)** is an excellent choice — it runs on 4 GB of RAM and handles structured extraction well:

```bash
ollama pull llama3.2
```

If you have 16 GB+ RAM, consider **Qwen 2.5 (7B)** or **Mistral (7B)** for better accuracy on complex emails:

```bash
ollama pull qwen2.5:7b
```

### Verify It Works

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Say hello in one sentence.",
  "stream": false
}'
```

You should get a JSON response with a `response` field containing the model's reply. That's your AI engine, running locally, no API key needed.

## Step 2: Set Up n8n with Docker Compose

n8n is a workflow automation tool — think of it as a visual programming environment where you connect nodes (triggers, transformations, actions) into a pipeline. It's self-hostable, fair-code licensed, and has a free community edition.

Create a `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Bring it up:

```bash
docker compose up -d
```

Open `http://localhost:5678` in your browser. You'll be walked through a quick owner-account setup. Once you're in, you'll see the workflow canvas — a blank grid where you'll build your automation.

## Step 3: Build the Workflow — Node by Node

### Node 1: Email Trigger (IMAP)

1. Click **Add first step** → search for **Email Trigger (IMAP)**
2. Configure your IMAP credentials:
   - **Host:** `imap.yourprovider.com` (e.g., `imap.fastmail.com`, `imap.tutamail.com`)
   - **Port:** 993 (SSL)
   - **Username:** your email address
   - **Password:** your email password or app-specific password
3. Set **Polling interval** to `2 minutes`
4. Set **Action on email** to `Mark as read` (so it doesn't re-process messages)

Test the node — it should fetch the most recent emails from your inbox. Each email becomes a JSON object with `subject`, `textBody`, `from`, and `date` fields.

### Node 2: Ollama (Information Extraction)

This is where the AI does its work. We'll send each email's body to Ollama with a prompt that asks for structured extraction.

1. Add a new node → search for **HTTP Request**
2. Configure:
   - **Method:** `POST`
   - **URL:** `http://host.docker.internal:11434/api/generate`
   - **Body Content Type:** `JSON`
   - **Body:**

```json
{
  "model": "llama3.2",
  "stream": false,
  "prompt": "You are an email analysis assistant. Extract key information from the following email and respond in valid JSON only — no markdown, no explanation.\n\nEmail subject: {{ $json.subject }}\nEmail from: {{ $json.from.value[0].address }}\nEmail body:\n{{ $json.textBody }}\n\nExtract these fields:\n- sender_name: the sender's display name\n- inquiry_type: one of [sales, support, billing, general, other]\n- summary: a one-sentence summary of what they're asking\n- urgency: one of [low, medium, high]\n- key_dates: any dates or deadlines mentioned, or null\n\nRespond with valid JSON only."
}
```

> **Note on `host.docker.internal`:** Since n8n runs inside Docker and Ollama runs on the host, use `host.docker.internal` to reach the host machine. On Linux, you may need to add `extra_hosts: ["host.docker.internal:host-gateway"]` to your n8n service in docker-compose.yml.

Test the node. You should get back something like:

```json
{
  "sender_name": "Jane Smith",
  "inquiry_type": "sales",
  "summary": "Requesting a quote for automation consulting services for a 20-person firm.",
  "urgency": "medium",
  "key_dates": "Needs response by August 25"
}
```

### Node 3: Format the Message

Add a **Set** node (now called "Edit Fields" in newer n8n versions) to construct the chat message:

| Field Name | Value |
|---|---|
| `message_text` | (expression below) |

```
📧 New inquiry from {{ $json.sender_name }}

Type: {{ $json.inquiry_type }}
Urgency: {{ $json.urgency }}
Summary: {{ $json.summary }}
{{ $json.key_dates ? '⏰ ' + $json.key_dates : '' }}

— Routed by AI · n8n workflow
```

### Node 4: Post to Chat (Mattermost or Slack)

**For Mattermost (incoming webhook):**

1. Add an **HTTP Request** node
2. **Method:** `POST`
3. **URL:** `https://your-mattermost.com/hooks/your-webhook-key`
4. **Body Content Type:** `JSON`
5. **Body:**

```json
{
  "text": "{{ $json.message_text }}"
}
```

**For Slack (incoming webhook):**

Same setup, just point the URL at your Slack webhook URL. The body format is identical — both platforms accept `{"text": "..."}`.

### Enable the Workflow

Click the toggle in the top-right corner to **activate** the workflow. n8n will now poll your inbox every 2 minutes, process new emails through the LLM, and post summaries to your chat.

## Step 4: Test It End-to-End

Send a test email to your monitored inbox:

> **Subject:** Interested in automation services
>
> Hi there,
>
> I'm Jane Smith, Operations Manager at Acme Consulting. We're a 20-person firm and we're drowning in manual data entry. I'd like a quote for automation consulting — specifically around invoice processing and CRM automation.
>
> We're evaluating vendors and need responses by August 25.
>
> Thanks,
> Jane

Within 2 minutes, you should see a notification in your chat channel:

```
📧 New inquiry from Jane Smith

Type: sales
Urgency: medium
Summary: Requesting a quote for automation consulting services for a 20-person firm.
⏰ Needs response by August 25

— Routed by AI · n8n workflow
```

That's a complete, working AI automation — built with entirely open-source tools, running on your infrastructure.

## Common Pitfalls (and How to Avoid Them)

### 1. Ollama is slow on first call

The first request to a model loads it into memory, which can take 10-30 seconds. Subsequent calls are fast. If your workflow times out, increase the HTTP Request node's timeout setting (default is 30s — bump it to 60s for safety).

### 2. LLM outputs inconsistent JSON

Small models sometimes wrap JSON in markdown code blocks or add explanatory text. Add a **Code** node after Ollama to strip non-JSON content:

```javascript
// Extract JSON from potentially noisy LLM output
const raw = $input.item.json.response;
const match = raw.match(/\{[\s\S]*\}/);
if (match) {
  return [{ json: JSON.parse(match[0]) }];
}
return [{ json: { error: "Could not parse LLM output", raw } }];
```

### 3. Emails pile up when you're testing

Set the Email Trigger to **fetch only unread messages** and **mark as read** after processing. During testing, send yourself fresh emails rather than re-processing old ones.

### 4. n8n can't reach Ollama inside Docker

If `host.docker.internal` doesn't resolve on Linux, add this to your n8n service in docker-compose.yml:

```yaml
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Then restart with `docker compose up -d`.

## How Much Does This Cost?

Here's the honest breakdown:

| Component | Cost |
|---|---|
| n8n (self-hosted community edition) | Free |
| Ollama + Llama 3.2 | Free (open source) |
| Mattermost (self-hosted, up to 10K users) | Free |
| VPS to run it all (4 vCPU, 8 GB RAM) | ~$12–20/month |
| Email account with IMAP | ~$3–5/month (Fastmail, Tuta, etc.) |
| **Total** | **~$15–25/month, flat** |

Compare that to a SaaS automation platform charging $0.01–0.10 per workflow execution plus $0.002 per 1K LLM tokens. At 500 emails/month, you'd pay $5–50 in execution fees plus token costs. At 5,000 emails/month, the SaaS bill could hit $500+ while your self-hosted cost stays at $25.

The trade-off is that you're responsible for keeping the server running. But for most small businesses, a single VPS with Docker Compose and `restart: unless-stopped` is remarkably reliable.

## Where to Go from Here

Once you have this workflow running, you can extend it in dozens of directions:

- **Add sentiment analysis** — have the LLM flag angry or confused customers for priority handling
- **Route by inquiry type** — send sales inquiries to your sales channel, support tickets to a support queue
- **Auto-respond** — generate a draft reply and post it for human review before sending
- **Log to a database** — store every inquiry in Directus or Odoo for analytics
- **Weekly digest** — add a scheduled trigger that summarizes the week's inquiries into a report

The beauty of n8n's node-based architecture is that each extension is just another node on the canvas. You're not rewriting code — you're snapping together building blocks.

## Why This Approach Beats Cloud-Only Automation

There's nothing wrong with using cloud APIs if that fits your constraints. But for many businesses — especially those handling customer communications, legal documents, financial data, or healthcare records — keeping AI processing on your own infrastructure matters:

- **No data leaves your network.** Emails, customer information, and LLM interactions stay on your server.
- **No per-message costs.** Run 10 emails or 10,000 — the cost is the same.
- **No vendor lock-in.** Swap models, swap chat platforms, swap email providers — the workflow structure stays the same.
- **Full audit trail.** Every step runs on infrastructure you control, with logs you can inspect.

## Wrapping Up

You now have a working AI automation: it reads emails, extracts structured information with a local language model, and posts clean summaries to your team chat. The whole thing runs on open-source software, on infrastructure you control, for about $20/month.

The workflow we built here is a starting point, not a destination. The same pattern — trigger → AI extraction → formatting → action — applies to hundreds of business processes. Invoice processing. Customer support triage. Lead qualification. Meeting notes. Contract review. Once you're comfortable with the pattern, the applications multiply.

If you'd like help designing or implementing an automation workflow tailored to your business, [reach out through our contact form](/). We specialize in open-source, self-hosted AI automation for small and mid-sized businesses — no cloud lock-in, no per-message fees, no black boxes.