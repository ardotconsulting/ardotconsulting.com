# PR Review Notes — Please Fix Before Merge

Reviewed by @ardot_automator on 2026-08-02.

## Must Fix

1. **`addressCountry: "Remote-first"` is invalid** — Schema.org expects a 2-letter ISO 3166-1 code. Remove the `address` block entirely (business is remote-first) or use `"US"`.

2. **Missing `og:image` / `twitter:image` tags** — social sharing cards will be text-only. Add an OG image and meta tags:
   ```html
   <meta property="og:image" content="https://www.ardotconsulting.com/og-image.png">
   <meta name="twitter:image" content="https://www.ardotconsulting.com/og-image.png">
   ```

3. **Missing trailing newlines** in `robots.txt` and `sitemap.xml` — add `
` at end of both files.

4. **Plausible `data-domain` mismatch** — `data-domain="ardotconsulting.com"` but canonical URL is `www.ardotconsulting.com`. Plausible treats these as different domains. Either:
   - Change to `data-domain="www.ardotconsulting.com"` to match canonical, or
   - Configure subdomain tracking in the Plausible dashboard for `ardotconsulting.com`

5. **Plausible Cloud needs an account** — script loads from `plausible.io` (paid SaaS, not self-hosted). A Plausible account must be created and the domain registered before analytics will collect data. The TODO comment about self-hosting via Docker is good — document this as a setup requirement.

## Non-Blocking (Optional)

- **Google Fonts** still loaded from `fonts.googleapis.com` — user prefers avoiding Google services. Consider self-hosting the Inter font (download woff2 files, serve locally).
- **Decision log says "Bing Webmaster Tools"** — that's Microsoft, not open source. The robots.txt/sitemap.xml approach is correct; just update the log wording to not mention Bing.
- **`<meta name="keywords">`** — Google has ignored this since 2009. Harmless but provides no SEO benefit.

## Verdict

Approve with minor fixes. Core approach is sound — Plausible for analytics, robots.txt + sitemap.xml for indexing, JSON-LD for structured data.

---

*Delete this file after addressing the notes.*
