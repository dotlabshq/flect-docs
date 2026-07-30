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

> **Maturity:** Pages is v1. The build/serve pipeline (`packages/pages`) is in
> place; the first-class `flect pages` CLI + `[[pages]]` in `flect.toml` are
> still landing. Today a page is deployed by running the **pages builder** with
> the env below. Prefer this skill for docs sites; use it as the source of truth
> for how Pages resolves content.

## What it is

Point Pages at a Git repo of Markdown. The builder clones it, drops the Markdown
into an Astro + Starlight docs template, builds a static site, and serves it at
`<name>-<shortid>.up.flect.run` (or a custom domain) behind TLS.

- **Content = Markdown/MDX.** Files without frontmatter get a title from the
  filename; nested folders become nested pages.
- **Skipped files:** `AGENTS.md`, `README.md`, `CHANGELOG.md`, `LICENSE.md`,
  `CONTRIBUTING.md`, `SKILL.md` — repo meta never becomes a page.
- Built-in full-text search (Pagefind) and the Starlight sidebar/nav.

## Deploy (pages builder)

The builder image reads a small set of env vars and builds + serves the site:

| var | required | purpose |
|---|---|---|
| `PAGE_ID` | ✅ | stable id for the page (build/serve dir) |
| `REPO_URL` | ✅ | Git URL of the Markdown repo to publish |
| `PAGE_HOSTNAME` | ✅ | generated host, e.g. `docs-ab12cd.up.flect.run` |
| `PAGE_TITLE` | — | site title (default `Docs`) |
| `PAGE_GITHUB` | — | repo link shown in the header |
| `PAGE_DOMAIN` | — | custom domain (CNAME → the generated host) |

The build output is written to the serve dir and fronted by Traefik with an
auto-issued certificate, exactly like a public app.

## Authoring tips

- One `# H1` per page or a `title:` in frontmatter; folders group the sidebar.
- Relative links between Markdown files resolve to site routes.
- Keep repo meta (`README.md`, `AGENTS.md`, `SKILL.md`) out of the nav — they're
  skipped automatically.

## Related skills

- [flect-app](../flect-app/SKILL.md) — a dynamic container instead of static content.
- [flect-store](../flect-store/SKILL.md) — raw object hosting instead of a built site.
- [flect](../flect/SKILL.md) — scopes, domains, deploy model.
