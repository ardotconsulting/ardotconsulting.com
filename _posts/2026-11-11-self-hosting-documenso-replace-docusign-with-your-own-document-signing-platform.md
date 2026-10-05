---
layout: post
title: "Self-Hosting Documenso: Replace DocuSign With Your Own Document Signing Platform"
date: 2026-11-11
author: "ARDOT Consulting"
tags: [documenso, document-signing, self-hosting, open-source, e-signature, docker, agpl]
excerpt: "DocuSign costs add up fast. Documenso is a 15,000-star open source e-signature platform you can self-host — here's how to set it up with Docker, what you get, and when it makes sense for your business."
---

# Self-Hosting Documenso: Replace DocuSign With Your Own Document Signing Platform

Every business signs documents. Contracts, NDAs, offer letters, vendor agreements, client onboarding forms — the list never ends. And for most businesses, that means paying DocuSign $15–$45 per user per month, forever, for the privilege of sending PDFs through someone else's servers.

What if you could run your own e-signature platform? One you control, with no per-envelope fees, no per-seat costs, and no third party holding your signed contracts?

That's exactly what **Documenso** offers. It's an open source document signing platform with over 15,000 GitHub stars, a polished web interface, templates, team management, API access, and compliance features that rival commercial alternatives. And you can self-host it on a single server for the cost of a $10/month VPS.

Let's walk through what Documenso is, how it compares to DocuSign, and how to get it running on your own infrastructure.

## What Is Documenso?

Documenso is an open source e-signature platform founded in 2022. It's built with TypeScript, Next.js, Prisma, and PostgreSQL — a modern stack that makes it straightforward to deploy and extend. The project is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**, which means you can use it, modify it, and self-host it freely, but if you modify the source code and provide it as a service to others, you must share those modifications back under the same license.

The community edition (CE) is the free, self-hostable version. Documenso also offers a cloud-hosted version with a free tier and paid plans, plus an enterprise license for organizations that need advanced compliance and support. But the community edition is fully functional — you're not running a crippled demo.

Key features in the self-hosted community edition include:

- **Document signing workflows** — upload a PDF, drag fields onto it (signature, text, date, checkbox), and send it to one or more recipients
- **Templates** — create reusable document templates so you're not starting from scratch every time
- **Team management** — invite team members, assign roles, share documents across your organization
- **Signing certificates** — each completed document includes a tamper-evident audit trail with timestamps, IP addresses, and signer details
- **API and webhooks** — integrate signing into your existing workflows programmatically
- **White-label** — remove Documenso branding and use your own domain
- **Multi-language support** — the interface has been translated into multiple languages by the community

## Documenso vs DocuSign: A Practical Comparison

Before diving into setup, let's be honest about where Documenso stands relative to the market leader.

| Feature | DocuSign | Documenso (Self-Hosted CE) |
|---------|----------|---------------------------|
| **Monthly cost** | $15–$45/user/month | Server cost only (~$10/month) |
| **Envelope limits** | Tier-dependent (100–unlimited) | Unlimited |
| **User limits** | Per-seat pricing | Unlimited |
| **Templates** | Yes (higher tiers) | Yes |
| **API access** | Yes (higher tiers) | Yes, included |
| **Webhooks** | Yes (higher tiers) | Yes, included |
| **White-label** | Enterprise only | Included |
| **Audit trail** | Yes | Yes (signing certificate) |
| **Compliance certifications** | SOC 2, HIPAA, eIDAS, 21 CFR Part 11 | ESIGN Act, UETA, 21 CFR Part 11 (enterprise tier) |
| **Data residency** | US/EU regions (varies by plan) | Wherever you deploy it |
| **Source code access** | Closed source | Open source (AGPL-3.0) |
| **Setup time** | 5 minutes (SaaS) | 30–60 minutes (Docker) |
| **Maintenance** | Zero | Periodic updates, backups |

The trade-off is clear. DocuSign is zero-maintenance and has deeper compliance certifications out of the box. Documenso self-hosted gives you unlimited everything, full data control, and no recurring per-user fees — but you're responsible for updates, backups, and security.

For a 10-person team on DocuSign's Business Standard plan at $45/user/month, that's **$5,400 per year**. A self-hosted Documenso instance on a $10/month VPS costs **$120 per year**. Even adding a few hours of setup and maintenance, the savings are significant.

## When Self-Hosting Makes Sense (And When It Doesn't)

**Self-host Documenso if:**

- You send a high volume of documents and per-envelope or per-seat fees are eating into margins
- You need full control over where signed documents are stored (regulated industries, government contracts, international data residency requirements)
- You already self-host other tools (n8n, Odoo, Nextcloud) and have the infrastructure in place
- You want to integrate e-signatures into automated workflows via API without paying for DocuSign's API tier
- You're comfortable with Docker and basic server administration

**Stick with DocuSign (or Documenso Cloud) if:**

- You need SOC 2 Type II or HIPAA certification documentation for clients or regulators (the self-hosted CE doesn't include these)
- You sign fewer than 10 documents per month — the setup cost isn't worth it
- You don't have anyone on your team who can maintain a server
- You need eIDAS Qualified Electronic Signatures (QES) for EU legal compliance — Documenso doesn't support QES yet

## Prerequisites

To self-host Documenso, you'll need:

- A server running Linux (Ubuntu 22.04+ or Debian 12+) with at least 2GB RAM
- Docker and Docker Compose installed
- A domain name (e.g., `sign.yourcompany.com`)
- An SMTP server for sending email notifications (you can use a transactional email service like Postmark, Resend, or self-hosted Mailcow)
- Optional: A PostgreSQL database — the Docker setup includes one, but for production you may want a managed instance

## Step 1: Clone and Configure

Start by cloning the Documenso repository:

```bash
git clone https://github.com/documenso/documenso.git
cd documenso
```

Copy the example environment file:

```bash
cp .env.example .env
```

Open `.env` and configure the key variables. Here's what a production setup looks like:

```bash
# Database
DATABASE_URL=postgresql://documenso:your_secure_password@localhost:5432/documenso
DIRECT_DATABASE_URL=postgresql://documenso:your_secure_password@localhost:5432/documenso

# Next.js public URL — your domain
NEXTAUTH_URL=https://sign.yourcompany.com
NEXTAUTH_SECRET=generate_a_random_32_char_string

# SMTP for email notifications
NEXT_PRIVATE_SMTP_HOST=smtp.yourprovider.com
NEXT_PRIVATE_SMTP_PORT=587
NEXT_PRIVATE_SMTP_USERNAME=your_username
NEXT_PRIVATE_SMTP_PASSWORD=your_password
NEXT_PRIVATE_SMTP_FROM_NAME=Your Company
NEXT_PRIVATE_SMTP_FROM_ADDRESS=noreply@yourcompany.com

# Document storage — local filesystem by default
NEXT_PRIVATE_DOCUMENT_STORAGE_PROVIDER=fs
NEXT_PRIVATE_DOCUMENT_STORAGE_LOCAL_ROOT_DIRECTORY=/data/documents

# Optional: S3-compatible storage instead of local
# NEXT_PRIVATE_DOCUMENT_STORAGE_PROVIDER=s3
# NEXT_PRIVATE_DOCUMENT_STORAGE_S3_BUCKET=documenso
# NEXT_PRIVATE_DOCUMENT_STORAGE_S3_REGION=us-east-1
# NEXT_PRIVATE_DOCUMENT_STORAGE_S3_ACCESS_KEY_ID=your_key
# NEXT_PRIVATE_DOCUMENT_STORAGE_S3_SECRET_ACCESS_KEY=your_secret
```

Generate a secure `NEXTAUTH_SECRET`:

```bash
openssl rand -base64 32
```

## Step 2: Deploy with Docker

Documenso provides a Docker setup in the `docker/` directory. Create a `docker-compose.yml` in your project root:

```yaml
version: "3.8"

services:
  documenso-db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: documenso
      POSTGRES_PASSWORD: your_secure_password
      POSTGRES_DB: documenso
    volumes:
      - documenso-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U documenso"]
      interval: 10s
      timeout: 5s
      retries: 5

  documenso:
    image: documenso/documenso:latest
    restart: unless-stopped
    depends_on:
      documenso-db:
        condition: service_healthy
    env_file: .env
    ports:
      - "3000:3000"
    volumes:
      - documenso-data:/data/documents

volumes:
  documenso-db-data:
  documenso-data:
```

Start the stack:

```bash
docker compose up -d
```

Check that it's running:

```bash
docker compose ps
docker compose logs -f documenso
```

You should see the app start up on port 3000. The first run will run database migrations automatically.

## Step 3: Reverse Proxy with SSL

You'll want to put a reverse proxy in front of Documenso to handle SSL and route traffic. Using Caddy is the simplest option — it handles Let's Encrypt certificates automatically:

```bash
# Install Caddy
sudo apt install caddy -y
```

Create or edit `/etc/caddy/Caddyfile`:

```
sign.yourcompany.com {
    reverse_proxy localhost:3000
}
```

Restart Caddy:

```bash
sudo systemctl restart caddy
```

Caddy will automatically provision an SSL certificate and redirect HTTP to HTTPS. Within a minute, your Documenso instance should be accessible at `https://sign.yourcompany.com`.

If you prefer Nginx, here's an equivalent config:

```nginx
server {
    listen 80;
    server_name sign.yourcompany.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name sign.yourcompany.com;

    ssl_certificate /etc/letsencrypt/live/sign.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sign.yourcompany.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Use `certbot --nginx -d sign.yourcompany.com` to provision the SSL certificate.

## Step 4: Create Your First Document

Once Documenso is running, navigate to your domain and create an account. The first registered user becomes the administrator.

1. Click **Sign Up** and create your account
2. Click **Create Document**
3. Upload a PDF (e.g., a contract template)
4. Drag signature, text, and date fields onto the document
5. Add recipient email addresses
6. Click **Send**

Your recipients will receive an email with a link to review and sign the document. Once everyone has signed, Documenso generates a signed PDF with a completion certificate showing who signed, when, and from what IP address.

## Step 5: Set Up Templates for Repeated Use

If you send the same type of document frequently (NDAs, offer letters, service agreements), create a template:

1. Go to **Templates** → **New Template**
2. Upload your standard PDF
3. Place fields in the same locations every time
4. Define which fields are fixed (your content) vs. variable (recipient-specific)
5. Save the template

Next time you need to send that document, just select the template, enter the recipient's email, and fill in any variable fields. This alone saves 10–15 minutes per document compared to starting from scratch.

## Step 6: Automate with the API

This is where Documenso gets really powerful for businesses with automated workflows. The API lets you create and send documents programmatically — no manual UI interaction required.

Here's a Python example using the Documenso API:

```python
import requests

DOCU_MENSO_URL = "https://sign.yourcompany.com"
API_KEY = "your_api_key"  # Generate in Settings → API

# Create a document from an uploaded PDF
response = requests.post(
    f"{DOCU_MENSO_URL}/api/v1/documents",
    headers={
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    },
    json={
        "title": "Service Agreement — Acme Corp",
        "recipient": {
            "email": "client@acmecorp.com",
            "name": "Jane Smith",
        },
        "fields": [
            {
                "type": "signature",
                "position": {"x": 72, "y": 600, "width": 200, "height": 50},
                "page": 1,
            },
            {
                "type": "date",
                "position": {"x": 350, "y": 600, "width": 100, "height": 30},
                "page": 1,
            },
        ],
    },
)

document = response.json()
print(f"Document created: {document['id']}")
print(f"Signing link sent to client@acmecorp.com")
```

You can also set up **webhooks** to be notified when a document is signed, declined, or completed. This lets you trigger downstream workflows automatically — for example:

- When a contract is signed → create a project in Odoo
- When an NDA is signed → grant the signer access to a shared folder in Nextcloud
- When an offer letter is signed → trigger onboarding in your HR system

Pair Documenso with **n8n** and you can build a fully automated document pipeline: generate a PDF from a template, send it via Documenso, wait for signature, and trigger the next step — all without human intervention.

Here's an n8n workflow concept:

```
[Trigger: New CRM Deal Won]
    → [HTTP Request: Generate PDF from template]
    → [HTTP Request: Create Documenso document via API]
    → [Wait for Webhook: Document signed]
    → [HTTP Request: Create project in Odoo]
    → [Send Slack/Mattermost notification]
```

## Backup and Maintenance

Self-hosting means you're responsible for backups. Don't skip this.

**Database backups** — set up a daily cron job to dump the PostgreSQL database:

```bash
#!/bin/bash
# /opt/documenso-backup.sh
DATE=$(date +%Y%m%d)
docker exec documenso-documenso-db-1 pg_dump -U documenso documenso > /backups/documenso-db-$DATE.sql
# Keep last 30 days
find /backups -name "documenso-db-*.sql" -mtime +30 -delete
```

Add to crontab:

```bash
0 2 * * * /opt/documenso-backup.sh
```

**Document storage** — if you're using local filesystem storage, back up the Docker volume:

```bash
# Copy documents to backup location
docker run --rm -v documenso_documenso-data:/data -v /backups:/backup alpine \
  tar czf /backup/documenso-documents-$DATE.tar.gz /data
```

Better yet, use S3-compatible storage (MinIO, Backblaze B2, Wasabi) for documents so they're stored off-site by default.

**Updates** — Documenso is actively developed (over 4,200 commits and regular releases). To update:

```bash
cd documenso
git pull
docker compose pull
docker compose up -d
```

Check the [release notes](https://github.com/documenso/documenso/releases) before updating, and always test on a staging instance first if you're running production workloads.

## Security Checklist

Before going live with a self-hosted Documenso instance, run through this checklist:

- [ ] **SSL/TLS enabled** — use Caddy or Nginx with Let's Encrypt
- [ ] **Firewall configured** — only expose ports 80/443; keep PostgreSQL (5432) internal
- [ ] **Strong database password** — at least 24 characters, randomly generated
- [ ] **NEXTAUTH_SECRET set** — a unique, random 32+ character string
- [ ] **SMTP credentials secured** — use app-specific passwords, not main email passwords
- [ ] **Regular database backups** — automated daily, tested monthly
- [ ] **Document storage backed up** — S3 or regular volume snapshots
- [ ] **Access restricted** — consider IP allowlisting for internal-only instances
- [ ] **Updates scheduled** — check for new releases monthly, update quarterly at minimum
- [ ] **User management** — remove inactive users, review team membership regularly

## The Business Case

Let's put real numbers on this. Say you're a 15-person consulting firm that sends 200 documents per month for signature — contracts, NDAs, statements of work, change orders.

**DocuSign Business Premium** at $45/user/month for 15 users = **$8,100/year**

**Self-hosted Documenso:**
- VPS (4GB RAM): $24/month = $288/year
- Domain: $12/year
- Backblaze B2 storage (10GB): ~$0.50/month = $6/year
- Setup time (one-time): ~3 hours at $100/hour = $300
- Maintenance (4 hours/year): $400
- **Total year 1: $1,006**
- **Total year 2+: $706/year**

**Annual savings: ~$7,094**

That's not a hypothetical. Those are the actual costs for a real deployment. The savings scale linearly — add more users or more documents, and the DocuSign bill goes up while the Documenso bill stays flat.

## Integrations Worth Knowing About

Documenso plays well with other open source tools:

- **n8n** — automate document creation, sending, and post-signature workflows via the REST API
- **Odoo** — trigger contract generation when deals are won in the CRM
- **Nextcloud** — store signed documents directly in shared folders
- **Mattermost** — notify channels when documents are signed or declined
- **BookStack** — embed signing links in your internal knowledge base for standard forms
- **Cal.com** — send signature requests after booking confirmation (e.g., consultation agreements)

## Limitations to Be Aware Of

Documenso is excellent, but it's not perfect. Here's what to know before committing:

1. **No qualified electronic signatures (QES)** — DocuSign and some competitors offer QES for EU compliance, which carries higher legal weight. Documenso uses standard electronic signatures, which are legally binding under ESIGN/UETA in the US but may not satisfy all EU requirements.

2. **Compliance certifications** — the self-hosted CE doesn't ship with SOC 2 or HIPAA documentation. If your clients require these, you'll need the enterprise license or a different solution.

3. **Mobile experience** — the web interface works on mobile browsers, but there's no native mobile app. For most business use cases this is fine, but field workers who sign frequently may prefer a native app.

4. **Bulk sending** — sending to many recipients at once (e.g., 500 contractors) requires API scripting. The UI is designed for individual or small-group signing.

5. **AGPL license implications** — if you modify Documenso's source code and provide it as a service, you must share your modifications. For internal use, this doesn't apply. But if you're building a signing service for clients on top of modified Documenso code, consult a lawyer.

## Wrapping Up

Documenso has matured into a genuine DocuSign alternative for businesses that want control over their document signing process without paying per-seat SaaS fees forever. The self-hosted community edition gives you unlimited documents, unlimited users, API access, templates, and a polished interface — all for the cost of a small VPS.

If you're already self-hosting other tools, adding Documenso to your stack is a natural fit. If you're new to self-hosting, this is a good starting project — the Docker setup is straightforward, and the payoff (thousands of dollars in annual savings) is immediate and measurable.

The question isn't really whether you *can* self-host your document signing. The question is whether the $5,000–$8,000 you're paying DocuSign every year is buying you anything you can't get for free.

---

*Want help setting up Documenso or integrating it with your existing workflows? [Contact ARDOT Consulting](/) — we specialize in open source automation for small and mid-size businesses. No hype, no vendor lock-in, just practical solutions that save you money.*