---
title: SDK Reference
description: createEnv() and the resource clients — the entire application-facing API.
---

`@flect/sdk` is the TypeScript client for accessing Flect-managed resources.
Its whole surface is one function and three accessors. It hands you the
**official client** for each resource — a real `@libsql/client`, `ioredis`, or
`@aws-sdk/client-s3` instance — already pointed at the right endpoint and scoped
to your project. There is no Flect-specific query wrapper to learn.

```bash
npm install @flect/sdk
```

## `createEnv(options?)`

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()
```

`createEnv()` is I/O-free — it does no network calls until you resolve a
binding. By default it reads:

- `FLECT_BROKER_URL` — the broker to resolve bindings through
- `FLECT_TOKEN` — the runtime token (both injected automatically on deploy)

When no broker URL is configured, it switches to **local mode** and resolves
from `flect.local.json` (see [Local development](/guides/local-dev/)).

### Options

| Option | Description |
|--------|-------------|
| `brokerUrl` | Broker base URL. Default: `FLECT_BROKER_URL`, else `http://localhost:8787`. |
| `authToken` | Runtime token. Default: `FLECT_TOKEN`. |
| `local` | `true` to force local mode (`flect.local.json` / `FLECT_LOCAL_CONFIG`), or a config object. |
| `retry` | Retry policy for broker resolution calls. |
| `fetch` | Custom `fetch` implementation (tests, proxies). |

Each accessor returns a `Promise` — resolution happens on first use, and clients
are cached until their lease expires.

---

## Database — `env.db(binding)`

Returns an official [`@libsql/client`](https://github.com/tursodatabase/libsql-client-ts)
instance. Install it as a peer dependency: `npm install @libsql/client`.

```typescript
const db = await env.db('DB')

// libsql API — parameterized queries, batches, transactions
const { rows } = await db.execute('SELECT id, title FROM notes ORDER BY created_at DESC')

await db.execute({
  sql: 'INSERT INTO notes (id, title) VALUES (?, ?)',
  args: [crypto.randomUUID(), 'Hello'],
})

await db.batch([
  { sql: 'INSERT INTO posts (id, title) VALUES (?, ?)', args: [id, title] },
  { sql: 'UPDATE users SET post_count = post_count + 1 WHERE id = ?', args: [userId] },
])
```

Isolation is transparent: Flect stamps the resource's sqld namespace on every
request, so a binding only ever sees its own data. Because it's the standard
client, it drops straight into `drizzle-orm/libsql`:

```typescript
import { drizzle } from 'drizzle-orm/libsql'
const orm = drizzle(await env.db('DB'))
```

---

## KV — `env.kv(binding)`

Returns an official [`ioredis`](https://github.com/redis/ioredis) instance.
Install it as a peer dependency: `npm install ioredis`.

```typescript
const cache = await env.kv('CACHE')

await cache.set('session:abc', 'active', 'EX', 3600)   // 1h TTL
const val = await cache.get('session:abc')
await cache.del('session:abc')

await cache.incr('counter')
const many = await cache.mget('a', 'b', 'c')
```

Every key is transparently prefixed with the resource's namespace, so stores
never collide. The full ioredis command set is available.

---

## Object storage — `env.store(binding)`

Returns an official [`@aws-sdk/client-s3`](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/)
`S3Client`, pre-configured with the resource's endpoint, region, bucket, and
credentials. Install the peers: `npm install @aws-sdk/client-s3`.

```typescript
import { PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3'

const s3 = await env.store('FILES')
const bucket = process.env.FILES_BUCKET   // the resolved bucket name

await s3.send(new PutObjectCommand({
  Bucket: bucket,
  Key: 'avatars/alice.png',
  Body: buffer,
  ContentType: 'image/png',
}))

const obj = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: 'avatars/alice.png' }))
```

For presigned URLs, use `@aws-sdk/s3-request-presigner` with the same client —
it's a standard `S3Client`.

---

## Typing the client

Each accessor takes an optional type parameter if you want to annotate the
returned client:

```typescript
import type { Client } from '@libsql/client'
const db = await env.db<Client>('DB')
```

---

## Why it works this way

Your code depends only on well-known, well-documented libraries — Flect owns
none of your data-access API. The platform's job is resolution and isolation:
turning `env.db('DB')` into a scoped, short-lived, correctly-namespaced client,
without a URL or secret ever appearing in your source. See
[Platform architecture](/platform/) for how resolution works.
