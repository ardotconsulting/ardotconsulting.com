---
layout: post
title: "Automating Year-End Reporting: Close Your Books Faster with Open Source Tools"
date: 2026-12-17
author: "ARDOT Consulting"
tags: [year-end, reporting, finance, erpnext, metabase, n8n, ollama, automation]
excerpt: "Year-end reporting doesn't have to mean weeks of manual spreadsheet work. Here's how to automate financial close, dashboard generation, and narrative summaries using open source tools you can self-host."
---

# Automating Year-End Reporting: Close Your Books Faster with Open Source Tools

If December fills you with dread because it means weeks of exporting data, pasting it into spreadsheets, reconciling numbers, and manually building reports for your accountant — you're not alone. Year-end reporting is one of the most time-consuming tasks small businesses face, and it's also one of the most error-prone.

But here's the thing: most of the work is repetitive. Pull data, format it, check it, send it. That's exactly the kind of workflow that automation handles well. And you don't need expensive accounting software or a data team to do it. Open source tools like ERPNext, Metabase, n8n, and Ollama can handle the entire pipeline — from data collection to report generation to narrative summaries — and they run on hardware you already own.

This guide walks through a practical year-end reporting automation stack, step by step.

## The Problem with Manual Year-End Reporting

Before we get to solutions, let's name the pain. A typical small business year-end process looks something like this:

1. **Export invoices and expenses** from your invoicing tool (or worse, compile them manually)
2. **Reconcile bank statements** against internal records
3. **Categorize transactions** that were miscoded during the year
4. **Generate financial statements** — Profit & Loss, Balance Sheet, Cash Flow
5. **Build management reports** — departmental breakdowns, year-over-year comparisons
6. **Write a narrative summary** for stakeholders or your board
7. **Send everything to your accountant** and answer follow-up questions

Each of these steps takes hours when done manually. Worse, they're prone to copy-paste errors, version confusion (which spreadsheet is the latest?), and the "single person who knows how it works" problem — if your bookkeeper is out, nobody else can run the process.

Automation doesn't eliminate all of this work. Your accountant still needs to review and sign off. But it compresses what used to take two weeks into a process that runs mostly on its own, with humans reviewing exceptions rather than typing numbers.

## The Open Source Year-End Stack

Here's the toolchain we'll build:

| Tool | Role | License |
|------|------|---------|
| **ERPNext** | ERP and accounting system — source of truth for financial data | GPLv3 |
| **Metabase** | BI dashboard — visual reports and scheduled exports | AGPLv3 |
| **n8n** | Workflow automation — orchestrates the reporting pipeline | Sustainable Use License |
| **Ollama** | Local LLM — generates narrative summaries and flags anomalies | MIT |
| **Invoice Ninja** | Invoicing data — feeds into ERPNext or directly into reports | AAL |

> **Note on licenses:** n8n uses the Sustainable Use License, which is source-available but not OSI-certified open source. You can use it freely for internal business purposes, but commercial redistribution is restricted. ERPNext, Metabase, and Ollama are fully OSI-certified open source.

If you've been following this blog, you may already have some of these tools running. The beauty of this stack is that each tool does one job well, and they connect through standard APIs and database connections — no vendor lock-in.

## Step 1: Centralize Your Financial Data in ERPNext

The first step is making sure all your financial data lives in one system. If you're using Invoice Ninja for invoicing, you have two options:

**Option A: Use ERPNext's built-in invoicing.** ERPNext has a full accounting module with invoicing, expense tracking, bank reconciliation, and financial reporting. If you migrate your invoicing here, everything is in one place.

**Option B: Sync Invoice Ninja data into ERPNext.** Use n8n to periodically pull invoices from Invoice Ninja's API and push them into ERPNext. This works if you prefer Invoice Ninja's invoicing UI but want ERPNext's reporting.

Here's a simple n8n workflow node that syncs invoices:

```javascript
// n8n Code Node: Sync Invoice Ninja invoices to ERPNext
const invoices = $input.all();

for (const item of invoices) {
  const invoice = item.json;
  
  // Create Sales Invoice in ERPNext via REST API
  await this.helpers.httpRequest({
    method: 'POST',
    url: 'http://erpnext.local/api/resource/Sales Invoice',
    headers: {
      'Authorization': `token ${process.env.ERPNEXT_API_KEY}:${process.env.ERPNEXT_API_SECRET}`,
      'Content-Type': 'application/json'
    },
    body: {
      customer: invoice.client_name,
      posting_date: invoice.date,
      items: invoice.line_items.map(li => ({
        item_code: li.product_key || 'CONSULTING',
        qty: li.quantity,
        rate: li.cost
      }))
    },
    json: true
  });
}

return [{ json: { synced: invoices.length } }];
```

Once your data is in ERPNext, the financial statements are available out of the box:

- **Profit & Loss Statement** — revenue, expenses, net profit for the fiscal year
- **Balance Sheet** — assets, liabilities, and equity as of year-end
- **Cash Flow Statement** — operating, investing, and financing cash flows

You can generate these directly in ERPNext's Accounts module, filtered by fiscal year. No spreadsheet required.

### Period Closing in ERPNext

ERPNext has a built-in **Period Closing Voucher** feature. This lets you:

1. Freeze the fiscal year so no new entries can be posted
2. Transfer P&L balances to retained earnings automatically
3. Generate the closing entries without manual journal adjustments

Navigate to **Accounts > Closing Statement > Period Closing Voucher**, select your fiscal year, and submit. ERPNext handles the accounting logic — you just review the result.

## Step 2: Build Visual Dashboards in Metabase

ERPNext's built-in reports are good for accountants. But if you want visual dashboards — charts, trends, year-over-year comparisons — for your management team or board, Metabase is the tool.

Connect Metabase to your ERPNext database (ERPNext runs on MariaDB or PostgreSQL). Once connected, you can build queries visually or with SQL.

### Key Year-End Dashboards to Build

Here are five dashboards worth setting up:

**1. Revenue Summary**
- Monthly revenue bar chart for the fiscal year
- Top 10 customers by revenue
- Revenue by product/service category

**2. Expense Breakdown**
- Expenses by category (pie chart)
- Month-over-month expense trend (line chart)
- Largest expense entries (table with filters)

**3. Profit Margins**
- Gross margin by month
- Net profit trend
- Comparison to previous fiscal year

**4. Accounts Receivable Aging**
- Outstanding invoices grouped by age (0-30, 31-60, 61-90, 90+ days)
- Total AR as of year-end
- Top overdue accounts

**5. Cash Position**
- Cash balance trend through the year
- Cash inflows vs. outflows by month
- Ending cash balance

Here's a SQL query you can use in Metabase for a revenue-by-month chart (assuming ERPNext on MariaDB):

```sql
SELECT 
  DATE_FORMAT(posting_date, '%Y-%m') AS month,
  SUM(base_grand_total) AS revenue
FROM `tabSales Invoice`
WHERE docstatus = 1 
  AND posting_date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY month
ORDER BY month;
```

### Schedule Automatic Report Delivery

Metabase supports **dashboard subscriptions** — you can schedule any dashboard to be emailed or sent to Slack on a recurring schedule. For year-end reporting:

1. Build your year-end dashboard
2. Click the **Sharing** icon (top right of the dashboard)
3. Select **Dashboard subscriptions**
4. Choose email or Slack, set the recipients, and pick the schedule

For year-end, you might schedule it to run on December 31st and January 1st — once for the preliminary close, once for the final numbers. Metabase sends a PDF or PNG of the dashboard along with a link.

This means your management team gets a visual year-end report automatically, without anyone manually exporting charts and pasting them into a deck.

## Step 3: Orchestrate the Full Pipeline with n8n

Metabase handles dashboards. ERPNext handles accounting. But year-end reporting also involves coordination — triggering the close, generating exports, notifying people, and assembling the final package. That's where n8n comes in.

Here's a year-end reporting workflow you can build in n8n:

```
[Schedule Trigger: Dec 31]
    ↓
[ERPNext: Generate Trial Balance] ← REST API call
    ↓
[ERPNext: Create Period Closing Voucher]
    ↓
[Metabase: Trigger Dashboard Refresh] ← Metabase API
    ↓
[Metabase: Export Year-End Dashboard as PDF]
    ↓
[Ollama: Generate Narrative Summary] ← Local LLM
    ↓
[Send Email: Package + Summary to Accountant]
    ↓
[Slack/Mattermost: Notify Team]
```

### The n8n Workflow in Detail

**Trigger:** Use a Schedule node set to run on December 31st at 5 PM (or whenever your fiscal year ends). You can also trigger it manually for mid-year reviews.

**ERPNext nodes:** Use HTTP Request nodes to call ERPNext's REST API. Generate the Trial Balance, then create and submit the Period Closing Voucher.

**Metabase nodes:** Call Metabase's API to refresh the dashboard data and export it as a PDF. Metabase's API supports exporting dashboards and individual questions.

**Ollama node:** Send the key financial figures to a local Ollama instance and ask it to draft a narrative summary. More on this below.

**Email node:** Attach the PDF dashboard, the narrative summary, and a link to the ERPNext financial statements. Send it to your accountant and internal stakeholders.

**Notification node:** Post a summary message to Mattermost or Slack so the team knows the year-end package has been generated.

Here's the n8n workflow exported as JSON (you can import this directly into n8n):

{% raw %}
```json
{
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": "custom",
          "triggerAt": "31 17 31 12 *"
        }
      },
      "name": "Year-End Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1
    },
    {
      "parameters": {
        "url": "http://erpnext.local/api/method/erpnext.accounts.doctype.trial_balance.trial_balance.get_trial_balance",
        "options": {}
      },
      "name": "Get Trial Balance",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4
    },
    {
      "parameters": {
        "url": "http://erpnext.local/api/resource/Period Closing Voucher",
        "method": "POST",
        "body": "={{ JSON.stringify({\"fiscal_year\": \"2026\", \"company\": \"ARDOT Consulting\"}) }}",
        "options": {}
      },
      "name": "Close Period",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4
    },
    {
      "parameters": {
        "url": "http://metabase.local/api/dashboard/5/export",
        "format": "pdf",
        "options": {}
      },
      "name": "Export Dashboard",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4
    }
  ]
}
```
{% endraw %}

> **Tip:** You'll need to add authentication headers to each HTTP Request node. Store your API keys in n8n's built-in credentials manager — never hard-code them in the workflow.

## Step 4: Generate Narrative Summaries with Ollama

One of the most tedious parts of year-end reporting is writing the narrative — the summary that explains what the numbers mean. "Revenue grew 23% year-over-year, driven primarily by..." This is where a local LLM shines.

Ollama runs entirely on your own hardware. Your financial data never leaves your network. That matters for confidentiality — you don't want to send your P&L to a third-party AI API.

### Setting Up Ollama for Financial Summaries

First, install Ollama and pull a model:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model good for structured text generation
ollama pull llama3.2
```

Then, in your n8n workflow, add an HTTP Request node that calls Ollama's local API:

```javascript
// n8n Code Node: Generate narrative summary with Ollama
const trialBalance = $input.first().json;

const prompt = `You are a financial analyst. Write a concise year-end summary 
(300-500 words) for management based on the following financial data:

Total Revenue: $${trialBalance.total_revenue}
Total Expenses: $${trialBalance.total_expenses}
Net Profit: $${trialBalance.net_profit}
Top Revenue Sources: ${JSON.stringify(trialBalance.top_customers)}
Largest Expense Categories: ${JSON.stringify(trialBalance.top_expenses)}
Previous Year Revenue: $${trialBalance.prev_year_revenue}

Format the summary with:
1. One-paragraph executive summary
2. Key highlights (3-4 bullet points)
3. Areas of concern (if any)
4. Outlook for next year

Keep the tone professional but accessible. No hype.`;

const response = await this.helpers.httpRequest({
  method: 'POST',
  url: 'http://localhost:11434/api/generate',
  headers: { 'Content-Type': 'application/json' },
  body: {
    model: 'llama3.2',
    prompt: prompt,
    stream: false
  },
  json: true
});

return [{ json: { summary: response.response } }];
```

The output is a plain-text narrative summary that you can include in the email to your accountant, attach to the dashboard PDF, or paste into a board report. You'll want to review and edit it — Ollama is good at structure and first drafts, but it doesn't know context that isn't in the prompt.

### Hardware Requirements

Running Ollama locally doesn't require a data center. For a 7B or 8B parameter model like Llama 3.2:

- **Minimum:** 8 GB RAM, any modern CPU (slow generation, ~5-10 tokens/second)
- **Recommended:** 16 GB RAM, GPU with 6+ GB VRAM (fast generation, ~30-50 tokens/second)
- **For larger models (13B+):** 32 GB RAM, GPU with 12+ GB VRAM

If you're already running ERPNext and Metabase on a server, you can run Ollama on the same machine — just be mindful of memory. A model loaded into RAM takes 4-8 GB depending on quantization.

## Step 5: Automate Anomaly Detection

Beyond generating reports, automation can catch problems before your accountant does. Set up a simple anomaly check in n8n that runs monthly — not just at year-end — so issues are caught early.

```javascript
// n8n Code Node: Flag unusual transactions for review
const transactions = $input.all();
const flagged = [];

for (const item of transactions) {
  const txn = item.json;
  
  // Flag transactions 3x larger than the category average
  if (txn.amount > txn.category_avg * 3) {
    flagged.push({
      date: txn.posting_date,
      description: txn.description,
      amount: txn.amount,
      category: txn.category,
      reason: `Amount is ${Math.round(txn.amount / txn.category_avg)}x the category average`
    });
  }
  
  // Flag uncategorized transactions
  if (!txn.category || txn.category === 'Uncategorized') {
    flagged.push({
      date: txn.posting_date,
      description: txn.description,
      amount: txn.amount,
      reason: 'Uncategorized transaction — needs manual classification'
    });
  }
}

return [{ json: { anomalies: flagged, total_flagged: flagged.length } }];
```

Route the flagged items to a Mattermost channel or email digest so someone reviews them before they compound into a year-end headache.

## Putting It All Together

Here's what the complete automated year-end process looks like:

| Step | Tool | Time | Human Involvement |
|------|------|------|-------------------|
| 1. Sync invoices and expenses | n8n + Invoice Ninja + ERPNext | Automated (runs weekly) | None |
| 2. Reconcile accounts | ERPNext | ~1 hour | Review exceptions only |
| 3. Close fiscal period | ERPNext Period Closing Voucher | Automated | Review and approve |
| 4. Generate visual reports | Metabase dashboards | Automated (scheduled) | None |
| 5. Generate narrative summary | Ollama via n8n | Automated | Edit and refine |
| 6. Assemble report package | n8n (email + attachments) | Automated | None |
| 7. Review with accountant | Email/Meeting | ~2 hours | Accountant reviews |

Compare that to the manual process: 20-40 hours of bookkeeper time, another 5-10 hours of management time, and the inevitable back-and-forth when numbers don't match. The automated version replaces most of that with review time — humans checking the machine's work rather than doing the grunt work themselves.

## Getting Started: A Practical Migration Path

If you're starting from scratch (or from spreadsheets), don't try to build the entire stack at once. Here's a phased approach:

**Phase 1 (Week 1-2): Centralize data.** Install ERPNext via Docker. Import your chart of accounts, customers, and suppliers. Start using it for day-to-day bookkeeping.

**Phase 2 (Week 3-4): Add dashboards.** Connect Metabase to ERPNext's database. Build your first dashboard (start with Revenue Summary — it's the most universally useful). Set up a monthly email subscription.

**Phase 3 (Week 5-6): Add workflow automation.** Install n8n. Build a simple monthly workflow that exports the Trial Balance from ERPNext and emails it to you. This proves the pipeline works before you build the complex year-end version.

**Phase 4 (Week 7-8): Add AI summaries.** Install Ollama. Test it with a sample Trial Balance. Iterate on the prompt until the output is useful. Wire it into the n8n workflow.

**Phase 5 (Ongoing): Refine and automate more.** Add anomaly detection. Schedule the year-end workflow. Build departmental dashboards. Each quarter, review what's still manual and automate the next piece.

## Docker Compose for the Full Stack

If you want to run everything on a single server, here's a starting point:

```yaml
version: '3.8'

services:
  erpnext:
    image: frappe/erpnext:v15
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - REDIS_CACHE=redis-cache:6379
    depends_on:
      - db
      - redis-cache
    volumes:
      - erpnext_data:/home/frappe/frappe-bench/sites

  db:
    image: mariadb:10.6
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}
    volumes:
      - mariadb_data:/var/lib/mysql

  redis-cache:
    image: redis:7-alpine

  metabase:
    image: metabase/metabase:latest
    ports:
      - "3000:3000"
    environment:
      - MB_DB_TYPE=postgres
      - MB_DB_DBNAME=metabase
      - MB_DB_PORT=5432
      - MB_DB_USER=metabase
      - MB_DB_PASS=${METABASE_DB_PASSWORD}
      - MB_DB_HOST=metabase-db
    depends_on:
      - metabase-db

  metabase-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=metabase
      - POSTGRES_USER=metabase
      - POSTGRES_PASSWORD=${METABASE_DB_PASSWORD}
    volumes:
      - metabase_db_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n.local
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://n8n.local:5678
    volumes:
      - n8n_data:/home/node/.n8n

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama

volumes:
  erpnext_data:
  mariadb_data:
  metabase_db_data:
  n8n_data:
  ollama_data:
```

> **Resource note:** Running all five services on one machine works for testing and small businesses. For production, consider at least 16 GB RAM and 100 GB SSD. If you add GPU support for Ollama, pass `--gpus all` in the Docker run command or use the `deploy.resources` section in Docker Compose.

## Common Pitfalls

**Don't automate bad data.** If your chart of accounts is messy, automation will produce messy reports faster. Clean up your categorization before building dashboards.

**Don't skip human review.** The goal is to eliminate data entry, not judgment. Your accountant should still review the final numbers. The automation handles the assembly; the human handles the approval.

**Don't forget backups.** Your ERPNext database is now your financial system of record. Back it up daily. We covered this in detail in our [disaster recovery guide](/blog/2026/11/17/backing-up-your-self-hosted-stack-a-practical-disaster-recovery-guide/).

**Don't over-customize ERPNext.** The standard accounting module handles 95% of small business needs. Custom fields and workflows are tempting but create maintenance debt. Start with defaults and only customize when you hit a real wall.

## The Bottom Line

Year-end reporting is a perfect candidate for automation because it's complex, repetitive, and high-stakes. The open source stack we've outlined — ERPNext for accounting, Metabase for dashboards, n8n for orchestration, and Ollama for narrative generation — replaces what used to be weeks of manual work with a pipeline that runs largely on its own.

The real payoff isn't just time saved at year-end. It's that your financial data is always current, always accessible, and always in a format that supports decision-making — not just tax filing. When your CEO asks "how did Q3 compare to last year?" in January, the answer is already on a dashboard. No spreadsheet hunting required.

---

*Ready to automate your year-end reporting and stop dreading December? [Contact ARDOT Consulting](/#contact) — we help businesses design, deploy, and maintain open source automation stacks tailored to their workflows.*