---
layout: post
title: "From Manual to Automated: A 30-Day AI Automation Transformation"
date: 2026-09-14
author: "ARDOT Consulting"
tags: [case-study, transformation, 30-day-plan, n8n, ollama, odoo]
excerpt: "A realistic week-by-week case study of a 15-person consulting firm that went from fully manual operations to 60% automated in 30 days using n8n, Ollama, and Odoo — with honest numbers, stumbles, and lessons learned."
---

# From Manual to Automated: A 30-Day AI Automation Transformation

Most AI automation case studies read like victory laps. Everything worked. ROI was 400%. The team was thrilled from day one.

That's not how it actually goes. Real automation is messy. The first week is confusing. Something breaks in week two. Someone on the team resists the change. By week three you're questioning whether this was worth it. And somewhere around day 25, it starts clicking — not because the technology got better, but because you learned how to use it.

This is a realistic case study of a 15-person consulting firm that went from fully manual operations to roughly 60% automated in 30 days. The company is fictional, but every detail is drawn from patterns we've seen across real engagements. The tools are real: n8n for workflow automation, Ollama for local AI, Odoo for CRM and operations. The timeline is real. The stumbles are real.

If you're considering a similar transformation, this is what it actually looks like.

---

## The Starting Point

**Meridian Consulting** is a 15-person firm that does market research and due diligence for mid-market companies. Their work involves:

- **Client intake** — New clients fill out a PDF questionnaire. An associate manually transcribes the answers into a project folder, creates a Slack channel, and sets up a shared drive.
- **Research compilation** — Analysts pull data from 10–15 sources per project, copy it into spreadsheets, and write summary memos. A typical due diligence report takes 20–30 hours of analyst time.
- **Report generation** — Senior consultants review the analyst's work, request revisions, and assemble a final PDF report. Three rounds of revisions is normal.
- **Invoicing and follow-up** — The office manager sends invoices manually, tracks payments in a spreadsheet, and chases overdue accounts by email.

None of this is unusual. It's how most small professional services firms operate. The problem isn't that the work is hard — it's that it's repetitive, and repetitive work is where errors creep in and time leaks out.

**The baseline numbers:**

| Metric | Before |
|--------|--------|
| Client onboarding time | 45 minutes per client |
| Research memo first draft | 8 hours per project |
| Invoice processing | 3 hours/week (manual) |
| Report revision cycles | 3 rounds average |
| Overdue invoices (30+ days) | 22% of total |

The goal wasn't to replace anyone. It was to free up the 15 people to do higher-value work — the judgment work that clients actually pay for.

---

## Week 1: Foundation and First Win

### Days 1–2: Audit and Prioritize

Before touching any tool, the team spent two days mapping their actual workflows. Not the idealized version in the SOP document — the real version, including all the shortcuts and workarounds people had invented.

They listed every repetitive task and ranked them by two criteria: **how much time it eats** and **how predictable it is**. Automation loves predictable. If a task follows the same steps every time, it's a candidate. If it requires case-by-case judgment, it's not.

The priority list:

1. Client onboarding (45 min × 4 new clients/month = 3 hours/month, but more importantly, it delayed project starts)
2. Invoice processing and follow-up (3 hours/week, fully predictable)
3. Research source compilation (6 hours/project, semi-predictable)
4. Report formatting and assembly (2 hours/project, predictable)

### Days 3–5: Set Up the Tools

The team stood up three tools on a single server (a modest machine with 32GB RAM and an older GPU):

- **n8n** (self-hosted via Docker) — the workflow engine that would connect everything
- **Ollama** (running a 7B parameter model) — local AI for text extraction and summarization, no data leaves the server
- **Odoo Community Edition** (Docker) — CRM for tracking clients, projects, and invoices

This took longer than expected. Docker networking between the three services required a half-day of troubleshooting. The team's IT lead (one of the analysts, not a dedicated IT person) spent an afternoon on n8n credential management. None of it was hard in hindsight, but the first setup always takes longer than the tutorial suggests.

### Days 6–7: First Automation — Client Onboarding

The first workflow was the highest-impact, most predictable task: client onboarding.

The n8n workflow:

1. **Trigger:** New client questionnaire PDF arrives in a designated email inbox
2. **Extract:** Ollama reads the PDF and extracts structured data (client name, project type, scope, contacts, deadlines) into JSON
3. **Create:** Odoo API creates a new client record and project
4. **Notify:** A message posts to a Slack channel alerting the project team
5. **File:** The original PDF and extracted data are saved to the project folder

**Result:** Onboarding went from 45 minutes of manual work to about 90 seconds of automated processing. The associate who used to do this now reviews the extracted data for accuracy (2 minutes) instead of typing it in.

This was the first win, and it mattered more for morale than for time saved. The team saw that automation worked — and that it didn't require a software engineering degree to set up.

---

## Week 2: The Invoicing Automation (and the First Failure)

### Days 8–10: Invoice Processing

The office manager's invoicing workflow was the next target. The n8n workflow:

1. **Trigger:** Scheduled — runs every Monday at 9 AM
2. **Query:** Odoo API pulls all completed projects from the previous week
3. **Generate:** For each project, create an invoice record in Odoo with predefined rates
4. **Send:** Email the invoice PDF to the client (generated from an Odoo template)
5. **Log:** Record the invoice in a tracking spreadsheet via n8n's spreadsheet node

This worked — mostly. The first run generated invoices correctly, but the email step failed silently for two clients because their email addresses in Odoo had typos. The workflow completed "successfully" but those invoices never arrived.

**The lesson:** Automation does exactly what you tell it to. It doesn't notice that "jon@@client.com" looks wrong. A human would have caught that. The fix was adding a validation step that checks email format before sending, and an alert that fires if any step fails. Two hours of work to add safeguards that should have been there from the start.

### Days 11–14: Payment Follow-up

The second invoicing workflow handled overdue accounts:

1. **Trigger:** Scheduled — runs every Wednesday
2. **Query:** Odoo API pulls invoices overdue by 15, 30, and 60 days
3. **Branch:** Send different reminder templates based on overdue duration
4. **Escalate:** For 60+ days, notify the managing partner instead of emailing the client directly

**Result:** Overdue invoice follow-up went from a weekly manual chore to automatic. More importantly, the consistency improved — every overdue invoice got a reminder exactly on schedule, not "when someone remembers." Overdue rates started dropping within two weeks.

But there was pushback. The office manager felt the reminder emails were "too robotic." The templates were technically correct but tonally flat. The fix: the office manager rewrote the three email templates in her own voice, and the workflow uses those. Automation handles the timing and targeting; humans handle the tone.

---

## Week 3: Research Compilation — Where AI Actually Helps

### Days 15–18: Source Gathering Automation

Research compilation was the hardest target because it's less predictable than onboarding or invoicing. But a chunk of it — gathering sources, formatting them, creating a structured outline — is repetitive enough to automate.

The n8n workflow for research prep:

1. **Trigger:** New project created in Odoo (from the onboarding workflow)
2. **Parse:** Ollama reads the project scope and identifies research categories (market size, competitor landscape, regulatory environment, financial health)
3. **Search:** For each category, query predefined data sources via their APIs (public databases, industry repositories)
4. **Compile:** Gather results into a structured markdown document, one section per category
5. **Deliver:** Post the document to the project's shared folder and notify the assigned analyst

This didn't replace the analyst. It replaced the first 2–3 hours of their work — the gathering and formatting phase. The analyst still evaluates which sources are relevant, identifies gaps, and writes the actual analysis.

**Result:** First-draft research memos went from 8 hours to 5 hours. The 3 hours saved is the gathering and formatting work that no one enjoyed and that added no strategic value.

### Days 19–21: The Summary Assistant

The team added a second AI-assisted step: a workflow where an analyst can drop a long document (a 50-page annual report, a regulatory filing) into a folder, and Ollama produces a structured summary — key findings, financial highlights, risk factors — in a standard format.

This one had a quality learning curve. The first summaries were generic. The fix was giving Ollama a detailed prompt template that specified exactly what to extract and in what format, along with examples of good summaries. Prompt engineering isn't magic — it's specification. The better you define what you want, the better the output.

---

## Week 4: Report Assembly and the Honest Assessment

### Days 22–26: Report Formatting

The final automation target was report assembly. Senior consultants were spending 2+ hours per report formatting sections, generating a table of contents, inserting charts, and producing the final PDF.

The workflow:

1. **Trigger:** Analyst marks the project as "ready for review" in Odoo
2. **Assemble:** n8n pulls all section documents from the project folder in the correct order
3. **Format:** A Pandoc-based step merges the sections, generates a table of contents, and applies the firm's report template
4. **Deliver:** The formatted draft PDF lands in the senior consultant's review folder

**Result:** Report assembly dropped from 2 hours to 20 minutes (most of which is the consultant reviewing the formatting, not doing it manually). Revision cycles dropped from 3 to 2 on average, because the formatting was consistent every time — no more "the TOC is wrong on version 2."

### Days 27–30: Measurement and Adjustment

At day 30, the team measured where they stood:

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Client onboarding | 45 min | 2 min | −96% |
| Research memo first draft | 8 hrs | 5 hrs | −37% |
| Invoice processing | 3 hrs/week | 15 min/week | −92% |
| Report assembly | 2 hrs | 20 min | −83% |
| Revision cycles | 3 rounds | 2 rounds | −33% |
| Overdue invoices (30+ days) | 22% | 11% | −50% |

Total time saved across the firm: roughly **18 hours per week**, or about 1.2 hours per person. That's a 60% reduction in the repetitive work they targeted — not 60% of all work, but 60% of the specific tasks they set out to automate.

The team didn't shrink. Those 18 hours got redirected — toward deeper client analysis, business development, and actually leaving the office at a reasonable hour.

---

## What Went Wrong (Honestly)

1. **The first invoice automation failed silently.** Two invoices didn't send because of bad email data. Automation trust is earned through reliability, and silent failures destroy it. Fix: error alerts on every workflow, validated at the start.

2. **The office manager resisted the email templates.** Not because she didn't want automation — because the automated emails didn't sound like her. Fix: humans own the tone, machines own the timing. This is a general principle, not a one-off.

3. **Ollama summaries were generic until we wrote better prompts.** "Summarize this document" produces a generic summary. "Extract the three most important financial risks, the revenue trend over 5 years, and any regulatory citations" produces a useful one. The tool was fine; the instructions were bad.

4. **One analyst refused to use the research prep workflow for two weeks.** He preferred his own source-gathering method. This is normal and fine. The workflow ran alongside his manual process, and after he saw a colleague finish a memo 3 hours faster, he tried it. Adoption is social, not technical.

5. **The server went down once.** A Docker container restart loop took n8n offline for 4 hours. No work was lost (n8n stores execution data), but scheduled workflows didn't fire. Fix: a simple monitoring alert that pages someone if n8n's health check fails for more than 5 minutes.

---

## The Lessons That Apply to Any Business

**Start with the most predictable, highest-time task.** Not the most impressive one, not the one that sounds cool in a demo. The predictable one. Client onboarding and invoicing are boring. They're also where automation delivers immediate, measurable wins that build momentum.

**Automate the gathering, not the judgment.** AI can collect, format, summarize, and route. It can't decide whether a due diligence finding is material or whether a client relationship is at risk. Keep humans in the judgment loop. This is not a limitation — it's the correct division of labor.

**Expect a 30% quality discount on first attempts.** Your first workflow will have a bug. Your first AI summary will be generic. Your first automated email will sound flat. Budget time for iteration. The second version is always dramatically better than the first.

**Data quality is the foundation.** Bad email addresses, inconsistent project names, missing fields in Odoo — every automation failure in this case study traced back to data that was "good enough" for manual work but not for automated work. Clean your data before you automate.

**Adoption takes longer than setup.** The tools were running in a week. Full team adoption took three. People need to see it work for someone else before they trust it. Plan for this. Don't mandate — demonstrate.

---

## The 30-Day Plan, Condensed

If you want to replicate this approach, here's the compressed roadmap:

**Week 1:** Audit your workflows. Rank by time and predictability. Set up n8n, Ollama, and Odoo on a single server. Automate your most predictable task (onboarding or intake).

**Week 2:** Automate invoicing and payment follow-up. Expect your first failure. Add error alerts. Let a human own the communication tone.

**Week 3:** Tackle a knowledge-work task — research prep, document summaries, report drafting. Invest in prompt templates. Accept that quality improves with iteration.

**Week 4:** Automate report assembly or output formatting. Measure everything. Compare to baseline. Identify the next 30% to automate.

You won't get to 100% automated, and you shouldn't try. The goal is to remove the repetitive 60% so your team can focus on the valuable 40%.

---

## The Tools, For Reference

All open source. All self-hosted. No per-seat fees, no vendor lock-in, no data leaving your infrastructure.

- **[n8n](https://n8n.io/)** — Workflow automation. Connects your tools and triggers actions. The visual editor means you can build workflows without writing code.
- **[Ollama](https://ollama.com/)** — Run AI models locally. Your data never leaves your server. A 7B parameter model runs on modest hardware and handles extraction, summarization, and classification well.
- **[Odoo Community Edition](https://www.odoo.com/page/editions)** — CRM, project management, invoicing. The open source edition is surprisingly full-featured. Integrates with n8n via REST API.
- **[Docker](https://www.docker.com/)** — Runs all of the above in isolated containers. Makes setup, updates, and backups manageable.

Total infrastructure cost for this setup: one server (existing hardware or ~$50/month cloud) and the team's time. No SaaS subscriptions. No per-user pricing. No API fees for AI inference, because Ollama runs locally.

---

## Want This for Your Business?

This kind of transformation isn't reserved for tech companies. The firm in this case study had no developers — one analyst with Docker experience set up the tools over a weekend. The workflows are visual, the AI runs on your own hardware, and the entire stack is open source.

If you're spending significant time on repetitive work — intake, invoicing, research compilation, report formatting, customer communication — there's a 30-day version of this for your business. We help small firms identify what to automate, set up the tools, and train the team.

[Get in touch](/#contact) and tell us where your time is going. We'll tell you honestly what's worth automating, what isn't, and what a realistic 30-day plan looks like for your specific operations.