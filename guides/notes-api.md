---
title: "Example: Notes API"
---

A complete REST API built with [Hono](https://hono.dev/), using a Flect database for persistence and a Flect KV store for caching.

Source: [`examples/notes-api`](https://github.com/dotlabshq/baseworks-ts/tree/main/examples/notes-api)  
Live demo: [https://notes-yc8sx2.up.flect.run](https://notes-yc8sx2.up.flect.run)

## What it demonstrates

- Database CRUD with `@flect/sdk`
- KV cache-first pattern (list cached 60s, single item 120s)
- Cache invalidation on write
- `flect.toml` binding configuration
- Docker multi-stage build with tsup

## Setup

### 1. Create resources

```bash
flect org create --name flect-examples && flect org use flect-examples
flect ws create --name default && flect ws use default
flect proj create --name notes && flect proj use notes
flect env create --name production && flect env use production
```

### 2. Create database and KV store

```bash
flect db create --name notes-db
flect kv create --name notes-cache
```

### 3. Run migrations

```bash
flect db migrate notes-db --dir migrations
```

The migration creates the `notes` table:

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

### 4. Create the app

```bash
flect app create --name notes
```

### 5. Deploy

```bash
flect app deploy notes --image ghcr.io/dotlabshq/notes-api:0.1.4
```

## API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/notes` | List all notes (cached) |
| `POST` | `/notes` | Create a note |
| `GET` | `/notes/:id` | Get a note (cached) |
| `PATCH` | `/notes/:id` | Update a note |
| `DELETE` | `/notes/:id` | Delete a note |
| `GET` | `/healthz` | Health check |

### Create a note

```bash
curl -X POST https://notes-yc8sx2.up.flect.run/notes \
  -H 'Content-Type: application/json' \
  -d '{"title": "My first note", "body": "Hello from Flect!"}'
```

```json
{
  "note": {
    "id": "cdb564dc-7fe5-4e6b-b545-5fd534ed038f",
    "title": "My first note",
    "body": "Hello from Flect!",
    "created_at": 1782594882769,
    "updated_at": 1782594882769
  }
}
```

### List notes

```bash
curl https://notes-yc8sx2.up.flect.run/notes
```

```json
{
  "notes": [...],
  "source": "cache"
}
```

`source` is `"db"` on first load, `"cache"` on subsequent requests within 60 seconds.

## Code walkthrough

### `flect.toml`

```toml
name = "notes-api"
port = 3000

[[databases]]
binding        = "DB"
name           = "notes-db"
migrations_dir = "migrations"

[[kv]]
binding = "CACHE"
name    = "notes-cache"
```

### `src/index.ts`

```typescript
import { createEnv } from '@flect/sdk'

const env   = createEnv()
const db    = env.db('DB')      // connects via FLECT_TOKEN → flect-proxy → sqld
const cache = env.kv('CACHE')   // connects via FLECT_TOKEN → flect-proxy → Valkey

// cache-first list
app.get('/notes', async (c) => {
  const cached = await cache.getJson<Note[]>('all')
  if (cached) return c.json({ notes: cached, source: 'cache' })

  const notes = await db.query<Note>('SELECT * FROM notes ORDER BY created_at DESC')
  await cache.setJson('all', notes, { ttl: 60 })
  return c.json({ notes, source: 'db' })
})

// invalidate cache on write
app.post('/notes', async (c) => {
  // ... insert into db
  await cache.del('all')
  return c.json({ note }, 201)
})
```

### Dockerfile

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@latest --activate

COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY examples/notes-api/package.json          examples/notes-api/package.json
COPY projects/flect/packages/sdk/package.json projects/flect/packages/sdk/package.json

RUN pnpm install --frozen-lockfile

COPY projects/flect/packages/sdk/ projects/flect/packages/sdk/
COPY examples/notes-api/          examples/notes-api/

RUN pnpm --filter @flect/sdk build
RUN pnpm --filter notes-api build

FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/examples/notes-api/dist/index.cjs ./index.cjs
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "index.cjs"]
```

Key points:
- Multi-stage build keeps the final image minimal
- `tsup` with `format: ['cjs']` and `noExternal: [/.*/]` bundles everything into a single file — no `node_modules` in the final image
- `@flect/sdk` is built first, then bundled into the app
