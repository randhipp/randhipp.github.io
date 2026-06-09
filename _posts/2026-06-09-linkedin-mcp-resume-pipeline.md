---
layout: post
title: "How I Used Hermes Agent + LinkedIn MCP to Audit My Profile and Generate a Resume"
description: "I connected Hermes Agent to a LinkedIn MCP server, audited my entire profile programmatically, rewrote my headline and experience, then generated a single-page A4 PDF resume — all from the terminal."
date: 2026-06-09 22:02:18 +0700
categories: automation hermes-agent linkedin mcp resume
permalink: /blog/linkedin-mcp-resume-pipeline/
---

I had been putting off updating my LinkedIn profile for months. The headline was vague. The experience section read like a pile of HR buzzwords. And my resume? An outdated two-pager I hadn't touched since 2024.

Instead of spending an afternoon clicking around linkedin.com and fiddling with margins in Google Docs, I decided to use the tools I already had: Hermes Agent, a LinkedIn MCP server running on my Mac, and headless Chromium on my server. Here's the full end-to-end pipeline.

---

## The Setup: LinkedIn MCP Server Over Streamable HTTP

The LinkedIn scraper I use — [linkedin-scraper-mcp](https://github.com/nicedouble/linkedin-scraper-mcp) (v3.4.2) — is an MCP server that wraps LinkedIn's internal API and exposes it as a set of tools: `get_my_profile`, `search_jobs`, `search_people`, `get_feed`, and about 15 others.

I run it on my Mac (because it needs a logged-in Chromium session with LinkedIn cookies) and expose it to the internet via Cloudflare at `https://tunnel.randhi.pro/mcp`. This was the first trick: the server uses **streamable HTTP transport**, not stdio, so it needs a real web endpoint — not just a local process.

On the Hermes side, I added this to `config.yaml`:

```yaml
mcp_servers:
  linkedin:
    url: "https://tunnel.randhi.pro/mcp"
    timeout: 120
    connect_timeout: 60
```

No `command`, no `args` — just a URL. Hermes talks to it using the standard MCP JSON‑RPC protocol over HTTP, with a session ID captured from the `initialize` handshake.

This means I can now call `get_my_profile` or `search_jobs` from *anywhere* Hermes runs — my server, Telegram, the CLI — and it routes through my Mac as the authenticated client. No API keys, no scraping proxies, no LinkedIn API quota limits.

---

## Step 1: Profile Audit

I asked Hermes to pull my full LinkedIn profile:

```
get_my_profile(sections=about,experience,education,skills)
```

It returned everything: headline, about section, all experience entries with descriptions, education, skills, certifications, even my banner image status. In seconds I had a structured dump of my entire professional presence.

Then I asked Hermes to audit it against my target: roles in **AI Fullstack / Product Engineering**. Here's what it found:

1. **Headline was too generic** — `Lead Software Engineer at Blooming Surveys` said nothing about my actual stack or focus areas.
2. **Experience descriptions were pure HR-speak** — `"Led cross-functional engineering initiatives driving digital transformation"` is what you write when you forget what you actually built.
3. **No skill endorsements** — the skills were there, but none had endorsements from colleagues.
4. **Stale certifications** — Alibaba Cloud certs from 2022 that I hadn't renewed.
5. **Education told an incomplete story** — two one-year stints with no context about what I studied or why.

I also got a list of quick wins: banner image, "Open to Work" settings, pronunciation recording, portfolio links.

---

## Step 2: Rewriting the Headline and Experience

I gave Hermes the audit report and asked for rewrites.

For the **headline**, it generated five options ranging from technical-specialist to founder-flavored. I picked:

> Lead Software Engineer | Fullstack, AI Products & Cloud | TypeScript · React · Go · GCP

It's dense, keyword-searchable, and tells a recruiter in one line exactly what I do.

For the **experience section**, Hermes rewrote my Thought&Function entry with concrete bullet points — real projects, real stacks, placeholders for metrics I need to fill in from production data:

- Built a checkout flow and automated order processing pipeline for Ted's Health, [reducing drop-off by X%]
- Designed a multi-tenant NestJS + Zenstack backend for Little Journey with admin dashboard and Sanity CMS integration
- Built an event‑driven GraphQL notification system for Equiyd [improving transaction success rate from X% to Y%]

The bracketed placeholders are deliberate — I need to pull actual numbers from production before pasting these into LinkedIn. But the structure, the storytelling, and the technical depth are already there.

> **Note:** The LinkedIn MCP tools are read‑only for profiles. You can `get_my_profile` but there's no `update_headline` or `edit_experience` tool. So all profile edits still happen manually on linkedin.com — the MCP just gave me the data and the agent drafted the text.

---

## Step 3: Generating a Single-Page Resume PDF

This was the fun part. I wanted a one‑page A4 PDF resume tailored for AI/Product Engineering roles — clean typography, dark navy accents, two‑column layout with experience on the left and skills + education on the right.

### The HTML approach

Hermes wrote a self‑contained HTML file with embedded CSS (`@page` rules, print‑only media, Inter and System UI font stacks). The layout uses CSS flexbox with a 1.6:1 ratio between the experience column and the sidebar.

Key CSS decisions for fitting one page:
- **7–8pt body text** — small but still readable in print
- **14mm margins** all around (A4 print area is 182×269mm)
- **1.4 line-height** — tight but not cramped
- **6pt spacing** between sections instead of the usual 12–16pt

```css
@page {
  size: A4;
  margin: 14mm 12mm 12mm 12mm;
}
```

### Headless Chromium rendering

The server is headless Linux — no display, no GUI. I used Chromium (installed alongside the LinkedIn scraper) with Xvfb for the virtual framebuffer:

```bash
xvfb-run -a "$CHROME" \
  --headless --disable-gpu --no-sandbox \
  --print-to-pdf=/home/ubuntu/randhi-putra-resume.pdf \
  --no-pdf-header-footer \
  /home/ubuntu/randhi-resume.html
```

### The header/footer gotcha

My first PDF had `6/9/26, 10:57 PM` stamped at the top and `file:///home/ubuntu/randhi-resume.html 1/1` at the bottom. That's Chrome's default print header/footer.

I tried `--print-to-pdf-no-header` first — it only suppressed the top header, not the footer. The fix was `--no-pdf-header-footer`, which kills both. File size dropped from 76K to 68K, and the PDF came out clean.

The final PDF has:
- Dark navy (`#1e3a5f`) accents on section titles, company names, and bullet symbols
- A footer line: `Randhi Putra · randhi.pro · hey@randhi.pro`
- No Chrome‑generated timestamps, titles, or file paths
- All content on a single A4 page

---

## The Full Pipeline in One Flow

```
[My Mac]                          [Hermes Server]
linkedin-scraper-mcp              Hermes Agent
  │                                  │
  │  Cloudflare tunnel               │
  │  tunnel.randhi.pro/mcp ←────┤ MCP config → url transport
  │                                  │
  │  get_my_profile ────────────────→│ Agent audits profile
  │                                  │ Agent rewrites headline/experience
  │                                  │ Agent writes resume HTML
  │                                  │ Agent calls headless Chromium + Xvfb
  │                                  │ PDF generated (1-page A4, 68KB)
  │                                  │
  └── Profile edits pasted           │
      manually on linkedin.com       ↓
                                 randhi-putra-resume.pdf
                                 (scp → ~/Downloads)
```

No resume builders. No Canva. No Google Docs margin battles. Just a terminal, an HTML file, and Chromium.

---

## What I'd Do Differently Next Time

1. **Profile writes via MCP** — if `linkedin-scraper-mcp` ever adds `update_profile` or `edit_headline` tools, the entire pipeline becomes fully automated. Until then, manual paste.
2. **Metrics from production data** — the bracketed placeholders need real numbers. Next iteration: pull GA4, Stripe, and Sentry data into the same session so the agent can fill them in.
3. **Multiple resume variants** — same profile data, different HTML templates for different roles (engineering manager vs IC, startup vs enterprise).

The whole process — audit, rewrite, resume generation — took about 45 minutes of agent interaction. Most of that was me reviewing drafts and picking between options. The actual tool calls and rendering took under 30 seconds.

If you want to try this yourself: install Hermes Agent, set up the LinkedIn MCP server on a machine with an active LinkedIn session, expose it over HTTP, and point your Hermes config at it. The resume HTML and CSS approach works with any headless Chromium — just watch out for that `--no-pdf-header-footer` flag.
