---
layout: post
title: "AI Automation for Law Firms: Document Review and Client Intake"
date: 2026-08-18
author: "ARDOT Consulting"
tags: [legal, document-intelligence, automation, ocr, ollama, privacy]
excerpt: "How small law firms can automate document review and client intake with open source AI — keeping sensitive data on-premises instead of sending it to cloud APIs."
---

Law firms run on documents. Contracts, briefs, discovery files, intake forms, correspondence — the volume of text that flows through even a small practice is staggering. And every document needs to be read, categorized, summarized, and filed.

Historically, that work fell to paralegals and junior associates. It's painstaking, repetitive, and expensive. Today, AI can handle a significant portion of it — but for law firms, there's a catch that most AI vendors gloss over: **confidentiality**.

When you send client documents to a cloud AI API, you're handing privileged information to a third party. The ABA's formal opinion on AI (Formal Opinion 508, issued July 2024) explicitly warns that lawyers must ensure client data isn't disclosed to unauthorized parties. Cloud AI services that may use your data to train their models are a serious risk.

The solution isn't to avoid AI — it's to run it locally. Open source tools like **Ollama** (for local LLM inference) and **Tesseract** (for OCR) let you process legal documents with AI that never touches the internet. Everything stays on your server, under your control.

This guide walks through two practical automations for small law firms: **automated document review** and **streamlined client intake**.

## The Toolkit: Open Source Legal AI Stack

Before diving into workflows, here's what you need:

| Tool | What It Does | Why It Matters for Law |
|------|-------------|----------------------|
| **Ollama** | Runs LLMs locally on your server | No client data leaves your network |
| **Tesseract OCR** | Extracts text from scanned PDFs/images | Handles the 40% of legal docs that are scans |
| **n8n** | Workflow automation engine | Connects everything without custom code |
| **Directus** | Headless CMS / database | Stores structured matter and client data |
| **Cal.com** | Open-source scheduling | Automates consultation booking |

All five are open source, self-hostable, and free for small deployments. Combined, they form a legal automation stack that costs less than a single monthly SaaS subscription.

## Automation 1: Intelligent Document Review

### The Problem

A personal injury firm receives 50–200 pages of medical records, police reports, and insurance correspondence per new case. An associate reads everything, identifies key facts (dates of treatment, diagnoses, policy limits, statement dates), and creates a case summary memo. This takes 3–6 hours per case.

### The Solution

A workflow that:
1. Watches a designated folder (or email attachment inbox) for new documents
2. Runs OCR on any scanned PDFs
3. Sends the extracted text to a local LLM with a structured extraction prompt
4. Files the structured data into a matter management database
5. Generates a summary memo automatically

### Step 1: OCR with Tesseract

Many legal documents arrive as scanned PDFs — image files, not text-searchable documents. Tesseract handles the conversion:

```bash
# Install Tesseract
sudo apt install tesseract-ocr tesseract-ocr-eng

# Convert a scanned PDF to text
# First, extract images from the PDF with pdftoppm
pdftoppm -png input.pdf page
# Then OCR each page
for f in page-*.png; do
  tesseract "$f" "${f%.png}" --psm 1
done
# Combine all pages
cat page-*.txt > extracted_text.txt
```

For a n8n workflow, you'd use the **Execute Command** node to run this script, or wrap it in a simple Python service.

### Step 2: Structured Extraction with a Local LLM

Once you have clean text, send it to Ollama for extraction. Here's a prompt designed for medical records review:

```python
import requests

text = open("extracted_text.txt").read()[:8000]  # Truncate for context limits

prompt = f"""You are a legal document analyst. Extract key information from the 
following document and respond in valid JSON only.

Document text:
{text}

Extract these fields:
- document_type: one of [medical_record, police_report, insurance_letter, 
  correspondence, contract, court_filing, other]
- date_of_document: the document's date in YYYY-MM-DD format, or null
- parties_mentioned: list of names found
- key_facts: list of important factual statements (max 10)
- dates_of_treatment: list of treatment dates, or null
- diagnoses: list of diagnoses mentioned, or null
- policy_limits: dollar amount if mentioned, or null
- potential_evidence: list of items that could be evidentiary, or null

Respond with valid JSON only — no markdown, no explanation."""

response = requests.post(
    "http://localhost:11434/api/generate",
    json={
        "model": "qwen2.5:7b",
        "prompt": prompt,
        "stream": False,
        "format": "json"
    }
)

result = response.json()["response"]
print(result)
```

> **Why Qwen 2.5 7B?** For legal document analysis, accuracy matters more than speed. Qwen 2.5's 7B parameter model handles structured extraction well and fits in 8 GB of RAM. If you have 16 GB+, consider a 14B model for even better accuracy on complex documents.

### Step 3: File in Matter Database

The extracted JSON goes directly into Directus (or any database). Here's the n8n HTTP Request node configuration:

```
POST http://localhost:8055/items/documents
Content-Type: application/json
Authorization: Bearer YOUR_DIRECTUS_TOKEN

Body:
{
  "matter_id": "{{ $json.matter_id }}",
  "document_type": "{{ $json.document_type }}",
  "date_of_document": "{{ $json.date_of_document }}",
  "key_facts": "{{ $json.key_facts }}",
  "parties_mentioned": "{{ $json.parties_mentioned }}",
  "raw_extraction": "{{ JSON.stringify($json) }}",
  "source_file": "{{ $json.filename }}",
  "processing_date": "{{ $now }}"
}
```

### Step 4: Generate Summary Memo

Add another LLM call that synthesizes all extracted documents for a matter into a single case summary:

```python
summary_prompt = f"""You are a legal assistant preparing a case summary memo.
Given the following extracted document data, write a professional summary 
memo suitable for an attorney's case file.

Documents (JSON):
{json.dumps(all_documents, indent=2)}

Write the memo in standard legal format with:
1. RE: line with matter reference
2. FACTUAL SUMMARY section
3. KEY DATES timeline
4. OPEN ISSUES section
5. RECOMMENDED NEXT STEPS

Keep it under 500 words. Be factual — do not speculate."""
```

### Time Saved

A workflow like this turns a 4-hour document review into a 15-minute review of the AI-generated summary. The attorney still reviews everything — but they're reviewing a structured summary, not reading 200 pages cold. That's a **90%+ time reduction** on the most tedious part of case preparation.

## Automation 2: Client Intake

### The Problem

Potential clients call or email asking for a consultation. A receptionist or paralegal manually:
1. Collects basic information (name, contact, matter type, brief description)
2. Checks for conflicts of interest
3. Schedules a consultation
4. Creates a new matter file
5. Sends a confirmation email with intake forms

This takes 20–30 minutes per inquiry and is entirely manual.

### The Solution

An n8n workflow that:
1. Receives intake submissions from a web form (or email)
2. Uses a local LLM to classify the matter type and assess urgency
3. Runs a conflict check against the existing client database
4. Offers scheduling via Cal.com
5. Creates a preliminary matter record in Directus
6. Sends an automated confirmation with intake instructions

### The Workflow in n8n

```
Web Form Submission
        │
        ▼
┌─────────────────┐
│  LLM: Classify  │──▶ matter_type, urgency, conflict_keywords
│  & Assess       │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│  Directus:      │──▶ conflict_found: yes/no
│  Conflict Check │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│  Condition:     │──▶ (no conflict) ──▶ Cal.com scheduling link
│  Conflict?      │──▶ (conflict)   ──▶ Decline email
└─────────────────┘
        │
        ▼
┌─────────────────┐
│  Directus:      │──▶ New matter record created
│  Create Matter  │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│  Email: Send    │──▶ Confirmation + intake forms
│  Confirmation   │
└─────────────────┘
```

### The Classification Prompt

```python
intake_prompt = f"""You are a legal intake assistant. Analyze the following 
potential client inquiry and extract structured information.

Inquiry:
Name: {name}
Email: {email}
Matter description: {description}

Respond in JSON with:
- matter_type: one of [personal_injury, family_law, criminal_defense, 
  estate_planning, business_law, employment, real_estate, other]
- urgency: one of [immediate, this_week, this_month, routine]
- brief_summary: one-sentence summary of the legal issue
- conflict_keywords: list of names and entities to check for conflicts
- estimated_consultation_length: 30 or 60 (minutes)

Valid JSON only."""
```

### Conflict Check

The conflict check is a simple database query — no AI needed:

```sql
SELECT client_name, matter_name, status 
FROM clients 
WHERE client_name ILIKE ANY(%s) 
   OR matter_name ILIKE ANY(%s)
```

If results come back, the workflow routes to a "potential conflict — manual review needed" path instead of auto-scheduling. This is critical: **the AI assists with intake, but conflict checking must be deterministic, not probabilistic.** Never let an LLM make the final conflict determination — use it only to extract search terms.

## Privacy and Compliance Considerations

For law firms, privacy isn't just best practice — it's an ethical obligation. Here's how the open source approach addresses key concerns:

### Data Residency

With Ollama running on your server, document text never leaves your network. Compare this to cloud AI APIs, where your data transits to a third party's servers, may be stored for varying periods, and may be used for model training (depending on the provider's terms).

### Access Control

Self-hosted tools give you granular access control:
- **n8n** supports per-user permissions and SSO
- **Directus** has role-based access control down to the field level
- **Ollama** can be firewalled to only accept local connections

### Audit Trail

Every step in an n8n workflow is logged with timestamps, input/output data, and execution status. You can demonstrate exactly what the AI saw, what it produced, and when — which matters if you ever need to defend your process.

### ABA Compliance Checklist

| Requirement | How Open Source Tools Address It |
|---|---|
| Duty of confidentiality (Rule 1.6) | Data stays on-premises; no third-party API calls |
| Competence in AI use (Rule 1.1) | Attorney reviews all AI output; AI assists, doesn't decide |
| Supervision of AI tools (Rule 5.3) | Workflow logs provide full audit trail of AI actions |
| Communication about AI use (Rule 1.4) | Intake forms can disclose AI-assisted document processing |
| Security of client data (Rule 1.6(c)) | Self-hosted, encrypted at rest, access-controlled |

> **Important:** This is general guidance, not legal advice. Consult your state bar's specific AI guidance and your firm's ethics counsel before implementing any AI workflow.

## Hardware Requirements

For a small firm (1–5 attorneys) processing moderate document volumes:

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 16 GB | 32 GB |
| Storage | 100 GB SSD | 500 GB NVMe |
| GPU | None (CPU inference works) | 1× RTX 3060 (12 GB VRAM) for faster LLM |
| OS | Ubuntu 22.04 LTS | Same |

Without a GPU, a 7B model processes about 5–15 tokens/second — fast enough for document-by-document processing. With a GPU, you'll get 50–100+ tokens/second, which matters if you're batch-processing discovery files.

Estimated cost: **$500–1,500** for a dedicated server, or **$20–40/month** for a cloud VPS with similar specs. Either way, it's less than a single month of most legal AI SaaS subscriptions.

## Realistic Expectations

AI won't replace your paralegals. What it does is eliminate the lowest-value, most time-consuming parts of their work — so they can focus on tasks that actually require human judgment:

- ✅ **AI handles well:** Document categorization, fact extraction, date compilation, summary generation, intake classification
- ⚠️ **AI assists but human decides:** Conflict checking (AI extracts names, human verifies), legal research pointers, draft memoranda
- ❌ **AI should not do:** Legal advice, strategy decisions, client communication, ethical determinations, final document review

The goal isn't to remove humans from the process. It's to ensure that the humans in your firm spend their time on work that actually requires a law degree — not reading through 200 pages of medical records to find treatment dates.

## Getting Started

If you're a solo practitioner or small firm interested in exploring AI automation, start small:

1. **Install Ollama** on a spare computer and test it with a few sample documents
2. **Set up n8n** in Docker and build a simple document-categorization workflow
3. **Run it in parallel** with your manual process for 2–3 weeks to build confidence
4. **Gradually expand** to intake, conflict checking, and summary generation

The technology is accessible enough that you can prototype in an afternoon. The harder part — and the part worth investing time in — is designing workflows that genuinely improve your practice without compromising client confidentiality or professional judgment.

If you'd like help designing an AI automation system tailored to your firm's practice areas and document workflows, [reach out through our contact form](/). We specialize in open-source, self-hosted AI for professional services — keeping your client data on your terms, not a vendor's.