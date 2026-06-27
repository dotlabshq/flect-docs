---
title: Local Development
---

Run your Flect app locally while pointing at real or local resources.

## Option 1: Local services with Docker

Run sqld and Valkey locally:

```bash
# sqld (libSQL server)
docker run -d \
  --name sqld \
  -p 8080:8080 \
  ghcr.io/tursodatabase/libsql-server:latest \
  /bin/sqld --enable-namespaces

# Valkey
docker run -d \
  --name valkey \
  -p 6379:6379 \
  valkey/valkey:8-alpine \
  valkey-server --save "" --appendonly no
```

Then set env vars:

```bash
# .env.local
DB_URL=http://localhost:8080
DB_NAMESPACE=myapp
CACHE_URL=redis://localhost:6379
CACHE_PREFIX=myapp:
```

No `FLECT_TOKEN` needed — the SDK skips proxy auth when `FLECT_TOKEN` is absent.

## Option 2: Point at your production resources

You can run your app locally while talking to your real Flect database and KV store, bypassing the proxy. Get the connection details:

```bash
flect db get myapp-db --output json
flect kv get myapp-cache --output json
```

Then set:

```bash
# .env.local (direct, no proxy)
DB_URL=http://100.64.0.1:8081
DB_NAMESPACE=<sqld-namespace>
CACHE_URL=redis://100.64.0.1:6380
CACHE_PREFIX=<key-prefix>
```

> **Note:** Direct access bypasses flect-proxy. Suitable for development only — do not expose these addresses publicly.

## Running your app

With [tsx](https://github.com/privatenumber/tsx) for TypeScript:

```bash
node --env-file=.env.local --import tsx/esm src/index.ts
```

Or with the script in `package.json`:

```json
{
  "scripts": {
    "dev": "node --env-file=.env.local --import tsx/esm src/index.ts"
  }
}
```

## Migrations in development

Run migrations against your local sqld:

```bash
flect db migrate myapp-db --dir migrations
```

Or apply SQL files directly if working locally:

```bash
sqlite3 local.db < migrations/0001_create_notes.sql
```
