---
title: SDK Reference
---

`@flect/sdk` is the TypeScript client library for accessing Flect-managed databases and KV stores from your application.

## Installation

```bash
npm install @flect/sdk
```

## Setup

The SDK reads configuration from environment variables that are automatically injected at deploy time. No manual configuration needed in production.

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()
```

For local development, set the env vars yourself (see [Local Development](../guides/local-dev.md)).

---

## Database

Get a database client for a binding defined in `flect.toml`:

```typescript
const db = env.db('DB')
```

This reads:
- `DB_URL` — sqld endpoint (via flect-proxy in production)
- `DB_NAMESPACE` — the isolated namespace for this database
- `FLECT_TOKEN` — proxy auth token (injected automatically at deploy)

### `db.query<T>(sql, params?)`

Execute a SELECT query and return typed rows.

```typescript
const users = await db.query<{ id: string; name: string }>(
  'SELECT id, name FROM users WHERE active = ?',
  [true]
)
```

Parameters are passed as an array. Supported types: `string`, `number`, `boolean`, `null`.

### `db.execute(sql, params?)`

Execute an INSERT, UPDATE, or DELETE statement.

```typescript
const { rowsAffected } = await db.execute(
  'INSERT INTO users (id, name) VALUES (?, ?)',
  [crypto.randomUUID(), 'Alice']
)
```

### `db.batch(statements)`

Execute multiple statements atomically.

```typescript
await db.batch([
  { sql: 'INSERT INTO posts (id, title) VALUES (?, ?)', params: [id, title] },
  { sql: 'UPDATE users SET post_count = post_count + 1 WHERE id = ?', params: [userId] },
])
```

---

## KV Store

Get a KV client for a binding defined in `flect.toml`:

```typescript
const cache = env.kv('CACHE')
```

This reads:
- `CACHE_URL` — Valkey endpoint with auth credentials (via flect-proxy)
- `FLECT_TOKEN` — proxy auth token

All keys are automatically namespaced — you cannot accidentally read or write another app's keys.

### `kv.get(key)`

```typescript
const value = await cache.get('session:abc123')
// → string | null
```

### `kv.getJson<T>(key)`

```typescript
const user = await cache.getJson<User>('user:abc123')
// → T | null
```

### `kv.set(key, value, options?)`

```typescript
await cache.set('session:abc123', 'active')
await cache.set('session:abc123', 'active', { ttl: 3600 }) // expires in 1h
```

### `kv.setJson(key, value, options?)`

```typescript
await cache.setJson('user:abc123', { id: 'abc123', name: 'Alice' }, { ttl: 60 })
```

### `kv.del(key)`

```typescript
await cache.del('session:abc123')
```

### `kv.exists(key)`

```typescript
const exists = await cache.exists('session:abc123')
// → boolean
```

### `kv.keys(pattern?)`

```typescript
const keys = await cache.keys('session:*')
// → string[]
```

Pattern matching uses Redis glob syntax. Keys are returned without the internal namespace prefix.

---

## Local Development

In production, all env vars are injected automatically. For local development, you need to provide them manually.

If you have a Flect account, you can point directly at the shared sqld and Valkey instances for development:

```bash
# .env.local
DB_URL=http://100.64.0.1:8081
DB_NAMESPACE=<your-namespace>
CACHE_URL=redis://100.64.0.1:6380
CACHE_PREFIX=<your-prefix>
```

Or run sqld and Valkey locally with Docker:

```bash
docker run -d -p 8080:8080 ghcr.io/tursodatabase/libsql-server:latest
docker run -d -p 6379:6379 valkey/valkey:8-alpine
```

```bash
DB_URL=http://localhost:8080
CACHE_URL=redis://localhost:6379
```

> Note: `FLECT_TOKEN` is not required in local development. The SDK skips proxy auth when the URL points directly to sqld or Valkey.
