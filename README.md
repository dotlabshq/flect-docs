# Flect

Flect is a developer platform for deploying apps, databases, key-value stores, and static pages. It runs on your own infrastructure via Nomad and exposes a CLI + SDK for building and deploying projects.

## Resources

| Resource | Description |
|----------|-------------|
| **App** | Containerized application (Docker → Nomad) |
| **Database** | SQLite-compatible database via sqld/libsql |
| **KV** | Redis-compatible key-value store via Valkey |
| **Page** | Static site deployed from a GitHub repo |

## Quick Start

```bash
# Install
npm install -g @flect/cli

# Login
flect login

# Create resources
flect db create my-db
flect kv create my-cache

# Deploy an app
flect app deploy my-app --image ghcr.io/you/my-app:1.0.0

# Deploy a static page
flect page create my-docs
flect page deploy my-docs --repo github.com/you/my-docs
```

## SDK

```typescript
import { db, kv } from '@flect/sdk'

// SQLite-compatible DB
const rows = await db().execute('SELECT * FROM users')

// Redis-compatible KV
await kv().set('key', 'value')
const val = await kv().get('key')
```

## Docs

- [Quickstart](guides/quickstart.md)
- [CLI Reference](cli/index.md)
- [SDK Reference](sdk/index.md)
- [Platform Architecture](platform/index.md)
- [flect.toml Reference](platform/flect-toml.md)
