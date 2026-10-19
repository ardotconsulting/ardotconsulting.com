---
layout: post
title: "Multi-Location Inventory Automation With Open-Source Tools"
date: 2026-12-07
author: "ARDOT Consulting"
tags: [inventory, multi-location, open-source, odoo, inventree, automation, stock-management, self-hosting]
excerpt: "Tracking stock across two or three locations is where spreadsheets break. Here's how to automate multi-location inventory with open-source tools that sync stock levels, trigger reorders, and give every site a real-time view — without a NetSuite subscription."
---

# Multi-Location Inventory Automation With Open-Source Tools

One warehouse is manageable. Two locations is where most businesses start losing money they can't see. Three or more, and you're almost certainly carrying dead stock in one building while another location stock-outs on the same item and expedite-orders it from a supplier.

The symptom is familiar: a customer asks whether an item is in stock, and the answer is "let me check with the other site." That phone call takes 15 minutes. The answer is often wrong by the time it arrives, because someone moved two pallets this morning and didn't update the spreadsheet. Multiply that by 40 SKUs and five locations, and you've got a full-time job that exists only because the system can't keep up.

The fix isn't more spreadsheets. It's a real inventory system that every location can see and write to in real time, with automated reorder points and stock-transfer tracking. The good news: you don't need NetSuite, SAP, or a $40,000/year SaaS subscription to get there. Open-source tools can do this today, on hardware you own.

This guide walks through the problem, the tools, and a realistic setup that a small business can run in a few weeks.

## Why Multi-Location Inventory Breaks First

Single-location inventory is hard enough. The moment you add a second site, three things break at once:

**1. The "truth" question.** When location A says you have 80 units on hand and location B says 120, which is right? Without a central database, you can't tell. Each site maintains its own spreadsheet, and they drift. The drift accelerates whenever stock moves between locations — which is exactly when accuracy matters most.

**2. The reorder problem.** Reorder points only work if you're looking at the right pool of stock. If location A hits its reorder point but location B has 400 units sitting idle, you'll over-order from your supplier and tie up cash. A central system with transfer-suggestion logic catches this. A spreadsheet doesn't.

**3. The visibility gap.** Your sales team can't sell what they can't see. If a customer wants 200 units and location A has 50, the sales rep needs to know location B has 180 — and whether those can be transferred in time. Without a shared view, the rep either under-promises (lost sale) or over-promises (angry customer).

Add three or more locations, and these problems compound. A 4-site distributor we worked with was carrying $180,000 in excess stock — almost all of it duplicate safety stock at each location, because nobody trusted the central number. They were also expediting roughly $4,000/month in supplier orders that could have been filled from another site. That's real money, every month, from a visibility problem.

## What a Real Multi-Location Inventory System Needs

Before we get to tools, let's be clear about the minimum bar. A system worth implementing has to do all of these:

- **Centralized stock ledger.** One database, every location reads and writes to it. No "the master spreadsheet lives on Sarah's laptop."
- **Per-location stock levels.** You can see on-hand, committed, and available for each SKU at each site — not just a company-wide total.
- **Stock transfers with tracking.** When stock moves from A to B, the system records the transfer, the quantity, the date, and who initiated it. The "in transit" state exists, so location A's stock drops before location B's stock rises, and you can reconcile if a transfer goes missing.
- **Automated reorder points per location.** Each location has its own min/max for each SKU, based on its own demand. The system flags when any location drops below its reorder point.
- **Transfer suggestions.** When location A needs stock, the system checks whether another location has excess before suggesting a supplier order.
- **Barcode/QR support.** Manual data entry is where accuracy dies. A system that can't accept a $30 barcode scanner or a phone camera scan isn't worth deploying.
- **Audit trail.** Every stock movement — receipt, transfer, sale, adjustment, write-off — is logged with a timestamp and a user. When the numbers don't match, you can find out why.

Anything that misses two or more of these isn't a multi-location system. It's a single-location system with extra steps.

## The Open-Source Options

There are three open-source tools worth serious consideration for multi-location inventory. They serve different needs, and the right one depends on how much else you want the system to do.

### Odoo Community Edition (Inventory module)

**Best for:** businesses that want inventory tied to sales, purchasing, and accounting in one system.

Odoo's Inventory module (called "Stock" internally) is part of the broader Odoo Community ERP. The Community Edition is genuinely open source under LGPL-3.0. The Inventory app supports:

- Multiple warehouses and locations, with a hierarchical structure (warehouse → location → sub-location)
- Real-time double-entry stock — every movement is a debit and credit, so the ledger always balances
- Automated reorder rules (min/max, or make-to-order) per location
- Transfer routes between locations, with full traceability
- Barcode scanning via the Odoo mobile app (Android/iOS) or a USB scanner
- Integration with purchasing, sales, and (if you enable it) accounting

The catch: Odoo Community's Inventory is powerful but the broader Odoo ecosystem has a learning curve. If you're already running Odoo for CRM or sales, adding multi-location inventory is a natural extension. If Odoo is new to you, expect a 2–4 week setup with a consultant or a determined internal person.

One important note: Odoo's "Enterprise" edition adds features like multi-step routes and predictive reorder — but Community Edition covers the basics above. Don't pay for Enterprise unless you've hit a specific wall in Community.

### InvenTree

**Best for:** businesses whose core need is inventory and parts management, without the full ERP overhead.

InvenTree is a dedicated open-source inventory management system, MIT-licensed, purpose-built for tracking stock across locations. It's lighter than Odoo and faster to stand up. Key capabilities:

- Multi-location stock tracking with a clean tree structure
- Stock transfers between locations, with full history
- Automatic low-stock notifications per part per location
- Build orders (BOMs) — useful if you assemble or kit products
- Barcode generation and scanning built in (works with phone cameras and USB scanners)
- A REST API and a Python library, so you can wire it into other systems

InvenTree doesn't do accounting, CRM, or sales order management. That's the trade-off. If you already have a sales system (Shopify, a custom storefront, a CRM) and you just need a serious inventory backbone, InvenTree is often the cleanest choice. We've deployed it for a 3-site electronics distributor in under a week.

### ERPNext (Stock module)

**Best for:** businesses that want a full ERP but prefer the Frappe ecosystem over Odoo.

ERPNext's Stock module covers the same ground as Odoo's Inventory — multi-warehouse tracking, stock transfers, reorder levels, barcode support, full audit trail — and it sits inside a complete ERP (accounting, HR, manufacturing, CRM). We covered ERPNext in detail in our [recent post on replacing NetSuite and SAP](/blog/2026/12/05/self-hosting-erpnext-replace-netsuite-and-sap-with-your-own-open-source-erp/), so we won't rehash the full setup here.

The choice between Odoo and ERPNext for inventory usually comes down to which ecosystem you prefer and which one your team finds easier to use. Both are capable. Try both on a test server before committing.

## How to Choose: A Quick Decision Guide

| You need… | Start with |
|---|---|
| Just inventory, fast, no ERP | **InvenTree** |
| Inventory + sales + purchasing in one system, and you're new to ERP | **Odoo Community** |
| Inventory + full accounting + HR + manufacturing | **ERPNext** |
| Inventory + you already run Odoo | Add Odoo's Inventory module |
| Inventory + you already run ERPNext | Add ERPNext's Stock module |

The wrong move is to pick the most feature-rich system (ERPNext) when you only need inventory. You'll spend six months configuring modules you don't use. Match the tool to the actual scope.

## A Realistic Setup: 3 Locations, InvenTree + n8n

Let's walk through a concrete deployment for a 3-site distributor — the kind of business that has outgrown spreadsheets but isn't ready for a full ERP. We'll use InvenTree for the inventory core and n8n for the automation layer (reorder alerts, supplier email, transfer suggestions).

**Step 1: Stand up InvenTree on a VPS (1 day).**
- Rent a VPS with 4GB RAM and 80GB storage (~$20/month from most providers — Hetzner, OVH, DigitalOcean)
- Deploy InvenTree with Docker Compose (official image, ~30 minutes)
- Configure 3 locations: `warehouse-main`, `warehouse-east`, `warehouse-west`
- Import your SKU list via CSV. InvenTree's importer handles part numbers, descriptions, units, and starting quantities.

**Step 2: Set reorder points per location (half day).**
For each SKU at each location, set a minimum stock level and a target (max) level. This is the boring part, but it's what makes automation work. Use your last 6 months of sales data to set realistic minimums — a rule of thumb is "average lead time in weeks × average weekly usage + 25% safety stock."

**Step 3: Wire up low-stock alerts with n8n (half day).**
Install n8n (self-hosted, Docker) and create a workflow that runs every morning:
1. Poll InvenTree's REST API for any parts below their minimum at any location
2. For each low-stock item, check whether another location has excess (available stock > 1.5× its own max)
3. If another location has excess → create a **transfer suggestion** (email to the warehouse manager, with a one-click approve link)
4. If no location has excess → create a **supplier reorder alert** (email to purchasing, with the suggested order quantity)

This is where the multi-location magic happens. The system is now actively looking for internal stock before it asks a supplier for money. That single workflow, for the 4-site distributor we mentioned earlier, eliminated about 70% of their expedited supplier orders within the first month.

**Step 4: Deploy barcode scanning (1 day).**
- Buy 3 USB barcode scanners (~$30 each) — one per location
- Generate barcode labels in InvenTree for each SKU and storage bin
- Train staff to scan in on receipt and scan out on pick. Every movement is now logged.

The scanning step is non-negotiable. Without it, the system relies on manual data entry, and manual data entry at a busy warehouse is unreliable. The scanner is the cheapest, highest-leverage piece of hardware in the whole stack.

**Step 5: Add stock-transfer tracking (built in).**
When stock needs to move from `warehouse-main` to `warehouse-east`, the warehouse manager creates a transfer in InvenTree. The stock leaves `main` immediately and shows as "in transit" until it's received at `east`. If a transfer sits in transit for more than 3 days, an n8n workflow pings both locations. Missing transfers — the ones where a truck got delayed or a pallet was misrouted — surface within days instead of at the next quarterly audit.

## What This Costs

Running this stack for a 3-location business:

| Item | One-time | Monthly |
|---|---|---|
| VPS (4GB RAM, 80GB) | — | $20 |
| Domain + TLS (if new) | $15 | — |
| 3 USB barcode scanners | $90 | — |
| Barcode label printer | $120 | — |
| Labels (roll of 5,000) | $15 | — |
| InvenTree + n8n licenses | $0 | $0 |
| **Total** | **~$240** | **~$20** |

Compare that to a mid-tier SaaS inventory platform, which typically runs $200–$600/month for multi-location, plus per-user fees, plus the cost of exporting your data if you ever want to leave. The open-source stack pays for itself in the first month and keeps paying every month after.

The real cost isn't the hardware. It's the time to set it up correctly — typically 5–10 days of focused work for a 3-location deployment, spread over 2–3 weeks so staff can adapt at each location.

## What Automation Will and Won't Fix

Be honest about scope. An inventory system fixes visibility, accuracy, and reorder discipline. It does not fix:

- **Bad supplier lead times.** If your supplier says 5 days and delivers 14, your safety stock math is wrong regardless of the system. Address the supplier, not the software.
- **Poor receiving discipline.** If staff don't scan in deliveries the day they arrive, the system's numbers are fiction. This is a management problem, not a tooling problem.
- **SKU proliferation.** If you have 12 variants of the same gasket because three different buyers each created their own part number, no inventory system will untangle that. Clean the SKU list first.
- **Demand you can't forecast.** A new product with no sales history will blow through your reorder points. Automation needs data; the first 90 days of a new SKU are still manual.

Automate the counting, the alerting, and the transfer tracking. Leave the supplier relationships, the staff training, and the SKU governance to humans. The system supports those decisions — it doesn't replace them.

## A 30-Day Rollout Plan

If you're going to do this, here's a realistic sequence:

- **Days 1–5:** Deploy InvenTree on a VPS. Import your SKU list. Set up the 3 locations. Load opening stock counts (do a physical count first — this is painful but necessary).
- **Days 6–10:** Set reorder points for your top 100 SKUs (by movement). Don't try to do all 2,000 at once. The 80/20 rule applies: your top 20% of SKUs drive 80% of the pain.
- **Days 11–15:** Deploy n8n. Build the low-stock alert workflow and the transfer-suggestion workflow. Test with real data.
- **Days 16–20:** Deploy barcode scanners at all 3 locations. Print labels. Train the receiving and picking staff. Expect resistance — scanning feels like extra work until it becomes habit, usually around day 25.
- **Days 21–30:** Run the system live. Meet weekly with each location's manager to review discrepancies. Fix the data entry gaps. By day 30, your stock accuracy should be above 90% (from a typical spreadsheet baseline of 60–75%).

After 30 days, the system runs itself. You spend an hour a week reviewing alerts, not 15 hours reconciling spreadsheets.

## The Bottom Line

Multi-location inventory is one of the clearest cases for automation in a growing business. The problem is well-defined, the tools are mature and open-source, the payback period is measured in weeks, and the alternative — carrying duplicate safety stock and expediting supplier orders — is a slow, invisible tax that compounds as you add locations.

The technology is no longer the barrier. The barrier is the discipline to deploy it properly: do the physical count, set the reorder points, enforce the scanning, and clean the SKU list. None of that is fun. All of it pays off.

If you're running stock across multiple sites and the spreadsheet is starting to lie to you, that's the signal. The fix exists, it's open source, and it runs on hardware that costs less than a week of excess inventory carrying costs.

---

If your business is outgrowing spreadsheets across multiple locations — or you've looked at the SaaS inventory platforms and balked at the pricing — [reach out to us](/#contact). We help businesses design and deploy open-source inventory stacks that sync across sites, automate reordering, and keep your data on your own hardware. No per-user fees, no vendor lock-in, no "export your data" exit tax.