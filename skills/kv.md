# Flect KV Skill

## What It Is

Flect KV stores are **Redis-compatible**, powered by Valkey. Each KV store gets a key prefix enforced at the proxy layer — apps can only access their own keys even though all stores share the same Valkey instance.

Compatible with: ioredis, node-redis, any Redis client.

## Create a KV Store

```bash
flect kv create <name>          # e.g. flect kv create my-cache
flect kv list                   # show KV stores in current env
```

## Bind to an App

In `flect.toml`:

```toml
[[kv]]
binding = "CACHE"       # becomes CACHE_URL env var in the app
name    = "my-cache"    # KV slug from flect kv list
```

Multiple stores:
```toml
[[kv]]
binding = "SESSION_STORE"
name    = "sessions"

[[kv]]
binding = "RATE_LIMIT"
name    = "rate-limiter"
```

## Use in Code

Install SDK: `npm install @flect/sdk`

```typescript
import { kv } from '@flect/sdk'

// kv() returns an ioredis client, pre-configured from CACHE_URL
const cache = kv()

// String operations
await cache.set('user:123', JSON.stringify(user))
await cache.set('session:abc', token, 'EX', 3600)   // expires in 1h
const val = await cache.get('user:123')
await cache.del('session:abc')

// Counters
await cache.incr('hits:today')
const count = await cache.get('hits:today')

// Lists
await cache.lpush('queue:jobs', JSON.stringify(job))
const next = await cache.rpop('queue:jobs')

// Hash maps
await cache.hset('config', 'theme', 'dark', 'lang', 'tr')
const theme = await cache.hget('config', 'theme')

// Sets
await cache.sadd('active:users', userId)
const isActive = await cache.sismember('active:users', userId)

// TTL
await cache.expire('temp:key', 300)   // 5 minutes
const ttl = await cache.ttl('temp:key')
```

### With Raw ioredis

```typescript
import Redis from 'ioredis'

const cache = new Redis(process.env['CACHE_URL']!)

// Keys are automatically prefixed by flect-proxy — use short keys
await cache.set('my-key', 'value')  // stored as <prefix>:my-key in Valkey
```

## Common Patterns

### Session Storage

```typescript
import { kv } from '@flect/sdk'

const store = kv()

async function createSession(userId: string): Promise<string> {
  const sessionId = crypto.randomUUID()
  await store.set(`session:${sessionId}`, userId, 'EX', 86400)  // 24h
  return sessionId
}

async function getSession(sessionId: string): Promise<string | null> {
  return store.get(`session:${sessionId}`)
}
```

### Rate Limiting

```typescript
async function checkRateLimit(ip: string, limit = 100): Promise<boolean> {
  const key   = `rl:${ip}:${Math.floor(Date.now() / 60000)}`  // per minute
  const count = await store.incr(key)
  if (count === 1) await store.expire(key, 60)
  return count <= limit
}
```

### Caching

```typescript
async function getCached<T>(key: string, fetch: () => Promise<T>, ttl = 300): Promise<T> {
  const cached = await store.get(key)
  if (cached) return JSON.parse(cached) as T
  const data = await fetch()
  await store.set(key, JSON.stringify(data), 'EX', ttl)
  return data
}
```

## Local Development

```bash
# .env.local — local Redis/Valkey or redis-memory
CACHE_URL=redis://localhost:6379

# Or use a local Valkey via Docker
docker run -d -p 6379:6379 valkey/valkey:8-alpine
```

## Key Isolation

Keys are automatically prefixed at the proxy layer. You do not need to manually prefix keys — `KEYS *` and `SCAN` also only return keys belonging to your KV store.

## Limits

- All Redis-compatible commands supported
- Key prefix isolation enforced at proxy — no cross-store access possible
- Max memory: shared Valkey instance, configured per environment
