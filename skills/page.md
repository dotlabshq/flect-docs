# Flect Page Skill

## What It Is

Flect Pages deploy static sites from a GitHub repo. Point it at a repo containing markdown files and Flect builds a Starlight docs site automatically. Supports custom Astro sites too.

Hosted at `<name>-<id>.up.flect.run` with HTTPS.

## Workflow

```
1. Create a GitHub repo with markdown files
2. Add flect.toml with [page] section (optional, has defaults)
3. flect page create <name>
4. flect page deploy <name> --repo github.com/<owner>/<repo>
```

## Quick Start

```bash
# 1. Create page record
flect page create my-docs

# 2. Deploy from GitHub repo
flect page deploy my-docs --repo github.com/you/my-docs

# 3. Check status
flect page list
```

## Repo Structure

Simplest possible setup — just markdown files:

```
my-docs/
  index.md          # home page
  guides/
    quickstart.md
    installation.md
  reference/
    api.md
  flect.toml        # optional config
```

## flect.toml `[page]` Section

```toml
[page]
template = "docs"                              # only template currently available
title    = "My Project"                        # site title
github   = "https://github.com/you/repo"      # shown in navbar

[page.build]
# Optional: use your own build system instead of the default template
command = "npm run build"
out     = "dist"
```

## Templates

### `docs` (default)

Astro Starlight — auto-generates sidebar from your directory structure.

- Drop any `.md` or `.mdx` files in the repo root
- Subdirectories become sidebar sections
- `index.md` becomes the home page

### Custom Astro Site

If your repo has `astro.config.mjs`, Flect runs your own build:

```toml
[page.build]
command = "npm run build"
out     = "dist"
```

## Markdown Tips

### Frontmatter

```md
---
title: Getting Started
description: Install and configure the project.
---

# Getting Started

Content here...
```

### Sidebar Order

Use numeric prefixes for ordering:

```
01-introduction.md
02-installation.md
03-configuration.md
```

## Management

```bash
flect page list                   # list all pages
flect page deploy <name> --repo github.com/you/repo   # redeploy (picks up new commits)
flect page delete <name>
```

## Custom Domain

```bash
flect page domain add <name> docs.mycompany.com
# Add CNAME: docs.mycompany.com → <name>-<id>.up.flect.run
```

## Pages vs Apps

| | Page | App |
|--|------|-----|
| Source | GitHub repo (markdown) | Docker image |
| Runtime | Static files via Caddy | Node.js / any |
| DB / KV | ✗ | ✓ |
| Build | Astro (auto) | Your Dockerfile |
| Use case | Docs, marketing sites | APIs, web apps |
