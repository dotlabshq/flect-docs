---
title: "Example: Notes API"
description: A complete Hono REST API using a Flect database and a Flect KV cache.
sidebar:
  order: 3
---

A small REST API built with [Hono](https://hono.dev/), backed by a Flect
database for persistence and a Flect KV store for caching. It shows the whole
loop: declare bindings, write `createEnv()` code, deploy.

## What it demonstrates

- Database CRUD with the official `@libsql/client` from `env.db()`
- A cache-first read pattern with `ioredis` from `env.kv()`
- Cache invalidation on writes
- `flect.toml` binding configuration
- A single-file Docker image

## 1. Declare resources in `flect.toml`

```toml
name    = "notes-api"
runtime = "node"
port    = 3000

[[databases]]
binding        = "DB"
name           = "notes-db"
migrations_dir = "migrations"

[[kv]]
binding = "CACHE"
name    = "notes-cache"
```

The `notes-db` and `notes-cache` resources are created automatically on
`flect deploy` if they don't exist yet (or create them ahead of time with
`flect db create notes-db` / `flect kv create notes-cache`).

## 2. The schema

```sql
-- migrations/0001_create_notes.sql
CREATE TABLE IF NOT EXISTS notes (
  id         TEXT PRIMARY KEY,
  title      TEXT NOT NULL,
  body       TEXT NOT NULL DEFAULT '',
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
```

## 3. The app

```typescript
import { Hono } from 'hono'
import { createEnv } from '@flect/sdk'

const env   = createEnv()
const db    = await env.db('DB')      // official @libsql/client
const cache = await env.kv('CACHE')   // official ioredis

const app = new Hono()

interface Note {
  id: string; title: string; body: string
  created_at: number; updated_at: number
}

// cache-first list
app.get('/notes', async (c) => {
  const cached = await cache.get('all')
  if (cached) return c.json({ notes: JSON.parse(cached), source: 'cache' })

  const { rows } = await db.execute('SELECT * FROM notes ORDER BY created_at DESC')
  await cache.set('all', JSON.stringify(rows), 'EX', 60)   // cache for 60s
  return c.json({ notes: rows, source: 'db' })
})

// create + invalidate cache
app.post('/notes', async (c) => {
  const { title, body = '' } = await c.req.json()
  const now = Date.now()
  const note: Note = { id: crypto.randomUUID(), title, body, created_at: now, updated_at: now }

  await db.execute({
    sql: 'INSERT INTO notes (id, title, body, created_at, updated_at) VALUES (?, ?, ?, ?, ?)',
    args: [note.id, note.title, note.body, note.created_at, note.updated_at],
  })
  await cache.del('all')
  return c.json({ note }, 201)
})

app.get('/healthz', (c) => c.json({ ok: true }))

export default app
```

`db.execute()` and `cache.get/set/del` are the **real** libsql and ioredis
APIs — Flect doesn't wrap them. Anything those libraries (or `drizzle-orm/libsql`)
support works unchanged.

## 4. Deploy

```bash
docker build -t ghcr.io/you/notes-api:1.0.0 .
docker push ghcr.io/you/notes-api:1.0.0

flect deploy
```

```
✓ provisioned  database  notes-db-a1b2c3d4
✓ provisioned  kv        notes-cache-e5f6g7h8
✓ deployed     notes-api https://notes-api-3f9k2a.up.flect.run   running
```

## API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/notes` | List all notes (cached 60s) |
| `POST` | `/notes` | Create a note (invalidates cache) |
| `GET` | `/healthz` | Health check |

```bash
curl -X POST https://notes-api-3f9k2a.up.flect.run/notes \
  -H 'Content-Type: application/json' \
  -d '{"title":"My first note","body":"Hello from Flect!"}'
```

The `source` field on `GET /notes` is `"db"` on first load and `"cache"` on
subsequent requests within 60 seconds.

## Dockerfile

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build          # bundle to dist/index.js (e.g. with tsup/esbuild)

FROM node:22-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/index.js"]
```
