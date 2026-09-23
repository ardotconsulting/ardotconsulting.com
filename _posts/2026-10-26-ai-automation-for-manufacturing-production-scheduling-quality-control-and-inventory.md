---
layout: post
title: "AI Automation for Manufacturing: Production Scheduling, Quality Control, and Inventory"
date: 2026-10-26
author: "ARDOT Consulting"
tags: [manufacturing, automation, odoo, n8n, ollama, inventory, quality-control, production-scheduling]
excerpt: "How small and mid-size manufacturers can automate production scheduling, quality inspections, and inventory tracking using open source tools — without expensive enterprise software."
---

# AI Automation for Manufacturing: Production Scheduling, Quality Control, and Inventory

If you run a small or mid-size manufacturing operation, you already know the pain. Production schedules change because a machine goes down. Quality inspections eat hours of skilled labor. Inventory counts are perpetually "close enough." And the ERP system you were quoted costs more than your forklift.

The good news: the open source ecosystem has matured to the point where you can build a genuinely useful manufacturing automation stack for the cost of a modest server and some configuration time. No enterprise software contracts. No per-seat licensing. No sending your production data to a third party.

In this post, we'll walk through three high-impact automation areas for manufacturers — production scheduling, quality control, and inventory management — and show you how to combine open source tools like Odoo, n8n, Ollama, and Tesseract into a working system.

## The Tool Stack

Before diving into use cases, here's the stack we'll be working with:

| Tool | What It Does | Why It Matters for Manufacturing |
|------|-------------|--------------------------------|
| **Odoo (Community Edition)** | ERP with manufacturing, inventory, and procurement modules | Replaces expensive ERP systems; self-hosted; modular |
| **n8n** | Workflow automation platform | Connects your tools together; handles triggers and actions |
| **Ollama** | Local LLM runtime | Processes text data (work orders, inspection reports) on your own hardware |
| **Tesseract OCR** | Optical character recognition | Extracts text from shipping labels, packing slips, and printed work orders |
| **Directus** | Headless database/API layer | Exposes your data through a clean API for custom dashboards and integrations |
| **Metabase** | Business analytics and dashboards | Visualizes production metrics, defect rates, and inventory trends |

All of these are open source, self-hostable, and can run on a single server for a small operation. Let's see how they fit together.

## 1. Production Scheduling: From Whiteboard to Automated

### The Problem

Most small manufacturers manage production schedules on a whiteboard, a spreadsheet, or — if they've invested — a basic ERP module that doesn't talk to anything else. When a machine goes down, a supplier delivers late, or a rush order comes in, someone manually reshuffles the board. That someone is usually the plant manager, who has ten other things to do.

### The Automation Approach

Odoo's Manufacturing module includes a Master Production Schedule (MPS) that lets you plan production orders based on forecasted demand. But where it gets powerful is when you connect it to n8n for automated triggers.

Here's a practical workflow:

1. **A rush order comes in** through your website or CRM
2. **n8n catches the trigger** (via webhook or database watch)
3. **n8n checks Odoo's production schedule** via the Odoo API
4. **Ollama analyzes the impact** — "If we insert this order, what gets delayed and by how much?"
5. **n8n creates a revised work order** in Odoo and sends a notification to the floor supervisor

The key insight: you're not replacing human judgment on the schedule. You're automating the data-gathering and impact-analysis steps so the human decision is faster and better-informed.

### Connecting Odoo and n8n

Odoo exposes a JSON-RPC/XML-RPC API. Here's how n8n can talk to it:

```python
# n8n Code node: Fetch today's production orders from Odoo
import xmlrpc.client

url = 'http://your-odoo-server:8069'
db = 'manufacturing_db'
uid = 1  # admin user id
password = 'your_api_key'

models = xmlrpc.client.ServerProxy(f'{url}/xmlrpc/2/object')

# Get all production orders scheduled for today
orders = models.execute_kw(db, uid, password,
    'mrp.production', 'search_read',
    [[['date_planned_start', '>=', '2026-10-26 00:00:00'],
      ['date_planned_start', '<=', '2026-10-26 23:59:59'],
      ['state', 'in', ['planned', 'progress']]]],
    {'fields': ['name', 'product_id', 'product_qty', 'workorder_ids', 'state'],
     'limit': 100}
)

# Return for next n8n node (e.g., Ollama analysis)
return [{"json": order} for order in orders]
```

Once n8n has the current schedule, it can pass the data to Ollama for analysis:

```python
# n8n Code node: Ask Ollama to analyze schedule impact
prompt = f"""
You are a production scheduling assistant. Here are today's planned 
production orders: {json.dumps(orders)}

A rush order for 500 units of Product X has arrived, requiring approximately 
3 hours of Machine A time. Machine A is currently booked from 9 AM to 2 PM.

Which orders should be shifted, and what are the downstream impacts on 
delivery dates? Provide a concise summary.
"""

# This gets sent to Ollama via n8n's HTTP Request node
# POST http://your-ollama-server:11434/api/generate
# Model: qwen2.5:7b (good balance of speed and reasoning)
```

The output isn't a final decision — it's a recommendation the floor supervisor reviews in a notification. But it turns a 30-minute analysis into a 30-second review.

### Practical Tip: Start with Notifications, Not Decisions

Don't try to fully automate scheduling decisions on day one. Start by having n8n send a Slack or Mattermost notification whenever the schedule changes:

> 🏭 **Schedule Alert**: Work Order WO-0142 (Widget A, 1000 units) has been delayed by 2 hours due to Machine B maintenance. Impact: 3 downstream orders may miss their delivery dates. [Review in Odoo →](http://your-odoo-server:8069/mrp/production)

Once your team trusts the notifications, you can layer in the AI-assisted impact analysis.

## 2. Quality Control: Automating Inspections and Defect Tracking

### The Problem

Quality control in small manufacturing is often paper-based. An inspector fills out a form, someone types the results into a spreadsheet, and by the time anyone spots a trend in defects, you've already shipped 200 bad parts.

### The Automation Approach

There are two layers to automate here: **data capture** (getting inspection results into a system quickly) and **trend detection** (spotting problems before they compound).

#### Layer 1: Automated Data Capture with Tesseract OCR

If your inspectors use paper forms (and many do — shop floors are rough environments), you can use Tesseract OCR to digitize them. A simple workflow:

1. Inspector photographs the completed form with a tablet or phone
2. Photo is uploaded to a shared folder (or Nextcloud instance)
3. n8n detects the new file and runs it through Tesseract
4. Extracted data is structured and posted to Odoo's quality control module or a Directus-backed database

Here's a basic Tesseract setup:

```bash
# Install Tesseract on your server
sudo apt-get install tesseract-ocr tesseract-ocr-eng

# Basic OCR on an inspection form image
tesseract inspection_form.jpg output_text --psm 6

# For structured forms, use Tesseract with a custom config
# that recognizes common inspection fields
tesseract inspection_form.jpg output_text \
  --psm 6 \
  -c tessedit_char_whitelist=0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ.-:/\ 
```

For more structured extraction, pipe Tesseract output through Ollama:

```python
# n8n Code node: Structure raw OCR text into inspection data
raw_text = items[0].json.ocr_text  # from Tesseract

prompt = f"""
Extract structured quality inspection data from this OCR text. 
Return a JSON object with these fields:
- inspector_name
- date
- part_number
- batch_number
- dimensions_pass (boolean)
- surface_finish_pass (boolean)
- weight_grams (number)
- defect_count (number)
- notes

OCR text:
{raw_text}
"""

# Send to Ollama via HTTP Request node
# Model: llama3.2:3b — fast enough for real-time use on the shop floor
```

This turns a 5-minute data entry task into a 5-second photo upload.

#### Layer 2: Trend Detection with Metabase

Once your inspection data is in a database (Directus makes this easy — it wraps any SQL database with a clean API and admin UI), connect Metabase to visualize trends:

| Dashboard Panel | What It Shows | Why It Matters |
|----------------|---------------|----------------|
| Defect rate by machine | % of failed inspections per machine over 30 days | Spot degrading equipment before it fails |
| Defect rate by operator | Failed inspections grouped by inspector/operator | Identifies training gaps, not blame |
| Defect type frequency | Pareto chart of defect categories | Focus improvement efforts on the top 3 causes |
| Trend alerts | Automated email when defect rate exceeds threshold | Catches problems before they compound |

Set up a Metabase alert that fires when the defect rate for any machine exceeds your threshold for 3 consecutive days:

```
Alert: Machine C defect rate has exceeded 5% for 3 consecutive days.
Current rate: 7.2%. Historical average: 2.1%.
Recommended action: Schedule maintenance check for Machine C.
```

This is the kind of early warning that saves a recall.

### Quality Control Workflow Diagram

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Inspector│───▶│ Nextcloud│───▶│   n8n    │───▶│Tesseract │
│  (photo) │    │  (upload)│    │ (trigger)│    │   (OCR)  │
└──────────┘    └──────────┘    └──────────┘    └────┬─────┘
                                                      │
                   ┌──────────┐    ┌──────────┐       │
                   │  Odoo /  │◀───│  Ollama  │◀──────┘
                   │ Directus │    │(structure)│
                   │ (record) │    └──────────┘
                   └────┬─────┘
                        │
                   ┌──────────┐
                   │ Metabase │
                   │(dashboard│
                   │ & alerts)│
                   └──────────┘
```

## 3. Inventory Management: Real-Time Tracking Without the Spreadsheet

### The Problem

Small manufacturers track inventory in one of three ways: a spreadsheet that's always out of date, a whiteboard that someone updates "when they get a chance," or an ERP module that nobody trusts because the data is never synced with what's actually on the shelf.

### The Automation Approach

Odoo's Inventory module handles the basics: stock levels, locations, moves, and valuation. The trick is keeping it accurate without manual counting. Here's where automation makes the difference.

#### Automated Reorder Points

Odoo supports minimum stock rules — when inventory drops below a threshold, it automatically generates a purchase order or manufacturing order. But you can make this smarter with n8n:

```python
# n8n Code node: Smart reorder check
# Runs every 2 hours via n8n Cron trigger

# 1. Fetch current stock levels from Odoo
low_stock_items = models.execute_kw(db, uid, password,
    'stock.quant', 'search_read',
    [[['quantity', '<=', 'reorder_point']]],
    {'fields': ['product_id', 'quantity', 'reorder_point', 'location_id']}
)

# 2. Check lead times and current supplier pricing
# (from your supplier database in Directus)

# 3. Ask Ollama to prioritize reorders
prompt = f"""
Based on these low-stock items, production schedule for the next 7 days,
and supplier lead times, which items should be reordered immediately 
vs. which can wait 3-5 days?

Low stock items: {json.dumps(low_stock_items)}
Lead times: {supplier_lead_times}

Rank by urgency (production impact if stockout occurs).
"""
```

The output goes to the purchasing manager as a prioritized list, not a raw alert:

> **Reorder Priority — Oct 26**
> 1. **Steel rod 12mm** — 200 units left, 3-day lead time, scheduled for Job WO-0151 tomorrow. **Order today.**
> 2. **Aluminum sheet 2mm** — 50 sheets left, 7-day lead time, next use in 5 days. **Order today.**
> 3. **Packaging boxes** — 300 left, 2-day lead time, next use in 8 days. *Can wait until Thursday.*

#### Barcode Scanning Integration

If you use barcode labels on your parts (and you should), n8n can process scan events from USB or Bluetooth scanners connected to a Raspberry Pi on the shop floor:

```yaml
# docker-compose.yml: Shop floor barcode scanner station
version: '3.8'
services:
  scanner-bridge:
    image: node:20-slim
    volumes:
      - ./scanner-app:/app
    environment:
      - N8N_WEBHOOK_URL=http://your-n8n-server:5678/webhook/scan
      - ODOO_URL=http://your-odoo-server:8069
    command: node /app/scanner.js
    restart: unless-stopped
    # Connect USB scanner device
    devices:
      - /dev/input/event0
```

When a worker scans a part moving from raw materials to work-in-progress, n8n:
1. Receives the barcode via webhook
2. Creates a stock move in Odoo
3. Updates the work order status
4. Logs the transaction in Metabase for real-time WIP tracking

No manual data entry. No clipboard. No end-of-week reconciliation.

#### Cycle Count Automation

Instead of shutting down for a full physical inventory count every quarter, automate cycle counting:

1. **n8n selects 20 random SKUs per day** (weighted by value and turnover)
2. **Sends a cycle count task** to the warehouse team via Mattermost
3. **Team counts and scans** the selected items
4. **n8n compares** scanned counts to Odoo's recorded quantities
5. **Discrepancies over 5%** trigger an automatic investigation ticket

This keeps your inventory accuracy above 95% without ever shutting down the line.

## Putting It All Together: A Reference Architecture

Here's what the complete system looks like for a small manufacturer running on a single server (or two):

```
                    ┌─────────────────────────────────┐
                    │        Shop Floor / Office       │
                    │                                  │
                    │  ┌──────┐  ┌──────┐  ┌────────┐│
                    │  │Tablet│  │Scanner│  │Browser ││
                    │  │(QC   │  │(stock │  │(Odoo   ││
                    │  │photo)│  │ moves)│  │ UI)    ││
                    │  └──┬───┘  └──┬───┘  └───┬────┘│
                    └─────┼─────────┼──────────┼─────┘
                          │         │          │
                    ┌─────▼─────────▼──────────▼─────┐
                    │        Internal Network         │
                    │                                 │
                    │  ┌──────┐   ┌──────┐  ┌──────┐│
                    │  │Odoo  │   │ n8n  │  │Ollama││
                    │  │(ERP) │◀──│(auto)│─▶│(LLM) ││
                    │  └──┬───┘   └──┬───┘  └──────┘│
                    │     │          │               │
                    │  ┌──▼───┐  ┌──▼────┐           │
                    │  │Directus│ │Tesseract│         │
                    │  │(API)  │  │(OCR)  │           │
                    │  └──┬───┘  └───────┘           │
                    │     │                            │
                    │  ┌──▼───────┐                   │
                    │  │Metabase  │                   │
                    │  │(dashboard)│                  │
                    │  └──────────┘                   │
                    └─────────────────────────────────┘
```

### Hardware Requirements

For a 20-50 person operation:

| Component | Spec | Approximate Cost |
|-----------|------|-----------------|
| Main server (Odoo, n8n, Directus, Metabase) | 8-core CPU, 32GB RAM, 500GB SSD | $2,000–$3,500 |
| AI server (Ollama with GPU) | 4-core CPU, 16GB RAM, RTX 4060 (8GB VRAM) | $1,200–$2,000 |
| Shop floor tablets (2-3) | Android tablets, rugged cases | $200–$400 each |
| Barcode scanners (2-3) | USB or Bluetooth, industrial grade | $150–$300 each |
| Raspberry Pi scanner bridges (optional) | Pi 4, 4GB | $75 each |

**Total infrastructure cost: roughly $5,000–$8,000** — less than one year of most enterprise ERP licenses.

Compare that to a typical small-business ERP quote: $15,000–$50,000 per year in licensing, plus implementation fees, plus per-user costs that scale with your headcount.

## Implementation Roadmap: 30 Days

Don't try to build everything at once. Here's a realistic phased approach:

### Week 1: Foundation
- Deploy Odoo Community Edition via Docker
- Import your product catalog, BOMs (bills of materials), and current inventory
- Set up user accounts for your team
- Deploy n8n alongside Odoo on the same server

### Week 2: Inventory Automation
- Configure minimum stock rules in Odoo
- Set up n8n cron jobs to check stock levels every 2 hours
- Connect Ollama for reorder prioritization
- Deploy barcode scanner bridges if using physical scanners

### Week 3: Quality Control
- Set up Tesseract OCR on the server
- Create the n8n workflow: photo upload → OCR → Ollama structuring → Odoo/Directus record
- Deploy Metabase and connect it to your database
- Build the quality dashboard (defect rate, trends, alerts)

### Week 4: Production Scheduling
- Configure Odoo's MRP module with your work centers and routings
- Set up n8n triggers for schedule change notifications
- Add Ollama-powered impact analysis for rush orders
- Test the full workflow end-to-end with a real production run

## Common Pitfalls (and How to Avoid Them)

**Pitfall 1: Over-automating quality control.** OCR and LLMs are good at structuring data, but they shouldn't replace human inspection of safety-critical parts. Use automation for data capture and trend detection — keep the pass/fail decision with a qualified inspector.

**Pitfall 2: Ignoring data quality in Odoo.** If your BOMs are wrong, your production schedules will be wrong, and your automated reorder alerts will be wrong. Spend the first week cleaning up your master data before building automations on top of it.

**Pitfall 3: Running Ollama on the same server as Odoo.** LLM inference is CPU/GPU intensive and can slow down your ERP. Run Ollama on a separate machine — even a modest desktop with a consumer GPU will outperform trying to share resources on your main server.

**Pitfall 4: Not training your team.** The most sophisticated automation stack is useless if your floor workers and inspectors don't use it. Budget time for training. Start with one workflow (usually inventory scanning) and expand from there.

## The Bottom Line

Manufacturing automation doesn't require a six-figure ERP implementation or a team of data scientists. With Odoo, n8n, Ollama, and Tesseract, you can build a system that handles production scheduling, quality control data capture, and inventory tracking — for a fraction of what traditional manufacturing software costs.

The key is to start small, automate the most painful manual process first, and build out from there. Every shop floor is different, but the tools are flexible enough to adapt to yours.

---

*Want help designing an automation stack for your manufacturing operation? ARDOT Consulting specializes in open source AI automation for small and mid-size businesses. [Contact us](https://www.ardotconsulting.com/#contact) to schedule a free consultation — we'll map your workflows and recommend a phased implementation plan.*