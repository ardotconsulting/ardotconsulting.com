---
layout: post
title: "AI Automation Security: Protecting Your Workflows and Data"
date: 2026-09-16
author: "ARDOT Consulting"
tags: [security, best-practices, automation, privacy, self-hosting]
excerpt: "A practical security checklist for businesses running AI automation: API key management, network segmentation, encryption, access control, and secure deployment of n8n and Ollama."
---

You've automated your invoice processing. Your n8n workflows route customer emails. Ollama runs locally to summarize documents. Everything hums along nicely—until you realize your API keys are stored in plaintext, your n8n instance is accessible from the open internet, and your Ollama server has no authentication at all.

Sound familiar? You're not alone. Most small businesses get AI automation running first and think about security second. This guide flips that order. We'll walk through the concrete security measures that matter most when you're self-hosting AI tools, with real configuration examples you can apply today.

## Why Self-Hosted AI Has Different Security Needs

When you use a cloud AI service, the vendor handles security for you—encryption, access control, network isolation. When you self-host with tools like n8n, Ollama, and Odoo, **you** are the security team. That's both the risk and the opportunity: you control every layer, but only if you actually configure them.

The good news? Self-hosting means your data never leaves your network. No third party can read your customer records, scan your financials, or train on your documents. The bad news? If you expose those tools without protection, anyone on the internet can.

Let's go layer by layer.

## Layer 1: API Key and Secret Management

Every automation workflow uses credentials—API keys for services like Stripe, SMTP passwords for email, database connection strings. These are the keys to your business, and they deserve real protection.

### The Problem

n8n stores credentials in its database. By default, if someone gets access to your n8n instance, they can see every credential you've ever saved. That's your Stripe key, your email password, your database connection string—all sitting in the n8n UI.

### The Fix: Encryption at Rest

n8n supports credential encryption out of the box, but you need to make sure it's configured. Set the encryption key via an environment variable rather than letting n8n auto-generate one:

```bash
# In your n8n environment file or docker-compose
N8N_ENCRYPTION_KEY=<a-strong-32+character-random-string>
```

Generate a strong key with:

```bash
openssl rand -base64 32
```

If you don't set this explicitly, n8n generates one on first run and stores it in the database—meaning anyone with database access can decrypt your credentials. Setting it as an environment variable keeps the key separate from the data.

### The Fix: Use a Secrets Manager

For teams with more than one automation, don't hardcode secrets in workflow definitions. Use a dedicated secrets manager. [Infisical](https://infisical.com/) is an open source option that integrates well:

```yaml
# docker-compose excerpt for Infisical
services:
  infisical:
    image: infisical/infisical:latest
    ports:
      - "8080:8080"
    environment:
      - ENCRYPTION_KEY=${INFISICAL_ENCRYPTION_KEY}
      - JWT_SIGNUP_SECRET=${INFISICAL_JWT_SECRET}
    volumes:
      - infisical-data:/var/lib/postgresql/data
volumes:
  infisical-data:
```

Store your API keys in Infisical, then reference them in n8n using environment variables. This way, credentials live in one encrypted vault—not scattered across workflow definitions.

### The Fix: Rotate Keys Regularly

Set a calendar reminder every 90 days to rotate your most sensitive keys (payment processors, email, database). If a key leaks, rotation limits the damage window.

## Layer 2: Network Segmentation

This is the single most impactful security measure for self-hosted tools, and it's the one most small businesses skip.

### The Problem

Many guides tell you to expose n8n on port 5678 so you can access it from anywhere. Similarly, Ollama's default setup binds to `0.0.0.0:11434`, making it reachable from any machine on your network—or the internet if your firewall is permissive.

### The Fix: Don't Expose Anything You Don't Have To

Your automation tools should **not** be directly accessible from the public internet. Here's a network architecture that works:

```
Internet
  │
  ├── Reverse Proxy (Caddy/Nginx) — only entry point, port 443
  │     ├── ardotconsulting.com         → Jekyll site
  │     ├── n8n.internal.ardotconsulting.com → n8n (behind auth)
  │     └── (Ollama is NOT exposed here)
  │
  ├── Internal Docker Network (no public access)
  │     ├── n8n (port 5678, internal only)
  │     ├── ollama (port 11434, internal only)
  │     ├── postgres (port 5432, internal only)
  │     └── redis (port 6379, internal only)
  │
  └── VPN (WireGuard) — for admin access
```

Key principles:

1. **Only the reverse proxy touches the public internet.** Everything else lives on an internal Docker network.
2. **Ollama is never exposed externally.** Only n8n and other internal services can reach it.
3. **Admin access goes through a VPN**, not through public URLs.

### Docker Compose with Internal Networking

```yaml
version: "3.8"

services:
  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    networks:
      - public
      - internal
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data

  n8n:
    image: n8nio/n8n:latest
    environment:
      - N8N_HOST=n8n.internal.ardotconsulting.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.internal.ardotconsulting.com/webhook/
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
    networks:
      - internal
    volumes:
      - n8n-data:/home/node/.n8n

  ollama:
    image: ollama/ollama:latest
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
    networks:
      - internal
    volumes:
      - ollama-data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

networks:
  public:
    internal: false
  internal:
    internal: true   # No external access

volumes:
  caddy-data:
  n8n-data:
  ollama-data:
```

Notice the `internal: true` on the `internal` network. Docker creates a network that has no route to the outside world. Even if a container tries to bind to `0.0.0.0`, only other containers on the same network can reach it.

### Caddyfile for Secure Reverse Proxy

```
n8n.internal.ardotconsulting.com {
    reverse_proxy n8n:5678
    basicauth / {
        admin <bcrypt-hash-here>
    }
    # Only allow access from known IPs
    @blocked not remote_ip 10.0.0.0/8 192.168.0.0/16
    respond @blocked 403
}
```

This adds basic authentication and IP allowlisting. Even someone who discovers the subdomain can't get in without credentials and a whitelisted IP.

### Ollama Security: Bind Locally

By default, Ollama binds to `127.0.0.1:11434` in recent versions. Verify this in your environment:

```bash
# Check what Ollama is listening on
ss -tlnp | grep 11434
# Should show 127.0.0.1:11434, NOT 0.0.0.0:11434

# If it shows 0.0.0.0, fix it:
OLLAMA_HOST=127.0.0.1:11434 ollama serve
```

If n8n needs to reach Ollama, they should be on the same Docker network (as in the compose file above), not on a publicly exposed port.

## Layer 3: Access Control

### n8n User Management

n8n has built-in user management. Turn it on and create separate accounts for each team member:

```bash
# In your n8n environment
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=<strong-password-here>
```

Better yet, if you're running n8n 0.200+, use the full user management system with role-based access control. Go to **Settings > Users** and invite team members with appropriate roles:

| Role | Can Edit Workflows | Can View Executions | Can Manage Credentials |
|------|-------------------|--------------------|-----------------------|
| Admin | Yes | Yes | Yes |
| Editor | Yes | Yes | No |
| Viewer | No | Yes | No |

Limit credential management to admins. Most team members only need to edit workflows or view execution logs, not touch the API keys.

### Ollama Access Control

Ollama has no built-in authentication. If it's on an internal-only Docker network, that's your access control. If you need to call Ollama from outside the Docker network (say, from a different server), put it behind a reverse proxy with authentication:

```
# Caddyfile
llm.internal.ardotconsulting.com {
    reverse_proxy ollama:11434
    basicauth / {
        api-user <bcrypt-hash-here>
    }
}
```

Now API calls to Ollama require a username and password:

```bash
curl -u api-user:password http://llm.internal.ardotconsulting.com/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Summarize this document securely."
}'
```

## Layer 4: Data Encryption

### Encryption at Rest

For your PostgreSQL database (used by n8n, Odoo, Infisical), enable full-disk encryption on the host or use encrypted Docker volumes.

For Linux hosts, LUKS full-disk encryption is the standard:

```bash
# Check if your disk is encrypted
lsblk -f
# Look for "crypto_LUKS" under FSTYPE
```

If you're setting up a new server, enable encryption during OS installation. For existing servers, you can encrypt specific directories:

```bash
# Encrypt the Docker data directory (example using fscrypt)
fscrypt encrypt /var/lib/docker
```

### Encryption in Transit

All internal communication between containers should use encrypted connections even on the internal network. Configure:

- **PostgreSQL**: Require SSL connections
- **n8n to Ollama**: Use HTTPS if going through a reverse proxy
- **All external traffic**: Force HTTPS via Caddy (automatic with Let's Encrypt)

PostgreSQL SSL configuration:

```bash
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/postgres.crt'
ssl_key_file = '/etc/ssl/private/postgres.key'

# pg_hba.conf — require SSL for all connections
hostssl all all 0.0.0.0/0 md5
```

## Layer 5: Monitoring and Auditing

Security isn't a one-time setup. You need to know when something goes wrong.

### Audit Logs in n8n

n8n logs every workflow execution. Configure execution logging to persist (not just in-memory):

```bash
N8N_LOG_OUTPUT=console
N8N_LOG_LEVEL=info
N8N_METRICS=true
```

Forward these logs to a central log server. [Loki](https://grafana.com/oss/loki/) with Grafana is a solid open source combination:

```yaml
# Add to your docker-compose
  loki:
    image: grafana/loki:latest
    networks:
      - internal
    volumes:
      - loki-data:/loki

  grafana:
    image: grafana/grafana:latest
    networks:
      - public
      - internal
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
```

Set up Grafana alerts for:

- Failed authentication attempts on n8n (possible brute force)
- Unusual execution spikes (a compromised workflow running repeatedly)
- Ollama API errors (potential resource exhaustion attack)

### Ollama Usage Monitoring

Track who's calling Ollama and how often:

```bash
# Simple log monitoring
docker logs ollama --since 24h | grep "POST /api/generate" | wc -l
# Number of inference calls in the last 24 hours
```

A sudden spike in API calls could indicate a compromised service or a workflow stuck in a loop.

## Layer 6: Backup Security

Backups are your safety net—but they're also a common attack vector. An unencrypted backup file is as dangerous as the live data.

### Encrypted Backups with Restic

[Restic](https://restic.net/) is an open source backup tool with built-in encryption:

```bash
# Initialize an encrypted backup repository
restic init --repo /backups/n8n

# Backup n8n data
restic backup --repo /backups/n8n /var/lib/docker/volumes/n8n-data/_data

# The backup is encrypted automatically—no plaintext on disk
```

Schedule backups via cron:

```bash
# /etc/cron.d/automation-backups
0 2 * * * root restic backup --repo /backups/n8n /var/lib/docker/volumes/n8n-data/_data
0 3 * * * root restic backup --repo /backups/ollama /var/lib/docker/volumes/ollama-data/_data
0 4 * * * root restic forget --repo /backups/n8n --keep-daily 7 --keep-weekly 4
```

This keeps 7 daily backups and 4 weekly backups, all encrypted.

## The Security Checklist

Here's a practical checklist to audit your AI automation stack:

| # | Security Measure | Status |
|---|-----------------|--------|
| 1 | n8n encryption key set via environment variable | ☐ |
| 2 | n8n behind reverse proxy (not directly exposed) | ☐ |
| 3 | n8n basic auth or user management enabled | ☐ |
| 4 | n8n credentials limited to admin role only | ☐ |
| 5 | Ollama bound to 127.0.0.1 or internal Docker network | ☐ |
| 6 | Ollama behind auth proxy if accessed outside Docker | ☐ |
| 7 | Internal Docker network marked `internal: true` | ☐ |
| 8 | All external traffic over HTTPS (Caddy + Let's Encrypt) | ☐ |
| 9 | PostgreSQL SSL required for all connections | ☐ |
| 10 | Full-disk or volume encryption on host | ☐ |
| 11 | API keys rotated every 90 days | ☐ |
| 12 | Audit logs forwarded to centralized logging | ☐ |
| 13 | Encrypted automated backups (Restic) | ☐ |
| 14 | VPN (WireGuard) for admin access | ☐ |
| 15 | Grafana alerts for failed auth and unusual activity | ☐ |

## Common Mistakes to Avoid

**Exposing n8n with default settings.** Out of the box, n8n has no authentication. If you deploy it on a public IP without a reverse proxy and auth, anyone who finds the port can access your workflows and credentials.

**Forgetting that Ollama has no auth.** Unlike n8n, Ollama doesn't even have optional authentication. Network isolation is your only protection. If Ollama is on a public network, it's open to everyone.

**Storing secrets in workflow JSON.** It's tempting to paste an API key directly into a workflow node. Don't. Use n8n's credential system with encryption, and keep the encryption key in an environment variable.

**Backing up without encrypting.** A backup file on an unencrypted volume defeats the purpose of encrypting your live data. Use Restic or similar tools that encrypt by default.

**No monitoring.** You won't know you've been compromised if you're not watching. Set up logging and alerting before you need it, not after.

## The Payoff

When you self-host AI tools with proper security, you get something cloud services can't offer: **complete control over your data's lifecycle.** Your customer data, financial documents, and business intelligence never leave your network. No third party can change their terms of service and start scanning your data. No API outage takes down your workflows.

The tradeoff is that security is your responsibility. But as this guide shows, the measures that matter most—network isolation, encryption, access control, monitoring—are well-documented and straightforward to implement. An afternoon of configuration work buys you a security posture that rivals any enterprise cloud contract.

Start with the checklist. Fix the gaps. Then sleep better knowing your automation stack is locked down.

---

*Want help securing your AI automation infrastructure? ARDOT Consulting specializes in self-hosted, open source deployments that keep your data under your control. [Contact us](https://www.ardotconsulting.com/#contact) to schedule a security review of your automation stack.*