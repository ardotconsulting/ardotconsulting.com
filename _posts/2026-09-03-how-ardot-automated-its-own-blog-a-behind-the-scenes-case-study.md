---
layout: post
title: "How ARDOT Automated Its Own Blog: A Behind-the-Scenes Case Study"
date: 2026-09-03
author: "ARDOT Consulting"
tags: [case-study, blog, automation, meta, jekyll, github-pages]
excerpt: "We eat our own dog food. Here's exactly how ARDOT Consulting uses AI agents, a cron job, and Jekyll to write and publish a blog post every two days — with no human in the loop."
---

## How ARDOT Automated Its Own Blog: A Behind-the-Scenes Case Study

We tell our clients that AI automation can handle repetitive, rules-based work. So we held ourselves to the same standard: if automation is as good as we say it is, it should be able to run our own blog.

This is the story of how we built that system — what it does, what it doesn't do, and what we learned along the way. No spin. If something breaks, we'll tell you.

---

## The Problem

Small consulting firms have a content problem. You know you should publish regularly — it drives inbound leads, establishes expertise, and helps SEO. But writing takes time. A good blog post takes 3–5 hours: research, drafting, editing, formatting, publishing. Multiply that by a biweekly cadence and you're spending 10+ hours a month on content, assuming nothing gets pushed back when client work gets busy.

Which it always does.

We'd tried the usual approaches: assigning posts to team members, hiring freelance writers, using content calendars that everyone ignores. The result was the same — a burst of posts, then silence for six weeks, then guilt, then another burst.

We needed consistency. Not brilliance every time, just a steady drumbeat of useful, honest content published on a predictable schedule. That's a workflow problem, not a creative problem. And workflow problems are exactly what automation is good at.

---

## The Architecture

Here's what the system looks like end to end:

```
Cron Job (every 2 days)
    │
    ▼
AI Agent (ardot-coder)
    ├── git pull latest from main
    ├── read EDITORIAL_CALENDAR.md for next topic
    ├── read GitHub issue for post details (title, tags, description)
    ├── check _posts/ to avoid duplicates
    │
    ├── WRITE the blog post (Markdown + front matter)
    │
    ├── git add, commit, push to main
    │
    ▼
GitHub Actions (auto-triggered on push)
    ├── build Jekyll site
    ├── deploy to GitHub Pages
    │
    ▼
Live blog post at ardotconsulting.com
    │
    ▼
Agent comments on the GitHub issue with the published URL
```

The entire pipeline has **zero human intervention**. The cron job fires, the agent does its work, the post goes live. We review posts after publication and edit if needed — but the publishing happens whether we're watching or not.

---

## The Pieces

### 1. Jekyll on GitHub Pages

Our blog is a static site built with [Jekyll](https://jekyllrb.com/) and hosted on GitHub Pages. We chose this for three reasons:

- **It's free.** GitHub Pages costs nothing for public repositories.
- **It's version-controlled.** Every post is a file in a Git repository. You can see the full history of every edit, revert mistakes, and branch experimental content.
- **It's simple.** A blog post is just a Markdown file in the `_posts/` directory. No database, no admin panel, no CMS login to manage. The filename format is `YYYY-MM-DD-title-with-dashes.md`, and the file starts with YAML front matter:

```yaml
---
layout: post
title: "Your Post Title"
date: 2026-09-03
author: "ARDOT Consulting"
tags: [example, markdown, jekyll]
excerpt: "A one-line summary for SEO and post listings."
---

Your post content in Markdown here...
```

That's it. If you can write a text file, you can publish a post.

### 2. The Editorial Calendar

Automation needs a plan to execute against. We created an `EDITORIAL_CALENDAR.md` file in the repository that lists 15 upcoming posts with their target dates, content pillar, tags, and a paragraph-length description of what each post should cover.

The calendar is the single source of truth. When the cron job fires, the agent reads the calendar, finds the next topic that doesn't have a corresponding file in `_posts/`, and writes it. If we want to change what gets written, we edit the calendar file. The agent adapts.

We also created individual GitHub issues for each post (#39 through #53), each containing the post details, SEO keywords, and requirements. The agent reads the relevant issue before writing, which gives it more context than the calendar alone.

### 3. The Cron Job

A scheduled job runs every two days. When it fires, it activates an AI agent — specifically, an instance of [Hermes Agent](https://hermes-agent.nousresearch.com/) configured as `ardot-coder`, our implementation lead agent.

The agent receives a detailed prompt that tells it:

1. Pull the latest code from `main`
2. Read the existing posts to understand tone and avoid duplicates
3. Check GitHub issue #21 for the content strategy
4. Pick the next unwritten topic from the editorial calendar
5. Write the post (1000–2000 words, practical tone, for business owners)
6. Commit and push to `main`
7. Comment on the relevant GitHub issue with the published URL
8. Report back with the title, URL, and next suggested topic

The agent has tools: it can read and write files, run shell commands, search the web, and make API calls to GitHub. It uses these to do real work — not to generate text in a vacuum, but to interact with the actual repository.

### 4. Agent Coordination via Kanban

ARDOT runs three AI agents:

- **ardot-coder** — writes code and blog posts, opens PRs
- **ardot-automator** — handles infrastructure, CI/CD, deploys
- **ardot-researcher** — does research, reviews posts for accuracy

They coordinate through a Kanban board. When `ardot-coder` publishes a post, it can create a task for `ardot-researcher` to review it for factual accuracy. When infrastructure changes are needed (like setting up the GitHub Actions workflow), `ardot-coder` creates a task for `ardot-automator`.

This isn't a chat room where agents talk to each other. It's a task board where work gets assigned, picked up, and completed — the same way a human team would coordinate, just faster and without standup meetings.

### 5. GitHub Actions for Auto-Deploy

When a commit lands on `main`, a GitHub Actions workflow builds the Jekyll site and deploys it to GitHub Pages. This is a standard GitHub Pages workflow — nothing custom. The post is live within 1–2 minutes of the push.

The key insight: the agent doesn't need to know how to deploy. It just pushes to `main`, and the existing CI/CD pipeline handles the rest. Separation of concerns works for agents too.

---

## What Works Well

**Consistency.** We've published on schedule for three weeks straight. That's longer than any human-driven effort lasted. The cron job doesn't get busy, doesn't get sick, doesn't decide that this week's post can wait.

**Speed.** From cron trigger to published post, the whole process takes 5–10 minutes. A human would spend hours. The agent reads the calendar, reads past posts for tone consistency, writes 1500 words, formats the front matter, commits, and pushes — all in a single session.

**Version control as safety net.** Every post is a Git commit. If the agent writes something bad, we can revert it in seconds. If we want to edit a published post, we make a pull request like any other code change. The history is permanent and auditable.

**Cost.** The marginal cost of a post is roughly the compute time for the agent (minutes) plus GitHub Pages (free). We're not paying freelance writers $200–500 per post. We're not paying for a CMS. We're not paying for a hosting platform.

---

## What Doesn't Work Well (Yet)

**Quality variance.** Some posts come out sharp and practical. Others are a bit generic. The agent follows the style guide, but it doesn't have the judgment of a human writer who knows when to push back on a topic or restructure an argument. We've found that about 1 in 5 posts needs a meaningful edit after publication — usually tightening the introduction or adding a more specific example.

**No original reporting.** The agent can't call a client for a quote or conduct an interview. Our case studies are realistic but composite, drawn from the kinds of problems we see across clients rather than a single named engagement. Real case studies with real clients still need a human.

**Topic drift.** The editorial calendar keeps things on track, but left to its own devices, the agent gravitates toward "listicle" structures (5 mistakes, 7 tools, 3 reasons). We have to deliberately design calendar entries that break that pattern — comparison posts, deep dives, narrative case studies.

**Front matter mistakes.** Early on, the agent occasionally mismatched the date in the filename vs. the front matter, or used the wrong excerpt format. We've mostly fixed this through clearer instructions in the prompt, but it's the kind of small format error that a human would never make.

---

## The Honest Assessment

Is this blog as good as one written entirely by humans? No. A great human writer brings editorial judgment, original research, and a voice that AI doesn't quite replicate. If we had a dedicated content team, the posts would probably be better.

But we don't have a dedicated content team. We're a small consulting firm. Without automation, this blog wouldn't exist at all. Zero posts is worse than good-enough posts published consistently.

The system buys us a baseline. We get useful, accurate, on-brand content every two days without lifting a finger. When we have time, we improve individual posts — better examples, deeper research, a stronger opening. The automation handles the 80% that gets us published. We handle the 20% that makes it great.

That's the right division of labor: machines do the repetitive work, humans do the judgment work. It's the same thing we recommend to our clients.

---

## What We'd Do Differently

If we were starting from scratch, here's what we'd change:

1. **More specific calendar entries.** Vague descriptions ("write about AI for accounting") produce vague posts. The more specific the calendar entry — with concrete examples, target audience, and key points — the better the output.

2. **A review step before publishing.** Right now, posts go live immediately on push. A better system would open a pull request, run an automated review (maybe a second agent checking for factual claims), and merge only after approval. We're working on this.

3. **Analytics feedback.** We self-host [Plausible Analytics](https://plausible.io/) for privacy-friendly metrics (see our [setup guide](https://www.ardotconsulting.com/blog/2026/08/26/self-hosting-plausible-analytics-a-step-by-step-docker-guide/)). Eventually, we'd like the agent to see which posts perform well and adjust future topics accordingly. Right now, topic selection is static — driven by the calendar, not by data.

4. **Human editing workflow.** A lightweight process where a human reviews each post within 24 hours of publication and submits edits as a follow-up commit. This would catch quality issues without blocking the publishing cadence.

---

## The Takeaway

You don't need a sophisticated AI system to automate content. You need:

- A **static site generator** (Jekyll, Hugo, Eleventy — all open source)
- A **content plan** (a Markdown file listing what to write and when)
- A **scheduled job** that triggers an AI agent
- A **deployment pipeline** (GitHub Pages, Netlify, or your own server)

That's four pieces, all open source, all free or nearly free. The hard part isn't the technology — it's defining the workflow clearly enough that an agent can execute it reliably. That's true for blog automation, and it's true for every other automation project we've done.

If you're spending time on repetitive content work — reports, summaries, documentation, customer communications — the same pattern applies. Define the workflow, automate the execution, keep humans in the judgment loop.

---

## Want This for Your Business?

We build automation systems like this for small businesses every day. If you have a repetitive workflow that's eating your team's time — whether it's content, reporting, data entry, or customer communication — we can help you design and implement an automation pipeline that runs on open source tools and your own infrastructure.

[Get in touch](/#contact) and tell us what's taking up your time. We'll tell you honestly whether it's a good fit for automation — and if it is, we'll show you exactly how we'd build it.