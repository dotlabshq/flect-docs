---
title: KV stores
description: Redis-compatible KV via Valkey, resolved as an official ioredis client.
---

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
import { createEnv } from '@flect/sdk'

const env   = createEnv()
const cache = await env.kv('CACHE')   // official ioredis client

// strings + TTL
await cache.set('session:abc', token, 'EX', 3600)   // 1h
const val = await cache.get('session:abc')
await cache.del('session:abc')

// counters, lists, hashes, sets — full ioredis API
await cache.incr('hits:today')
await cache.lpush('queue:jobs', JSON.stringify(job))
await cache.hset('config', 'theme', 'dark')
await cache.sadd('active:users', userId)
```

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
