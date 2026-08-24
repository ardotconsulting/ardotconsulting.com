---
layout: post
title: "AI Automation for E-Commerce: Inventory, Pricing, and Customer Service"
date: 2026-08-28
author: "ARDOT Consulting"
tags: [e-commerce, retail, automation, inventory, n8n, odoo, ollama]
excerpt: "How small online retailers can automate inventory sync, dynamic pricing, and customer service using open source tools — without sending data to third-party APIs."
---

## AI Automation for E-Commerce: Inventory, Pricing, and Customer Service

Running a small e-commerce business means juggling a dozen balls at once. Your inventory counts drift out of sync the moment a customer checks out on one channel but not another. Your prices go stale while competitors adjust theirs twice a day. Customer questions pile up faster than your team can answer them — and that's before you deal with returns, shipping updates, and restocking.

The good news? You don't need an enterprise platform or a six-figure budget to automate the most painful parts of e-commerce operations. Open source tools like **n8n**, **Odoo**, and **Ollama** can handle inventory sync, pricing updates, and customer service responses — all running on hardware you control.

This guide walks through three practical automation workflows you can build today.

---

### Why Open Source for E-Commerce?

Before diving in, it's worth addressing the elephant in the room: why not just use a commercial e-commerce platform's built-in automation?

Three reasons:

1. **Data ownership.** Every product detail, customer interaction, and pricing decision is your data. Commercial platforms lock it behind their API, charge for access, and can change terms anytime. Self-hosted tools keep data on your infrastructure.

2. **No per-transaction costs.** Most commercial automation platforms charge per task, per transaction, or per API call. When you're running hundreds of inventory checks per day, those costs add up fast. Open source tools have zero marginal cost.

3. **Integration flexibility.** n8n connects to virtually anything through HTTP nodes and webhooks. You're not limited to a platform's approved integration partners.

---

### Workflow 1: Multi-Channel Inventory Sync

**The problem:** You sell on your own Shopify store, an Etsy shop, and at a physical retail location. A customer buys the last unit of a product on Shopify. Five minutes later, someone orders the same product on Etsy. You're now oversold on a item you don't have.

**The solution:** A central inventory system (Odoo) that syncs stock levels across all channels in near real-time.

#### How it works

```
Customer buys on Shopify
        ↓
  Shopify webhook fires
        ↓
  n8n receives webhook
        ↓
  n8n updates Odoo stock
        ↓
  n8n pushes updated stock to Etsy API
        ↓
  n8n sends low-stock alert to Mattermost
```

#### Setting it up

**Step 1: Install Odoo Community Edition**

Odoo serves as your central inventory database. Run it with Docker:

```yaml
# docker-compose.yml
version: "3.8"
services:
  odoo:
    image: odoo:16.0
    ports:
      - "8069:8069"
    volumes:
      - odoo-data:/var/lib/odoo
    environment:
      - HOST=odoo-db
    depends_on:
      - odoo-db
  odoo-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=odoo
      - POSTGRES_PASSWORD=change_me
    volumes:
      - odoo-db-data:/var/lib/postgresql/data

volumes:
  odoo-data:
  odoo-db-data:
```

```bash
docker compose up -d
```

Open `http://localhost:8069`, create your database, and enable the Inventory app. Add your products with initial stock counts.

**Step 2: Create the n8n inventory sync workflow**

In n8n, create a new workflow with these nodes:

1. **Webhook node** — listens for Shopify's `order/create` webhook
2. **HTTP Request node** — calls Odoo's XML-RPC API to decrement stock
3. **HTTP Request node** — calls Etsy's REST API to update listing quantity
4. **Mattermost node** — sends an alert if stock drops below a threshold

Here's what the Odoo stock update node looks like:

```json
{
  "method": "POST",
  "url": "http://odoo:8069/xmlrpc/2/object",
  "body": {
    "jsonrpc": "2.0",
    "method": "call",
    "params": {
      "model": "stock.move",
      "method": "create",
      "args": [{
        "product_id": "={{ $json.line_items[0].product_id }}",
        "product_uom_qty": "={{ $json.line_items[0].quantity }}",
        "location_id": 8,
        "location_dest_id": 9
      }]
    }
  }
}
```

The Shopify webhook triggers this workflow within seconds of each order. Stock levels update across all channels automatically.

**Step 3: Set the low-stock threshold**

Add a Switch node after the Odoo update that checks if remaining stock is below your threshold (say, 5 units). If it is, send a Mattermost notification:

```
⚠️ Low stock alert: {{ $json.product_name }}
Remaining: {{ $json.stock }} units
Reorder point: 5 units
Action needed: Place restock order or mark as out of stock.
```

---

### Workflow 2: Dynamic Pricing Updates

**The problem:** Your competitors adjust prices based on demand, seasonality, and stock levels. You're still using the price you set six months ago. You're either leaving money on the table (priced too low during high demand) or losing sales (priced too high when competitors drop).

**The solution:** An automated pricing workflow that adjusts prices based on rules you define — competitor pricing, stock levels, time of year — without sending your sales data to a third-party API.

#### How it works

```
Scheduled trigger (every 4 hours)
        ↓
  n8n scrapes competitor prices
        ↓
  Ollama analyzes pricing context
        ↓
  n8n applies pricing rules
        ↓
  n8n updates prices in Odoo + storefront
```

#### Setting it up

**Step 1: Define your pricing rules**

Create a simple rules file that n8n references:

```yaml
# pricing-rules.yaml
rules:
  - name: "Clearance"
    condition: "stock > 20 AND days_listed > 90"
    action: "decrease by 15%"

  - name: "High demand"
    condition: "stock < 5 AND orders_last_7_days > 10"
    action: "increase by 5%"
    max_price: "MSRP * 1.1"

  - name: "Match competitor"
    condition: "competitor_price < current_price * 0.95"
    action: "set to competitor_price * 0.98"
    max_decrease: "10%"

  - name: "Seasonal"
    condition: "month in [11, 12]"
    action: "increase by 3%"
```

**Step 2: Use Ollama for pricing intelligence**

Instead of hard-coding every rule, use a local LLM to evaluate pricing context. Ollama runs on your own hardware, so your sales data never leaves your network:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model suitable for reasoning
ollama pull llama3.1
```

In n8n, add an HTTP Request node that calls Ollama's API:

```json
{
  "method": "POST",
  "url": "http://localhost:11434/api/generate",
  "body": {
    "model": "llama3.1",
    "prompt": "You are a pricing analyst. Given this data:\n\nProduct: {{ $json.product_name }}\nCurrent price: ${{ $json.current_price }}\nCompetitor price: ${{ $json.competitor_price }}\nCurrent stock: {{ $json.stock }}\nOrders last 7 days: {{ $json.recent_orders }}\nDays listed: {{ $json.days_listed }}\n\nRecommend a new price. Return only a JSON object with 'price' and 'reasoning' fields. Do not exceed a 10% change from current price.",
    "format": "json",
    "stream": false
  }
}
```

The LLM considers context that simple rules miss — like whether a product is trending, whether a competitor's price drop looks like a clearance or a promotion, and whether the season suggests higher demand.

**Step 3: Add guardrails**

Always add a guardrail node that caps price changes. No automated system should be allowed to change prices by more than a reasonable amount without human review:

```javascript
// n8n Code node — pricing guardrail
const newPrice = $('Ollama').item.json.price;
const currentPrice = $('Get Current Price').item.json.price;
const maxChange = currentPrice * 0.10; // 10% max change

if (Math.abs(newPrice - currentPrice) > maxChange) {
  // Flag for human review instead of auto-applying
  return {
    json: {
      action: "flag_for_review",
      product: $('Get Current Price').item.json.name,
      suggested_price: newPrice,
      current_price: currentPrice,
      change_pct: ((newPrice - currentPrice) / currentPrice * 100).toFixed(1)
    }
  };
}

return {
  json: {
    action: "apply",
    product: $('Get Current Price').item.json.name,
    new_price: newPrice
  }
};
```

---

### Workflow 3: Automated Customer Service Responses

**The problem:** Customers ask the same questions over and over: "Where is my order?", "Do you have this in size M?", "What's your return policy?" Your team spends hours copying and pasting the same answers.

**The solution:** An AI-powered customer service workflow that handles common questions automatically, escalates complex issues to humans, and works through your existing channels (email, contact form, chat).

#### How it works

```
Customer sends email / form submission
        ↓
  n8n receives the message
        ↓
  n8n classifies with Ollama
        ↓
  Simple query? → Auto-respond with order/return info
  Complex query? → Route to human + draft suggested response
```

#### Setting it up

**Step 1: Classify incoming messages**

Use Ollama to categorize each incoming message:

```json
{
  "method": "POST",
  "url": "http://localhost:11434/api/generate",
  "body": {
    "model": "llama3.1",
    "prompt": "Classify this customer message into one of these categories:\n\n- order_status: asking about shipment or delivery\n- return_refund: asking about returns, exchanges, or refunds\n- product_question: asking about product details, sizing, availability\n- billing: asking about invoices, payment issues\n- other: anything that doesn't fit above\n\nMessage: {{ $json.body }}\n\nReturn only the category name, nothing else.",
    "stream": false
  }
}
```

**Step 2: Auto-respond to simple queries**

For order status queries, n8n can look up the order in Odoo and respond directly:

```javascript
// n8n Code node — order status auto-response
const orderNumber = $('Extract Order Number').item.json.order_number;
const orderStatus = $('Check Odoo Order').item.json;

const responses = {
  "confirmed": `Hi {{ $json.customer_name }},\n\nYour order #${orderNumber} is confirmed and being prepared for shipment. Expected ship date: ${orderStatus.ship_date}.\n\nThanks for shopping with us!`,
  "shipped": `Hi {{ $json.customer_name }},\n\nYour order #${orderNumber} shipped on ${orderStatus.ship_date}. Tracking number: ${orderStatus.tracking}.\n\nYou can track it here: ${orderStatus.tracking_url}`,
  "delivered": `Hi {{ $json.customer_name }},\n\nYour order #${orderNumber} was delivered on ${orderStatus.delivery_date}. We hope you love your purchase!\n\nIf you have any issues, just reply to this email.`,
  "pending": `Hi {{ $json.customer_name }},\n\nWe received your order #${orderNumber} and it's pending payment confirmation. If you've already paid, please allow 24 hours for processing.`
};

return {
  json: {
    response: responses[orderStatus.state] || "Could not determine order status.",
    auto_send: orderStatus.state !== "unknown"
  }
};
```

**Step 3: Route complex queries to humans with a draft**

For messages classified as "other" or complex product questions, n8n routes to your support team via Mattermost, but includes an AI-drafted response they can review and send:

```json
{
  "method": "POST",
  "url": "http://localhost:11434/api/generate",
  "body": {
    "model": "llama3.1",
    "prompt": "Draft a helpful, friendly response to this customer message. Be concise and specific. Do not make up information — if you don't know an answer, say the team will follow up.\n\nCustomer message: {{ $json.body }}\nProduct context: {{ $json.product_info }}\nReturn policy: 30-day returns, free for defects, customer pays return shipping for change-of-mind.\n\nDraft response:",
    "stream": false
  }
}
```

The human agent reviews the draft, edits if needed, and sends. This cuts response time from 15 minutes to 2.

---

### Tool Comparison: E-Commerce Automation Stack

| Tool | Role | Self-Hosted? | Cost | Best For |
|------|------|-------------|------|----------|
| **n8n** | Workflow orchestration | Yes | Free (self-hosted) | Connecting all channels and tools |
| **Odoo Community** | Inventory + CRM + orders | Yes | Free (Community) | Central product database |
| **Ollama** | Local AI inference | Yes | Free | Classification, drafting responses |
| **Mattermost** | Team notifications | Yes | Free (self-hosted) | Alerts, escalations |
| **Directus** | Headless API layer | Yes | Free (self-hosted) | Custom storefront integration |
| **Cal.com** | Booking/scheduling | Yes | Free (self-hosted) | Appointment-based services |

All of these tools run on a single server. For a small e-commerce operation processing fewer than 1,000 orders per month, a VPS with 4 CPU cores and 8 GB RAM (around $40-60/month) is more than enough.

---

### Getting Started: A 7-Day Plan

If you want to implement these workflows, here's a practical week-by-week plan:

**Day 1:** Install Odoo and import your product catalog. Set initial stock levels.

**Day 2:** Install n8n (Docker) and connect it to your storefront's webhook. Test receiving order data.

**Day 3:** Build the inventory sync workflow. Start with just one channel (Shopify → Odoo). Test thoroughly before adding more.

**Day 4:** Install Ollama and pull a model. Test the classification prompt with sample customer messages.

**Day 5:** Build the customer service auto-response workflow. Start with just order status queries — the most common and lowest-risk automation.

**Day 6:** Build the pricing workflow. Run it in "dry run" mode (generates recommendations but doesn't apply them) for a week before enabling auto-updates.

**Day 7:** Set up Mattermost for alerts. Connect all workflows to send notifications on errors, low stock, and flagged items.

---

### Common Pitfalls to Avoid

**Don't auto-apply prices on day one.** Run the pricing workflow in observation mode for at least a week. Compare its suggestions to what you would have decided manually. Adjust the rules or prompt until you're comfortable.

**Don't auto-respond to every message.** Start with a narrow set — order status only — and expand gradually. Customers tolerate automated responses for simple factual queries but get frustrated when a robot tries to handle a complex complaint.

**Don't forget error handling.** If Odoo is down, n8n should queue the updates, not silently drop them. Add retry logic to every HTTP node and alert on consecutive failures.

**Don't skip the audit log.** Every price change, stock update, and auto-response should be logged. If a customer disputes a price, you need to know exactly when and why it changed. n8n's execution history handles this, but export it to a separate store regularly.

---

### The Bottom Line

E-commerce automation doesn't require an enterprise budget. With n8n orchestrating workflows, Odoo managing inventory, and Ollama handling AI tasks — all running on your own hardware — you can automate the three most time-consuming operations: inventory sync, pricing, and customer service.

The key is to start small, automate one workflow at a time, and always keep a human in the loop for edge cases. The tools are powerful, but they work best as an extension of your team — not a replacement for it.

---

*Want help setting up any of these workflows for your e-commerce business? [Get in touch with ARDOT Consulting](/contact/) — we specialize in open source automation for small businesses.*