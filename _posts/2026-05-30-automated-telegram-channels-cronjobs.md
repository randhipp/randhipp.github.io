---
layout: post
title: "How I Built 2 Automated Telegram Channels with Hermes Agent"
date: 2026-05-30 06:20:17 +0700
categories: automation telegram cronjob hermes-agent
permalink: /blog/automated-telegram-channels-cronjobs/
---

I have been running two fully automated Telegram channels for the past few days — no manual posting, no copy-pasting links, no waking up early to hit "send." Both are powered by cronjobs running inside [Hermes Agent](https://github.com/NousResearch/hermes-agent), an open-source AI agent framework that lives on my server and talks to Telegram, GitHub, LinkedIn, and basically anything with an API.

Here is how each one works.

---

## 1. Remote Software Engineer Jobs — @RemoteSWEJobs

**Channel:** [t.me/RemoteSWEJobs](https://t.me/RemoteSWEJobs)

The idea was simple: I wanted a firehose of remote software engineering jobs that I could scroll through with my morning coffee, without wading through LinkedIn's UI or missing posts because the algorithm decided to hide them.

### What the cronjob does

- **Scrapes LinkedIn every hour** (06:00–21:00 Bangkok time) using LinkedIn's guest API
- **Filters strictly for remote jobs** with `f_WT=2` plus extra keyword checks to weed out "hybrid" or on-site listings pretending to be remote
- **Extracts tech stack** from each job description by scanning for 73 predefined keywords (Python, Go, React, Kubernetes, RAG, etc.)
- **Pulls a requirements snippet** from the "Qualifications" or "About You" section
- **Deduplicates** against a local `history.json` so the same job never gets posted twice
- **Splits the digest into two parts:**
  1. 🇮🇩 Indonesia Remote jobs (up to 20)
  2. 🌍 Global Remote jobs — sorted by a priority score: UK (5) > US (4) > Australia (3) > Japan (2) > Singapore (1) > others (0)

### How it looks

Each message is clean Markdown:

- **Job title** as a clickable LinkedIn link
- Company name in italics
- Location with country flag
- 🛠 Tech stack tags (max 6)
- 📋 A short requirements excerpt (~120 chars)

If there are fewer than 5 new jobs in a run, the agent stays silent. No spam.

### The stack

- `scraper.py` — fetches and parses LinkedIn job listings
- `digest.py` — formats the JSON output into Telegram-ready Markdown
- `linkedin-daily.sh` — wrapper that ties them together
- Hermes Agent cronjob (`0 6-22 * * *`) in LLM-driven mode so it can auto-split long messages

---

## 2. Daily GitHub Trending Repos

**Channel:** [t.me/dailygithub](https://t.me/dailygithub) — *Daily Interesting Github Repo*

GitHub's official Trending page is great, but it is limited to a single language at a time and offers zero context. I wanted a curated list of 10 repos every morning, with a paragraph explaining *why* each one matters, cross-referenced against npm download stats and PyPI presence.

### What the cronjob does

- **Scrapes GitHub Trending** (daily + weekly tabs) for candidate repos
- **Enriches each repo** via the GitHub API to get full metadata, topics, creation date, and exact star counts
- **Scores repos** based on star velocity, total stars, and recency
- **Filters out dust** with age-based heuristics:
  - >2,000 stars in <14 days → suspicious
  - >5,000 stars in <30 days → suspicious
  - >15,000 stars in <90 days → suspicious
- **Maintains exclusion lists** for already-famous repos and artificially inflated ones
- **Caps language diversity** at 3 repos per language so the list is not just 10 JavaScript projects
- **Cross-checks npm and PyPI** for package download counts when applicable
- **Outputs a clean list:** just a URL + 1-paragraph summary per repo. No intro, no fluff.

### The stack

- `github-trending-reporter.py` (v4, ~370 lines) — the main scraper, scorer, and formatter
- `github-trending.sh` — wrapper that sources a GitHub token and runs the Python script
- Hermes Agent cronjob (`0 7 * * *` server time = 06:00 Bangkok) in `no_agent` mode — the script itself produces the exact message text, zero LLM tokens burned

---

## Why Hermes Agent?

Both channels run on the same machine, managed by a single `jobs.json` file. Hermes handles:

- **Scheduling** — cron expressions with timezone-aware execution
- **Delivery** — auto-posting to Telegram channels by numeric ID
- **LLM splitting** — for the LinkedIn digest, the agent breaks long output across multiple Telegram messages while preserving Markdown
- **Silent runs** — when there is nothing new, the job outputs `[SILENT]` and nothing gets posted
- **Retries and logs** — every run is timestamped and saved to `~/.hermes/cron/output/`

The whole setup is infrastructure-as-yaml. If I want to change the schedule, add a new filter, or spin up a third channel, it is one CLI command or one config edit away.

---

## The Channels

If you are a software engineer looking for remote work or just want a curated daily dose of interesting open-source projects:

- **🇮🇩🌍 Remote SWE Jobs:** [t.me/RemoteSWEJobs](https://t.me/RemoteSWEJobs)
- **🔥 Daily Interesting Github Repo:** [t.me/dailygithub](https://t.me/dailygithub)

Both are 100% automated, update daily, and will keep running as long as the server has power and the APIs stay open.

---

*Built with Hermes Agent, Python, curl, and too much coffee.*
