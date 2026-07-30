---
name: flect-kv
description: >-
  Flect KV — Redis-compatible (Valkey), resolved as an official ioredis client
  via env.kv(). Use for caching, counters, sessions, queues, rate limiting, or
  binding [[kv]] in flect.toml.
---

# Flect · KV stores

Part of the [flect](../flect/SKILL.md) skill set.

## What it is

Flect KV stores are Redis-compatible, powered by **Valkey**. Each store has its
own key namespace. `env.kv(binding)` returns an official
[`ioredis`](https://github.com/redis/ioredis) client whose keys are transparently
prefixed, so stores never collide.

## Create

```bash
flect kv create my-cache
flect kv list
flect kv status <ref>
flect kv delete <ref>
```

## Bind

```toml
[[kv]]
binding = "CACHE"
name    = "my-cache"
```

## Use in code

```typescript
import { createEnv } from '@getflect/sdk'

const env   = createEnv()
const cache = await env.kv('CACHE')   // official ioredis client

// strings + TTL — note the ioredis signature: set(key, val, 'EX', seconds)
await cache.set('session:abc', token, 'EX', 3600)
const val = await cache.get('session:abc')
await cache.del('session:abc')

// counters, lists, hashes, sets — full ioredis API
await cache.incr('hits:today')
await cache.lpush('queue:jobs', JSON.stringify(job))
await cache.hset('config', 'theme', 'dark')
```

**Don't invent methods** — it's the real ioredis. Use `cache.set(k,v,'EX',ttl)`,
not `setJson`/`getJson`.

## Common patterns

### Cache-aside

```typescript
async function getCached<T>(key: string, load: () => Promise<T>, ttl = 300): Promise<T> {
  const hit = await cache.get(key)
  if (hit) return JSON.parse(hit) as T
  const data = await load()
  await cache.set(key, JSON.stringify(data), 'EX', ttl)
  return data
}
```

Invalidate on write (`cache.del(key)`) so the next read repopulates from the DB.

### Rate limiting

```typescript
async function underLimit(ip: string, limit = 100): Promise<boolean> {
  const key = `rl:${ip}:${Math.floor(Date.now() / 60000)}`   // per minute
  const n = await cache.incr(key)
  if (n === 1) await cache.expire(key, 60)
  return n <= limit
}
```

## Isolation

Keys are prefixed with the store's namespace automatically — use short, natural
keys. `KEYS`/`SCAN` only ever see your store's keys.

## Local development

```bash
flect dev     # createEnv() resolves CACHE locally
docker run -d -p 6379:6379 valkey/valkey:8-alpine
# VALKEY_URL=redis://localhost:6379  (default)
```

## Related skills

- [flect-db](../flect-db/SKILL.md) — the source of truth behind a cache.
- [flect-app](../flect-app/SKILL.md) · [flect](../flect/SKILL.md).
