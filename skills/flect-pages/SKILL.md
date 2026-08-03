---
name: flect-pages
description: >-
  Flect Pages — host a static docs or marketing site built from a Markdown repo
  (Astro/Starlight) at an HTTPS hostname. Use when publishing documentation or a
  static site rather than running a container.
---

# Flect · Pages

Part of the [flect](../flect/SKILL.md) skill set. Where [flect-app](../flect-app/SKILL.md)
runs *containers*, Pages publishes a *static site* built from Markdown — no server,
no bindings, just content at a TLS hostname.

> **Maturity:** Pages is v1. A page is a first-class broker resource: declare it
> in `[page]` and deploy it with `flect page deploy`. Use this skill as the
> source of truth for how Pages resolves content.

## What it is

Point Pages at a Git repo of Markdown. The builder clones it, drops the Markdown
into an Astro + Starlight docs template, builds a static site, and serves it at
`<name>-<shortid>.up.flect.run` (or a custom domain) behind TLS.

- **Content = Markdown/MDX.** Files without frontmatter get a title from the
  filename; nested folders become nested pages.
- **Skipped files:** `AGENTS.md`, `README.md`, `CHANGELOG.md`, `LICENSE.md`,
  `CONTRIBUTING.md`, `SKILL.md` — repo meta never becomes a page.
- Built-in full-text search (Pagefind) and the Starlight sidebar/nav.

## Deploy

```toml
# flect.toml
[page]
name     = "docs"
repo     = "https://github.com/acme/docs"
ref      = "main"          # branch, tag or commit (default: main)
template = "docs"
title    = "Acme Docs"
github   = "https://github.com/acme/docs"
# domain = "docs.acme.com" # CNAME → the generated host
```

```bash
flect page deploy      # build + publish, waits for the build
flect page status      # status and build history
flect page logs        # the last build's log
flect page rollback    # serve a previous build, no rebuild
```

The site is fronted by Traefik with an auto-issued certificate, exactly like a
public app.

**Build isolation.** The clone-and-build step runs with no cluster credentials
and no access to any other page's content; a separate step publishes the result.
Each build lands in its own immutable directory and is swapped in atomically, so
a failed build never touches the live site and rollback is a pointer switch.

## Authoring tips

- One `# H1` per page or a `title:` in frontmatter; folders group the sidebar.
- Relative links between Markdown files resolve to site routes.
- Keep repo meta (`README.md`, `AGENTS.md`, `SKILL.md`) out of the nav — they're
  skipped automatically.

## Related skills

- [flect-app](../flect-app/SKILL.md) — a dynamic container instead of static content.
- [flect-store](../flect-store/SKILL.md) — raw object hosting instead of a built site.
- [flect](../flect/SKILL.md) — scopes, domains, deploy model.
