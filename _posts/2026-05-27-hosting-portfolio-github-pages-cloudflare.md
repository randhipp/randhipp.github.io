---
layout: post
title: "Hosting a Portfolio with Jekyll, GitHub Pages & Cloudflare DNS"
date: 2026-05-27 01:00:00 +0700
read_time: "3 min read"
description: "A complete walkthrough of how this minimalist developer portfolio is set up using Jekyll, hosted on GitHub Pages, and mapped to a custom domain via Cloudflare."
---

When building a personal portfolio, developers often face a trade-off: build a heavy React/Next.js application that requires continuous maintenance and hosting costs, or stick to a basic static HTML file that is hard to extend.

For this repository, we chose a third option: **a minimalist landing page integrated with Jekyll and hosted on GitHub Pages**. 

Here is exactly how this setup works and how it is routed through Cloudflare DNS to run on a custom domain.

---

## 🛠️ The Architecture

The project consists of three core components:
1. **GitHub Pages:** For free, secure, and ultra-fast static file hosting.
2. **Jekyll:** To dynamically generate blog posts from simple Markdown files without running a database.
3. **Cloudflare DNS:** For domain management, caching, CDN speed, and security.

---

## 🚀 Step-by-Step Setup

### 1. Repository Naming
To host a website at a root personal domain (`https://<username>.github.io`), GitHub requires a repository named exactly `<username>.github.io`. 

For this project, we initialized the public repository:
```bash
https://github.com/randhipp/randhipp.github.io
```

### 2. Jekyll Configuration
Jekyll needs a basic `_config.yml` at the root of the project to know how to process files and format URLs:

```yaml
title: Randhi Putra
email: hey@randhi.pro
permalink: /blog/:title/
markdown: kramdown
```

We created an `index.html` as the landing page and set up a custom post layout under `_layouts/post.html` that matches the dark-theme aesthetic.

### 3. Custom Domain Routing (randhi.pro)
To map the custom domain `randhi.pro` to GitHub Pages, we created a file named `CNAME` at the root of the repository containing exactly one line:
```text
randhi.pro
```

### 4. Cloudflare DNS Setup
In the Cloudflare dashboard for `randhi.pro`, we pointed the domain to GitHub's servers.

#### A Records (Apex Domain `@`)
We pointed the root domain directly to GitHub's official IP addresses:
* `185.199.108.153`
* `185.199.109.153`
* `185.199.110.153`
* `185.199.111.153`

#### CNAME Record (Subdomain `www`)
We pointed the `www` subdomain to the GitHub pages URL:
* **Name:** `www`
* **Target:** `randhipp.github.io`

---

## 🔒 Crucial SSL Settings in Cloudflare

One of the most common issues when routing GitHub Pages through Cloudflare is getting stuck in a **Redirect Loop** (`ERR_TOO_MANY_REDIRECTS`).

This happens if Cloudflare's SSL/TLS setting is set to **Flexible**. Since GitHub Pages enforces HTTPS, it will redirect any HTTP requests from Cloudflare back to HTTPS. In *Flexible* mode, Cloudflare always talks to the origin server (GitHub) over HTTP, creating an infinite loop.

**The Fix:**
In your Cloudflare dashboard under **SSL/TLS**, you must set the encryption mode to **Full** or **Full (strict)**. This ensures Cloudflare communicates with GitHub Pages over HTTPS, loading the page successfully.

---

## ✍️ Writing New Posts
Now, adding a new blog post is as simple as creating a new Markdown file in the `_posts` folder and pushing it:

```bash
git add _posts/new-post.md
git commit -m "feat: publish new post"
git push origin main
```
GitHub Pages automatically builds the site in the background, and the post is live in seconds!
