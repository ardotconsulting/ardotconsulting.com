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
| _(empty)_ | | | | |

---

## Completed

| Who | What | PR/Issue | Date |
|-----|------|----------|------|
| @ardot_coder_bot | Build complete landing page website | PR #1 | 2026-08-01 |
| @ardot_coder_bot | Add GitHub Pages deployment workflow | PR #2 | 2026-08-01 |
| @ardot_coder_bot | Wire up contact form (FormSubmit.co), update email, remove 24h promise | PR #4 (Issue #3) | 2026-08-01 |
| @ardot_coder_bot | Remove mailto fallback, form uses FormSubmit.co only | PR #7 (Issue #6) | 2026-08-01 |

---

## Decision Log

| Date | Decision | Context |
|------|----------|---------|
| 2026-08-01 | Use dark theme with emerald accent | Differentiate from generic purple/blue AI company sites (PR #1) |
| 2026-08-01 | Use FormSubmit.co for contact form | No backend on GitHub Pages; FormSubmit.co is free, no signup, forwards to email |
| 2026-08-01 | Contact email: ardotconsulting@tutamail.com | hello@ardotconsulting.com has no MX records; tutamail is the actual inbox |
| 2026-08-01 | Remove "24 hour response" promise | No mechanism to guarantee response time yet |
| 2026-08-01 | Form uses FormSubmit.co only, no mailto fallback | Mailto was opening user's email client in addition to FormSubmit submission |

---

## Open Questions

- [ ] Email hosting: MX records not set up for ardotconsulting.com — using tutamail.com as a workaround. Should we set up proper email at the domain?
- [ ] Sector specialization: researching which industries to target for AI automation consulting leads
- [ ] FormSubmit.co one-time confirmation: first submission triggers a confirmation email — needs to be completed once

---

## Infrastructure Notes

- **Token limitation:** Fine-grained PAT can push branches and create issues, but cannot create PRs or update issues via API. PRs must be created through the GitHub web UI or by an agent with broader token scopes.
- **Deployment:** Pushes to `main` trigger `.github/workflows/deploy.yml` which deploys to GitHub Pages automatically.
- **CNAME:** `www.ardotconsulting.com` is configured in GitHub Pages settings (not via a CNAME file in the repo).

---

*Last updated: 2026-08-01 by @ardot_automator*