---
layout: post
title: "AI Automation for Construction: RFIs, Daily Reports, and Subcontractor Coordination"
date: 2026-10-28
author: "ARDOT Consulting"
tags: [construction, automation, n8n, ollama, paperless-ngx, odoo, rfi, daily-reports, subcontractor, open-source]
excerpt: "How construction firms can automate RFI processing, daily field reports, and subcontractor coordination using open source tools — replacing manual paperwork and email chains with workflows that actually keep up with the job site."
---

# AI Automation for Construction: RFIs, Daily Reports, and Subcontractor Coordination

If you've spent any time on a construction project, you know the paperwork problem. Every day generates a new stack of RFIs, submittals, daily reports, change order requests, and subcontractor correspondence. Most of it flows through email. Some of it gets lost. A fraction of it ends up properly filed in your project management system — usually weeks after the fact, when someone has time to catch up.

The cost of this chaos isn't just frustration. Missed RFIs cause delays. Lost submittals cause rework. Incomplete daily reports weaken your position in disputes. And the people buried in this paperwork — project engineers, superintendents, office managers — are the same people you need focused on actually building the project.

The good news is that the open source automation ecosystem has reached a point where you can build a genuinely useful construction document workflow without buying enterprise construction software. In this post, we'll walk through three high-impact automation areas: RFI processing, daily field reports, and subcontractor coordination. We'll use tools you can self-host — n8n, Ollama, Paperless-ngx, and Odoo — and keep everything on infrastructure you control.

## The Tool Stack

Before we get into the workflows, here's what we're working with:

| Tool | What It Does | Role in Construction Automation |
|------|-------------|-------------------------------|
| **n8n** | Workflow automation engine | Routes documents, triggers actions, connects systems |
| **Ollama** | Local LLM runtime | Extracts data from documents, classifies RFIs, summarizes reports |
| **Paperless-ngx** | Document management | Stores, OCRs, and organizes scanned documents and PDFs |
| **Odoo (Community)** | ERP / project management | Tracks projects, subcontractors, RFIs, and tasks |
| **Tesseract OCR** | Optical character recognition | Converts scanned site documents and photos to text |

All of these are open source. All can run on a single server or split across two. None require per-seat licensing or sending your project data to a third-party cloud.

Let's walk through each automation area.

---

## 1. RFI Processing: From Inbox Chaos to Tracked Action Items

Requests for Information are the lifeblood of construction projects — and one of the biggest sources of administrative drag. A typical RFI arrives as an email with a PDF attachment, sometimes a photo, sometimes a sketch. Someone has to read it, figure out what it's asking, determine who needs to answer, log it, and track it until it's resolved.

Here's how to automate the first 80% of that process.

### The Workflow

**Step 1: Capture.** All RFI emails are sent to a dedicated address like `rfi@yourcompany.com`. n8n monitors this inbox via IMAP. When a new message arrives, n8n pulls the email body and any attachments.

**Step 2: Extract.** n8n sends the email body and any PDF/image attachments to Ollama, running a local model like Llama 3.1 (8B). The model is given a structured prompt:

```
You are a construction RFI classifier. Read the following RFI and extract:
1. RFI subject (one line)
2. What is being asked (2-3 sentences)
3. Which discipline is involved (structural, MEP, architectural, civil, other)
4. Urgency (high, medium, low) — based on language like "delaying work" or "holding up trade"
5. Suggested responder role (architect, structural engineer, MEP consultant, owner, other)

Return as JSON.
```

For scanned PDFs or photos, Tesseract OCR runs first to extract text, then that text goes to Ollama.

**Step 3: Log.** n8n takes the structured JSON output and creates a record in Odoo — either as a project task tagged "RFI" or in a custom RFI model if you've set one up. The original PDF is stored in Paperless-ngx with appropriate tags (project name, RFI number, discipline).

**Step 4: Route.** Based on the suggested responder role, n8n sends a notification to the right person — via email, or via a Mattermost message if you're using it (and if you've followed our [self-hosting Mattermost guide](https://www.ardotconsulting.com/blog/2026/10/08/self-hosting-mattermost-replace-slack-with-your-own-team-chat-platform/), you should be). The notification includes the RFI summary, a link to the full document in Paperless-ngx, and a link to the Odoo record.

**Step 5: Track.** n8n sets a reminder — if the RFI hasn't been marked as answered in Odoo within 48 hours (for high urgency) or 5 days (for normal), it sends a follow-up nudge to the responder.

### What This Saves

On a mid-size project generating 10-15 RFIs per week, manual processing typically eats 4-6 hours of a project engineer's time — reading, routing, logging, chasing. This workflow cuts that to maybe 30 minutes of review time (someone confirms the AI's classification is correct and tweaks it if needed). The rest is automated.

The bigger win is speed. RFIs get routed in minutes instead of hours. Urgent ones get flagged automatically. Nothing falls through the cracks because every RFI is logged the moment it arrives, not when someone gets around to it.

### What It Doesn't Do

This workflow doesn't write the RFI response. That still requires a human — an architect, engineer, or consultant with the expertise to answer the question. What the automation does is handle everything *around* the response: capture, classification, routing, logging, and follow-up. That's the part that's pure administrative overhead.

---

## 2. Daily Field Reports: From Scribbled Notes to Structured Data

Daily reports are one of those tasks everyone acknowledges is important but nobody enjoys. Superintendents and foremen fill them out at the end of a long day, often by hand or in a spreadsheet, and they vary wildly in quality and format. The office team then has to transcribe them, reconcile them with time cards, and file them — usually a week behind.

Here's how to streamline this with automation.

### The Workflow

**Step 1: Capture in the field.** Field staff submit daily reports via a simple web form (hosted on your Odoo instance or a custom form built with n8n's form trigger). The form captures:

- Date and project
- Weather conditions
- Crew present (names or trade headcount)
- Work performed (free text)
- Materials delivered
- Equipment on site
- Delays or issues (free text)
- Photos (uploaded)

The form is mobile-friendly — it works on a phone browser. No app to install.

**Step 2: Structure with AI.** When the form is submitted, n8n sends the free-text fields to Ollama with a prompt to structure and summarize:

```
You are a construction daily report assistant. Take the following field notes 
and produce:
1. A one-paragraph summary of the day's work
2. A structured list of trades active on site
3. Any delays or issues mentioned, categorized as: weather, material, labor, 
   equipment, design, or other
4. A flag if the notes mention anything that could affect the schedule 
   (e.g., "waiting on", "couldn't proceed", "stopped work")

Return as JSON.
```

**Step 3: Store and index.** n8n creates a daily report record in Odoo with the structured data. The original field notes are preserved alongside the AI-structured version — you always have the raw input. Photos are stored in Paperless-ngx tagged with the project and date.

**Step 4: Flag and alert.** If the AI flags a potential schedule impact, n8n sends an alert to the project manager. Instead of discovering a delay in next week's report review, the PM hears about it the same day.

**Step 5: Weekly rollup.** Every Friday, n8n compiles the week's daily reports into a summary — total crew-hours by trade, materials delivered, issues flagged — and posts it to Odoo as a weekly project update. This gives you a clean record for project meetings and for any future disputes.

### What This Saves

The big win here isn't just time — it's data quality and timeliness. Structured daily reports that are filed the same day, in a consistent format, are vastly more useful than a stack of handwritten notes transcribed a week later. If you ever need to reconstruct what happened on day 47 of a project — for a claim, a dispute, or a schedule analysis — you have a searchable, structured record instead of a shoebox of paper.

On the time side: a superintendent spending 20 minutes a day on a paper report, plus an office person spending 15 minutes transcribing it, adds up to about 3 hours per person per week. The form-based approach with AI structuring cuts that to 5-10 minutes of field input with zero transcription.

### A Note on Adoption

Field crews are understandably skeptical of new technology. The key to adoption is making the form faster than the paper version. Keep it simple — no more than 8-10 fields. Use dropdowns for common entries (trades, equipment, weather). Let them dictate work performed using their phone's voice-to-text if they prefer. And make it clear that the AI is structuring their notes, not replacing their judgment — they still review and approve the final report.

---

## 3. Subcontractor Coordination: Tracking Deadlines and Deliverables

Subcontractor coordination is where construction projects live or die. You've got dozens of subs — concrete, steel, MEP, drywall, finishes — each with their own schedule, their own submittals, their own deliverables. Tracking who needs to deliver what and when is a full-time job for a project engineer, and it's usually done with a spreadsheet that's out of date the moment it's saved.

### The Workflow

**Step 1: Centralize submittal tracking in Odoo.** Each submittal and subcontractor deliverable is a record in Odoo with:

- Subcontractor name
- Submittal type (shop drawings, product data, samples, certifications)
- Required date
- Status (not started, in progress, submitted, approved, rejected)
- Linked RFI (if applicable)

**Step 2: Automated reminders.** n8n runs a daily check on all open submittals. For any submittal due within 7 days that isn't marked "submitted," n8n sends a reminder to the subcontractor via email (and a copy to the project manager). For anything overdue, it sends an escalation to the PM and the general contractor's project lead.

**Step 3: Submittal intake.** When a subcontractor emails a submittal to `submittals@yourcompany.com`, n8n captures it, OCRs any PDFs with Tesseract, runs the document through Ollama to classify it (which submittal does this correspond to in Odoo?), and updates the matching record. The PM gets a notification that a submittal has arrived and is ready for review.

**Step 4: Approval routing.** Submittals typically need to go to the architect or engineer for approval. n8n can route the submittal to the appropriate reviewer based on discipline — the same classification logic we used for RFIs. When the reviewer responds (approved, approved as noted, rejected), n8n updates the Odoo record and notifies the subcontractor.

**Step 5: Weekly coordination report.** Every week, n8n generates a subcontractor coordination report: who's on schedule, who's behind, what's due next week, and what's blocking critical path activities. This report goes to the PM and is filed in Odoo.

### What This Saves

The manual version of this workflow involves a project engineer spending several hours a day checking the submittal log, sending reminder emails, forwarding submittals to reviewers, and updating the spreadsheet. It's tedious, error-prone work that scales poorly as the number of subcontractors grows.

The automated version handles all the routine tracking and routing. The PM and project engineer focus on exceptions — the submittals that are overdue, the ones that came back rejected, the ones that affect critical path. That's where their judgment actually matters.

On a project with 30 subcontractors and 200+ submittals, this can save 10-15 hours per week of coordination administrative work. More importantly, it reduces the risk of a missed submittal derailing the schedule — which is the kind of problem that costs far more than any automation stack.

---

## Putting It All Together: A Realistic Deployment

You don't need to build all three workflows at once. Here's how we'd recommend phasing it:

### Phase 1 (Week 1-2): Document Capture and RFI Processing

Start with Paperless-ngx and n8n. Get all project documents flowing into Paperless-ngx with proper tags. Set up the RFI intake workflow. This gives you immediate value — every RFI is captured, classified, and routed — and it gets your team used to the idea that documents are handled automatically.

**Hardware:** A single server with 16GB RAM and 4 CPU cores can run Paperless-ngx, n8n, and Odoo. Cost: a modest VPS ($20-40/month) or a repurposed office machine.

### Phase 2 (Week 3-4): Daily Reports

Add the daily report form and AI structuring workflow. This is where Ollama comes in — you'll want a separate machine for LLM inference if you're running it locally, or you can use a GPU-equipped server. A consumer GPU (RTX 3060 or better) running Llama 3.1 8B will handle document classification and structuring with plenty of speed for a construction firm's volume.

### Phase 3 (Week 5-6): Subcontractor Coordination

Build out the submittal tracking and automated reminder system in Odoo. This is the most complex piece because it involves your subcontractors — external parties who need to interact with the system (primarily by emailing the right address). Keep the subcontractor-facing side dead simple: they email a document to an address, and the system handles the rest.

### Total Cost

| Component | Cost |
|-----------|------|
| Server (VPS or on-premise) | $20-80/month |
| LLM inference machine (optional, for local AI) | One-time $500-1500 for a desktop with a GPU |
| Software (n8n, Ollama, Paperless-ngx, Odoo) | $0 — all open source |
| Setup time | 20-40 hours over 6 weeks |

Compare that to enterprise construction management software, which typically runs $50-200+ per user per month and requires you to store all your project data in someone else's cloud. For a 10-person firm, that's $6,000-24,000 per year in software costs alone — and you don't control your data.

---

## Pitfalls to Watch For

**Pitfall 1: Overestimating what the AI can do.** Ollama running a local model is good at classification, extraction, and summarization — not at making engineering judgments. Don't try to automate the actual RFI *response* or submittal *approval*. Those require human expertise. Automate the paperwork around them.

**Pitfall 2: Garbage in, garbage out.** If your subcontractors email submittals with vague filenames and no project reference, the AI's classification will struggle. Establish simple conventions — "ProjectName_SubmittalNumber_Trade" in the email subject — and the automation will work dramatically better.

**Pitfall 3: Not training your team.** Field staff, project engineers, and subcontractors all need to understand the new workflow. Budget real time for training. Start with one project, not your whole portfolio. Let the team see it working before you roll it out broadly.

**Pitfall 4: Ignoring the exception process.** Automation handles the 80% of cases that follow the pattern. You need a clear, simple process for the 20% that don't — the RFI that doesn't fit any discipline, the submittal that arrives without a matching record, the daily report with a critical safety issue that needs immediate human attention. Build the exception path before you go live.

**Pitfall 5: Forgetting backups.** Your project documents are now digital and centralized. That's great for access and search, but it means a server failure could be catastrophic. Set up automated backups of Paperless-ngx, Odoo, and n8n to a separate location. Test your restores. This isn't optional — it's the equivalent of having a fireproof filing cabinet.

---

## The Bottom Line

Construction is an industry where the paperwork can feel as heavy as the materials. RFIs, daily reports, submittals, and subcontractor coordination are all essential — and all consume hours of skilled people's time on tasks that don't require their expertise.

With open source tools like n8n, Ollama, Paperless-ngx, and Odoo, you can build an automation stack that handles the document capture, classification, routing, and tracking — leaving your project team to focus on the work that actually requires judgment. The cost is a fraction of enterprise construction software, and your data stays on your infrastructure.

Start with RFI processing. It's the highest-impact, most straightforward workflow, and it gives your team a tangible win that builds confidence in the approach. From there, daily reports and subcontractor coordination are natural next steps.

Every project is different, but the pattern is the same: capture documents automatically, use AI to structure and classify them, route them to the right people, and track them until they're resolved. The tools are flexible enough to adapt to how your firm actually works — not the other way around.

---

*Want help designing a construction automation workflow for your firm? ARDOT Consulting builds self-hosted, open-source automation systems tailored to your project workflows — no vendor lock-in, no per-seat fees, your project data stays on your infrastructure. [Get in touch](/#contact) and we'll map out what's possible for your team.*