---
name: flect
description: >-
  Build and deploy an app plus its backing resources (database, KV, object
  storage, static pages) on the Flect platform. Use when the user wants to ship a
  service on Flect, wire up a Flect database/KV/store, write flect.toml, use
  @getflect/sdk (createEnv), or run the flect CLI (login, scope, deploy).
---

# Flect

This skill is organized as a small library under [`skills/`](./skills/). Start
with the **hub**, then lazy-load the one component skill your task needs — each
is self-contained.

1. **Hub — [`skills/flect/SKILL.md`](./skills/flect/SKILL.md)** — the model,
   scopes, `createEnv()` rules, CLI, `flect.toml` shape, plan limits. Read this
   first.
2. **Components** (load on demand):

   | When you're… | Load |
   |---|---|
   | deploying a container / writing an app's `flect.toml` | [`skills/flect-app/SKILL.md`](./skills/flect-app/SKILL.md) |
   | adding a database, queries, migrations, Drizzle | [`skills/flect-db/SKILL.md`](./skills/flect-db/SKILL.md) |
   | adding a cache, counters, sessions, rate limits | [`skills/flect-kv/SKILL.md`](./skills/flect-kv/SKILL.md) |
   | storing files, presigned URLs, S3 | [`skills/flect-store/SKILL.md`](./skills/flect-store/SKILL.md) |
   | publishing a docs / static site from Markdown | [`skills/flect-pages/SKILL.md`](./skills/flect-pages/SKILL.md) |

Raw fetch base:
`https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/<name>/SKILL.md`.

Deeper references live alongside: `cli/index.md`, `sdk/index.md`,
`platform/index.md`, `platform/flect-toml.md`.
