---
layout: post
title: "Automating Procurement and Purchase Orders with Open Source Tools"
date: 2026-12-09
author: "ARDOT Consulting"
tags: [procurement, purchase-orders, automation, open-source, odoo, erpnext, n8n, ollama, workflows]
excerpt: "Manual procurement is where small businesses leak money — duplicated orders, missed approvals, maverick spending. Here's how to automate the full purchase-order cycle with open-source tools you can self-host, without a Procure-to-Pay SaaS subscription."
---

# Automating Procurement and Purchase Orders with Open Source Tools

If you've ever discovered that two different people ordered the same office supplies from two different vendors at two different prices — in the same week — you've met the procurement problem. It's not glamorous. It's not the kind of thing that shows up in quarterly strategy slides. But it's one of the most common ways small and mid-size businesses quietly leak money.

The typical procurement process in a company with 20–100 employees looks something like this: someone needs something, they send an email or Slack message to their manager, the manager says "go ahead," the employee finds a vendor, gets a quote, emails it to the manager, the manager forwards it to finance, finance cuts a check or enters a card payment, and nobody updates the budget spreadsheet until the end of the month — at which point the numbers don't match anyone's expectations.

That process has five handoffs, no audit trail, no spending visibility, and no enforcement of approval rules. It works when you're five people. It breaks at twenty. It's actively costing you money at fifty.

The fix isn't a $30/seat/month procurement SaaS. It's an automated purchase-order workflow built on open-source tools you self-host. Here's how to build one.

## Why Procurement Automation Matters

Before we get to tools, let's be specific about what you're fixing. A manual procurement process typically suffers from four problems:

1. **Maverick spending.** Employees buy from whichever vendor they find first, ignoring negotiated discounts or preferred suppliers. Studies put the cost of maverick spending at 10–20% of total procurement spend.
2. **No approval trail.** When someone asks "who approved this purchase?" the answer is usually "it was in an email thread somewhere." That's not an audit trail.
3. **Duplicate orders.** Without a centralized system, two people can order the same thing without knowing it. This is especially common for consumables and office supplies.
4. **Budget blindness.** Finance doesn't know what's been committed until invoices arrive. By then, the money is already spent.

An automated procurement workflow addresses all four: every purchase request goes through a defined approval chain, every order is logged in a central system, duplicate detection is automatic, and finance can see committed spend in real time.

## The Tools: Open Source, Self-Hosted, No Per-Seat Fees

You need three components: an ERP or purchasing system to hold purchase orders and vendor records, a workflow engine to route approvals, and optionally an AI model to classify and route requests intelligently. Here are the open-source tools we recommend:

| Tool | Role | License | Replaces |
|------|------|---------|----------|
| ERPNext | Purchase orders, vendor management, receiving | GPL-3.0 (open source) | SAP Ariba, Coupa |
| Odoo Community | Purchase requests, RFQs, vendor catalog | LGPL-3.0 (open source) | Procurify, PurchaseControl |
| n8n | Approval routing, notifications, integrations | Sustainable Use License (source-available) | Zapier, Microsoft Power Automate |
| Ollama | Local AI for request classification and routing | MIT (open source) | OpenAI API, Google AI APIs |

**A note on licenses:** ERPNext and Odoo Community are genuinely open source under OSI-approved licenses. n8n uses the Sustainable Use License, which is source-available — you can self-host it freely for internal use, but you can't resell it as a hosted product. For procurement automation within your own company, all three work without licensing concerns.

## The Architecture: How It Fits Together

Here's what the automated procurement workflow looks like end to end:

```
[Employee submits request]
        │
        ▼
  ┌─────────────┐
  │  n8n webhook │  ← Request form (OpnForm, Cal.com, or ERPNext portal)
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Ollama     │  ← Classifies: category, urgency, estimated cost
  │ (local LLM)  │     "office supplies, routine, ~$340"
  └──────┬──────┘
         │
         ▼
  ┌─────────────────────┐
  │ Approval routing     │  ← Under $500? Auto-approve to manager.
  │ (n8n switch node)    │     Over $500? Route to dept head + finance.
  └──────┬──────────────┘
         │
         ▼
  ┌─────────────┐
  │   ERPNext    │  ← Creates draft purchase order
  │  (PO module) │     Links to vendor, updates committed spend
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Notification │  ← Slack/Mattermost message to approver
  │  (n8n node)   │     Email confirmation to requester
  └─────────────┘
```

The flow is simple: an employee submits a purchase request through a form or the ERPNext portal. n8n picks it up, passes it to Ollama to classify (what category, how urgent, rough cost estimate), then routes it through the right approval chain based on amount and category. Once approved, ERPNext creates a draft purchase order linked to the vendor and updates the committed-spend ledger. The approver gets a Mattermost notification; the requester gets an email.

## Step 1: Set Up ERPNext for Purchase Orders

If you followed our [ERPNext self-hosting guide](/blog/2026/12/05/self-hosting-erpnext-replace-netsuite-and-sap-with-your-own-open-source-erp/), you already have the foundation. ERPNext has a built-in Purchase Order module that handles the full cycle: purchase requests → supplier RFQs → purchase orders → goods receipt → invoice matching.

The key setup steps:

1. **Create supplier records.** Add your vendors with their contact info, payment terms, and default currency. This takes 10–15 minutes per supplier and only needs to be done once.

2. **Set up item master.** Each recurring purchase item — office supplies, raw materials, services — gets a record with a default supplier and standard cost. This is what enables duplicate detection and maverick-spending alerts.

3. **Configure approval thresholds.** In ERPNext, you can set up approval rules based on amount. Under $500 goes to the department manager. Over $500 requires department head plus finance approval. Over $5,000 goes to the CEO.

```python
# Example: ERPNext approval rule via API
import requests

frappe_url = "https://erp.yourcompany.com"
api_key = "your_api_key"
api_secret = "your_api_secret"

# Create an approval threshold rule
rule = {
    "doctype": "Authorization Rule",
    "transaction_type": "Purchase Order",
    "based_on": "Grand Total",
    "applicable_for": "Purchase Order",
    "value": 500,
    "approving_role": "Purchase Manager"
}

response = requests.post(
    f"{frappe_url}/api/resource/Authorization Rule",
    json=rule,
    headers={
        "Authorization": f"token {api_key}:{api_secret}",
        "Content-Type": "application/json"
    }
)
print(f"Approval rule created: {response.status_code}")
```

## Step 2: Build the Approval Workflow in n8n

ERPNext handles the purchase order data. But the approval routing — the logic that says "this request is $1,200 for IT equipment, so it goes to the IT manager first, then finance" — is better handled in n8n, where you can build conditional branching without writing code.

Here's how to set up the workflow in n8n:

1. **Trigger node: Webhook.** Create a webhook that receives the purchase request. If you're using OpnForm (which we covered in [our form builder guide](/blog/2026/11/01/self-hosting-opnform-replace-typeform-with-your-own-form-builder/)), OpnForm can POST to this webhook on every submission.

2. **AI node: Ollama classification.** Pass the request description to a local Ollama model (we recommend Qwen 2.5 7B — it's fast and good at classification tasks). Ask it to return a JSON object with category, urgency, and estimated cost range.

```
Prompt: "Classify this purchase request. Return JSON with fields:
category (IT, office, raw-materials, services, other),
urgency (routine, urgent, critical),
cost_range (under-500, 500-2000, over-2000).

Request: '{request_description}'"
```

3. **Switch node: Route by cost range.** Three branches:
   - **Under $500:** Notify department manager. Auto-approve if no objection in 24 hours.
   - **$500–$2,000:** Require department manager approval, then notify finance.
   - **Over $2,000:** Require department manager + finance director approval.

4. **ERPNext node: Create purchase order.** On approval, call the ERPNext API to create a draft PO linked to the supplier and item.

5. **Notification node: Mattermost.** Send a message to the approver's channel with a one-click approve/reject link that calls back to n8n.

Here's what the n8n workflow JSON looks like for the routing logic:

```json
{
  "nodes": [
    {
      "type": "n8n-nodes-base.webhook",
      "name": "Purchase Request",
      "parameters": {
        "path": "procurement/request",
        "methods": ["POST"]
      }
    },
    {
      "type": "n8n-nodes-base.httpRequest",
      "name": "Ollama Classify",
      "parameters": {
        "url": "http://ollama:11434/api/generate",
        "method": "POST",
        "body": {
          "model": "qwen2.5:7b",
          "prompt": "={{ 'Classify this purchase request as JSON: ' + $json.body.description }}",
          "format": "json"
        }
      }
    },
    {
      "type": "n8n-nodes-base.switch",
      "name": "Route by Cost",
      "parameters": {
        "rules": [
          { "value2": "under-500", "outputKey": "low" },
          { "value2": "500-2000", "outputKey": "medium" },
          { "value2": "over-2000", "outputKey": "high" }
        ],
        "value1": "={{ JSON.parse($json.body.response).cost_range }}"
      }
    }
  ]
}
```

## Step 3: Add Duplicate Detection

One of the most valuable features of automated procurement is catching duplicate orders before they happen. Here's a simple approach using n8n:

When a new purchase request comes in, n8n queries ERPNext for all open purchase orders from the last 30 days. It passes the new request and the existing POs to Ollama with a prompt like:

```
"Does this new purchase request duplicate any of these existing
open purchase orders? Return JSON: {duplicate: true/false,
matching_po: 'PO-0123', reason: 'same item, same vendor'}"

New request: '{request_description}'
Existing POs: {existing_pos_json}
```

If Ollama flags a likely duplicate, n8n holds the new request and notifies the requester: "This may duplicate PO-0123 (Office supplies, $340, ordered 3 days ago). Do you still want to proceed?" This alone can save thousands of dollars per year in redundant orders.

## Step 4: Track Committed Spend in Real Time

The final piece is visibility. With manual procurement, finance doesn't see committed spend until invoices arrive — weeks or months after the purchase decision. With this automated setup, every approved purchase order immediately updates the committed-spend ledger in ERPNext.

You can build a simple Metabase dashboard (which we covered in our [self-hosting Metabase guide](/blog/2026/09/30/self-hosting-metabase-open-source-business-analytics-without-saas/)) that shows:

- **Committed vs. actual spend** by department, month-over-month
- **Open purchase orders** by supplier and age
- **Approval cycle time** — how long requests sit waiting for approval
- **Maverick spending alerts** — orders to non-preferred suppliers

This dashboard replaces the end-of-month spreadsheet ritual. Instead of reconstructing what happened, finance can see what's happening, in real time, on a single screen.

## What This Costs You

Let's compare the total cost of this self-hosted procurement stack versus a SaaS procurement platform:

| Component | Self-Hosted (annual) | SaaS Equivalent (annual, 25 users) |
|-----------|---------------------|-------------------------------------|
| ERP/purchasing (ERPNext) | $0 license + ~$600 server | Procurify: ~$7,000+ |
| Workflow engine (n8n) | $0 license + shared server | Zapier: $0 (limited) to $2,400+ |
| AI classification (Ollama) | $0 license + shared GPU/CPU | OpenAI API: ~$50–200/month |
| Analytics (Metabase) | $0 license + shared server | Embedded BI add-on: $3,000+ |
| **Total (year 1)** | **~$1,500–$3,000** | **~$12,000–$18,000** |
| **Total (year 2+)** | **~$1,200–$2,400** (server renewal) | **~$13,000–$20,000** (price increases) |

The self-hosted stack pays for itself in the first month — and you own the data, the software, and the infrastructure. No per-seat fees, no vendor lock-in, no surprise price hikes at renewal.

## Getting Started: A Practical First Week

Don't try to automate everything at once. Here's a realistic week-one plan:

**Day 1–2:** Install ERPNext via Docker (follow our [ERPNext guide](/blog/2026/12/05/self-hosting-erpnext-replace-netsuite-and-sap-with-your-own-open-source-erp/)). Add your top 10 suppliers and top 50 recurring purchase items.

**Day 3–4:** Set up the n8n webhook and Ollama classification node. Test it with sample purchase requests from different categories. Tune the prompt until classification is reliable.

**Day 5:** Configure the approval routing switch node. Start with a single rule: all requests over $500 go to the department manager. Keep it simple.

**Day 6–7:** Roll it out to one department — IT or facilities are good starting points. Collect feedback for two weeks, then expand to the rest of the company.

The goal of week one isn't perfection. It's getting a real workflow running with real users, so you can see where the friction is and iterate.

## Common Pitfalls

**Over-engineering the approval chain.** If every purchase needs three signatures, people will route around the system. Start with two tiers (manager + finance for large orders) and add complexity only when you see actual problems.

**Ignoring the vendor side.** Your automated system is great internally, but if you're still emailing POs to suppliers as PDFs, you've only automated half the process. ERPNext can email POs to suppliers automatically — turn that on.

**Classifying everything with AI.** Ollama is good at categorizing free-text requests, but don't use it for cost estimation on items that have a fixed price in your item master. Use the actual price from ERPNext; use AI only when the price is unknown.

**Forgetting to close the loop.** When goods arrive, someone needs to mark the PO as received in ERPNext. If that step is manual and gets skipped, your committed-spend numbers will drift from reality. Add an n8n reminder that pings the requester 3 days after the expected delivery date.

## The Bottom Line

Procurement automation isn't exciting. Nobody blogs about purchase order workflows. But it's one of the highest-ROI automations you can implement — because the savings are immediate, measurable, and recurring. Every duplicate order you catch, every maverick purchase you prevent, every approval cycle you shorten from five days to five hours — that's real money, every month, forever.

And you don't need a $15,000/year SaaS platform to do it. ERPNext, n8n, and Ollama give you a full procure-to-pay workflow that you own, that runs on your hardware, and that scales without per-seat fees. The setup takes a week. The savings start on day one.

---

*Ready to automate your procurement process? ARDOT Consulting helps businesses design and deploy open-source automation stacks tailored to their workflows — from purchase orders to invoice processing to inventory management. [Contact us](/) to schedule a free consultation and we'll map out your first automation in 30 minutes.*