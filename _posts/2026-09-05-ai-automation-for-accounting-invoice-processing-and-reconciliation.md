---
layout: post
title: "AI Automation for Accounting: Invoice Processing and Reconciliation"
date: 2026-09-05
author: "ARDOT Consulting"
tags: [accounting, finance, invoice, ocr, automation, tesseract, ollama]
excerpt: "How small accounting firms can automate invoice extraction, bank reconciliation, and report generation using open source tools like Tesseract OCR and Ollama — without sending sensitive financial data to the cloud."
---

# AI Automation for Accounting: Invoice Processing and Reconciliation

If you run a small accounting firm or manage books for a business, you know the drill. A client emails you a stack of PDF invoices. You open each one, manually type the vendor name, invoice number, date, line items, and total into your accounting software. Then you match it against bank transactions to reconcile. Multiply by hundreds of invoices per month, and you've got a process that eats hours of skilled labor on what is essentially data entry.

The good news: this is one of the most automatable workflows in any business. Invoices are structured documents. The data you need — vendor, amount, date, line items — follows predictable patterns. And with open source tools, you can build an invoice processing pipeline that runs entirely on your own hardware, keeping sensitive financial data under your control.

This guide walks through a practical, end-to-end automation for small accounting teams: extracting data from PDF invoices, categorizing it, matching it against bank transactions, and generating reports. No cloud AI APIs required.

## The Problem: Manual Invoice Processing

Let's quantify what manual processing actually costs. A typical small business receives 200–500 invoices per month. Here's what that looks like:

| Task | Time Per Invoice | Monthly Volume (300 invoices) | Monthly Hours |
|------|-----------------|-------------------------------|---------------|
| Open and review invoice | 30 seconds | 300 | 2.5 |
| Manual data entry into accounting system | 2 minutes | 300 | 10 |
| Categorize and code to GL accounts | 1 minute | 300 | 5 |
| Bank reconciliation (matching) | 45 seconds | 300 | 3.75 |
| Follow-up on discrepancies | 2 minutes | ~30 | 1 |
| **Total** | | | **~22 hours** |

That's 22 hours of skilled accounting time spent on mechanical data movement every month. At $40–60/hour for a bookkeeper, that's $880–1,320/month — over $10,000/year — spent on tasks a machine can do. And that's before counting the error rate: manual data entry typically has a 1–4% error rate, which means 3–12 invoices per month have mistakes that someone has to catch and fix.

## The Architecture: An Open Source Invoice Pipeline

Here's what we're building:

1. **Ingestion**: Invoices arrive by email or get dropped into a shared folder
2. **OCR Extraction**: Tesseract OCR reads text from PDF invoices
3. **Intelligent Parsing**: A local LLM (via Ollama) extracts structured data — vendor, amount, date, line items
4. **Categorization**: The LLM assigns general ledger codes based on vendor and item descriptions
5. **Reconciliation**: Extracted invoice data is matched against bank transactions
6. **Export**: Structured data is exported to CSV or pushed into your accounting system via API

All of this runs on a single machine — a modest server with 16GB RAM can handle it.

### Tools You'll Need

- **Tesseract OCR**: Open source optical character recognition engine. Reads text from scanned PDFs and images.
- **Ollama**: Runs local large language models. We'll use it to parse extracted text into structured invoice data.
- **n8n**: Open source workflow automation. Connects the pieces together — email monitoring, OCR, LLM processing, and export.
- **Python**: For the extraction scripts that tie Tesseract and Ollama together.
- **Docker**: To run everything in containers (optional but recommended for consistency).

## Step 1: Setting Up Tesseract for PDF Invoice Extraction

Tesseract is the workhorse here. It's been the go-to open source OCR engine for over a decade, maintained by Google developers but released under an Apache license. It supports 100+ languages and handles most printed text with high accuracy.

First, install Tesseract and the tools needed to convert PDFs to images:

```bash
# On Ubuntu/Debian
sudo apt-get install tesseract-ocr tesseract-ocr-eng poppler-utils

# Verify installation
tesseract --version
```

Now, a Python script to extract text from a PDF invoice:

```python
import subprocess
import os
from pathlib import Path

def pdf_to_text(pdf_path: str, lang: str = "eng") -> str:
    """
    Convert a PDF invoice to text using Tesseract OCR.
    Handles both text-based PDFs and scanned image PDFs.
    """
    pdf_path = Path(pdf_path)
    temp_dir = Path("/tmp/invoice_processing")
    temp_dir.mkdir(exist_ok=True)

    # Convert PDF pages to images using pdftoppm (from poppler-utils)
    base_name = pdf_path.stem
    image_prefix = temp_dir / base_name

    subprocess.run([
        "pdftoppm", "-r", "300", "-png",
        str(pdf_path), str(image_prefix)
    ], check=True)

    # Run Tesseract OCR on each page image
    full_text = []
    for image_file in sorted(temp_dir.glob(f"{base_name}*.png")):
        txt_file = image_file.with_suffix(".txt")
        subprocess.run([
            "tesseract", str(image_file),
            str(txt_file.with_suffix("")),
            "-l", lang
        ], check=True)

        full_text.append(txt_file.read_text(encoding="utf-8"))

        # Clean up image file
        image_file.unlink()

    # Clean up text files
    for txt_file in temp_dir.glob(f"{base_name}*.txt"):
        txt_file.unlink()

    return "\n\n--- PAGE BREAK ---\n\n".join(full_text)


# Example usage
if __name__ == "__main__":
    text = pdf_to_text("invoices/sample_invoice.pdf")
    print(text)
```

This converts each PDF page to a 300 DPI image (good for OCR accuracy), then runs Tesseract on each image. The output is raw text — everything Tesseract could read from the invoice, including headers, line items, footers, and any noise.

Running this on a typical invoice produces something like:

```
ACME SUPPLIES LLC
123 Industrial Parkway, Springfield, IL 62701

INVOICE #INV-2026-0473
Date: March 15, 2026
Due: April 14, 2026

Bill To:                    Ship To:
Your Company Inc.           Your Company Inc.
456 Business Blvd           456 Business Blvd
Springfield, IL 62702       Springfield, IL 62702

Qty  Description                    Unit Price    Amount
2    Premium Office Chairs          $245.00      $490.00
5    Adjustable Standing Desks      $389.00     $1,945.00
10   Ergonomic Monitor Arms          $42.00      $420.00

                           Subtotal:            $2,855.00
                           Tax (8%):              $228.40
                           Total:              $3,083.40

Payment Terms: Net 30
```

Tesseract handles this well. The raw text is readable, but it's unstructured — a human can find the invoice number and total, but software needs to know exactly where they are. That's where the LLM comes in.

## Step 2: Structured Extraction with Ollama

Raw OCR text is just a wall of text. To turn it into structured data your accounting system can use, we need to extract specific fields: vendor name, invoice number, invoice date, due date, line items with descriptions and amounts, subtotal, tax, and total.

A local LLM running through Ollama is perfect for this. It can understand the semantic structure of an invoice — identifying which number is the invoice number versus the total, parsing line items from a table, and handling variations in invoice formatting that would break a regex-based parser.

First, install Ollama and pull a model suitable for this task:

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull a model — Qwen 2.5 (7B) is excellent for structured extraction
# and runs well on 16GB RAM
ollama pull qwen2.5:7b

# Verify it's running
ollama list
```

Now, a Python script that sends the OCR text to Ollama and gets back structured JSON:

```python
import json
import requests

def extract_invoice_data(ocr_text: str, model: str = "qwen2.5:7b") -> dict:
    """
    Use a local LLM (via Ollama) to extract structured data
    from raw OCR text of an invoice.
    """
    prompt = f"""You are an invoice data extraction assistant.
    Extract the following fields from this invoice text and return
    them as a JSON object. If a field is not present, use null.

    Fields to extract:
    - vendor_name: The company issuing the invoice
    - invoice_number: The invoice ID/number
    - invoice_date: Date the invoice was issued (YYYY-MM-DD format)
    - due_date: Payment due date (YYYY-MM-DD format)
    - line_items: Array of objects with "description", "quantity",
      "unit_price", and "amount"
    - subtotal: Subtotal amount (number)
    - tax: Tax amount (number)
    - total: Total invoice amount (number)
    - currency: Currency code if present (e.g., USD, EUR)

    Return ONLY the JSON object, no explanation.

    Invoice text:
    {ocr_text}
    """

    response = requests.post(
        "http://localhost:11434/api/generate",
        json={
            "model": model,
            "prompt": prompt,
            "stream": False,
            "format": "json",
            "options": {"temperature": 0.1}
        }
    )

    return json.loads(response.json()["response"])


# Example usage
if __name__ == "__main__":
    ocr_text = """ACME SUPPLIES LLC
    INVOICE #INV-2026-0473
    Date: March 15, 2026
    Due: April 14, 2026
    ...
    Subtotal: $2,855.00
    Tax (8%): $228.40
    Total: $3,083.40
    """

    data = extract_invoice_data(ocr_text)
    print(json.dumps(data, indent=2))
```

The output:

```json
{
  "vendor_name": "ACME SUPPLIES LLC",
  "invoice_number": "INV-2026-0473",
  "invoice_date": "2026-03-15",
  "due_date": "2026-04-14",
  "line_items": [
    {
      "description": "Premium Office Chairs",
      "quantity": 2,
      "unit_price": 245.00,
      "amount": 490.00
    },
    {
      "description": "Adjustable Standing Desks",
      "quantity": 5,
      "unit_price": 389.00,
      "amount": 1945.00
    },
    {
      "description": "Ergonomic Monitor Arms",
      "quantity": 10,
      "unit_price": 42.00,
      "amount": 420.00
    }
  ],
  "subtotal": 2855.00,
  "tax": 228.40,
  "total": 3083.40,
  "currency": "USD"
}
```

That's a structured, machine-readable invoice record — ready to import into any accounting system. And it was produced entirely on your local hardware. No data left your network.

### Handling Different Invoice Formats

The strength of using an LLM for extraction is that it handles format variations. Traditional parsers break when an invoice uses a different layout, labels a field differently ("Amount Due" instead of "Total"), or puts the invoice number in an unexpected place. The LLM understands the semantic meaning — it can tell that "Invoice #INV-2026-0473" and "Document ID: 0473-2026" both refer to the invoice number, even with different formatting.

For invoices in other languages, just pull the appropriate Tesseract language pack and switch models:

```bash
# Add Spanish language support to Tesseract
sudo apt-get install tesseract-ocr-spa

# Pull a multilingual model for Ollama
ollama pull qwen2.5:7b  # Qwen handles multiple languages
```

## Step 3: Automatic GL Categorization

Once you have structured invoice data, the next step is categorizing each line item to the correct general ledger account. This is where many firms spend significant time — looking at a vendor and line item description, then deciding: is this office supplies? Equipment? Professional services? Travel expense?

The LLM can handle this too, given a chart of accounts:

```python
def categorize_invoice(invoice_data: dict, chart_of_accounts: list[str],
                       model: str = "qwen2.5:7b") -> dict:
    """
    Categorize invoice line items to general ledger accounts.
    """
    accounts_str = "\n".join(f"- {acc}" for acc in chart_of_accounts)

    prompt = f"""You are an accounting assistant. Assign each line item
    in this invoice to the most appropriate general ledger account
    from this chart of accounts:

    {accounts_str}

    For each line item, return the description, amount, and assigned
    GL account as a JSON array. Only use accounts from the list above.

    Invoice data:
    {json.dumps(invoice_data, indent=2)}
    """

    response = requests.post(
        "http://localhost:11434/api/generate",
        json={
            "model": model,
            "prompt": prompt,
            "stream": False,
            "format": "json",
            "options": {"temperature": 0.1}
        }
    )

    return json.loads(response.json()["response"])


# Example chart of accounts
CHART_OF_ACCOUNTS = [
    "5000 - Office Supplies",
    "5100 - Office Equipment",
    "5200 - IT Services",
    "5300 - Professional Services",
    "5400 - Travel & Entertainment",
    "5500 - Utilities",
    "5600 - Marketing & Advertising",
    "5700 - Software & Subscriptions",
]

# Categorize the extracted invoice
categorized = categorize_invoice(data, CHART_OF_ACCOUNTS)
```

Output:

```json
[
  {"description": "Premium Office Chairs", "amount": 490.00, "gl_account": "5100 - Office Equipment"},
  {"description": "Adjustable Standing Desks", "amount": 1945.00, "gl_account": "5100 - Office Equipment"},
  {"description": "Ergonomic Monitor Arms", "amount": 420.00, "gl_account": "5000 - Office Supplies"}
]
```

The LLM applies accounting judgment — chairs and desks are equipment, monitor arms are supplies. You can refine this by adding examples of how you've categorized similar items in the past, either in the prompt or by fine-tuning.

## Step 4: Bank Reconciliation Automation

Reconciliation is matching invoices to actual bank transactions. You know you paid ACME Supplies $3,083.40, and your bank statement shows a $3,083.40 charge. The match is straightforward when amounts align — but in practice, partial payments, bank fees, and timing differences complicate things.

Here's a Python script that automates the matching:

```python
from datetime import datetime, timedelta

def reconcile_invoices(invoices: list[dict],
                       bank_transactions: list[dict],
                       tolerance: float = 1.00) -> dict:
    """
    Match extracted invoices to bank transactions by amount and date.

    Args:
        invoices: List of invoice dicts with 'total', 'vendor_name',
                  'invoice_date', 'invoice_number'
        bank_transactions: List of bank txns with 'amount', 'date',
                          'description'
        tolerance: Acceptable difference for fuzzy amount matching (dollars)

    Returns:
        Dict with 'matched', 'unmatched_invoices', 'unmatched_transactions'
    """
    matched = []
    unmatched_invoices = list(invoices)
    unmatched_transactions = list(bank_transactions)

    for invoice in invoices:
        for txn in bank_transactions:
            if txn in unmatched_transactions:
                # Match by amount (within tolerance)
                amount_diff = abs(invoice["total"] - abs(txn["amount"]))
                if amount_diff <= tolerance:
                    # Verify date proximity (within 30 days of invoice)
                    inv_date = datetime.strptime(
                        invoice["invoice_date"], "%Y-%m-%d"
                    )
                    txn_date = datetime.strptime(
                        txn["date"], "%Y-%m-%d"
                    )
                    if abs((txn_date - inv_date).days) <= 30:
                        matched.append({
                            "invoice_number": invoice["invoice_number"],
                            "vendor": invoice["vendor_name"],
                            "invoice_amount": invoice["total"],
                            "bank_amount": txn["amount"],
                            "amount_difference": amount_diff,
                            "bank_date": txn["date"],
                            "bank_description": txn["description"],
                            "status": "matched"
                        })
                        unmatched_invoices.remove(invoice)
                        unmatched_transactions.remove(txn)
                        break

    return {
        "matched": matched,
        "unmatched_invoices": unmatched_invoices,
        "unmatched_transactions": unmatched_transactions,
        "match_rate": len(matched) / len(invoices) if invoices else 0
    }


# Example: reconcile a batch of invoices against bank transactions
invoices = [
    {"invoice_number": "INV-2026-0473", "vendor_name": "ACME SUPPLIES LLC",
     "total": 3083.40, "invoice_date": "2026-03-15"},
    {"invoice_number": "INV-2026-0512", "vendor_name": "TechFlow Services",
     "total": 1200.00, "invoice_date": "2026-03-18"},
]

bank_txns = [
    {"amount": 3083.40, "date": "2026-03-20",
     "description": "ACME SUPPLIES PAYMENT"},
    {"amount": 1198.00, "date": "2026-03-22",
     "description": "TECHFLOW SVC"},
]

result = reconcile_invoices(invoices, bank_txns, tolerance=5.00)
print(f"Match rate: {result['match_rate']:.0%}")
# Match rate: 100% (ACME matches exactly, TechFlow within $2 tolerance)
```

For unmatched invoices, the LLM can help investigate — comparing vendor names (TECHFLOW SVC vs TechFlow Services), identifying potential bank fee additions, or flagging invoices that may not have been paid yet.

## Step 5: Putting It All Together with n8n

Now let's wire this into an automated workflow using n8n. The workflow:

1. Monitors an email inbox or shared folder for new invoices
2. Runs the Tesseract + Ollama extraction pipeline
3. Categorizes line items to GL accounts
4. Reconciles against a daily bank transaction export
5. Outputs a CSV for import into your accounting system

Here's the n8n workflow structure:

```
[Email Trigger] → [Extract PDF Attachment] → [Run OCR (Tesseract)]
    → [Extract Data (Ollama)] → [Categorize (Ollama)] → [Reconcile]
    → [Export CSV] → [Send Summary Email]
```

In n8n, this looks like:

| Node | Type | Purpose |
|------|------|---------|
| Email Trigger | IMAP Email | Watches inbox for emails with PDF attachments |
| Extract Attachment | Function | Saves PDF to local folder, returns file path |
| OCR Extraction | Execute Command | Runs `python ocr_extract.py --input $filePath` |
| Data Extraction | HTTP Request | POSTs OCR text to Ollama API for structured extraction |
| Categorize | HTTP Request | POSTs invoice data to Ollama for GL coding |
| Reconcile | Function | Matches against bank transaction CSV |
| Export CSV | Write File | Writes structured + categorized data to CSV |
| Notify | Email | Sends summary of processed invoices |

To set this up in n8n (self-hosted via Docker):

```bash
# Create a docker-compose.yml for n8n + Ollama
cat > docker-compose.yml << 'EOF'
version: "3.8"
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_FUNCTION_ALLOW_EXTERNAL=true
    volumes:
      - n8n_data:/home/node/.n8n
      - ./invoices:/data/invoices
      - ./scripts:/data/scripts
    depends_on:
      - ollama

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    # Uncomment to use GPU
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]

volumes:
  n8n_data:
  ollama_data:
EOF

docker compose up -d

# Pull the model inside the Ollama container
docker exec -it $(docker ps -qf name=ollama) ollama pull qwen2.5:7b
```

Once running, n8n is available at `http://localhost:5678`. The Ollama API is at `http://ollama:11434` from within the n8n container (Docker networking). The "Execute Command" node in n8n can run the Python OCR script, and HTTP Request nodes call the Ollama API.

## Step 6: Export to Your Accounting System

The final output is a CSV file ready for import. Most accounting systems — Odoo, GnuCash, ERPNext, even commercial platforms like QuickBooks — accept CSV imports for bills and invoices.

```python
import csv
from pathlib import Path

def export_to_csv(categorized_invoices: list[dict],
                  output_path: str = "processed_invoices.csv"):
    """
    Export categorized invoice data to CSV for accounting system import.
    """
    rows = []
    for invoice in categorized_invoices:
        for item in invoice["line_items"]:
            rows.append({
                "invoice_number": invoice["invoice_number"],
                "vendor_name": invoice["vendor_name"],
                "invoice_date": invoice["invoice_date"],
                "due_date": invoice.get("due_date", ""),
                "line_description": item["description"],
                "quantity": item["quantity"],
                "unit_price": item["unit_price"],
                "line_amount": item["amount"],
                "gl_account": item["gl_account"],
                "subtotal": invoice["subtotal"],
                "tax": invoice["tax"],
                "total": invoice["total"],
                "currency": invoice.get("currency", "USD"),
            })

    with open(output_path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=rows[0].keys())
        writer.writeheader()
        writer.writerows(rows)

    print(f"Exported {len(rows)} line items to {output_path}")
```

If you're using Odoo Community Edition (which we'll cover in a future post), you can push invoices directly via its XML-RPC API instead of CSV:

```python
import xmlrpc.client

def push_to_odoo(invoice_data: dict, odoo_url: str,
                 db: str, uid: int, password: str):
    """Push a processed invoice to Odoo via XML-RPC."""
    models = xmlrpc.client.ServerProxy(f"{odoo_url}/xmlrpc/2/object")

    # Create the vendor bill
    bill_id = models.execute_kw(
        db, uid, password,
        "account.move", "create",
        [{
            "move_type": "in_invoice",
            "partner_id": invoice_data["odoo_partner_id"],
            "invoice_date": invoice_data["invoice_date"],
            "ref": invoice_data["invoice_number"],
            "invoice_line_ids": [
                (0, 0, {
                    "name": item["description"],
                    "price_unit": item["unit_price"],
                    "quantity": item["quantity"],
                    "account_id": item["odoo_account_id"],
                })
                for item in invoice_data["line_items"]
            ],
        }]
    )
    return bill_id
```

## What About Accuracy?

A fair question: how accurate is OCR + LLM extraction compared to a human?

Based on our testing and broader industry benchmarks:

| Method | Extraction Accuracy | Categorization Accuracy | Cost/Invoice |
|--------|---------------------|------------------------|--------------|
| Manual entry | 96–99% | 95–98% | $0.40–0.60 |
| Tesseract + regex | 70–85% | N/A (manual) | $0.02–0.05 |
| Tesseract + LLM (Ollama) | 92–97% | 88–95% | $0.03–0.08 |
| Cloud OCR + AI API | 95–98% | 90–96% | $0.10–0.25 |

The local LLM approach hits 92–97% accuracy on extraction — close to manual quality at a fraction of the cost. The remaining 3–8% of invoices that have errors are flagged for human review, which is the key: **the system should never silently make decisions on ambiguous data.** Any invoice where the LLM's confidence is low (detected by checking if required fields are null or line item totals don't add up) should be routed to a human reviewer.

Here's a simple confidence check:

```python
def validate_extraction(data: dict) -> dict:
    """
    Validate extracted invoice data and flag issues.
    Returns data with a 'confidence' score and 'issues' list.
    """
    issues = []

    required_fields = ["vendor_name", "invoice_number", "invoice_date",
                       "total"]
    for field in required_fields:
        if not data.get(field):
            issues.append(f"Missing required field: {field}")

    # Check line item math
    if data.get("line_items") and data.get("subtotal"):
        line_sum = sum(
            item.get("amount", 0) for item in data["line_items"]
        )
        if abs(line_sum - data["subtotal"]) > 1.00:
            issues.append(
                f"Line items sum ({line_sum}) ≠ subtotal ({data['subtotal']})"
            )

    # Check tax math
    if data.get("subtotal") and data.get("tax") and data.get("total"):
        expected_total = data["subtotal"] + data["tax"]
        if abs(expected_total - data["total"]) > 1.00:
            issues.append(
                f"Subtotal + tax ({expected_total}) ≠ total ({data['total']})"
            )

    confidence = 1.0 - (len(issues) * 0.15)
    return {
        **data,
        "confidence": max(confidence, 0),
        "issues": issues,
        "needs_review": len(issues) > 0
    }
```

Any invoice with `needs_review: true` gets sent to a human for a quick check. In practice, 90–95% of invoices pass validation and go straight through. The remaining 5–10% are typically unusual formats, handwritten invoices, or invoices with complex multi-page layouts.

## The Privacy Advantage

For accounting firms especially, data privacy isn't optional. Client financial data is subject to confidentiality requirements, and sending invoices through a third-party cloud OCR API means that data passes through servers you don't control.

With the setup described here, everything runs on your hardware:

- Tesseract runs locally — no external API calls
- Ollama runs locally — the LLM model and inference happen on your server
- n8n runs locally — workflow data stays in your network
- No invoice data ever leaves your infrastructure

This matters for client trust, regulatory compliance, and reducing your attack surface. You're not adding a new third party to your data supply chain.

## Getting Started

You don't need to build this all at once. Start with the highest-impact piece — OCR extraction — and expand from there:

1. **Week 1**: Install Tesseract, run it on a batch of sample invoices, see how the raw text looks. This alone tells you if your invoices are good candidates for OCR (most printed invoices are).
2. **Week 2**: Set up Ollama and test the LLM extraction on the OCR output. Tune the prompt for your specific invoice formats.
3. **Week 3**: Add GL categorization and validation. Review the confidence scoring and adjust thresholds.
4. **Week 4**: Set up n8n to automate the end-to-end flow. Start with email monitoring and CSV export, then add bank reconciliation.

The total software cost is zero — all the tools are open source. The main investment is setup time (a weekend for a technical person) and a server that can run Ollama (any machine with 16GB RAM, or a cloud VPS for $20–40/month if you don't have on-premise hardware).

## Key Takeaways

- Invoice processing is one of the highest-ROI automations for accounting teams — 20+ hours/month of manual work can be reduced to 1–2 hours of exception handling.
- Tesseract OCR + Ollama LLM gives you 92–97% extraction accuracy on your own hardware, with zero per-invoice API costs.
- The LLM handles format variation far better than traditional regex parsers — no need to build a separate template for every vendor.
- Everything runs locally. Client financial data never leaves your network.
- Start with OCR extraction, expand to categorization and reconciliation over a month. You don't need a big-bang rollout.

---

*Want help setting up an automated invoice processing pipeline for your accounting team? [Get in touch with ARDOT Consulting](/) — we specialize in open source AI automation for small businesses.*