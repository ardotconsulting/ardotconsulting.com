---
layout: post
title: "Building an Internal Knowledge Base with BookStack and AI: Make Company Knowledge Searchable"
date: 2026-10-04
author: "ARDOT Consulting"
tags: [knowledge-base, bookstack, ollama, search, open-source, docker]
excerpt: "Learn how to set up BookStack as a self-hosted company wiki and add AI-powered semantic search with Ollama so your team actually finds what they need."
---

Your company has a knowledge problem. You know it. Your team knows it. The answer to last week's customer question is sitting in someone's email, a Slack thread from March, a Google Doc that nobody can find, and a sticky note on Sandra's monitor. You've tried shared folders. You've tried a wiki. Nobody updates the wiki, and the search bar returns nothing useful because nobody remembers the exact title of the page they're looking for.

The fix isn't another SaaS subscription. It's a self-hosted knowledge base paired with AI-powered search that understands what your team is actually asking — not just keyword matching, but semantic search that finds "how do I process a refund" when the document is titled "Customer Return Policy."

In this guide, we'll set up **BookStack**, an open-source wiki platform, and pair it with **Ollama** for AI-powered semantic search. Everything runs on your own infrastructure. No per-seat fees, no data leaving your network, no vendor deciding to double their pricing next quarter.

## Why BookStack?

BookStack is a free, open-source documentation platform that sits somewhere between a wiki and a structured documentation site. It's built in PHP/Laravel, runs in Docker, and has a clean, approachable interface that doesn't scare non-technical staff.

Here's why it works for small businesses:

- **Organized by default:** Content is structured into Books → Chapters → Pages, so your team isn't dumping everything into one flat list
- **WYSIWYG editor:** No Markdown required (though it's supported). Your operations manager can actually use it
- **Role-based permissions:** Control who can view, edit, or administer different books
- **Search built in:** Full-text search out of the box — we'll make it smarter with AI
- **Self-hosted:** Runs on a $5/month VPS or an old office desktop

### How BookStack Compares to SaaS Alternatives

| Feature | BookStack (Self-Hosted) | Notion (SaaS) | Confluence (SaaS) |
|---------|------------------------|---------------|-------------------|
| Cost | Free (server cost only) | $8–$18/user/month | $5–$10/user/month |
| Data location | Your server | Vendor's cloud | Vendor's cloud |
| Offline access | Yes (on your network) | Limited | Limited |
| User limit | Unlimited | Plan-based | Plan-based |
| Customization | Full source code | Limited | Limited |
| Export format | HTML, PDF, Markdown | Limited | Limited |
| AI search | Add your own (this guide) | Vendor-controlled | Vendor-controlled |

For a 15-person team, that's $1,440–$3,240 per year in SaaS fees you're not paying. Plus, your company knowledge stays on your server.

## Step 1: Set Up BookStack with Docker

You'll need a server with Docker installed. This can be a VPS from Hetzner, OVH, or any provider you trust — or a spare machine in your office. We'll use the official BookStack Docker image.

Create a `docker-compose.yml` file:

```yaml
version: "3.8"

services:
  bookstack:
    image: lscr.io/linuxserver/bookstack:latest
    container_name: bookstack
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - DB_HOST=bookstack_db
      - DB_USER=bookstack
      - DB_PASS=change_this_password
      - DB_DATABASE=bookstack
    volumes:
      - ./bookstack_data:/config
    ports:
      - 6875:80
    restart: unless-stopped
    depends_on:
      - bookstack_db

  bookstack_db:
    image: lscr.io/linuxserver/mariadb:latest
    container_name: bookstack_db
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - MYSQL_ROOT_PASSWORD=change_this_root_password
      - MYSQL_DATABASE=bookstack
      - MYSQL_USER=bookstack
      - MYSQL_PASSWORD=change_this_password
    volumes:
      - ./db_data:/config
    restart: unless-stopped
```

Start it up:

```bash
docker compose up -d
```

Wait about 30 seconds for the database to initialize, then open `http://your-server-ip:6875` in your browser. The default login is `admin@admin.com` / `password`. **Change this immediately** on first login.

### Secure It with a Reverse Proxy

For production, don't expose port 6875 directly. Use Caddy as a reverse proxy with automatic HTTPS:

```yaml
# Add to your docker-compose.yml
  caddy:
    image: caddy:latest
    container_name: caddy
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
    restart: unless-stopped
    depends_on:
      - bookstack

volumes:
  caddy_data:
```

And a minimal `Caddyfile`:

```
wiki.yourcompany.com {
    reverse_proxy bookstack:80
}
```

Point your DNS to the server, restart with `docker compose up -d`, and Caddy handles the TLS certificate automatically. Your team now accesses the wiki at `https://wiki.yourcompany.com`.

## Step 2: Organize Your Knowledge

Before adding AI search, you need content worth searching. The biggest reason internal wikis fail is that nobody structures them. BookStack's hierarchy helps, but you still need a plan.

Here's a structure that works for most small businesses:

**Book 1: Company Operations**
- Chapter: Onboarding → Pages: Day 1 Checklist, Tools Setup, Key Contacts
- Chapter: Daily Procedures → Pages: Opening/Closing, Cash Handling, End-of-Day Reports
- Chapter: Emergency Procedures → Pages: System Outage, Customer Incident, Evacuation

**Book 2: Customer Guide**
- Chapter: Common Questions → Pages: Return Policy, Warranty Claims, Pricing Tiers
- Chapter: Account Management → Pages: How to Create Accounts, Reset Passwords, Update Billing

**Book 3: Internal Tools**
- Chapter: Systems → Pages: CRM Login, Email Setup, VPN Access
- Chapter: Troubleshooting → Pages: Printer Issues, Email Problems, VPN Won't Connect

The key principle: **every page should answer one question.** Not "Everything About Customer Service" but "How to Process a Refund." This makes both human search and AI search more effective.

## Step 3: Add AI-Powered Semantic Search with Ollama

BookStack's built-in search is decent — it does full-text matching. But it has the same limitation as every keyword search: if a user types "how do I handle a customer who wants their money back," it won't find a page titled "Refund Processing Procedure" unless those exact words appear.

Semantic search fixes this. Instead of matching keywords, it matches *meaning*. We'll use Ollama to generate embeddings (numerical representations of text meaning) for every page in your wiki, then search against those embeddings.

### Install Ollama

If you followed our [Ollama guide](/blog/2026/08/20/running-local-llms-with-ollama-a-practical-guide-for-businesses/), you already have it. If not:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Pull an embedding model — these are small and fast:

```bash
ollama pull nomic-embed-text
```

This model is about 270MB and runs on any machine with 4GB RAM. It doesn't need a GPU for embedding generation.

### Build a Search Index

We'll write a Python script that:

1. Pulls all pages from BookStack's REST API
2. Generates an embedding for each page
3. Stores embeddings in a simple SQLite database
4. Provides a search endpoint that compares query embeddings against stored ones

First, get an API token from BookStack: Settings → API Tokens → Create Token. Save the ID and secret.

Here's the indexing script, `build_index.py`:

```python
import requests
import json
import sqlite3
import os
import subprocess

# BookStack API config
BOOKSTACK_URL = os.environ.get("BOOKSTACK_URL", "http://localhost:6875")
API_TOKEN_ID = os.environ.get("BS_API_ID", "your_token_id")
API_TOKEN_SECRET = os.environ.get("BS_API_SECRET", "your_token_secret")

# Ollama config
OLLAMA_URL = "http://localhost:11434"
EMBED_MODEL = "nomic-embed-text"

def get_all_pages():
    """Fetch all pages from BookStack API."""
    headers = {"Authorization": f"Token {API_TOKEN_ID}:{API_TOKEN_SECRET}"}
    pages = []
    offset = 0
    while True:
        resp = requests.get(
            f"{BOOKSTACK_URL}/api/pages",
            headers=headers,
            params={"count": 100, "offset": offset}
        )
        resp.raise_for_status()
        data = resp.json().get("data", [])
        if not data:
            break
        pages.extend(data)
        offset += 100
    return pages

def get_embedding(text):
    """Generate embedding for text using Ollama."""
    resp = requests.post(
        f"{OLLAMA_URL}/api/embeddings",
        json={"model": EMBED_MODEL, "prompt": text[:8000]}
    )
    resp.raise_for_status()
    return resp.json()["embedding"]

def init_db(db_path="search_index.db"):
    """Initialize SQLite database for embeddings."""
    conn = sqlite3.connect(db_path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS page_embeddings (
            page_id INTEGER PRIMARY KEY,
            title TEXT,
            url TEXT,
            content_preview TEXT,
            embedding TEXT
        )
    """)
    conn.commit()
    return conn

def build_index():
    """Build the search index from all BookStack pages."""
    conn = init_db()
    pages = get_all_pages()
    print(f"Found {len(pages)} pages to index")

    for page in pages:
        # Combine title and text content for richer embeddings
        text = f"{page.get('name', '')}\n{page.get('text', '')}"
        embedding = get_embedding(text)

        url = f"{BOOKSTACK_URL}/books/{page.get('book_slug', '')}/page/{page.get('slug', '')}"

        conn.execute(
            "INSERT OR REPLACE INTO page_embeddings VALUES (?, ?, ?, ?, ?)",
            (
                page["id"],
                page.get("name", ""),
                url,
                text[:200],
                json.dumps(embedding)
            )
        )
        conn.commit()
        print(f"  Indexed: {page.get('name', '')}")

    conn.close()
    print("Index build complete")

if __name__ == "__main__":
    build_index()
```

Run it:

```bash
pip install requests
export BS_API_ID="your_token_id"
export BS_API_SECRET="your_token_secret"
python build_index.py
```

### Build the Search Interface

Now create a small Flask app that takes a natural-language query and returns the most relevant pages:

```python
# search_server.py
from flask import Flask, request, jsonify
import requests
import json
import sqlite3
import numpy as np

app = Flask(__name__)
OLLAMA_URL = "http://localhost:11434"
EMBED_MODEL = "nomic-embed-text"

def get_embedding(text):
    resp = requests.post(
        f"{OLLAMA_URL}/api/embeddings",
        json={"model": EMBED_MODEL, "prompt": text}
    )
    resp.raise_for_status()
    return np.array(resp.json()["embedding"])

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

@app.route("/search")
def search():
    query = request.args.get("q", "")
    if not query:
        return jsonify({"error": "No query provided"}), 400

    query_embedding = get_embedding(query)

    conn = sqlite3.connect("search_index.db")
    results = []
    for row in conn.execute("SELECT page_id, title, url, content_preview, embedding FROM page_embeddings"):
        stored_embedding = np.array(json.loads(row[4]))
        score = cosine_similarity(query_embedding, stored_embedding)
        results.append({
            "title": row[1],
            "url": row[2],
            "preview": row[3],
            "score": float(score)
        })
    conn.close()

    # Sort by similarity score, return top 5
    results.sort(key=lambda x: x["score"], reverse=True)
    return jsonify({"results": results[:5]})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Run the search server:

```bash
pip install flask numpy
python search_server.py
```

Now you can search:

```bash
curl "http://localhost:5000/search?q=how+do+I+process+a+refund"
```

And get back the most semantically relevant pages — even if none of them contain the exact word "refund." The search understands that "process a refund" and "customer return policy" are talking about the same thing.

### Rebuild the Index When Content Changes

BookStack content evolves. Set up a cron job to rebuild the index nightly:

```bash
# /etc/cron.d/bookstack-search
0 2 * * * root BS_API_ID=your_token_id BS_API_SECRET=your_token_secret /usr/bin/python3 /opt/bookstack-search/build_index.py >> /var/log/bookstack-search.log 2>&1
```

Or, for real-time updates, use BookStack's webhook feature to trigger a re-index of a single page whenever content is edited. That's more work but keeps the index always current.

## Step 4: Make It Usable for Your Team

A search API is great for developers but useless for your operations manager. You need a front end. Here are two approaches:

### Option A: Simple HTML Page

A single-page HTML form that calls your search API:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Company Knowledge Search</title>
    <style>
        body { font-family: sans-serif; max-width: 700px; margin: 40px auto; padding: 0 20px; }
        .search-box { width: 100%; padding: 12px; font-size: 16px; border: 1px solid #ddd; border-radius: 6px; }
        .result { padding: 16px 0; border-bottom: 1px solid #eee; }
        .result h3 { margin: 0 0 4px; }
        .result h3 a { color: #2196F3; text-decoration: none; }
        .result .preview { color: #666; font-size: 14px; margin-top: 4px; }
        .score { color: #999; font-size: 12px; }
    </style>
</head>
<body>
    <h1>🔍 Knowledge Search</h1>
    <input class="search-box" id="query" placeholder="Ask anything..." onkeyup="search(event)" />
    <div id="results"></div>

    <script>
        async function search(e) {
            if (e.key !== 'Enter') return;
            const q = document.getElementById('query').value;
            if (!q) return;
            const resp = await fetch(`http://your-server:5000/search?q=${encodeURIComponent(q)}`);
            const data = await resp.json();
            const html = (data.results || []).map(r => `
                <div class="result">
                    <h3><a href="${r.url}" target="_blank">${r.title}</a></h3>
                    <div class="preview">${r.preview}</div>
                    <div class="score">Match: ${(r.score * 100).toFixed(0)}%</div>
                </div>
            `).join('');
            document.getElementById('results').innerHTML = html;
        }
    </script>
</body>
</html>
```

Host this on your Caddy server alongside BookStack, and your team has a simple search bar that understands natural language.

### Option B: Integrate with Your Chat Platform

If your team uses Mattermost (open-source Slack alternative), you can create a bot that responds to `/search how do I process a refund` with the top results directly in chat. This is where the search gets the most use — people are already in chat when they have a question.

The bot is a simple Python script that listens for slash commands and queries your search API:

```python
from mattermost import MattermostAPI
import requests

mm = MattermostAPI("https://chat.yourcompany.com", "your_bot_token")

def handle_search_command(query, channel_id):
    resp = requests.get(
        "http://localhost:5000/search",
        params={"q": query}
    )
    results = resp.json().get("results", [])

    if not results:
        mm.create_post(channel_id, "No results found. Try different wording.")
        return

    message = "**Top results:**\n\n"
    for i, r in enumerate(results[:3], 1):
        message += f"{i}. [{r['title']}]({r['url']})\n   _{r['preview'][:100]}..._\n"

    mm.create_post(channel_id, message)
```

## Real-World Impact

Here's what this looks like in practice for a small business:

**Before:** A new hire needs to know how to process a return. They search the shared drive for "return" and get 47 files. They ask three colleagues. One says "check the wiki." The wiki search returns nothing because the page is called "Customer Refund Workflow." Twenty minutes lost.

**After:** The new hire types "how do I handle a customer return" into the knowledge search. The AI matches the semantic meaning to "Customer Refund Workflow" and returns it as the top result with a 94% match score. They click through and have their answer in 30 seconds.

Multiply that by 15 employees asking 5 questions per day, and you're saving roughly 18 hours of "where do I find..." time per week. That's nearly half a full-time position redirected from searching to actually working.

## Keeping It Alive

The biggest threat to any knowledge base isn't technology — it's entropy. Here's how to keep yours from going stale:

1. **Assign ownership:** Each Book (Company Operations, Customer Guide, etc.) has one person responsible for reviewing it monthly
2. **Page review dates:** BookStack doesn't have this natively, but you can add a "Last Reviewed: [date]" line at the top of each page and set a calendar reminder
3. **Tie updates to process changes:** When you change a business process, updating the wiki page is part of the change — not a separate task
4. **Watch the search logs:** If people search for something and get no results, that's a gap. Create a page for it
5. **Re-index regularly:** The nightly cron job handles this, but verify it's running: `tail -20 /var/log/bookstack-search.log`

## What This Setup Costs You

| Component | Cost |
|-----------|------|
| BookStack (open source) | $0 |
| Ollama + nomic-embed-text | $0 |
| VPS (4GB RAM, 2 vCPU) | ~$5–$10/month |
| Domain name | ~$10/year |
| Caddy (reverse proxy + TLS) | $0 |
| **Total** | **~$70–$130/year** |

Compare that to Notion at $1,440/year for 15 users, or Confluence at $600–$1,800/year. And your data never leaves your server.

## Wrapping Up

A knowledge base only works if people can find what they need. Traditional wikis fail because their search is keyword-based and people don't think in keywords — they think in questions. By combining BookStack's clean documentation structure with Ollama's semantic search, you give your team a system that understands what they're actually asking for.

The whole setup runs on infrastructure you control, costs less than a single SaaS seat per month, and scales to as many users as you have. The only ongoing work is keeping your content current — and that's a people problem, not a technology one.

If you want help setting up a self-hosted knowledge base with AI search for your business, [reach out through our contact form](/#contact). We'll assess your needs, handle the setup, and train your team on keeping it alive — no SaaS subscriptions required.