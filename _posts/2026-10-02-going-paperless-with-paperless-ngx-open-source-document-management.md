---
layout: post
title: "Going Paperless with Paperless-ngx: Open-Source Document Management for Your Business"
date: 2026-10-02
author: "ARDOT Consulting"
tags: [paperless-ngx, document-management, ocr, self-hosting, docker, open-source, paperless]
excerpt: "Every business drowns in paper and PDFs. Paperless-ngx is the open-source document management system that automatically scans, OCRs, tags, and files everything — self-hosted, no subscriptions, your data stays yours."
---

Every business produces documents. Invoices from vendors. Signed contracts. Receipts. Tax forms. Insurance certificates. Employee onboarding paperwork. Utility bills. Bank statements. They arrive as paper in the mail, as PDF attachments in email, as scanned images from a multifunction printer — and they all end up in the same place: a filing cabinet, a shared network drive, or worse, someone's desktop folder named "Stuff to deal with later."

You know the feeling. You need a specific invoice from eight months ago, and you spend forty-five minutes searching through folders, email threads, and physical binders before finally finding it — or giving up and asking the vendor to resend it. That's not just annoying. It's expensive. A study by IDC put the cost of searching for misplaced documents at around $20 per document in lost productivity. For a small business generating hundreds of documents a month, that adds up fast.

The commercial document management market is full of solutions — but they come with per-user pricing, storage limits, and the quiet assumption that you're comfortable handing your most sensitive business documents to a third-party cloud. Paperless-ngx is the open-source alternative. It's a self-hosted document management system that automatically ingests documents, runs optical character recognition (OCR) to make them searchable, assigns tags and categories, and stores everything in a clean, queryable archive. No per-user fees. No storage limits beyond your own server's disk. No vendor seeing your contracts.

In this guide, we'll cover what Paperless-ngx does, how it compares to commercial alternatives, and how to deploy it with Docker in an afternoon.

## What Paperless-ngx Actually Does

Paperless-ngx is a document management system built around a simple idea: documents should be searchable, organized, and stored on infrastructure you control. It was forked from the original Paperless project (which went dormant) and has become the de facto open-source standard for small-business and personal document management, with an active community and regular releases.

Here's what it does out of the box:

### 1. Document Ingestion

Paperless-ngx accepts documents from multiple sources:

- **Consume folder:** Drop a PDF or image into a watched directory, and Paperless picks it up automatically. This is how most scanner integrations work — your multifunction printer scans to a network share, Paperless sees the file, and the rest is automatic.
- **Email integration:** Point Paperless at an email account (via IMAP), and it will monitor incoming messages, download attachments, and ingest them. Set rules so only messages from specific senders or with specific subjects are processed. This is perfect for vendor invoices that arrive as email attachments.
- **Web upload:** Drag and drop files directly through the Paperless web interface. Useful for one-off documents or when you're cleaning out a filing cabinet.
- **API:** For the technically inclined, Paperless has a REST API. You can push documents programmatically from other systems — an n8n workflow, a script, or another application.

Supported formats include PDF, PNG, JPEG, TIFF, and even HEIC (Apple's image format). If it's a document, Paperless can probably handle it.

### 2. OCR and Full-Text Search

This is where Paperless earns its keep. Every document that enters the system runs through Tesseract OCR — the same open-source OCR engine we've recommended in our [accounting automation](/blog/2026/09/05/ai-automation-for-accounting-invoice-processing-and-reconciliation/) and [legal document](/blog/2026/08/18/ai-automation-for-law-firms-document-review-and-intake/) posts. Tesseract extracts the text from scanned images and PDFs, making everything searchable.

That means you can search for "Acme Corporation" and find every document — invoices, contracts, correspondence — that mentions that name, regardless of whether it was a born-digital PDF or a blurry scan of a faxed purchase order. The OCR supports over 100 languages, and you can configure multiple languages simultaneously if your business deals with multilingual documents.

The search is fast. Paperless uses Whoosh (a Python search library) for full-text indexing, and searches return results in milliseconds across thousands of documents.

### 3. Automatic Classification

Here's the feature that surprises people: Paperless can automatically tag, categorize, and assign correspondents to documents based on their content. You define rules — called "matching algorithms" — and Paperless applies them to every new document.

For example:

- Any document containing the words "invoice" and "Acme Corp" gets the tag "Invoice," the correspondent "Acme Corporation," and the document type "Invoice."
- Anything from the email address `tax@irs.gov` gets the tag "Tax" and is filed under the "Government" document type.
- Documents mentioning "certificate of insurance" are automatically tagged "Insurance" and assigned the storage path "Insurance / 2026."

Three matching algorithms are available:

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Any Word** | Matches if any of the specified keywords appear | Broad categories (e.g., "invoice," "receipt") |
| **All Words** | Matches only if all specified keywords appear | Specific document types (e.g., "certificate" AND "insurance") |
| **Exact Match** | Matches an exact string | Document IDs, specific phrases |

You can combine tags, correspondents, document types, and storage paths in a single matching rule. Once you've tuned your rules over the first few weeks, most documents are filed correctly without any human intervention.

### 4. Document Organization

Paperless organizes documents through four main constructs:

- **Correspondents:** Who sent or is associated with the document (e.g., "Acme Corporation," "City Water Department," "Jane Doe").
- **Document Types:** What kind of document it is (e.g., "Invoice," "Contract," "Receipt," "Tax Form").
- **Tags:** Flexible labels for anything else (e.g., "Urgent," "Q4 2026," "Pending Signature").
- **Storage Paths:** Where the document's original file lives on disk (e.g., "Contracts/2026/" or "Tax/2026/").

Every document can have multiple tags, one correspondent, one document type, and one storage path. The combination gives you powerful filtering: "Show me all invoices from Acme Corporation tagged 'Q4 2026'" is a two-click query.

### 5. Retention Policies

For businesses that need to retain documents for specific periods (tax records for seven years, employment records for varying durations depending on jurisdiction), Paperless supports retention policies. You can set rules like "delete receipts after 3 years" or "archive tax forms after 7 years." The policies apply based on document type and tags, so you can be as specific as you need.

This matters more than people realize. Holding onto documents longer than necessary creates clutter and, in some cases, legal risk (keeping records you were supposed to destroy). Holding them for less time than required creates compliance risk. Paperless lets you set it once and forget it.

## Paperless-ngx vs. Commercial Document Management

Let's be honest about the comparison. Commercial document management systems — things like DocuWare, M-Files, or even the document features baked into SharePoint — have more features. They have workflow engines for approval processes, tighter integration with enterprise identity providers, and dedicated sales teams.

But for a small or medium business, those features often come at a cost that's hard to justify. Here's how the comparison actually shakes out:

| Feature | Paperless-ngx | Commercial DMS (typical) |
|---------|--------------|--------------------------|
| **Cost** | Free (self-hosted) | $15–$50/user/month + storage fees |
| **Storage limits** | Your server's disk | Typically tiered, extra cost |
| **OCR** | Tesseract (unlimited) | Often per-page pricing above a quota |
| **Data location** | Your server | Vendor's cloud |
| **User accounts** | Unlimited | Per-seat licensing |
| **Search** | Full-text, all documents | Full-text (usually) |
| **Auto-classification** | Keyword matching | Often AI/ML-based (more accurate, more complex) |
| **Workflow/approvals** | Not built-in (use n8n) | Native |
| **Mobile access** | Responsive web app | Usually native apps |
| **Setup time** | 1–2 hours | Days to weeks (vendor onboarding) |

The tradeoff is clear: Paperless-ngx gives you 80% of what most small businesses need from a document management system at 0% of the licensing cost. What you give up is workflow automation (which you can fill with n8n) and the polish of a commercial product. For a 5-to-50-person business that wants to stop losing documents and start searching their archive, that's a trade worth making.

## Deploying Paperless-ngx with Docker

Paperless-ngx is designed to be self-hosted, and the recommended deployment method is Docker Compose. You'll need a server (a VPS with 2 vCPU and 4GB RAM is plenty for a small business), Docker, and Docker Compose installed.

### Step 1: Create the Project Directory

```bash
mkdir -p /opt/paperless-ngx && cd /opt/paperless-ngx
```

### Step 2: Create the Docker Compose File

```yaml
version: "3.8"

services:
  broker:
    image: docker.io/library/redis:7
    restart: unless-stopped
    volumes:
      - redisdata:/data

  db:
    image: docker.io/library/postgres:16
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: ${PAPERLESS_DB_PASSWORD}

  paperless:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - db
      - broker
    ports:
      - "8000:8000"
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - export:/usr/src/paperless/export
      - consume:/usr/src/paperless/consume
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_DBNAME: paperless
      PAPERLESS_DBUSER: paperless
      PAPERLESS_DBPASS: ${PAPERLESS_DB_PASSWORD}
      PAPERLESS_SECRET_KEY: ${PAPERLESS_SECRET_KEY}
      PAPERLESS_URL: https://docs.yourbusiness.com
      PAPERLESS_OCR_LANGUAGE: eng
      PAPERLESS_CONSUMER_POLLING: 60
      PAPERLESS_CONSUMER_RECURSIVE: true
      PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS: true
      PAPERLESS_TIME_ZONE: America/New_York

volumes:
  data:
  media:
  pgdata:
  redisdata:
  export:
  consume:
```

### Step 3: Create the Environment File

```bash
# .env file — keep this secret, don't commit it
PAPERLESS_DB_PASSWORD=change_this_to_a_strong_password
PAPERLESS_SECRET_KEY=change_this_to_a_random_50_char_string
```

Generate a strong secret key:

```bash
openssl rand -hex 24
```

### Step 4: Start the Services

```bash
docker compose up -d
```

### Step 5: Create Your Admin User

```bash
docker compose run --rm paperless manage createsuperuser
```

Follow the prompts to set your admin username and password.

### Step 6: Set Up HTTPS with a Reverse Proxy

Paperless-ngx itself runs on port 8000 over HTTP. For production, you should put it behind a reverse proxy with TLS. Caddy is the simplest option — it handles HTTPS certificates automatically:

```yaml
# Add to your Caddyfile or docker-compose
docs.yourbusiness.com {
    reverse_proxy paperless:8000
}
```

Caddy will automatically obtain and renew a Let's Encrypt certificate. No manual cert management required.

### Step 7: Start Scanning

Drop a PDF or image into the `consume` folder (or configure your scanner to save there), and within 60 seconds Paperless will pick it up, OCR it, and add it to your archive. Log into the web interface to review, tag, and organize.

## Practical Workflows for Small Business

Here's how businesses actually use Paperless-ngx day-to-day:

### Invoice Processing

Configure Paperless to monitor the email account where vendor invoices arrive. Every PDF attachment gets ingested, OCR'd, and tagged as "Invoice" based on your matching rules. Your bookkeeper logs in once a week, reviews the tagged invoices, and exports them — or better yet, use n8n to push the extracted data into your accounting system.

### Contract Management

Every signed contract gets uploaded (via scanner, email, or web interface). Tag by client name, contract type, and renewal date. When you need to review a contract before renewal, search by client name and filter by "Contract" — it's a five-second lookup instead of a forty-minute dig through filing cabinets.

### Tax Document Organization

Create a "Tax 2026" tag. Throughout the year, tag every tax-relevant document — 1099s, receipts for deductible expenses, property tax bills, charitable donation acknowledgments. When tax season arrives, filter by the tag and export everything for your accountant. Set a retention policy to archive the tag after seven years.

### Employee Records

Onboarding paperwork, performance reviews, training certificates — all tagged by employee name and document type. Set retention policies based on your legal requirements. When an employee leaves, you know exactly where everything is and when it can be destroyed.

## Integrating Paperless-ngx with Your Automation Stack

Paperless-ngx has a REST API, which means it plays well with the automation tools we've covered elsewhere on this blog. A few integration patterns:

**Paperless + n8n + Ollama:** When a new document is ingested, trigger an n8n workflow that sends the OCR'd text to a local LLM (via Ollama) for classification. The LLM can extract structured data — vendor name, invoice number, total amount, due date — and push it to your accounting system. This is a more sophisticated version of the keyword matching that Paperless does natively.

**Paperless + n8n + notifications:** Set up a webhook so that when a document matching certain criteria is added (e.g., a document tagged "Urgent"), n8n sends a notification to your team chat.

**Paperless + Metabase:** Export Paperless's document metadata to a database and connect Metabase to visualize it — how many invoices per month, average processing time, document volume by correspondent. This gives you analytics on your document workflow.

## Hardware Requirements and Performance

Paperless-ngx is not a heavy application. For a small business with a few thousand documents:

- **CPU:** 2 vCPU is sufficient. OCR is the most CPU-intensive task, and it's parallelized.
- **RAM:** 4GB minimum. 8GB if you're processing large batches of scanned documents.
- **Storage:** Plan for roughly 1–2MB per document (after compression). 10,000 documents = ~15GB. A 100GB disk gives you years of headroom.
- **Database:** PostgreSQL is recommended for any production deployment. SQLite works for testing but isn't recommended for multi-user use.

OCR processing time depends on document length and CPU. A typical 5-page scanned invoice takes 10–20 seconds on a 2 vCPU server. Paperless processes documents asynchronously, so this doesn't block the web interface.

## Migration: Getting Your Existing Documents In

If you're migrating from an existing system (filing cabinet, network drive, Google Drive export), the process is straightforward but takes time:

1. **Scan paper documents.** A sheet-fed scanner (anything from a $200 desktop scanner to a multifunction printer) can batch-scan years of paper. Save scans as PDF to the consume folder.
2. **Batch import digital files.** Paperless includes a `document_importer` management command that can ingest a directory of existing files, preserving creation dates from file metadata.
3. **Clean up as you go.** The first pass will have classification errors. Spend a week reviewing and correcting tags — your matching rules will improve as a result.

A realistic timeline: a small business with a filing cabinet and a few hundred digital documents can be fully migrated in a weekend. A larger archive (10,000+ documents) is a multi-week project, but you can do it incrementally — start using Paperless for new documents immediately, and backfill the archive over time.

## Backups: Non-Negotiable

Paperless-ngx stores your documents on your server. That's the whole point — your data, your control. But it also means you're responsible for backups. A single disk failure without a backup means losing your entire document archive.

At minimum:

1. **Back up the media volume** (where document files are stored) daily. This is the irreplaceable data.
2. **Back up the database** (PostgreSQL) daily. This stores all metadata, tags, and search indices.
3. **Store backups off-site.** Use `restic` or `borg` to push encrypted backups to a separate location — a different VPS, an S3-compatible storage service like Backblaze B2 or MinIO.
4. **Test your restores.** A backup you've never restored is a hope, not a backup.

Paperless also has a built-in export feature (`document_exporter`) that creates a complete export of all documents and metadata. Schedule it weekly via cron and include the export in your backup routine.

## Common Pitfalls

**Underestimating OCR quality.** Tesseract is good but not perfect. Handwritten notes, low-resolution scans, and documents with unusual fonts will produce OCR errors. Invest in a decent scanner — 300 DPI is the minimum for reliable OCR, 600 DPI is better for small text.

**Over-tagging.** It's tempting to create a tag for every possible categorization. Resist. Start with 5–10 tags and 3–5 document types. You can always add more. Too many tags makes the system harder to use, not easier.

**Ignoring the consume folder permissions.** If your scanner saves files as one user but Paperless runs as another, file permission issues will silently prevent ingestion. Make sure the consume folder is writable by the Paperless container user.

**No retention policy.** Without one, your archive grows indefinitely. Set retention policies early, even if they're conservative (e.g., "archive after 10 years"). You can tighten them later.

## The Bottom Line

Paperless-ngx solves a problem that every business has and most businesses handle badly. Document chaos isn't a technology problem — it's a process problem that technology can fix. The question isn't whether you need document management. You do. The question is whether you'll pay a recurring subscription for a commercial product that holds your most sensitive documents, or whether you'll spend an afternoon setting up an open-source system that gives you the same core capabilities on your own terms.

For most small and medium businesses, the answer is clear. Paperless-ngx gives you searchable, organized, automatically-classified document storage with no per-user fees, no storage limits, and no vendor looking at your files. Pair it with n8n for workflow automation and Ollama for intelligent classification, and you have a document management stack that rivals commercial systems at a fraction of the cost.

If you'd like help deploying Paperless-ngx or building a document automation workflow for your business, [reach out through our contact form](/#contact). We specialize in open-source automation for small and medium businesses — practical solutions, no vendor lock-in, your data stays yours.

---

*Paperless-ngx is open-source software released under the GPL-3.0 license. It's community-maintained and free to use. You can find the project documentation and source code at [docs.paperless-ngx.com](https://docs.paperless-ngx.com).*