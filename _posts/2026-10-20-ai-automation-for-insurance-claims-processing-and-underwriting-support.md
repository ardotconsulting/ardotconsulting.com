---
layout: post
title: "AI Automation for Insurance: Claims Processing and Underwriting Support"
date: 2026-10-20
author: "ARDOT Consulting"
tags: [insurance, claims, automation, ocr, ollama, underwriting, open-source]
excerpt: "Claims processing and underwriting are document-heavy, repetitive, and slow. Here's how small insurance teams can automate intake, extraction, triage, and first-pass underwriting using open source tools — without sending sensitive policyholder data to third-party AI APIs."
---

# AI Automation for Insurance: Claims Processing and Underwriting Support

If you work in insurance, you know the bottleneck. A customer files a claim. It lands in an inbox alongside 200 others that day. Someone has to open it, read the policy, check the damage photos, verify coverage, request more documents, and decide whether to approve, deny, or escalate. Multiply that across auto, home, commercial, and specialty lines, and you've got a claims department that spends most of its time on paperwork — not on making good decisions.

The same is true on the underwriting side. New business submissions arrive as PDFs with dozens of pages of loss runs, schedules of values, and applicant financials. An underwriter reads through them, pulls out the key numbers, checks them against guidelines, and writes a quote. It's skilled work buried under hours of manual extraction.

This is exactly the kind of work where open source AI automation shines. The documents are structured (or semi-structured), the rules are well-defined, and the volume is high. But insurance is also one of the most heavily regulated and data-sensitive industries there is. Sending policyholder data, medical records, and financial statements to a third-party AI API isn't just a compliance headache — in many jurisdictions, it may not even be legal without explicit disclosures and safeguards.

That's why this guide focuses on a fully self-hosted stack. No data leaves your infrastructure. You run the OCR, the language models, and the workflow engine on your own servers. Here's how to build it.

## Why Insurance Is a Prime Candidate for Automation

Let's look at what a typical claims process actually involves, and where the time goes:

| Task | Manual Process | Time Per Claim | Automation Potential |
|------|---------------|----------------|---------------------|
| **Intake** | Claims arrive by email, portal, phone | 5–10 min | High — automated ingestion |
| **Document review** | Open each PDF, verify completeness | 10–15 min | High — OCR + validation |
| **Policy lookup** | Search system for matching policy | 3–5 min | High — API/database lookup |
| **Coverage verification** | Read policy terms, check exclusions | 8–12 min | Medium — LLM-assisted |
| **Damage assessment** | Review photos, adjuster notes | 10–20 min | Medium — AI triage + human review |
| **Fraud screening** | Check databases, flag anomalies | 5–8 min | High — rule + ML screening |
| **Decision routing** | Approve, deny, or escalate to adjuster | 2–5 min | High — rules-based routing |
| **Total per claim** | | **43–75 min** | |

A small claims team handling 50 claims a day spends 35–60 person-hours on intake and initial processing alone. That's before any adjuster field work or complex decision-making. The goal of automation isn't to replace the adjuster — it's to clear the paperwork pile so the adjuster can focus on the claims that actually need human judgment.

## The Architecture: A Self-Hosted Claims Pipeline

Here's what we're building:

1. **Ingestion**: Claims documents arrive by email or get dropped into a monitored folder
2. **OCR Extraction**: Tesseract OCR reads text from scanned PDFs and photos
3. **Intelligent Parsing**: A local LLM (via Ollama) extracts structured data — claim number, date of loss, policy number, claimant info, damage description
4. **Policy Match**: Workflow engine looks up the matching policy in your system
5. **Coverage Check**: LLM compares claim details against policy terms and flags potential issues
6. **Fraud Screen**: Rules engine checks against known fraud indicators and databases
7. **Triage and Route**: Claim is auto-approved (low-value, clear coverage), denied (clear exclusion), or routed to an adjuster for review
8. **Notification**: All parties notified; adjuster gets a pre-built claim summary

The tools:

- **[n8n](https://n8n.io/)** — Open source workflow automation. Orchestrates the entire pipeline.
- **[Tesseract OCR](https://github.com/tesseract-ocr/tesseract)** — Open source optical character recognition. Reads text from scanned documents and photos.
- **[Ollama](https://ollama.com/)** — Runs large language models locally. Handles parsing, coverage analysis, and summarization.
- **[Paperless-ngx](https://docs.paperless-ngx.com/)** — Open source document management. Stores and indexes all claim documents.

No third-party APIs. No data sent to OpenAI, Anthropic, or anyone else. Everything runs on a server you control.

## Step 1: Document Intake and OCR

The first bottleneck is getting claim documents into a usable digital format. Claims arrive as email attachments, portal uploads, or faxes (yes, still). Here's how to automate the intake:

### Automated Ingestion with n8n

Set up an n8n workflow that monitors your claims inbox. Every incoming document — whether email attachment, portal upload, or fax — lands in a monitored folder and kicks off the processing pipeline automatically.

### OCR with Tesseract

Insurance documents are notoriously varied — scanned forms, phone photos of damage, PDFs with mixed text and images, handwritten adjuster notes. Tesseract handles most of this, though you'll want to pre-process images for best results:

```bash
# Install Tesseract
sudo apt install tesseract-ocr tesseract-ocr-eng

# Basic OCR on a scanned PDF (convert to images first)
pdftoppm -png -r 300 claim_form.pdf page
tesseract page-1.png claim_text -l eng

# For damage photos, add image preprocessing
convert photo.jpg -density 300 -depth 8 -threshold 60% photo_clean.png
tesseract photo_clean.png photo_text -l eng
```

For a production pipeline, you'd run this inside a Docker container triggered by n8n. Don't try to get perfect OCR on everything — get 90% accuracy and let the LLM clean up the rest. The LLM is far better at making sense of messy text than Tesseract is at producing perfect output.

## Step 2: Intelligent Parsing with Ollama

This is where the local LLM earns its keep. Tesseract gives you raw text — often messy, with OCR errors, jumbled columns, and missing sections. The LLM turns that into structured data.

### Setting Up Ollama

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model suitable for document understanding
ollama pull llama3.1:8b

# Start the API server (runs on port 11434)
ollama serve
```

For insurance document parsing, Llama 3.1 8B or Qwen 2.5 7B work well. They're small enough to run on a single server with 16GB RAM and fast enough for real-time processing. If you have a GPU, inference is near-instant.

### Extracting Structured Claim Data

Here's the prompt pattern that works. You feed the OCR text and ask the LLM to extract specific fields:

```python
import requests

ocr_text = """[paste Tesseract output here]"""

prompt = f"""You are an insurance claims processor. Extract the following fields
from the claim document text below. Return ONLY valid JSON, no commentary.

Fields to extract:
- claim_number
- policy_number
- claimant_name
- date_of_loss
- type_of_claim (auto/home/commercial/specialty)
- description_of_loss
- estimated_damage_amount (if mentioned)
- contact_phone
- contact_email

If a field is not present in the document, use null.

Document text:
{ocr_text}
"""

response = requests.post("http://localhost:11434/api/generate", json={
    "model": "llama3.1:8b",
    "prompt": prompt,
    "format": "json",
    "stream": False
})

claim_data = response.json()["response"]
print(claim_data)
```

The output is clean, structured JSON you can pass directly to your claims management system. This same pattern works for:

- **Loss runs** — extract 5-year loss history from broker submissions
- **ACORD forms** — extract all standard fields from industry forms
- **Police reports** — extract date, location, parties, narrative summary
- **Medical records** — extract diagnosis, treatment dates, provider info (with appropriate access controls)

## Step 3: Coverage Verification

This is where automation gets genuinely useful — and where you need to be careful. You're not asking the AI to make coverage decisions. You're asking it to do the first-pass comparison and flag issues for a human.

The workflow:

1. Pull the relevant policy document from your document management system
2. Send the claim details and policy terms to the LLM
3. Ask it to identify coverage limits, deductibles, exclusions, and conditions that apply
4. Flag anything ambiguous for adjuster review

Here's the prompt structure:

```python
coverage_prompt = f"""You are an insurance coverage analyst. Compare the claim
details against the policy terms below.

Claim details:
{claim_data}

Policy terms (relevant sections):
{policy_sections}

Provide:
1. Does the claim type appear to be covered? (yes/no/unclear)
2. What is the applicable deductible?
3. What is the coverage limit for this claim type?
4. Are there any exclusions that might apply?
5. What conditions must be met (e.g., proof of loss within 60 days)?
6. Flag any areas of ambiguity for human review.

Be conservative. If something is unclear, flag it — do not guess.
"""
```

The key phrase: **be conservative**. The LLM should surface information and flag potential issues, not make the call. A human adjuster reviews the summary and makes the coverage decision. What you've done is compress 20 minutes of reading into a 30-second scan of a structured summary.

## Step 4: Fraud Screening

Fraud detection in insurance runs on two layers: rules and pattern recognition. Automation handles both.

### Rules-Based Screening

Set up automated checks in your n8n workflow:

| Check | Logic | Action |
|-------|-------|--------|
| **Duplicate claim** | Compare claim number, date of loss, claimant against database | Flag if match found |
| **Early filing** | Date of loss vs. policy effective date | Flag if loss before policy start |
| **High-value claim** | Estimated damage > $50,000 | Route to senior adjuster |
| **Multiple claims** | Claimant has > 3 claims in past 24 months | Flag for review |
| **Suspicious timing** | Claim filed within 30 days of policy start | Flag for investigation |

These are deterministic rules — no AI needed. They catch the obvious stuff and run instantly.

### LLM-Assisted Anomaly Detection

For the subtler stuff, use the LLM to look for inconsistencies in the claim narrative:

```python
fraud_prompt = f"""Review this insurance claim for potential red flags.
Look for inconsistencies between the stated facts, unusual phrasing,
or details that don't align with the claim type.

Claim data:
{claim_data}

OCR text of supporting documents:
{ocr_text}

List any red flags found. If none, say "No red flags identified."
Do not make accusations — just note inconsistencies for review.
"""
```

The LLM won't catch sophisticated fraud, but it will catch things like a claimant who describes a "break-in" but lists items that don't match the reported point of entry, or a date of loss that contradicts the weather report for that location. These are the kind of cross-document inconsistencies a tired human reviewer might miss on claim number 47 of the day.

## Step 5: Triage and Routing

Not every claim needs the same level of attention. A $2,000 windshield repair with clear coverage and no red flags should be approved in minutes, not days. A $200,000 commercial property loss with ambiguous coverage needs a senior adjuster.

Here's a routing matrix you can implement in n8n:

| Claim Profile | Auto-Approve | Route To |
|---------------|-------------|----------|
| < $5,000, clear coverage, no flags | Yes — generate payment authorization | Auto-process |
| $5,000–$25,000, clear coverage, no flags | No — review summary | Junior adjuster |
| Any amount, coverage unclear | No | Senior adjuster |
| Any amount, fraud flags | No | SIU (fraud team) |
| > $50,000, any | No | Senior adjuster + supervisor |
| Property + injury | No | Complex claims team |

The auto-approve threshold is a business decision. Some insurers auto-approve up to $2,500. Others cap at $1,000. The point is that you're freeing your adjusters from the routine work so they can spend real time on the claims that need it.

## The Underwriting Side: Same Tools, Different Workflow

Everything above applies to underwriting too, just with different documents and decisions:

| Claims | Underwriting |
|--------|-------------|
| Claim form → extract data | Submission → extract data |
| Policy lookup → coverage check | Guidelines lookup → eligibility check |
| Fraud screen | Risk score |
| Route to adjuster | Route to underwriter (or auto-quote) |

A new business submission typically includes an ACORD application, loss runs for the past 5 years, and supporting schedules. The same OCR + LLM pipeline extracts the key fields: applicant name, operations, revenue, payroll, loss history. The LLM can summarize 5 years of loss runs into a clean table in seconds — a task that takes an underwriter 15–20 minutes per submission. For smaller, straightforward risks, the system can generate a quote recommendation that the underwriter reviews and approves. For complex risks, the underwriter gets a pre-built summary with all the key numbers extracted and organized.

## What This Costs to Run

Unlike a per-document SaaS pricing model (which can get expensive fast at insurance volumes), self-hosted automation has a fixed cost:

| Component | Hardware/Cost |
|-----------|---------------|
| **Server** | 1 VM with 8 vCPU, 16GB RAM, 100GB storage (~$80/month or existing hardware) |
| **Tesseract** | Free, open source |
| **Ollama + Llama 3.1 8B** | Free, runs on the server above |
| **n8n** | Free self-hosted, or $20/month for n8n Cloud |
| **Paperless-ngx** | Free, open source |
| **Total** | ~$80–100/month, flat |

Compare that to a commercial claims automation platform that charges $0.50–$2.00 per document processed. At 50 claims per day with an average of 4 documents each, that's $3,000–$12,000 per month. The self-hosted stack pays for itself in the first week.

## What Not to Automate

Do not automate final coverage decisions on complex claims, fraud determinations, denial letters with legal implications, or anything involving regulatory filings and consumer protection notices. The pattern: automation handles extraction, summarization, routing, and routine decisions. Humans handle judgment, accountability, and edge cases. If you're unsure whether to automate something, ask: "Would I be comfortable explaining to a regulator that a computer made this decision?" If the answer is no, keep a human in the loop.

## A Realistic Implementation Timeline

**Week 1**: Set up Ollama and Tesseract on a server. Test OCR + LLM extraction on 20 sample claim documents. Measure accuracy — this tells you whether your document quality is good enough for automation.

**Week 2**: Build the n8n ingestion workflow. Connect email/portal intake to the OCR + LLM pipeline. Start with document parsing only — no decisions yet.

**Week 3**: Add policy lookup and coverage verification. Have adjusters review the LLM summaries alongside their normal process. Compare. The goal is to validate that the summaries are accurate and useful.

**Week 4**: Add fraud screening and routing. Start with the rules-based checks (easy wins). Add LLM anomaly detection gradually. Begin auto-approving the smallest, clearest claims.

**Week 5–6**: Refine based on adjuster feedback. Tune the routing thresholds. Expand auto-approve limits as confidence builds.

By week 6, a small claims team should see a 40–60% reduction in time-to-first-contact on routine claims. The complex claims still take time — but your adjusters are working on them instead of doing data entry.

## Getting Started

You don't need a big budget or a dev team to start. You need a server, an afternoon, and a stack of sample documents to test against. The tools are free. The payoff is real. And because everything runs on your own infrastructure, your policyholder data stays exactly where it should — under your control.

If you want help designing a claims or underwriting automation pipeline for your team, [get in touch](/#contact). We specialize in self-hosted, open source solutions for regulated industries — and we'll help you build something that your compliance team will actually approve of.