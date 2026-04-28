# nikita.bolotov

Personal blog & dossier built with [Hugo](https://gohugo.io/) and deployed via GitHub Actions to GitHub Pages.

## Structure

```
├── .github/workflows/deploy.yml   # Auto-deploy on push to main
├── hugo.toml                      # Hugo configuration
├── content/
│   ├── about/index.md             # About page (markdown + HTML components)
│   └── posts/                     # Blog posts (pure markdown)
│       ├── observability-as-code.md
│       ├── ci-cd-platform-at-scale.md
│       └── supply-chain-security.md
├── layouts/
│   ├── _default/
│   │   ├── baseof.html            # Base template
│   │   ├── list.html              # Post list
│   │   └── single.html            # Single post
│   ├── about/single.html          # About layout
│   ├── index.html                 # Homepage
│   ├── 404.html                   # Custom 404
│   └── partials/
│       ├── boot.html
│       ├── head.html
│       ├── nav.html
│       └── footer.html
└── static/css/style.css           # Design system
```

## Quick Start

### 1. Create repo `zyv4yk.github.io`

Create a new repository named `zyv4yk.github.io` on GitHub.

### 2. Push the code

```bash
git init
git remote add origin git@github.com:zyv4yk/zyv4yk.github.io.git
git add .
git commit -m "feat: initial blog with CV"
git push -u origin main
```

### 3. Enable GitHub Pages

Go to **Settings → Pages → Source** → select **GitHub Actions**.

The workflow will automatically build and deploy on every push to `main`.

Site goes live at: **https://blcknb.tech**

### 4. Configure custom domain (one-time)

In **Settings → Pages**, set **Custom domain** to `blcknb.tech`. Add DNS records at your registrar pointing `blcknb.tech` at GitHub Pages (A records `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`, or a CNAME to `zyv4yk.github.io`). The `static/CNAME` file is committed in the repo so GitHub Pages picks it up on every deploy. Note: the deploy workflow passes `--baseURL` from the GitHub Pages output, so canonical link and OG tags will continue to reference `zyv4yk.github.io` until the custom domain is saved in Settings and the domain check passes.

## Writing a New Post

Create a markdown file in `content/posts/`:

```bash
hugo new posts/my-new-post.md
```

Or manually create `content/posts/my-new-post.md`:

```markdown
---
title: "My New Post Title"
date: 2025-01-15
description: "A short description that appears on the homepage card."
tags: ["Kubernetes", "Helm", "CI/CD"]
---

Your markdown content here. Supports **bold**, *italic*, `code`,
code blocks, lists, blockquotes — all standard markdown.

## Subheadings Work

So do code blocks:

\```yaml
apiVersion: v1
kind: ConfigMap
\```
```

Push to `main` → GitHub Actions builds → site updates automatically.

## Local Development

```bash
# Install Hugo (macOS)
brew install hugo

# Run local server
hugo server -D

# Build static files
hugo --minify
```

## Customization

| What | Where |
|------|-------|
| Colors, fonts, spacing | `static/css/style.css` → `:root` variables |
| Site title, links, availability | `hugo.toml` → `[params]` |
| Navigation menu | `hugo.toml` → `[[menu.main]]` |
| Homepage layout | `layouts/index.html` |
| About content | `content/about/index.md` |
| Post template | `layouts/_default/single.html` |

### Toggle Availability Status

In `hugo.toml`:

```toml
[params]
  available = true   # green dot + "Available"
  available = false  # red dot + "Not Available"
```

## Tech

- **Hugo** — static site generator (Go-based, fast)
- **GitHub Actions** — automated build & deploy
- **GitHub Pages** — hosting
- **Vanilla JS** — small inline scripts only (boot loader, cursor light, tag filter, reading progress); no frameworks or bundlers
- **Custom theme** — TRON-inspired cyan-on-near-black aesthetic, no third-party theme dependency
