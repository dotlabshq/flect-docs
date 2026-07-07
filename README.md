# Flect

Flect is a developer platform for deploying apps and their backing resources —
**databases, KV stores, and object storage** — on your own infrastructure
(Nomad + Traefik). You declare what your project needs in `flect.toml`; Flect
provisions it, binds it, and deploys your containers behind TLS.

Your code stays clean: it calls `createEnv()` and asks for a binding by name. It
never sees a URL, a port, or a secret. At deploy time Flect injects only
`FLECT_TOKEN` and `FLECT_BROKER_URL`; the SDK resolves each binding to a scoped,
short-lived connection at runtime.

## Resources

| Resource | Backed by | Accessor |
|----------|-----------|----------|
| **App** | Docker → Nomad | — |
| **Database** | sqld / libsql (SQLite-compatible) | `env.db('DB')` |
| **KV** | Valkey (Redis-compatible) | `env.kv('CACHE')` |
| **Object storage** | Garage (S3-compatible) | `env.store('FILES')` |

## Quick start

```bash
npm install -g @flect/cli

flect login                    # SSO, stored in ~/.flect/config.json
flect use acme/web/prod        # org / workspace / project / environment
flect init                     # scaffold flect.toml
flect deploy                   # provision resources, bind, deploy
```

## SDK

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()

const db    = await env.db('DB')       // an official @libsql/client
const cache = await env.kv('CACHE')    // an official ioredis
const files = await env.store('FILES') // an official @aws-sdk/client-s3

const { rows } = await db.execute('SELECT * FROM notes')
await cache.set('key', 'value')
```

The SDK returns the **official** client for each resource — there's no
Flect-specific query API to learn.

## Docs

- [Quickstart](guides/quickstart.md)
- [Local development](guides/local-dev.md)
- [CLI reference](cli/index.md)
- [SDK reference](sdk/index.md)
- [Platform architecture](platform/index.md)
- [flect.toml reference](platform/flect-toml.md)
