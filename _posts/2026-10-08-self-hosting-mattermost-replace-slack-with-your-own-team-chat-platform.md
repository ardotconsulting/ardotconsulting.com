---
layout: post
title: "Self-Hosting Mattermost: Replace Slack with Your Own Team Chat Platform"
date: 2026-10-08
author: "ARDOT Consulting"
tags: [mattermost, self-hosting, team-chat, slack-replacement, open-source, docker, communication]
excerpt: "Slack costs $7–$15 per user per month and puts your team's conversations on someone else's servers. Here's how to replace it with Mattermost — a self-hosted, open-source team chat platform you fully control."
---

Your team needs to communicate. Right now, that probably means Slack. You pay $7 to $15 per user per month for the privilege of storing every internal conversation, every decision, every shared file, and every strategic discussion on servers owned by Salesforce. When Slack changes its pricing model — and it has, multiple times — you absorb the increase. When Slack decides to retire a feature or change its API, you adapt on their timeline, not yours.

For a 20-person team on the Business plan at $8.75/user/month, that's $2,100 per year. The Pro plan at $7/user/month runs $1,680. Not catastrophic, but the real cost isn't the subscription. It's the dependency. Your team's entire communication history — the institutional knowledge, the decisions, the context that new hires need — lives in a system you don't control, can't audit, and can't run on your own infrastructure.

**Mattermost** is the open-source alternative. It's a team chat platform that does most of what Slack does — channels, direct messages, threads, file sharing, search, integrations, bots — and runs entirely on your own server. No per-user fees. No data leaving your network. No vendor making decisions about your communication stack.

This guide walks through setting up Mattermost with Docker, migrating from Slack, and configuring the features your team actually uses day-to-day.

## What Mattermost Replaces

Let's be straightforward about what Mattermost does well and where it falls short. No tool is a perfect drop-in replacement, and you should understand the trade-offs before committing.

| Slack Feature | Mattermost Equivalent | How Well It Works |
|---------------|----------------------|-------------------|
| Channels (public & private) | Channels (public & private) | Excellent — identical concept, same UX |
| Direct messages & group DMs | Direct messages & group messages | Excellent — full feature parity |
| Threads | Threaded replies | Good — functional, slightly different UX |
| Slack Connect (shared channels) | Shared channels (Enterprise edition) | Limited in open-source edition |
| Slack search | Mattermost search | Good — full-text search, slightly slower on large instances |
| Workflow Builder | Playbooks & integrations | Good — different approach, more flexible via webhooks |
| Slack calls (Huddle) | Plugin-based (Jitsi, etc.) | Requires setup — not built-in by default |
| App directory | Integrations & webhook ecosystem | Smaller ecosystem, but webhooks and bots cover most needs |
| File sharing | File sharing with preview | Excellent — full preview support, stored on your server |
| Emoji & reactions | Emoji & reactions | Excellent — custom emoji supported |
| Slackbot / AI features | Custom bots via API | You build what you need — more work, more control |

The gaps are real but manageable. The biggest one is the lack of a built-in voice/video call feature. Most teams running Mattermost integrate it with **Jitsi Meet** (also open source, also self-hosted) for video calls, or use it alongside a separate phone system. If your team relies heavily on Slack Huddles for quick voice chats, that's a workflow you'll need to replace.

The app ecosystem is smaller, but Mattermost supports incoming and outgoing webhooks, slash commands, and a full REST API. If you're already using n8n for automation (and if you've been reading this blog, you probably are), you can build integrations that go far beyond what Slack's App Directory offers. You're trading pre-built convenience for unlimited flexibility.

## Hardware Requirements

Mattermost is lighter than you might expect. The server only handles message routing and storage — your team's devices do the actual rendering.

| Team Size | RAM | CPU | Storage | Monthly Cost (VPS) |
|-----------|-----|-----|---------|-------------------|
| 1–50 users | 2 GB | 2 cores | 20 GB SSD | $10–20 |
| 50–200 users | 4 GB | 2 cores | 50 GB SSD | $20–40 |
| 200–500 users | 8 GB | 4 cores | 100 GB SSD | $40–80 |
| 500+ users | 16 GB | 8 cores | 200+ GB SSD | $80–160 |

Compare that to Slack Business: a 50-person team pays $4,375/year. A $20/month VPS costs $240/year. The break-even point on hardware is roughly one month of Slack fees.

## Setting Up Mattermost with Docker

The fastest path to a working Mattermost instance is Docker Compose. This setup uses PostgreSQL for the database and includes Nginx as a reverse proxy for SSL termination.

### Step 1: Create the Docker Compose File

```yaml
# docker-compose.yml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: mattermost
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: mattermost
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - mattermost-network

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: "postgres://mattermost:${POSTGRES_PASSWORD}@postgres:5432/mattermost?sslmode=disable"
      MM_SERVICESETTINGS_SITEURL: "https://chat.yourcompany.com"
      MM_SERVICESETTINGS_ENABLELOCALMODE: "true"
    ports:
      - "8065:8065"
    volumes:
      - mattermost_data:/mattermost/data
      - mattermost_config:/mattermost/config
      - mattermost_logs:/mattermost/logs
      - mattermost_plugins:/mattermost/plugins
      - mattermost_client_plugins:/mattermost/client/plugins
    networks:
      - mattermost-network

volumes:
  postgres_data:
  mattermost_data:
  mattermost_config:
  mattermost_logs:
  mattermost_plugins:
  mattermost_client_plugins:

networks:
  mattermost-network:
    driver: bridge
```

### Step 2: Create the Environment File

```bash
# .env
POSTGRES_PASSWORD=change-this-to-a-strong-password
```

Use a real password here — at least 24 characters, mixed case, numbers, symbols. Generate one with:

```bash
openssl rand -base64 32
```

### Step 3: Start the Server

```bash
docker compose up -d
```

Check that both containers are running:

```bash
docker compose ps
```

You should see both `postgres` and `mattermost` with status `Up`. If Mattermost keeps restarting, check the logs:

```bash
docker compose logs mattermost
```

The most common issue is a database connection failure — verify your `POSTGRES_PASSWORD` matches in both the `.env` file and the datasource string.

### Step 4: Set Up the Reverse Proxy

Mattermost listens on port 8065. For production, you need a reverse proxy with SSL. Here's a minimal Nginx config:

```nginx
# /etc/nginx/sites-available/chat.yourcompany.com
upstream mattermost_backend {
    server 127.0.0.1:8065;
    keepalive 32;
}

server {
    listen 80;
    server_name chat.yourcompany.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name chat.yourcompany.com;

    ssl_certificate /etc/letsencrypt/live/chat.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chat.yourcompany.com/privkey.pem;

    location / {
        proxy_pass http://mattermost_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
        proxy_http_version 1.1;
        proxy_read_timeout 90s;
    }
}
```

Get your SSL certificate with Let's Encrypt (free):

```bash
sudo certbot certonly --nginx -d chat.yourcompany.com
sudo ln -s /etc/nginx/sites-available/chat.yourcompany.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Step 5: Create Your Admin Account

Visit `https://chat.yourcompany.com` in your browser. You'll see the setup screen. Create your admin account, name your team, and you're in.

## Migrating from Slack

Mattermost includes a Slack import tool that converts your Slack export into Mattermost channels and messages. Here's how it works.

### Export from Slack

In Slack's admin dashboard (slack.com/admin), go to Settings & Permissions → Import/Export → Export. Choose the date range and download the ZIP file. On the free plan, you can export public channel messages. On paid plans, you can export private channels and DMs too.

### Import to Mattermost

Mattermost provides a CLI tool for importing:

```bash
docker exec -it mattermost mattermost import slack /mattermost/data/slack-export.zip --apply
```

The import preserves:
- Public channels (creates matching Mattermost channels)
- Channel messages and threads
- File attachments (if included in the export)
- User mappings (creates accounts or maps to existing ones)

It does **not** preserve:
- Slack-specific features (Workflow Builder flows, Canvas docs)
- App configurations and integrations
- Emoji (you can re-import custom emoji separately)

After the import, go to **System Console → Compliance → Compliance Export** to verify the data landed correctly. The import log will tell you how many channels, messages, and users were processed.

## Connecting Mattermost to n8n for Automation

This is where Mattermost shines over Slack. Because you control the server, you can build deep integrations without rate limits, API quotas, or third-party app approval processes.

Mattermost supports **incoming webhooks** (post messages to channels from external services) and **outgoing webhooks** (trigger external services when messages are posted). Both work perfectly with n8n.

### Example: Post Automated Reports to a Channel

In Mattermost, go to **Integrations → Incoming Webhooks** and create a webhook for the channel where you want reports posted. Copy the URL.

In n8n, create a workflow that:
1. Runs on a schedule (e.g., every Monday at 9 AM)
2. Queries your business data (from Metabase, Directus, or a database)
3. Formats the results as a Markdown message
4. Sends it to the Mattermost webhook

```json
{
  "text": "## Weekly Sales Summary",
  "attachments": [
    {
      "color": "#36a64f",
      "fields": [
        { "title": "Orders this week", "value": "147", "short": true },
        { "title": "Revenue", "value": "$28,450", "short": true },
        { "title": "New customers", "value": "23", "short": true },
        { "title": "Avg. order value", "value": "$193", "short": true }
      ],
      "footer": "Auto-generated by ARDOT automation pipeline",
      "ts": 1696118400
    }
  ]
}
```

POST that to your webhook URL, and the report appears in the channel automatically:

```
curl -X POST https://chat.yourcompany.com/hooks/your-webhook-id \
  -H "Content-Type: application/json" \
  -d @report.json
```

### Example: AI-Powered Meeting Notes Bot

Using Mattermost's bot framework and Ollama running locally, you can build a bot that:

1. Receives a message like `/summarize #product-meeting`
2. Fetches the last N messages from that channel via the Mattermost API
3. Sends them to a local LLM (Ollama) for summarization
4. Posts the summary back as a threaded reply

Here's a minimal Python bot using the Mattermost driver:

```python
import requests
import json

MATTERMOST_URL = "https://chat.yourcompany.com"
BOT_TOKEN = "your-bot-token"
OLLAMA_URL = "http://localhost:11434/api/generate"

def get_channel_messages(channel_id, limit=50):
    """Fetch recent messages from a Mattermost channel."""
    headers = {"Authorization": f"Bearer {BOT_TOKEN}"}
    resp = requests.get(
        f"{MATTERMOST_URL}/api/v4/channels/{channel_id}/posts",
        headers=headers,
        params={"per_page": limit}
    )
    posts = resp.json().get("order", [])
    messages = [resp.json()["posts"][p]["message"] for p in posts]
    return "\n".join(messages)

def summarize_with_ollama(text):
    """Send text to local Ollama instance for summarization."""
    payload = {
        "model": "llama3.2",
        "prompt": f"Summarize this team conversation in bullet points:\n\n{text}",
        "stream": False
    }
    resp = requests.post(OLLAMA_URL, json=payload)
    return resp.json()["response"].strip()

def post_to_channel(channel_id, message):
    """Post a message back to the Mattermost channel."""
    headers = {"Authorization": f"Bearer {BOT_TOKEN}"}
    requests.post(
        f"{MATTERMOST_URL}/api/v4/posts",
        headers=headers,
        json={"channel_id": channel_id, "message": message}
    )
```

The bot runs on your server alongside Mattermost. No external API calls, no data leaving your network. The LLM inference happens on your hardware through Ollama. Your team's conversations — which often contain sensitive business information — never touch a third-party server.

## Security Configuration

Running your own chat server means you're responsible for security. Here's a checklist of the essentials:

**Authentication:**
- Enable email verification for new accounts (System Console → Authentication → Email)
- Set up SSO if you use an identity provider (Mattermost supports SAML 2.0 and OpenID Connect)
- Disable open account creation if your team is small: System Console → Authentication → Signup → Enable Open Server: `false`
- Enforce password complexity: minimum 10 characters, require mixed case and numbers

**Data protection:**
- Enable TLS for the database connection (add `sslmode=require` to the datasource string)
- Set up automatic database backups (cron + `pg_dump`):
```bash
# Daily backup at 2 AM
0 2 * * * docker exec mattermost-postgres pg_dump -U mattermost mattermost | gzip > /backups/mattermost_$(date +\%Y\%m\%d).sql.gz
```
- Configure retention policies for message deletion (System Console → Compliance → Data Retention)
- Enable audit logging (System Console → Compliance → Auditing)

**Network security:**
- Restrict port 8065 to localhost only in docker-compose.yml (the reverse proxy handles external traffic)
- Set up fail2ban to block brute-force login attempts
- Use a firewall (ufw) to only expose ports 22, 80, and 443

```bash
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

## Cost Comparison: Slack vs Mattermost Over 3 Years

Let's run the numbers for a realistic scenario: a 25-person company over three years.

| Cost Category | Slack Business ($8.75/user/mo) | Mattermost (Self-Hosted) |
|---------------|-------------------------------|-------------------------|
| Subscription (3 years) | $7,875 | $0 |
| VPS hosting (3 years) | $0 | $864 ($24/mo × 36) |
| SSL certificate | $0 (included) | $0 (Let's Encrypt) |
| Initial setup time | 0 hours | 4–8 hours |
| Migration effort | Minimal | 2–4 hours |
| Backup storage | Included | $60/year (~$180) |
| **3-year total** | **$7,875** | **~$1,044** |

That's a savings of roughly $6,800 over three years for a 25-person team. For a 100-person team, the savings scale to over $30,000.

The hidden cost is maintenance. You'll spend an hour or two per month on updates, backup verification, and occasional troubleshooting. If that's a dealbreaker, Mattermost also offers a hosted Enterprise edition — but at that point, you're back to paying per user, just to a different vendor.

## When to Choose Mattermost vs Slack

Mattermost isn't the right choice for every team. Here's an honest assessment:

**Choose Mattermost if:**
- Data sovereignty matters — you're in healthcare, legal, finance, or defense, and team conversations can't leave your infrastructure
- You have more than 15–20 users (the cost crossover happens fast)
- You already self-host other tools (Nextcloud, Odoo, n8n) and have the infrastructure
- You want deep, custom integrations without API rate limits
- You're comfortable with basic Docker and Linux administration

**Stick with Slack if:**
- Your team is smaller than 10 people and the free plan covers your needs
- You rely heavily on Slack-specific features (Canvas, Huddles, Workflow Builder)
- You don't have anyone on the team who can manage a Linux server
- Your team needs the app ecosystem (thousands of pre-built integrations)
- You value zero-setup convenience over cost savings and control

There's no wrong answer. The question is whether the trade-offs — more setup work, more maintenance responsibility, smaller app ecosystem — are worth the cost savings and data control. For a growing business that's already investing in self-hosted infrastructure, the answer is usually yes.

## Making the Switch

If you've decided to move, here's a phased rollout plan that minimizes disruption:

**Week 1:** Set up the server, configure SSL, create your admin account. Invite 2–3 technical team members as beta users. Import your Slack history. Test core workflows — messaging, file sharing, search.

**Week 2:** Set up the integrations you rely on. If you use n8n, create Mattermost webhooks to replace Slack notifications. If you use GitHub, configure commit notifications to post to Mattermost channels. Test everything with the beta group.

**Week 3:** Invite the rest of the team. Keep Slack running in parallel — this is critical. People need time to build the habit of checking Mattermost. Pin a welcome message in your main channel with links to short guides (Mattermost's docs are excellent).

**Week 4:** Start winding down Slack. Archive channels, set Slack statuses to "We've moved to Mattermost — find us at chat.yourcompany.com." Give it one more week, then cancel the Slack subscription.

Don't rush the transition. The technology is the easy part. The hard part is changing habits, and that takes time regardless of which tool you're switching to.

## Wrapping Up

Slack is a good product. It works, it's polished, and it requires zero maintenance. But it's also a recurring tax on your team's communication, and it puts your institutional knowledge in someone else's hands. Mattermost gives you the same core functionality — channels, threads, search, file sharing, integrations — on your own server, for the cost of a modest VPS.

If you're already on the self-hosting path with tools like Nextcloud, n8n, or Odoo, adding Mattermost to the stack is a natural next step. The integration story is actually better than Slack's, because you're not constrained by rate limits or app approval processes — you can build exactly what your team needs with webhooks, bots, and n8n workflows.

The question isn't whether Mattermost can replace Slack. It can. The question is whether your team is ready to own its communication infrastructure. If the answer is yes, the path is clear.

---

*Want help setting up Mattermost or migrating your team from Slack? ARDOT Consulting specializes in open-source automation and self-hosted infrastructure for small and medium businesses. [Get in touch](/#contact) — we'll handle the setup, migration, and integration so your team can focus on the work, not the tools.*