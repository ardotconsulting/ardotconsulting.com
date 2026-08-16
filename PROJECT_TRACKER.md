# ARDOT Consulting — Project Tracker

> **Purpose:** Persistent coordination document for all agents and humans working on the ARDOT Consulting website and infrastructure. Updated whenever work starts, progresses, or completes.
>
> **Rule:** Before starting any work, read this file. After completing work, update it.

---

## Active Agents

| Agent | Role | Access |
|-------|------|--------|
| @ardot_automator | Infrastructure, deployment, ops automation | Repo admin, GitHub PAT (fine-grained) |
| @ardot_coder_bot | Website development, PRs, feature work | Repo access, PR creation |
| @ardot_researcher_bot | Research, strategy, content | TBD |

---

## Coordination Protocol

1. **Before starting work:** Read this file. Check "In Progress" and "Decision Log" sections.
2. **Claim work:** Add an entry to "In Progress" with your name and what you're doing.
3. **File a GitHub issue** for anything that needs tracking beyond a quick fix.
4. **Open a PR** (not direct push to main) for any code changes.
5. **After completing work:** Move your entry from "In Progress" to "Completed" and update the "Decision Log" if a decision was made.

---

## Website Status

**URL:** https://www.ardotconsulting.com
**Repo:** https://github.com/ardotconsulting/ardotconsulting.com
**Hosting:** GitHub Pages (workflow-based deployment from `main`)
**DNS:** Hetzner (hydrogen.ns.hetzner.com)
**SSL:** GitHub-managed, expires 2026-10-29

### Current State
- Single-page marketing site (dark theme, emerald accent, Inter font)
- Sections: Hero, Services (6 cards), Process (4 steps), Industries (9), About, CTA, Contact (form)
- Contact form uses FormSubmit.co → ardotconsulting@tutamail.com (no mailto fallback)
- Scroll reveal animations via IntersectionObserver
- Responsive with mobile nav toggle

### Known Issues
- See [GitHub Issues](https://github.com/ardotconsulting/ardotconsulting.com/issues)

---

## In Progress

| Who | What | Branch/Issue | Started | Status |
|-----|------|-------------|---------|--------|
| @ardot_coder_bot | Migrate site to Jekyll + add blog | feat/jekyll-blog (Issue #20) | 2026-08-02 | PR open |

---

## Completed

| Who | What | PR/Issue | Date |
|-----|------|----------|------|
| @ardot_coder_bot | Build complete landing page website | PR #1 | 2026-08-01 |
| @ardot_coder_bot | Add GitHub Pages deployment workflow | PR #2 | 2026-08-01 |
| @ardot_coder_bot | Wire up contact form (FormSubmit.co), update email, remove 24h promise | PR #4 (Issue #3) | 2026-08-01 |
| @ardot_coder_bot | Remove mailto fallback, form uses FormSubmit.co only | PR #7 (Issue #6) | 2026-08-01 |
| @ardot_coder_bot | Add open-source analytics, SEO, and search indexing | PR #16 (Issue #11) | 2026-08-02 |

---

## Decision Log

| Date | Decision | Context |
|------|----------|---------|
| 2026-08-01 | Use dark theme with emerald accent | Differentiate from generic purple/blue AI company sites (PR #1) |
| 2026-08-01 | Use FormSubmit.co for contact form | No backend on GitHub Pages; FormSubmit.co is free, no signup, forwards to email |
| 2026-08-01 | Contact email: ardotconsulting@tutamail.com | hello@ardotconsulting.com has no MX records; tutamail is the actual inbox |
| 2026-08-01 | Remove "24 hour response" promise | No mechanism to guarantee response time yet |
| 2026-08-01 | Form uses FormSubmit.co only, no mailto fallback | Mailto was opening user's email client in addition to FormSubmit submission |
| 2026-08-02 | Use Plausible for analytics (open source, no cookies) | Per Issue #11: avoid Google Analytics. Plausible is open source, privacy-first, self-hostable. Using hosted SaaS initially; will self-host via Docker when VPS is available. |
| 2026-08-02 | Use Jekyll as blog engine | Native GitHub Pages support, Markdown posts, no custom build needed. Chosen by Ryan over Astro/Eleventy/Hugo. |
| 2026-08-02 | Self-host Inter font instead of Google Fonts | User prefers avoiding Google services. Downloaded woff2 files (latin subset) served locally from /fonts/. Eliminates fonts.googleapis.com dependency. |
| 2026-08-02 | Use Bing Webmaster Tools for search indexing (not Google) | Per Issue #11: avoid Google Search Console. Sitemap submitted via robots.txt for all crawlers. Bing Webmaster Tools is Microsoft (proprietary) but is the main non-Google option for active submission. robots.txt + sitemap.xml work for all search engines regardless. |

---

## Open Questions

- [ ] Email hosting: MX records not set up for ardotconsulting.com — using tutamail.com as a workaround. Should we set up proper email at the domain?
- [ ] Sector specialization: researching which industries to target for AI automation consulting leads
- [ ] FormSubmit.co one-time confirmation: first submission triggers a confirmation email — needs to be completed once

---


## Blog Setup — Planned Work

| Issue | Title | Assignee | Dependencies | Status |
|-------|-------|----------|--------------|--------|
| #22 | Evaluate and choose a static site generator | TBD | — | Open |
| #24 | Design blog layout and styling | TBD | #22 | Open |
| #26 | Set up blog content structure, templates, RSS | TBD | #22, #24 | Open |
| #27 | Update deployment workflow for SSG build | TBD | #22, #26 | Open |
| #28 | Write first blog post (launch announcement) | TBD | #22, #24, #26, #27 | Open |
| #29 | Add blog link to main site navigation | TBD | #22, #24 | Open |

**Coordination notes:**
- @ardot_researcher_bot should draft content for #28
- @ardot_coder_bot should handle implementation (#22, #24, #26, #27, #29)
- @ardot_automator should verify deployment end-to-end (#27, #28)
- All tools must be open source — no Google dependencies
- Self-hosted Inter font must continue to work across blog pages

## Infrastructure Notes

- **Token limitation:** Fine-grained PAT can push branches and create issues, but cannot create PRs or update issues via API. PRs must be created through the GitHub web UI or by an agent with broader token scopes.
- **Deployment:** Pushes to `main` trigger `.github/workflows/deploy.yml` which deploys to GitHub Pages automatically.
- **CNAME:** `www.ardotconsulting.com` is configured in GitHub Pages settings (not via a CNAME file in the repo).

---

*Last updated: 2026-08-02 by @ardot_automator*