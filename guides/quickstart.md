---
title: Quickstart
---

Deploy your first app on Flect in under 5 minutes.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) — to build and push your image
- A container registry (GitHub Container Registry, Docker Hub, etc.)
- Flect CLI installed

## 1. Install the CLI

```bash
npm install -g @flect/cli
```

## 2. Log in

```bash
flect login
```

This opens a browser window. Sign in with your account and the CLI saves your token locally.

## 3. Set up your workspace

```bash
# create or select an org
flect org create --name my-org
flect org use my-org

# create a workspace and project
flect ws create --name default
flect ws use default

flect proj create --name myproject
flect proj use myproject

# create a production environment
flect env create --name production
flect env use production
```

## 4. Create your resources

```bash
# database (SQLite, isolated per app)
flect db create --name myapp-db

# run migrations
flect db migrate myapp-db --dir ./migrations

# KV store (Redis-compatible cache)
flect kv create --name myapp-cache
```

## 5. Create an app

```bash
flect app create --name myapp
# → myapp-abc123.up.flect.run
```

## 6. Add a `flect.toml`

In your project root:

```toml
name = "myapp"
port = 3000

[[databases]]
binding = "DB"
name    = "myapp-db"

[[kv]]
binding = "CACHE"
name    = "myapp-cache"
```

The `binding` is the env var prefix your app uses. The `name` is the resource name in Flect.

## 7. Use the SDK in your app

```bash
npm install @flect/sdk
```

```typescript
import { createEnv } from '@flect/sdk'

const env   = createEnv()
const db    = env.db('DB')      // reads DB_URL, DB_NAMESPACE, FLECT_TOKEN
const cache = env.kv('CACHE')   // reads CACHE_URL, FLECT_TOKEN
```

## 8. Deploy

Build and push your image, then deploy from the directory containing `flect.toml`:

```bash
docker build -t ghcr.io/you/myapp:1.0.0 .
docker push ghcr.io/you/myapp:1.0.0

flect app deploy myapp --image ghcr.io/you/myapp:1.0.0
```

```
✓ Deployed: abc123
  id      abc123
  status  running
  url     https://myapp-abc123.up.flect.run
  image   ghcr.io/you/myapp:1.0.0
```

Your app is live. All env vars (`DB_URL`, `CACHE_URL`, `FLECT_TOKEN`) are injected automatically — your code needs no configuration.

## Next steps

- [CLI Reference](../cli/index.md) — all available commands
- [SDK Reference](../sdk/index.md) — database and KV API
- [flect.toml Reference](../platform/flect-toml.md) — full config options
- [Notes API example](./notes-api.md) — complete working example
