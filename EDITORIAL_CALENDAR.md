# ARDOT Consulting — Editorial Calendar

> **Purpose:** Guide blog post creation to sustain a 1-post-every-2-days cadence.
> **Last updated:** 2026-08-16 by @ardot_automator

## Content Pillars

1. **Practical AI (How-To & Education)** — Step-by-step guides, tutorials, best practices
2. **Industry-Specific Automation** — Legal, healthcare, finance, retail, e-commerce
3. **AI Strategy & Decision-Making** — Build vs buy, ROI, risk, security
4. **Open Source Tool Reviews** — n8n, Ollama, Huginn, Plausible, Odoo, etc.
5. **Case Studies & Results** — Real (or realistic) automation transformations

## Style Guide

- **Tone:** Practical, approachable, no hype. Show, don't tell.
- **Audience:** Business owners and operations managers — curious about AI but not technical
- **Length:** 1,500–2,500 words
- **Rules:**
  - NO Google tools/services — lead with open source alternatives
  - Reference real tools: n8n, Ollama, Huginn, Plausible, Odoo, Directus, Tesseract, Cal.com
  - Include a CTA linking to the contact form on the homepage
  - Include code snippets, comparison tables, or practical examples where relevant
  - Jekyll frontmatter: `layout: post`, title, date, author, tags, excerpt

## Published Posts

| Date | Title | Author |
|------|-------|--------|
| 2026-08-02 | What AI Can Actually Do for Your Business Today | @ardot_coder_bot |
| 2026-08-02 | 5 Repetitive Tasks Every Small Business Should Automate | @ardot_automator |
| 2026-08-05 | Open Source AI Tools for Small Business Automation | @ardot_automator |

## Upcoming Posts (30-Day Calendar)

| Date | Title | Pillar | Tags | Agent |
|------|-------|--------|------|-------|
| 2026-08-17 | How to Build Your First AI Automation Workflow with n8n | Practical AI (How-To) | n8n, automation, tutorial, workflow | cron |
| 2026-08-19 | AI Automation for Law Firms: Document Review and Intake | Industry-Specific | legal, document-intelligence, automation | cron |
| 2026-08-21 | Build vs Buy: When to Custom-Build AI Automation vs Use a Platform | AI Strategy | build-vs-buy, strategy, decision-making | cron |
| 2026-08-23 | Running Local LLMs with Ollama: A Practical Guide for Businesses | Open Source Tool Review | ollama, local-llm, self-hosting, privacy | cron |
| 2026-08-25 | AI Automation for Healthcare: Patient Scheduling and Records | Industry-Specific | healthcare, scheduling, automation, privacy | cron |
| 2026-08-27 | The Real ROI of AI Automation: How to Measure It | AI Strategy | roi, metrics, business-case | cron |
| 2026-08-29 | Self-Hosting Plausible Analytics: A Step-by-Step Docker Guide | Open Source Tool Review | plausible, analytics, self-hosting, docker, privacy | cron |
| 2026-08-31 | AI Automation for E-Commerce: Inventory, Pricing, and Customer Service | Industry-Specific | e-commerce, retail, automation, inventory | cron |
| 2026-09-02 | 5 Common AI Automation Mistakes (And How to Avoid Them) | Practical AI | mistakes, best-practices, automation | cron |
| 2026-09-04 | Huginn vs n8n: Which Open Source Automation Tool Is Right for You? | Open Source Tool Review | huginn, n8n, comparison, automation | cron |
| 2026-09-06 | How ARDOT Automated Its Own Blog: A Behind-the-Scenes Case Study | Case Study | case-study, blog, automation, meta | cron |
| 2026-09-08 | AI Automation for Accounting: Invoice Processing and Reconciliation | Industry-Specific | accounting, finance, invoice, ocr | cron |
| 2026-09-10 | Why Open Source AI Is Safer for Your Business Data | Thought Leadership | open-source, privacy, data-security, strategy | cron |
| 2026-09-12 | Setting Up Odoo Community Edition: CRM Automation for Small Business | Open Source Tool Review | odoo, crm, automation, self-hosting | cron |
| 2026-09-14 | From Manual to Automated: A 30-Day AI Automation Transformation | Case Study | case-study, transformation, 30-day-plan | cron |
| 2026-09-16 | AI Automation Security: Protecting Your Workflows and Data | Practical AI | security, best-practices, automation, privacy | cron |

## Topic Descriptions

### 2026-08-17 — How to Build Your First AI Automation Workflow with n8n
**Pillar:** Practical AI (How-To)
**Tags:** n8n, automation, tutorial, workflow
**Description:** Step-by-step guide: install n8n (self-hosted via Docker), create a workflow that monitors an email inbox, extracts key info with a local LLM via Ollama, and posts to a Slack-compatible channel. Include screenshots/diagrams and code snippets.

### 2026-08-19 — AI Automation for Law Firms: Document Review and Intake
**Pillar:** Industry-Specific
**Tags:** legal, document-intelligence, automation
**Description:** How AI can automate document review, client intake forms, and contract analysis for small law firms. Reference open source OCR (Tesseract), local LLMs (Ollama) for privacy-sensitive legal data. Real-world workflow examples.

### 2026-08-21 — Build vs Buy: When to Custom-Build AI Automation vs Use a Platform
**Pillar:** AI Strategy
**Tags:** build-vs-buy, strategy, decision-making
**Description:** Decision framework for choosing between building custom AI automation (n8n + Ollama) vs buying a SaaS platform. Cost analysis, maintenance burden, data privacy, flexibility. Include a comparison table and decision tree.

### 2026-08-23 — Running Local LLMs with Ollama: A Practical Guide for Businesses
**Pillar:** Open Source Tool Review
**Tags:** ollama, local-llm, self-hosting, privacy
**Description:** Complete guide to running Ollama locally: installation, model selection (Llama, Qwen, Mistral), hardware requirements, integrating with n8n workflows, and when local inference beats cloud APIs. Include benchmark comparisons.

### 2026-08-25 — AI Automation for Healthcare: Patient Scheduling and Records
**Pillar:** Industry-Specific
**Tags:** healthcare, scheduling, automation, privacy
**Description:** How small clinics can automate appointment scheduling, patient reminders, and records management with open source tools. Emphasize HIPAA considerations and why self-hosted AI (Ollama) matters for healthcare data.

### 2026-08-27 — The Real ROI of AI Automation: How to Measure It
**Pillar:** AI Strategy
**Tags:** roi, metrics, business-case
**Description:** Practical framework for measuring automation ROI: time saved, error reduction, throughput increase, employee satisfaction. Include a downloadable calculator template (link to a simple spreadsheet). Real numbers from small business examples.

### 2026-08-29 — Self-Hosting Plausible Analytics: A Step-by-Step Docker Guide
**Pillar:** Open Source Tool Review
**Tags:** plausible, analytics, self-hosting, docker, privacy
**Description:** How to self-host Plausible Analytics with Docker Compose instead of paying for Plausible Cloud. Includes docker-compose.yml, config for custom domain (stats.ardotconsulting.com), and migration from cloud to self-hosted.

### 2026-08-31 — AI Automation for E-Commerce: Inventory, Pricing, and Customer Service
**Pillar:** Industry-Specific
**Tags:** e-commerce, retail, automation, inventory
**Description:** How small online retailers can automate inventory sync, dynamic pricing, and customer service responses using n8n, Ollama, and Odoo. Include a workflow diagram showing the integration points.

### 2026-09-02 — 5 Common AI Automation Mistakes (And How to Avoid Them)
**Pillar:** Practical AI
**Tags:** mistakes, best-practices, automation
**Description:** Real-world mistakes businesses make when implementing AI automation: over-automating, ignoring edge cases, not training staff, choosing wrong tools, no monitoring. Each mistake with a concrete fix.

### 2026-09-04 — Huginn vs n8n: Which Open Source Automation Tool Is Right for You?
**Pillar:** Open Source Tool Review
**Tags:** huginn, n8n, comparison, automation
**Description:** Head-to-head comparison of Huginn (agent-based) vs n8n (workflow-based). Architecture, ease of use, community, licenses, best use cases. Include a comparison table and recommendation for different business sizes.

### 2026-09-06 — How ARDOT Automated Its Own Blog: A Behind-the-Scenes Case Study
**Pillar:** Case Study
**Tags:** case-study, blog, automation, meta
**Description:** Meta post: how we use AI agents to research, write, and publish blog posts every 2 days automatically. The cron job, the Jekyll setup, the Kanban coordination between 3 AI agents. Real architecture diagram.

### 2026-09-08 — AI Automation for Accounting: Invoice Processing and Reconciliation
**Pillar:** Industry-Specific
**Tags:** accounting, finance, invoice, ocr
**Description:** How small accounting firms can automate invoice extraction (Tesseract OCR + Ollama), bank reconciliation, and report generation. Include a working code snippet for PDF invoice extraction.

### 2026-09-10 — Why Open Source AI Is Safer for Your Business Data
**Pillar:** Thought Leadership
**Tags:** open-source, privacy, data-security, strategy
**Description:** Argument piece: why self-hosted open source AI (Ollama, n8n) is safer than sending data to OpenAI/Google APIs. Data sovereignty, no vendor lock-in, audit-able code, no surprise price hikes. Reference real incidents.

### 2026-09-12 — Setting Up Odoo Community Edition: CRM Automation for Small Business
**Pillar:** Open Source Tool Review
**Tags:** odoo, crm, automation, self-hosting
**Description:** Guide to installing and configuring Odoo Community Edition for small business CRM. Lead capture, automated follow-ups, pipeline management. Docker setup guide + integration with n8n for extended automation.

### 2026-09-14 — From Manual to Automated: A 30-Day AI Automation Transformation
**Pillar:** Case Study
**Tags:** case-study, transformation, 30-day-plan
**Description:** Fictional but realistic case study: a 15-person consulting firm goes from fully manual to 60% automated in 30 days. Week-by-week breakdown, tools used (n8n, Ollama, Odoo), time saved, lessons learned.

### 2026-09-16 — AI Automation Security: Protecting Your Workflows and Data
**Pillar:** Practical AI
**Tags:** security, best-practices, automation, privacy
**Description:** Security best practices for AI automation: API key management, network segmentation for self-hosted tools, encrypting data at rest, access control in n8n, secure Ollama deployment. Include a security checklist.

## Notes

- The cron job (`b105859080fa`) runs every 2 days and picks the next topic from this calendar
- Posts are automatically written, pushed to `_posts/`, and deployed via GitHub Pages
- @ardot_researcher_bot should review posts for accuracy after publication
- Topics can be reordered or swapped — just update this file
- After all 15 topics are published, create a new calendar for the next 30 days
