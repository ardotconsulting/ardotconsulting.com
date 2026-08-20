---
layout: post
title: "Running Local LLMs with Ollama: A Practical Guide for Businesses"
date: 2026-08-20
author: "ARDOT Consulting"
tags: [ollama, local-llm, self-hosting, privacy, ai]
excerpt: "A complete guide to running AI language models on your own hardware with Ollama — no cloud API, no subscription, no data leaving your network."
---

If you've been curious about AI language models but hesitant to send your company's data to a third-party API, there's good news: you can run capable AI models on your own hardware, entirely offline, using an open source tool called Ollama. No subscription fees, no data leaving your network, no rate limits.

This guide walks through everything you need to know: what Ollama is, what hardware you'll need, how to install it, how to pick the right model, and how to connect it to other automation tools like n8n.

## What Is Ollama?

[Ollama](https://ollama.com) is an open source tool that lets you run large language models (LLMs) locally on your own computer or server. Think of it as a lightweight, user-friendly wrapper around the complex infrastructure normally required to run AI models. You install it, pull a model with a single command, and start chatting — no GPU programming experience needed.

Ollama handles model downloading, quantization (compressing models so they fit in less memory), and exposes a REST API that other applications can call. It runs on macOS, Linux, and Windows.

### Why local matters for your business

When you use a cloud AI API — OpenAI, Anthropic, or any SaaS AI product — your data leaves your network. Every prompt, every document you upload, every question you ask gets sent to someone else's servers. For many businesses, that's a dealbreaker:

- **Legal firms** can't send client documents to third-party servers without risk of privilege waiver
- **Healthcare providers** are bound by HIPAA and can't transmit patient data externally
- **Financial services** handle regulated data with strict residency requirements
- **Any business** with trade secrets, proprietary processes, or competitive intelligence

Running models locally with Ollama eliminates these concerns. The data never leaves your machine. You control the hardware, the model, and the logs.

## Hardware Requirements: What Do You Actually Need?

This is the question everyone asks first. The answer depends on which model you want to run, but here's a practical breakdown.

### The key metric: RAM (and VRAM)

LLMs are memory-bound. The size of the model determines how much RAM you need. Models are measured in **parameters** — billions of them. A 7-billion-parameter model (7B) needs roughly 4–5 GB of RAM to run. A 13B model needs about 8 GB. Larger models need more.

Here's a rough guide:

| Model Size | RAM Required | Example Models | Good For |
|------------|-------------|----------------|----------|
| 3B–4B | 3–4 GB | Phi-3, Qwen2.5-3B | Lightweight tasks, older hardware |
| 7B–8B | 5–6 GB | Llama 3.1, Qwen2.5-7B | General-purpose, most businesses |
| 13B–14B | 8–10 GB | Qwen2.5-14B | Higher-quality output, more nuanced tasks |
| 32B+ | 20+ GB | Qwen2.5-32B, Llama 3.1 70B (quantized) | Complex reasoning, professional writing |

### CPU vs GPU: Do you need a graphics card?

You **can** run Ollama on a CPU-only machine. It works. But it's slower — typically 5–15 tokens per second for a 7B model on a modern CPU, compared to 40–80+ tokens per second on a decent GPU.

For most business use cases (document summarization, data extraction, email drafting, internal Q&A), CPU-only performance is perfectly usable. You're not building a real-time chatbot for thousands of concurrent users — you're processing documents and generating text at a pace humans can read.

**If you want GPU acceleration:**
- **macOS (Apple Silicon):** Ollama automatically uses the M-series unified memory. An M2/M3 Mac with 16 GB unified memory can comfortably run 7B–13B models. This is one of the best local AI setups you can buy.
- **Linux/Windows with NVIDIA:** Any RTX-series card with 8 GB+ VRAM works well. A used RTX 3060 (12 GB) is an excellent budget choice.
- **No GPU:** Fine for 7B and smaller models. Expect to wait a few seconds longer per response.

## Installing Ollama

### macOS or Windows

Download the installer from [ollama.com](https://ollama.com) and run it. That's it. Ollama runs as a background service and is available from your terminal immediately.

### Linux

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This installs the Ollama binary, sets it up as a systemd service, and starts it automatically. Verify it's running:

```bash
ollama --version
# ollama version is 0.x.x
```

### Docker (recommended for servers)

If you're running Ollama on a server, Docker is the cleanest approach:

```bash
docker run -d \
  --name ollama \
  -p 11434:11434 \
  -v ollama_data:/root/.ollama \
  --gpus=all \
  ollama/ollama
```

Omit `--gpus=all` if you don't have a GPU. The volume mount (`ollama_data`) persists your downloaded models across container restarts.

## Pulling Your First Model

Ollama uses a simple pull command, similar to Docker:

```bash
ollama pull llama3.1
```

This downloads the Llama 3.1 8B model (about 4.7 GB) and stores it locally. Once downloaded, you can run it:

```bash
ollama run llama3.1
```

You'll get an interactive prompt. Try asking it something:

```
>>> Summarize the key points of a vendor agreement in 5 bullet points.
```

The model runs entirely on your machine. No internet connection required after the initial download.

## Choosing the Right Model

Ollama's model library has dozens of options. Here are the ones we recommend for business use cases, ranked by practical value:

### Llama 3.1 (8B) — The All-Rounder

Meta's Llama 3.1 is currently the best general-purpose open model in the 8B class. It handles summarization, Q&A, drafting, and extraction well. It's fast, fits in 6 GB of RAM, and is a great default choice.

```bash
ollama pull llama3.1
```

### Qwen 2.5 (7B or 14B) — The Detail-Oriented One

Qwen 2.5, from Alibaba, excels at structured tasks: extracting data from documents, formatting output as JSON, following multi-step instructions precisely. If you're building automation workflows that need reliable structured output, Qwen is worth trying.

```bash
ollama pull qwen2.5
```

### Phi-3 Mini (3.8B) — The Lightweight Option

Microsoft's Phi-3 is remarkably capable for its size. At under 3 GB of RAM, it runs on almost anything — including older laptops and small VMs. It's ideal for simple tasks like classification, basic summarization, or triage.

```bash
ollama pull phi3
```

### Mistral (7B) — The European Option

Mistral is a French AI company producing high-quality open models. Mistral 7B is a solid alternative to Llama 3.1 with comparable performance. If you're operating under EU data regulations, using a European model from a European company has practical and symbolic value.

```bash
ollama pull mistral
```

### A practical recommendation

Start with **Llama 3.1 8B**. It's the safest default. If it's too slow on your hardware, switch to Phi-3. If you need higher-quality output and have the RAM, try Qwen 2.5 14B. You can install multiple models and switch between them freely — Ollama doesn't care.

## The Ollama API: Connecting to Other Tools

Ollama exposes a REST API on port 11434 by default. This is what makes it powerful — other applications can call it just like they'd call OpenAI's API, except everything stays local.

### A simple API call

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "Extract the company name and total amount from this invoice: 'ACME Corp, Invoice #1234, Total: $4,500.00'",
  "stream": false
}'
```

The response:

```json
{
  "model": "llama3.1",
  "response": "Company Name: ACME Corp\nTotal Amount: $4,500.00",
  "done": true,
  "total_duration": 1234567890,
  "eval_count": 15
}
```

### Connecting to n8n for automation workflows

If you're using [n8n](https://n8n.io) for automation (we covered this in our [n8n tutorial](/blog/2026/08/17/how-to-build-your-first-ai-automation-workflow-with-n8n/)), you can add Ollama as an HTTP Request node:

1. Add an **HTTP Request** node in your n8n workflow
2. Set method to `POST`
3. URL: `http://localhost:11434/api/generate`
4. Body:
```json
{
  "model": "llama3.1",
  "prompt": "={{ $json.document_text }}",
  "stream": false
}
```
5. The response is available as JSON in subsequent nodes

This lets you build workflows like: "When a new email arrives → extract the key information with Ollama → create a task in Odoo." All running locally, no external API calls.

### Connecting to other tools

Ollama's API is OpenAI-compatible (with the `/v1/chat/completions` endpoint). This means any tool that supports custom OpenAI API endpoints can use Ollama instead:

- **VS Code** with Continue extension — local AI code completion
- **Obsidian** with AI plugins — summarize notes locally
- **Any custom application** — just point it at `http://localhost:11434/v1`

## A Real-World Example: Document Q&A Bot

Here's a practical scenario: you want a system where employees can ask questions about your company's internal policies, and get answers grounded in your actual documents — without uploading those documents anywhere.

### Step 1: Install Ollama and pull a model

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.1
```

### Step 2: Create a custom prompt template

Ollama lets you create custom "modelfiles" (like Dockerfiles for models):

```bash
cat > Modelfile << 'EOF'
FROM llama3.1
SYSTEM """
You are an internal assistant for [Your Company]. Answer questions based on the context provided.
If you don't know the answer from the context, say "I don't have that information in the provided documents."
Be concise and professional.
"""
EOF

ollama create company-assistant -f Modelfile
```

### Step 3: Test it

```bash
ollama run company-assistant "What is our remote work policy?"
```

For full document Q&A (searching across many documents), you'd pair Ollama with a vector database like ChromaDB or Qdrant. That's a more advanced setup we'll cover in a future post, but the core insight is simple: Ollama handles the language understanding, the vector database handles the document retrieval, and everything runs on your hardware.

## Managing and Updating Models

### Listing installed models

```bash
ollama list
```

```
NAME            ID            SIZE     MODIFIED
llama3.1:latest 8a7c1b2c...   4.7 GB   2 days ago
qwen2.5:latest  7a3b9c1d...   4.5 GB   1 week ago
phi3:latest     4d2a8e3f...   2.2 GB   2 weeks ago
```

### Removing a model to free disk space

```bash
ollama rm phi3
```

### Updating models

Ollama doesn't auto-update models. When a new version is released, pull it again:

```bash
ollama pull llama3.1
```

If there's a newer version, it downloads the update. If not, it confirms you're already current.

## Cost Comparison: Local vs Cloud API

Let's talk numbers. Here's a realistic comparison for a small business processing ~500 documents per month with AI:

| Factor | Cloud API (OpenAI) | Local (Ollama) |
|--------|-------------------|-----------------|
| Setup cost | $0 | $0–500 (hardware if needed) |
| Monthly API cost | $50–200 (usage-based) | $0 |
| Data privacy | Data leaves network | Data stays local |
| Latency | 200–500ms (network) | 100ms–3s (local compute) |
| Maintenance | Zero | Low (updates, monitoring) |
| Vendor lock-in | High (API-specific) | None (open models) |
| Scalability | Unlimited (pay more) | Limited by hardware |

The break-even point is typically 3–6 months. After that, local inference is free apart from electricity. For businesses with sensitive data, the privacy benefit alone justifies the switch — even if the cloud API were cheaper.

## When Local Makes Sense (And When It Doesn't)

### Local is right for you if:

- You handle sensitive, regulated, or confidential data
- You want predictable costs with no monthly API bills
- You need full control over the model and its behavior
- You're building internal tools, not customer-facing applications
- You value data sovereignty and auditability

### Cloud API might be better if:

- You need the absolute highest-quality model available (frontier models like GPT-4-class still outperform open models on complex reasoning)
- You have bursty, unpredictable usage that would require expensive hardware to handle peaks
- You're building customer-facing applications with many concurrent users
- Your team has no IT capacity to manage infrastructure

There's no either/or here. Many businesses run a **hybrid setup**: Ollama for sensitive internal tasks, cloud APIs for less sensitive customer-facing features. The best architecture uses the right tool for each job.

## Security Considerations

Running Ollama locally is more secure than using cloud APIs, but it's not automatically secure. A few best practices:

**1. Don't expose Ollama to the internet.** By default, Ollama listens on `127.0.0.1:11434` (localhost only). If you need other machines on your network to access it, bind to your internal network IP — never to `0.0.0.0` without a firewall.

**2. Use a reverse proxy with authentication.** If multiple services need to call Ollama, put it behind a reverse proxy (like Caddy or nginx) with basic auth or API key validation.

**3. Be mindful of model prompts in logs.** Ollama logs are local, but if you're processing sensitive documents, ensure log files are access-controlled and rotated.

**4. Keep Ollama updated.** Like any software, security patches are released regularly. Run `ollama --version` periodically and update when new versions are available.

## Getting Started: A 15-Minute Plan

If you want to try Ollama today, here's the fastest path:

1. **Install Ollama** — one command or one download (5 min)
2. **Pull Llama 3.1** — `ollama pull llama3.1` (5 min, depends on bandwidth)
3. **Test it interactively** — `ollama run llama3.1` and ask it to summarize a document (2 min)
4. **Call the API from a script** — use the curl example above with one of your own documents (3 min)

That's it. You're running a capable AI model on your own hardware, with no subscription and no data leaving your network.

## Conclusion

Local LLMs have crossed the threshold from "interesting experiment" to "practical business tool." Ollama makes the technical barrier remarkably low — if you can run a Docker container or install a package, you can run an AI model locally.

For businesses that handle sensitive data, care about cost predictability, or want to reduce dependency on external vendors, running local LLMs with Ollama is worth serious consideration. The models are good enough for most internal tasks, the tooling is mature, and the cost savings compound over time.

The best way to evaluate it is to try it. Install Ollama, pull a model, and run it on a few real documents from your business. You'll know within an hour whether local AI meets your quality bar — and if it does, you've eliminated an API bill and a data privacy concern in one move.

---

*Want help setting up local AI for your business? ARDOT Consulting designs and deploys self-hosted automation systems tailored to your workflows. [Get in touch through our contact form](/#contact) — we'll help you go from "interested in AI" to "running your own models" without the learning curve.*