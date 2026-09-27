---
layout: post
title: "AI Automation for Logistics: Route Planning, Fleet Tracking, and Delivery Coordination"
date: 2026-09-27
author: "ARDOT Consulting"
tags: [logistics, fleet-management, automation, n8n, ollama, odoo, open-source, route-planning, delivery-tracking]
excerpt: "Route planning spreadsheets, driver text-message check-ins, and delivery status emails that nobody reads — logistics paperwork eats hours every day. Here's how to automate route planning, fleet tracking, and delivery coordination with open source tools you host yourself."
---

# AI Automation for Logistics: Route Planning, Fleet Tracking, and Delivery Coordination

If you run a small logistics operation — a fleet of five to fifty vehicles, a dispatch desk, a warehouse with a loading dock — you already know where the time goes. It's not in the driving. It's in the coordination. Someone spends the first two hours of every morning building route plans in a spreadsheet. Someone else spends the afternoon chasing drivers by phone to find out where truck #14 actually is. The customer service inbox fills with "where's my delivery?" emails that take four minutes each to answer, because the answer requires calling a driver who doesn't pick up because they're driving.

This is the logistics version of the paperwork problem that haunts every industry we cover. The work that requires judgment — deciding which loads to prioritize, handling a breakdown, managing a difficult customer — gets squeezed out by the work that's purely procedural: updating status, confirming ETAs, logging miles, reconciling delivery receipts.

The open source automation stack has gotten good enough to take most of that procedural work off your plate. In this post we'll walk through three high-impact automation areas for small logistics operations: route planning, fleet tracking, and delivery coordination. The tools — n8n, Ollama, Odoo, and a few specialized open source projects — are all self-hostable, with no per-vehicle licensing and no sending your operational data to a third-party cloud.

## The Tool Stack

Before we get into the workflows, here's what we're working with:

| Tool | What It Does | Role in Logistics Automation |
|------|-------------|------------------------------|
| **n8n** | Workflow automation engine | Routes data between systems, triggers alerts, syncs status |
| **Ollama** | Local LLM runtime | Parses delivery instructions, classifies emails, summarizes exceptions |
| **Odoo (Community)** | ERP / fleet module | Tracks vehicles, drivers, maintenance, and delivery orders |
| **Traccar** | Open source GPS tracking | Real-time vehicle location from cheap GPS hardware or driver phones |
| **OSRM** | Open Source Routing Machine | Calculates optimal routes between stops using open map data |

All of these are open source. Traccar and OSRM are purpose-built for logistics. Odoo has a fleet management module that's included in the Community edition. Together they replace a surprising amount of what fleet management SaaS vendors charge $15-40 per vehicle per month for.

## Workflow 1: Route Planning That Doesn't Take Two Hours

The morning route-planning ritual is classic automation territory. You have a list of deliveries for the day — addresses, time windows, priorities, vehicle assignments. Building an efficient route by hand requires either years of local knowledge or a lot of guessing. Most dispatchers do a rough geographic sort and move on, which is why the actual driving rarely matches the plan.

Here's what an automated version looks like:

**Step 1 — Deliveries land in Odoo.** When a delivery order is created in Odoo (manually, or automatically from an e-commerce system via n8n), it includes the destination address, delivery window, and priority. This is your source of truth.

**Step 2 — n8n pulls the day's deliveries and sends them to OSRM.** A scheduled n8n workflow runs at 6:00 AM, queries Odoo for all unassigned deliveries scheduled for today, and sends the list of waypoints to a self-hosted OSRM instance. OSRM calculates the optimal route — the actual road-network-optimized sequence of stops, not a straight-line approximation.

**Step 3 — Routes are assigned to vehicles.** The workflow groups deliveries by region and assigns each group to a vehicle based on capacity and availability (tracked in Odoo's fleet module). The optimized stop sequence is written back to each delivery order.

**Step 4 — Drivers get their route.** The workflow generates a simple route manifest for each driver — stop number, address, delivery window, special instructions — and sends it. You can deliver this as a PDF via email, a message through a chat platform like Mattermost, or even a printed copy at the dispatch desk. No driver app required.

The key insight here is that OSRM does the hard math. You're not asking an LLM to plan a route — that's not what LLMs are good at. You're using a dedicated routing engine that understands road networks, turn restrictions, and real distances. The LLM (Ollama) only comes in when there's unstructured text to parse — special delivery instructions like "leave at the back loading dock, call foreman John on arrival" that need to be extracted from the order notes and formatted cleanly for the driver manifest.

**What this saves:** A dispatcher who currently spends 90-120 minutes every morning on route planning gets that time back. The routes are also genuinely better — OSRM's optimization typically reduces total driving distance by 10-20% compared to manual geographic sorting, which means lower fuel costs and more deliveries per shift.

## Workflow 2: Fleet Tracking Without the Per-Vehicle SaaS Tax

Commercial fleet tracking services charge $15-40 per vehicle per month. For a 20-vehicle fleet, that's $3,600-9,600 per year — and you're sending your vehicles' location history to a vendor's cloud, where it's subject to their retention policies and their price increases.

Traccar is a mature, open source GPS tracking platform that does the same core job. It supports hundreds of GPS hardware devices (from $30 OBD trackers to professional hardwired units), and it can also use the Traccar Client app on a driver's phone as a tracking device — useful if you're not ready to buy hardware for every vehicle.

Here's how it fits into the automation stack:

**Hardware layer:** Each vehicle gets either a GPS tracker (hardwired or OBD) or the driver installs the Traccar Client app. The device reports position to your self-hosted Traccar server at a configurable interval — every 30 seconds when moving, every 5 minutes when stopped, for example.

**Traccar server:** Runs on your VPS alongside n8n and Odoo. It stores location history, shows a live map, and supports geofencing — you can draw a circle around your warehouse, a customer's facility, or a delivery zone and trigger events when vehicles enter or leave.

**n8n integration:** Traccar can send webhooks to n8n on geofence events. This is where the automation kicks in:

- **Vehicle leaves the warehouse** → n8n updates the delivery order status in Odoo to "in transit" and sends the customer an automated "your delivery is on the way, ETA is X" message.
- **Vehicle arrives at the delivery site** → n8n logs the arrival time on the delivery order and starts a timer. If the vehicle is still at the site after 45 minutes (configurable), n8n alerts the dispatcher — a possible delay or problem.
- **Vehicle returns to the warehouse** → n8n reconciles the day's deliveries against the route plan, flags any stops that were missed or completed out of sequence, and generates an end-of-day summary.

The beauty of this setup is that the tracking data never leaves your infrastructure. You control how long it's retained, who can see it, and what it's used for. There's no vendor deciding to raise prices or discontinue a feature you depend on.

**What this saves:** The direct replacement of fleet tracking SaaS is the obvious one — $3,600-9,600/year for a 20-vehicle fleet. But the bigger saving is the dispatcher's time. Instead of calling drivers to ask "where are you?", the answer is on the Traccar map. And the customer-facing automation (ETAs, arrival notifications) cuts down the "where's my delivery?" phone calls that eat customer service time all afternoon.

## Workflow 3: Delivery Coordination and Proof of Delivery

The last mile of logistics is where the communication overhead really piles up. Every delivery generates a chain of status updates: dispatched, in transit, arrived, completed. Every completion generates proof of delivery — a signature, a photo, a timestamp. And every exception (failed delivery, partial delivery, damaged goods) generates a cascade of phone calls, emails, and notes that need to be entered into the system.

Here's how to automate the common paths:

**The happy path — delivery completed:** When a driver completes a delivery, they need a way to capture proof. The simplest approach that works with the stack above: the driver's Traccar Client app (or a simple web form served by n8n) lets them tap "Delivered" and optionally attach a photo. n8n receives the event, updates the Odoo delivery order to "completed," attaches the photo and timestamp as proof of delivery, and sends the customer an automated delivery confirmation.

You can make this as simple or as sophisticated as your drivers will tolerate. A web form with three buttons — Delivered, Partial, Failed — and a photo upload covers 90% of cases. Don't overcomplicate it. Drivers are not going to fill out a ten-field form on a loading dock.

**The exception path — failed or partial delivery:** When a driver taps "Failed" or "Partial," n8n routes the event differently. It creates an exception ticket in Odoo, notifies the dispatcher immediately (via Mattermost message or email), and uses Ollama to do something useful: if the driver typed a brief note ("customer site closed, no one to receive"), Ollama categorizes the failure reason and suggests the next action based on your business rules. "Customer not available" might trigger an automatic re-scheduling workflow. "Address incorrect" might flag the order for dispatcher review.

The LLM is not making decisions here — it's classifying and structuring the driver's free-text note so your business rules can act on it. That's a good use of an LLM. Asking it to decide whether to re-route a delivery or send the driver back to the warehouse would be a bad use.

**The reconciliation path — end of day:** At the end of the shift, n8n generates a reconciliation report: planned deliveries vs. completed deliveries, exceptions with reasons, total drive time per vehicle (from Traccar), and any missed stops. This goes to the dispatcher and the operations manager. It's the same report someone is probably building by hand right now, except it's accurate, it's ready when the last truck pulls in, and it doesn't take 45 minutes to assemble.

## A Phased Rollout

Don't try to build all three workflows at once. Logistics operations are sensitive to disruption, and your drivers and dispatchers need time to trust the system.

### Phase 1 (Weeks 1-3): Fleet Tracking

Start with Traccar. Get GPS tracking on every vehicle — even if it's just the phone app to start. This gives you immediate visibility, builds the data foundation for the later workflows, and doesn't change anyone's daily routine. Dispatchers start using the live map instead of calling drivers. That alone is a meaningful win.

**Hardware:** A $20-40/month VPS runs Traccar comfortably for a small fleet. GPS hardware runs $30-80 per vehicle for OBD trackers, or $0 if you use the phone app.

### Phase 2 (Weeks 4-6): Route Planning

Add the OSRM + n8n route planning workflow once Traccar is stable and your deliveries are consistently logged in Odoo. Run the automated routes in parallel with the dispatcher's manual planning for a week — compare the results, tune the parameters (delivery windows, vehicle capacities, region groupings), and switch over when the team is confident.

**Hardware:** OSRM is memory-hungry during route calculation but fine on the same VPS if your fleet is small. For 50+ vehicles, a separate $10/month instance dedicated to OSRM is worth it.

### Phase 3 (Weeks 7-8): Delivery Coordination

Add proof-of-delivery capture and the customer notification workflow. This is the most driver-facing change, so introduce it carefully. Start with one vehicle or one route, get feedback on the form, and adjust. Drivers will tell you if the form is too slow or the photo upload doesn't work in low signal areas — listen to them.

### Total Cost

| Component | Cost |
|-----------|------|
| Server (VPS for Traccar, n8n, Odoo, OSRM) | $20-60/month |
| GPS hardware (optional, per vehicle) | $30-80 one-time per vehicle |
| Software (Traccar, OSRM, n8n, Odoo, Ollama) | $0 — all open source |
| Setup time | 25-45 hours over 8 weeks |

Compare that to commercial fleet management platforms, which typically run $15-40 per vehicle per month plus setup fees and integration costs. For a 20-vehicle fleet, that's $3,600-9,600 per year — every year — and your tracking data lives in someone else's database.

## Pitfalls to Watch For

**Pitfall 1: Bad address data breaks everything.** OSRM can only optimize routes to the addresses you give it. If your Odoo delivery orders have typos, missing apartment numbers, or old customer addresses, the routes will be wrong. Clean your address data before you automate. Spend a day normalizing addresses and adding geocodes. It's unglamorous work that makes every downstream workflow dramatically better.

**Pitfall 2: Over-relying on phone tracking.** The Traccar Client app works, but phones run out of battery, get left in the cab, or have location services disabled. For permanent tracking, hardwired GPS units are more reliable. Use phone tracking to get started and prove the value, then migrate to hardware for vehicles where reliability matters.

**Pitfall 3: Automating exceptions that need humans.** Failed deliveries, damaged goods, customer disputes — these need human judgment. Use automation to *surface* exceptions quickly and route them to the right person, not to resolve them. The goal is to make sure the dispatcher knows about a failed delivery within 30 seconds, not to have an AI decide what to do about it.

**Pitfall 4: Ignoring driver input.** Your drivers know things the routing engine doesn't — which loading docks are hard to access, which customers need morning deliveries, which roads to avoid at certain times. Build a feedback loop: if a driver consistently reorders their stops, figure out why and adjust the routing parameters. The system should learn from your drivers, not override them blindly.

**Pitfall 5: No offline fallback.** If your n8n server goes down at 6:00 AM, the route plans don't generate. Have a manual fallback — even if it's just yesterday's spreadsheet template — and make sure the dispatcher knows how to use it. Automation should make your operation more resilient, not create a single point of failure that paralyzes the morning dispatch.

## The Bottom Line

Logistics is an industry where small inefficiencies compound across every vehicle, every delivery, every day. A route that's 10% longer than it needs to be, a delivery status that takes three phone calls to confirm, a failed delivery that isn't logged until the driver gets back at 7 PM — these are the costs that quietly erode margins.

With open source tools like Traccar, OSRM, n8n, and Odoo, you can build an automation stack that handles route optimization, real-time tracking, and delivery coordination — the procedural work that eats your dispatchers' and drivers' time. The cost is a fraction of commercial fleet management SaaS, and your operational data stays on your infrastructure.

Start with fleet tracking. It's the lowest-friction workflow, it delivers immediate value, and it builds the data foundation for everything else. From there, route planning and delivery coordination are natural extensions. Every logistics operation is different, but the pattern is the same: capture data automatically, use the right tool for each job (OSRM for routing, Traccar for tracking, Ollama for parsing text, n8n for tying it together), and let your people focus on the decisions that actually require judgment.

If you want help mapping this out for your specific operation — your fleet size, your delivery patterns, your existing systems — [get in touch](/#contact). We'll walk through your current workflow and show you exactly where automation fits and where it doesn't.