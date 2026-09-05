---
layout: post
title: "Building a Customer Support Chatbot with Open Source AI: A Step-by-Step Guide"
date: 2026-09-24
author: "ARDOT Consulting"
tags: [chatbot, customer-support, ollama, n8n, open-source, tutorial, self-hosting]
excerpt: "Learn how to build a self-hosted customer support chatbot using Ollama and n8n — no API fees, no vendor lock-in, and your customer data never leaves your server."
---

Every small business owner has been there: the same five questions landing in your inbox every single day. "What are your hours?" "Do you ship internationally?" "How do I reset my password?" "Can I get a refund?" "Where's my order?"

You don't need a team of agents to handle these. You don't need to pay $200/month for a SaaS chatbot platform that sends your customer conversations to a third-party server. And you don't need to write a single line of Python.

In this guide, we'll build a customer support chatbot using two open source tools — **Ollama** for the AI brain and **n8n** for the workflow glue — that runs entirely on your own server. No API fees. No data leaving your infrastructure. No vendor lock-in.

## What We're Building

Here's the architecture at a glance:

```
Customer Message (Web/Email)
        │
        ▼
   ┌─────────┐
   │   n8n   │  ← Webhook receives message
   │ Workflow │
   └────┬────┘
        │
        ▼
   ┌─────────┐
   │ Ollama  │  ← Local LLM generates response
   │ (Llama  │    using your knowledge base
   │  3.2)   │
   └────┬────┘
        │
        ▼
   ┌─────────┐
   │   n8n   │  ← Sends response back to
   │ Workflow │    customer via web/email
   └─────────┘
```

The chatbot will:
1. Accept messages from a web widget or email
2. Check a knowledge base of your FAQ documents
3. Generate a helpful response using a local LLM
4. Escalate to a human when it's not confident

## Why Open Source for Customer Support?

Before we dive in, let's address the elephant in the room. There are dozens of SaaS chatbot platforms — Intercom, Zendesk AI, Drift, Tidio. Why build your own?

| Factor | SaaS Chatbot | Self-Hosted (Ollama + n8n) |
|--------|-------------|---------------------------|
| Monthly cost | $50–$500+/month | ~$20/month (VPS hosting) |
| Data privacy | Conversations sent to vendor | Stays on your server |
| Customization | Limited to platform features | Full control |
| Vendor lock-in | High — migration is painful | Low — open standards |
| Setup time | 1–2 hours | 3–4 hours (one-time) |
| Ongoing maintenance | Vendor handles it | You handle updates |

The tradeoff is clear: a bit more setup work in exchange for lower costs, full data ownership, and no dependency on a vendor's pricing decisions. For businesses handling sensitive customer information — healthcare, legal, finance — keeping data on your own server isn't just cheaper. It's often the legally safer option.

## Prerequisites

You'll need:

- A Linux server (VPS) with at least 8GB RAM and 4 CPU cores — a $20/month VPS from Hetzner, OVH, or DigitalOcean works fine
- Docker and Docker Compose installed
- Basic comfort with the terminal (copy-paste level)
- About 30 minutes of your FAQ and policy documents in text form

If you've never used Docker before, don't worry — we'll use `docker-compose` files that just work.

## Step 1: Set Up Ollama

Ollama runs large language models locally. Think of it as a lightweight, self-hosted alternative to the OpenAI API — except there's no API key, no per-token billing, and no data leaving your server.

Create a directory for your chatbot stack:

```bash
mkdir -p ~/support-bot && cd ~/support-bot
```

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

volumes:
  ollama_data:
```

Start Ollama:

```bash
docker compose up -d
```

Now pull a model. For customer support, **Llama 3.2 (3B parameter)** is an excellent choice — it's fast, capable enough for FAQ-style responses, and runs comfortably on 8GB RAM:

```bash
docker exec -it ollama ollama pull llama3.2
```

This downloads about 2GB and takes a few minutes. Once it's done, verify the model works:

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "What are your business hours?",
  "stream": false
}'
```

You should get a JSON response with a generated answer. Ollama is running.

### Choosing the Right Model

Llama 3.2 (3B) is our default recommendation, but here's how to decide if you need something bigger:

| Model | Size | RAM Needed | Best For |
|-------|------|-----------|----------|
| Llama 3.2 (3B) | 2GB | 8GB | FAQ-style support, simple Q&A |
| Llama 3.1 (8B) | 4.7GB | 16GB | Complex responses, multi-turn conversations |
| Qwen 2.5 (7B) | 4.7GB | 16GB | Strong multilingual support |
| Mistral (7B) | 4.1GB | 16GB | Concise, professional tone |

For most small business support bots, the 3B model is more than enough. You can always upgrade later by pulling a bigger model — no need to change your workflow.

## Step 2: Set Up n8n

n8n is the workflow automation tool that ties everything together. It receives customer messages, sends them to Ollama, and returns the response.

Add n8n to your `docker-compose.yml`:

```yaml
version: "3.8"

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=ChangeThisPassword123
      - WEBHOOK_URL=https://support.yourdomain.com/
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  ollama_data:
  n8n_data:
```

**Important:** Change the password before deploying. Use a strong, unique password.

Restart the stack:

```bash
docker compose up -d
```

Open `http://YOUR_SERVER_IP:5678` in your browser. Log in with the credentials you set. You're now in the n8n workflow editor.

### Securing the Setup

Before going live, put n8n behind a reverse proxy with HTTPS. Using Caddy (the simplest option):

```bash
# Add Caddy to docker-compose.yml
  caddy:
    image: caddy:latest
    container_name: caddy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
    restart: unless-stopped
```

Create a `Caddyfile`:

```
support.yourdomain.com {
    reverse_proxy n8n:5678
}
```

Caddy automatically provisions and renews Let's Encrypt TLS certificates. Now your webhook URLs are HTTPS-encrypted end to end.

## Step 3: Build the Knowledge Base

The chatbot is only as good as the information you give it. We'll create a system prompt that contains your business knowledge.

Create a text file with your FAQs, policies, and common questions. Here's a template — replace with your own information:

```text
BUSINESS: Acme Widgets Inc.
HOURS: Monday–Friday, 9 AM – 5 PM EST. Closed weekends and holidays.
SHIPPING: Free shipping on orders over $50. Standard delivery 3–5 business days. Express delivery 1–2 business days ($15 extra). We ship to US and Canada only.
RETURNS: 30-day return policy. Items must be unused and in original packaging. Contact support@acmewidgets.com for a return label.
PASSWORD RESET: Go to acmewidgets.com/reset, enter your email, click the link in the email. If you don't receive the email within 5 minutes, check spam folder.
ORDER STATUS: Go to acmewidgets.com/track and enter your order number. Order numbers start with "AW" followed by 6 digits.
REFUNDS: Refunds processed within 5–7 business days to the original payment method.
CONTACT: support@acmewidgets.com or call 1-800-555-0100 during business hours.
ESCALATION: If the customer is angry, mentions legal action, or asks something not covered above, tell them you'll connect them with a human agent.
```

This is your **knowledge base document**. Keep it concise — under 2,000 words works best with smaller models. Every few weeks, review what questions the bot couldn't answer and add those to this document.

## Step 4: Build the n8n Workflow

Now for the fun part. In the n8n editor, create a new workflow.

### Node 1: Webhook Trigger

Add a **Webhook** node:
- **HTTP Method:** POST
- **Path:** `chat`
- **Response Mode:** Using 'Respond to Webhook' node

This creates a URL like `https://support.yourdomain.com/webhook/chat`. When a customer sends a message, it hits this endpoint.

### Node 2: Build the Prompt

Add a **Set** node (now called "Edit Fields") to construct the prompt for Ollama:

```json
{
  "system_prompt": "You are a customer support agent for Acme Widgets Inc. Use ONLY the information below to answer questions. Be friendly, concise, and helpful. If you don't know the answer, say you'll connect them with a human agent. Never make up information.\n\nKNOWLEDGE BASE:\n{{ $json.knowledge_base }}",
  "user_message": "{{ $json.body.message }}"
}
```

In practice, you'll store the knowledge base text in a separate file or n8n variable and reference it here. This keeps the prompt manageable.

### Node 3: Call Ollama

Add an **HTTP Request** node:
- **Method:** POST
- **URL:** `http://ollama:11434/api/chat`
- **Body Type:** JSON

```json
{
  "model": "llama3.2",
  "stream": false,
  "messages": [
    {
      "role": "system",
      "content": "{{ $json.system_prompt }}"
    },
    {
      "role": "user",
      "content": "{{ $json.user_message }}"
    }
  ],
  "options": {
    "temperature": 0.3,
    "num_predict": 300
  }
}
```

**Why temperature 0.3?** Low temperature means the model produces more consistent, factual responses — exactly what you want for customer support. Higher temperatures produce more creative but less reliable answers.

**Why num_predict 300?** This limits the response to roughly 300 tokens (about 200 words). Support responses should be concise. Nobody wants a chatbot that writes essays.

### Node 4: Check for Escalation

Add an **If** node to check whether the response mentions escalation:

```json
{
  "conditions": {
    "string": [
      {
        "value1": "{{ $json.message.content }}",
        "operation": "contains",
        "value2": "human agent"
      }
    ]
  }
}
```

If the condition is true (the bot wants to escalate), route to a node that:
- Sends an email or Mattermost notification to your support team
- Tells the customer a human will follow up shortly

If false, route directly to the response node.

### Node 5: Respond to Webhook

Add a **Respond to Webhook** node:
- **Respond With:** JSON

```json
{
  "reply": "{{ $json.message.content }}",
  "status": "ok"
}
```

This sends the chatbot's response back to whoever called the webhook.

### The Complete Workflow

Here's what the final workflow looks like:

```
Webhook (POST /chat)
    │
    ▼
Set Node (build prompt with knowledge base)
    │
    ▼
HTTP Request (call Ollama API)
    │
    ▼
If Node (contains "human agent"?)
    ├── YES → Email Notification → Respond with "escalating" message
    └── NO  → Respond to Webhook with bot reply
```

Save and activate the workflow. Your chatbot endpoint is now live.

## Step 5: Connect a Web Widget

The easiest way to add a chat widget to your website is with a small HTML snippet. You don't need a fancy framework — just a simple form that posts to your webhook:

```html
<div id="chat-widget">
  <div id="chat-messages"></div>
  <input type="text" id="chat-input" placeholder="Ask a question..." />
  <button onclick="sendMessage()">Send</button>
</div>

<script>
async function sendMessage() {
  const input = document.getElementById('chat-input');
  const messages = document.getElementById('chat-messages');

  const message = input.value.trim();
  if (!message) return;

  // Show user message
  messages.innerHTML += `<p><strong>You:</strong> ${message}</p>`;
  input.value = '';

  // Call the n8n webhook
  const response = await fetch('https://support.yourdomain.com/webhook/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: message })
  });

  const data = await response.json();

  // Show bot response
  messages.innerHTML += `<p><strong>Bot:</strong> ${data.reply}</p>`;
  messages.scrollTop = messages.scrollHeight;
}
</script>
```

Add some CSS to style it, and you have a functional chat widget. No JavaScript framework. No npm packages. Just a form that talks to your self-hosted AI.

### Connecting via Email Instead

If you'd rather handle email-based support, add an **IMAP Trigger** node in n8n that monitors your support inbox. When a new email arrives, the same workflow runs — Ollama generates a response, and an **Send Email** node replies to the customer. This turns your support@ inbox into an AI-assisted auto-responder.

## Step 6: Test and Tune

Before going live, test with real customer questions. Here's a quick test script:

```bash
# Test 1: Basic FAQ
curl -X POST https://support.yourdomain.com/webhook/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What are your business hours?"}'

# Test 2: Policy question
curl -X POST https://support.yourdomain.com/webhook/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Can I return a product after 40 days?"}'

# Test 3: Escalation trigger
curl -X POST https://support.yourdomain.com/webhook/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "I want to speak to your manager about a lawsuit"}'
```

Expected behavior:
- **Test 1:** Responds with your business hours from the knowledge base
- **Test 2:** Explains the 30-day return policy and suggests contacting support
- **Test 3:** Escalates to a human agent

If the bot gives wrong answers, the fix is almost always in the knowledge base document, not the model. Add the missing information and test again. This is the iterative improvement loop:

```
Bot gives wrong answer
    │
    ▼
Identify what information was missing
    │
    ▼
Add it to the knowledge base document
    │
    ▼
Test the same question again
    │
    ▼
Repeat until accurate
```

## Monitoring and Maintenance

### Track What the Bot Can't Answer

Add a **Postgres** or **SQLite** node to log every conversation where the bot escalated to a human. Review these weekly — they're your roadmap for what to add to the knowledge base next.

```sql
CREATE TABLE escalation_log (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT NOW(),
    customer_message TEXT,
    bot_response TEXT,
    resolved BOOLEAN DEFAULT FALSE
);
```

### Watch Server Resources

Ollama uses CPU and RAM when generating responses. Monitor with:

```bash
docker stats ollama
```

If responses are slow (more than 5–8 seconds), consider:
- Upgrading to a VPS with more CPU cores
- Switching to a smaller model (if accuracy allows)
- Adding a GPU (Ollama supports NVIDIA GPUs natively)

### Update Regularly

```bash
# Update Ollama
docker compose pull ollama && docker compose up -d ollama

# Update n8n
docker compose pull n8n && docker compose up -d n8n

# Pull a newer model if available
docker exec -it ollama ollama pull llama3.2
```

Set a calendar reminder to do this monthly. Updates take about 5 minutes.

## Cost Breakdown

Here's what this setup actually costs per month:

| Item | Cost |
|------|------|
| VPS (8GB RAM, 4 cores) | $15–$25/month |
| Domain name (if new) | $1/month (amortized) |
| Ollama | $0 (open source) |
| n8n (self-hosted) | $0 (open source) |
| Caddy (reverse proxy) | $0 (open source) |
| LLM API costs | $0 (running locally) |
| **Total** | **$15–$25/month** |

Compare that to Intercom's cheapest plan at $74/month (with limited AI features) or Zendesk Suite at $55/month per agent. Even factoring in the one-time 3–4 hour setup, you break even within the first month.

## Limitations (Be Honest About These)

This setup is powerful, but it's not magic. Here's what it **can't** do:

- **Access your database in real-time.** It can't look up a specific customer's order. For that, you'd need to add a database query node in n8n that fetches order data and includes it in the prompt.
- **Handle complex multi-turn conversations.** Llama 3.2 (3B) is good for single-question responses but loses context in long back-and-forth chats. For multi-turn support, consider Llama 3.1 (8B) and add conversation history to the prompt.
- **Understand images or attachments.** The 3B model is text-only. If you need image understanding (e.g., customers sending photos of damaged products), use Llama 3.2 Vision or a multimodal model like LLaVA.
- **Be 100% accurate.** All LLMs hallucinate occasionally. The escalation trigger catches most issues, but you should review logs weekly and refine the knowledge base.

## Going Further

Once the basic chatbot is working, here are incremental upgrades you can add:

1. **Database integration:** Add a Postgres node in n8n to look up customer orders and include real data in the prompt. This transforms the bot from a FAQ responder to a personalized support agent.

2. **Conversation logging:** Store every conversation in a database for quality review. Use Metabase (open source) to build dashboards showing common questions, escalation rates, and response times.

3. **Multi-channel:** Connect the same n8n workflow to Telegram, Mattermost, or Signal using their respective n8n trigger nodes. One bot, multiple channels.

4. **RAG (Retrieval-Augmented Generation):** For larger knowledge bases (100+ documents), set up a vector database like Qdrant or ChromaDB alongside Ollama. This lets the bot search your documents semantically rather than stuffing everything into the system prompt.

5. **Sentiment detection:** Add a second LLM call that classifies sentiment (happy, neutral, frustrated, angry). Route angry customers to humans immediately, before the support bot even tries to respond.

## Wrapping Up

A self-hosted customer support chatbot is one of the highest-ROI automation projects a small business can undertake. The setup takes an afternoon, the monthly cost is negligible, and the payoff — hours saved on repetitive questions — is immediate.

The key insight is this: **you don't need a $500/month AI platform to handle "what are your hours?"** You need a lightweight model, a workflow tool, and a well-maintained knowledge base. The open source tools exist. The hosting is cheap. The only thing standing between you and an automated support bot is a few hours of setup.

And unlike a SaaS platform, this one is yours. No vendor can raise prices on you. No one can shut off your API access. Your customer conversations stay on your server, under your control.

---

*Ready to automate your customer support without the SaaS price tag? [Contact ARDOT Consulting](/) — we help small businesses design, deploy, and maintain open source AI automation that fits their needs and their budget.*