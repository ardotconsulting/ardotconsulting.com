---
layout: post
title: "AI Automation for HR: Recruitment, Screening, and Onboarding"
date: 2026-09-26
author: "ARDOT Consulting"
tags: [hr, recruitment, automation, onboarding, ai, open-source]
excerpt: "How small and mid-size businesses can automate resume screening, interview scheduling, and onboarding paperwork using open source AI tools — without sending sensitive employee data to third-party APIs."
---

# AI Automation for HR: Recruitment, Screening, and Onboarding

If you've ever posted a job opening and watched 300 resumes flood your inbox within 48 hours, you know the problem. Someone has to read every single one. Someone has to schedule phone screens with the promising candidates. Someone has to chase down offer letters, tax forms, and policy acknowledgments. And that someone is usually you — or an HR team already drowning in other work.

Human resources is one of the most automation-friendly functions in any business. The processes are repetitive, document-heavy, and rule-based. But HR is also where data privacy matters most. Resumes contain personal information. Onboarding involves tax documents and identification. Sending all of that to a third-party AI service isn't just expensive — it's a compliance risk.

That's where open source AI tools change the equation. You can run the entire pipeline on your own infrastructure: resume parsing with local AI models, automated screening with custom criteria, interview scheduling that syncs with your calendar, and onboarding workflows that generate and track paperwork automatically. No data leaves your servers.

This guide walks through how to build that pipeline using tools you can self-host today.

## The HR Automation Landscape

Before diving into implementation, let's map the territory. A typical hiring process has five stages, and each one has automation potential:

| Stage | Manual Process | Automated Approach |
|-------|---------------|-------------------|
| **Job posting** | Copy-paste to multiple boards | n8n workflow distributes to multiple channels |
| **Resume screening** | Human reads every CV | Ollama-powered parser scores and ranks resumes |
| **Interview scheduling** | Email back-and-forth | Cal.com integration with automated booking |
| **Assessment** | Manual test grading | Automated scoring with custom rubrics |
| **Onboarding** | Paperwork, accounts, equipment | n8n orchestrates document generation and provisioning |

The goal isn't to remove humans from hiring. It's to remove the paperwork and scheduling friction so your team can focus on the parts that actually require judgment — interviews, culture fit, and final decisions.

## Tool Stack Overview

Here's what we'll use and why:

| Tool | Role | Why Open Source |
|------|------|-----------------|
| **n8n** | Workflow orchestration | Self-hosted, no per-task pricing, visual builder |
| **Ollama** | Local LLM inference | Resumes and employee data never leave your server |
| **Cal.com** | Interview scheduling | Open-source scheduling, integrates with n8n |
| **Directus** | Candidate database | Headless CMS that stores applicant data securely |
| **Mattermost** | Team notifications | Open source Slack alternative for hiring team alerts |

All five can run on a single server with 16GB RAM. Ollama is the heaviest component — it needs a GPU for fast inference, but CPU-only mode works fine for the batch processing that resume screening involves.

## Stage 1: Resume Screening with Ollama

The first bottleneck in any hiring process is resume triage. When you get 200 applications for one role, reading them all is a full day's work. Here's how to automate it.

### Setting Up the Parser

First, install Ollama and pull a model suitable for text analysis:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model — Qwen2.5 works well for structured extraction
ollama pull qwen2.5:7b
```

Next, write a prompt that extracts structured data from each resume. The key is to ask for JSON output with consistent fields:

```python
import json
import subprocess

def screen_resume(resume_text, job_description):
    prompt = f"""
You are an HR assistant. Evaluate this resume against the job description.
Return a JSON object with these fields:
- name: candidate name
- email: contact email
- years_experience: estimated years of relevant experience
- skills_matched: list of required skills found
- skills_missing: list of required skills not found
- score: 0-100, how well the resume matches the job
- summary: 2-3 sentence assessment
- recommendation: "interview", "maybe", or "reject"

Job Description:
{job_description}

Resume:
{resume_text}
"""
    result = subprocess.run(
        ["ollama", "run", "qwen2.5:7b", prompt],
        capture_output=True, text=True, timeout=60
    )
    return json.loads(result.stdout.strip())

# Example usage
resume = open("candidate_resume.txt").read()
job = open("job_description.txt").read()
evaluation = screen_resume(resume, job)
print(f"{evaluation['name']}: Score {evaluation['score']} — {evaluation['recommendation']}")
```

### Batch Processing with n8n

Running this one resume at a time is fine for testing, but for production you want a workflow. Here's how to build it in n8n:

1. **Trigger node**: Watch an email inbox (or a shared drive folder) for new resume attachments
2. **PDF extraction node**: Use a function node with `pdfplumber` to extract text from PDF resumes
3. **Ollama node**: Send the extracted text to your local Ollama instance via HTTP request
4. **Directus node**: Store the parsed candidate data in your Directus database
5. **Filter node**: Route candidates based on score — above 80 goes to "interview," 60-80 goes to "maybe," below 60 gets a polite rejection
6. **Mattermost node**: Post high-scoring candidates to a `#hiring` channel for the team to review

Here's the HTTP request configuration for the Ollama node:

```json
{
  "method": "POST",
  "url": "http://localhost:11434/api/generate",
  "headers": { "Content-Type": "application/json" },
  "body": {
    "model": "qwen2.5:7b",
    "prompt": "={{ $json.prompt }}",
    "format": "json",
    "stream": false
  }
}
```

The `format: "json"` parameter tells Ollama to return valid JSON, which makes parsing reliable. No more guessing where the model's output starts and ends.

### Handling Bias and Fairness

This is where you need to be careful. An LLM can inherit biases from its training data — it might score resumes from certain universities higher, or penalize career gaps. Here are practical mitigations:

- **Strip identifying information** before scoring. Remove name, address, and university names from the resume text before sending it to Ollama. This forces the model to evaluate skills and experience only.
- **Use the model as a triage tool, not a final decision-maker.** Have it sort candidates into buckets (strong match, possible match, likely not a fit) but always have a human review the "maybe" pile.
- **Audit periodically.** Every few weeks, manually review 20 randomly selected resumes that the model scored. Check whether the scoring aligns with your actual hiring outcomes.

```python
def anonymize_resume(text):
    """Remove identifying info before AI screening."""
    import re
    # Remove email addresses
    text = re.sub(r'\S+@\S+', '[EMAIL]', text)
    # Remove phone numbers
    text = re.sub(r'\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}', '[PHONE]', text)
    # Remove names (common patterns at top of resume)
    text = re.sub(r'^[A-Z][a-z]+ [A-Z][a-z]+', '[NAME]', text)
    return text
```

## Stage 2: Automated Interview Scheduling

Once you've identified candidates worth interviewing, the scheduling dance begins. "Are you available Tuesday at 2?" "No, how about Wednesday morning?" "I have a meeting then..." — this back-and-forth wastes hours.

### Cal.com Integration

Cal.com is an open-source scheduling tool you can self-host. It lets candidates book time slots directly, eliminating the email tennis match.

Here's the setup:

1. **Install Cal.com** via Docker:
```yaml
# docker-compose.yml (simplified)
version: "3.8"
services:
  calcom:
    image: calcom/cal.com:latest
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://calcom:password@db:5432/calcom
      - NEXTAUTH_SECRET=your-secret-here
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=calcom
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=calcom
```

2. **Create event types** for different interview stages: 15-minute phone screen, 45-minute technical interview, 30-minute culture fit call.

3. **Connect to n8n**: When Cal.com receives a booking, it sends a webhook to n8n. The workflow then:
   - Creates a candidate record in Directus (or updates an existing one)
   - Sends a confirmation email to the candidate
   - Notifies the interviewer via Mattermost
   - Generates a interview prep sheet by pulling the candidate's parsed resume data

4. **Automated reminders**: n8n sends a reminder 24 hours before the interview to both the candidate and the interviewer, with the video link and the candidate's summary.

### The n8n Workflow for Scheduling

Here's what the webhook-triggered workflow looks like:

```
Cal.com Webhook → 
  Filter (event_type = "phone_screen") → 
    Directus (update candidate status to "scheduled") →
      Email (send confirmation to candidate) →
        Mattermost (notify interviewer with prep sheet) →
          Schedule (set reminder for 24h before)
```

The interview prep sheet is generated by sending the candidate's parsed resume data back to Ollama with a prompt like: "Generate 5 interview questions tailored to this candidate's background and the job requirements."

## Stage 3: Onboarding Automation

Onboarding is where automation delivers the most time savings. A new hire requires 15-30 administrative tasks across different systems — and missing one can mean compliance issues or a poor first-day experience.

### Building the Onboarding Workflow

Here's a comprehensive n8n workflow that triggers when a candidate is marked as "hired" in Directus:

```
Directus (status = "hired") →
  ┌─ Email (welcome email with first-day instructions)
  ├─ Directus (create employee record)
  ├─ Document Generation (offer letter, NDA, tax forms)
  ├─ Cal.com (schedule orientation sessions)
  ├─ Mattermost (create #welcome-[name] channel, notify team)
  ├─ Odoo (create HR record, assign to department)
  └─ Schedule (day-1, day-7, day-30 check-in reminders)
```

### Document Generation

Onboarding paperwork is a perfect candidate for automation. Most offer letters, NDAs, and policy acknowledgments are template documents with variable fields (name, start date, salary, role). Here's how to generate them automatically:

```python
from docx import Document
from datetime import datetime

def generate_offer_letter(candidate_data, template_path="templates/offer_letter.docx"):
    doc = Document(template_path)
    
    replacements = {
        "{{name}}": candidate_data["name"],
        "{{role}}": candidate_data["position"],
        "{{salary}}": f"${candidate_data['salary']:,}",
        "{{start_date}}": candidate_data["start_date"],
        "{{date}}": datetime.now().strftime("%B %d, %Y"),
    }
    
    for paragraph in doc.paragraphs:
        for key, value in replacements.items():
            if key in paragraph.text:
                paragraph.text = paragraph.text.replace(key, value)
    
    output_path = f"output/offer_letter_{candidate_data['name'].replace(' ', '_')}.docx"
    doc.save(output_path)
    return output_path

# Example
candidate = {
    "name": "Jane Smith",
    "position": "Operations Manager",
    "salary": 75000,
    "start_date": "October 15, 2026"
}
path = generate_offer_letter(candidate)
print(f"Offer letter generated: {path}")
```

In n8n, this runs as a function node. The generated document gets stored in Directus and emailed to the candidate for e-signature.

### The 30-60-90 Day Check-In

Onboarding doesn't end on day one. A good process includes check-ins at 30, 60, and 90 days. n8n can automate these:

- **Day 30**: Send a structured feedback form to the new hire and their manager. Use Ollama to summarize the responses and flag any concerns.
- **Day 60**: Schedule a one-on-one via Cal.com. Send the manager a summary of the 30-day feedback.
- **Day 90**: Generate a probationary review document. If both the manager and employee responses are positive, trigger the "confirm permanent role" workflow. If concerns are flagged, alert HR for a manual review.

```
Schedule (30 days after start) →
  Email (send feedback form to employee) →
  Email (send feedback form to manager) →
  Wait (7 days) →
  Ollama (summarize responses, flag concerns) →
  Directus (store review) →
  Mattermost (notify HR if concerns flagged)
```

## Measuring the Impact

How much time does this actually save? Let's break it down with realistic numbers for a company hiring 2-3 people per month:

| Task | Manual Time | Automated Time | Monthly Savings |
|------|------------|----------------|-----------------|
| Resume screening (200/month) | 16 hours | 1 hour (review only) | 15 hours |
| Interview scheduling (15/month) | 5 hours | 0.5 hours | 4.5 hours |
| Onboarding paperwork (3/month) | 6 hours | 1 hour | 5 hours |
| Check-in follow-ups | 3 hours | 0.5 hours | 2.5 hours |
| **Total** | **30 hours** | **3 hours** | **27 hours/month** |

27 hours per month — that's more than three full work days reclaimed. For a solo HR person or a business owner handling hiring themselves, that's the difference between drowning and staying afloat.

And that's before counting the qualitative improvements: faster response times to candidates (better hiring outcomes), no missed onboarding steps (better retention), and consistent processes (easier compliance).

## Getting Started: A Minimal Viable Pipeline

You don't need to build everything at once. Here's a pragmatic rollout:

**Week 1**: Install Ollama and test resume screening with 10 sample resumes. Tune the prompt until the scoring matches your judgment.

**Week 2**: Set up n8n and connect it to your email inbox. Automate resume intake — even if you just use it to sort candidates into folders, that alone saves hours.

**Week 3**: Install Cal.com and create interview slots. Replace email-based scheduling with self-serve booking.

**Week 4**: Build the onboarding workflow. Start with just the welcome email and document generation. Add the 30-60-90 day check-ins in week 5.

The beauty of this stack is that each piece works independently. You can adopt them one at a time and still see immediate benefits.

## Common Pitfalls

**Over-automating rejection emails.** Candidates talk. If every rejection is an obviously automated template, your employer brand suffers. Use Ollama to draft personalized rejection emails that reference something specific from the candidate's resume, and have a human review them before sending — or at least the ones for candidates who made it to the interview stage.

**Ignoring data retention.** Resumes contain personal data. In many jurisdictions, you're legally required to delete candidate data after a certain period (6-24 months depending on your location). Build a n8n workflow that automatically purges rejected candidate records from Directus after your retention period expires.

**Treating AI scores as gospel.** The Ollama model will occasionally score a great candidate low because their resume uses different terminology than the job description. Always have a human skim the top 20% of the "maybe" pile. The model is a filter, not an oracle.

**Forgetting the candidate experience.** Automation should make things faster for candidates too, not just for you. Automated confirmations, clear next steps, and timely updates (even automated ones) make candidates feel respected. Ghosting — even automated ghosting — damages your reputation.

## Conclusion

HR automation isn't about replacing human judgment in hiring. It's about removing the administrative weight that makes hiring painful. When your team spends 27 fewer hours per month on paperwork and scheduling, they can spend that time actually talking to candidates, evaluating culture fit, and making better hiring decisions.

The open source stack — n8n, Ollama, Cal.com, Directus, Mattermost — gives you enterprise-grade automation without sending sensitive employee data to third-party servers. You own the infrastructure. You control the data. You set the rules.

Start small. Automate resume screening first. Then scheduling. Then onboarding. Each step compounds, and within a month you'll have a hiring process that feels less like administrative drudgery and more like what HR is supposed to be: finding the right people and helping them succeed.

---

*Want help setting up an automated HR pipeline for your business? ARDOT Consulting specializes in open source AI automation for small and mid-size businesses. [Contact us](/) to schedule a free consultation — we'll map your current hiring process and show you exactly where automation can save you time.*