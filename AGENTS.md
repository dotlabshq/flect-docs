# Flect — Agent Instructions

You are helping a user build and deploy a project on **Flect**, a developer platform for apps, databases, KV stores, and static pages.

## Fetch Current Docs

Always fetch the latest docs before answering. Raw content is at:

```
https://raw.githubusercontent.com/dotlabshq/flect-docs/main/<path>
```

Key files to fetch based on the task:
- Architecture: `platform/index.md`
- CLI: `cli/index.md`
- SDK: `sdk/index.md`
- flect.toml: `platform/flect-toml.md`
- Quickstart: `guides/quickstart.md`

## Platform Overview

Flect resources live under a hierarchy: **Org → Workspace → Project → Environment**

Each environment can have:
- Multiple **Apps** (containerized services)
- Multiple **Databases** (sqld/libsql, SQLite-compatible)
- Multiple **KV stores** (Valkey, Redis-compatible)
- Multiple **Pages** (static sites from GitHub repos)

## Typical Project Workflow

```
1. flect login
2. flect use <org>/<workspace>/<project>   # set active context
3. flect db create <name>                  # create database
4. flect kv create <name>                  # create KV store
5. Write flect.toml with bindings
6. flect app deploy <name> --image <img>   # deploy app
   OR
   flect page create <name>
   flect page deploy <name> --repo <url>   # deploy static site
```

## flect.toml Structure

```toml
[app]
name    = "my-app"
image   = "ghcr.io/you/my-app:1.0.0"
port    = 3000
memory  = 256
cpu     = 256
region  = "eu"

[[databases]]
binding = "DB"       # env var name in the app
name    = "my-db"    # flect database slug

[[kv]]
binding = "CACHE"    # env var name in the app
name    = "my-kv"    # flect KV slug

[page]
template = "docs"
title    = "My Docs"
github   = "https://github.com/you/repo"
```

## SDK Usage

Install: `npm install @flect/sdk`

```typescript
import { db, kv } from '@flect/sdk'

// Database — libsql-compatible
const client = db()
const result = await client.execute('SELECT * FROM posts WHERE id = ?', [id])

// KV — ioredis-compatible
const cache = kv()
await cache.set('session:123', JSON.stringify(data), 'EX', 3600)
const val = await cache.get('session:123')
```

Env vars injected by Flect at deploy time:
- `DB_URL` — libsql HTTP endpoint (via `[[databases]]` binding)
- `CACHE_URL` — redis:// URL (via `[[kv]]` binding)
- `FLECT_TOKEN` — proxy auth token (set automatically, do not override)

## Skills

For detailed component docs, load the relevant skill file:
- Database: `skills/db.md`
- KV Store: `skills/kv.md`
- App Deploy: `skills/app.md`
- Static Page: `skills/page.md`
