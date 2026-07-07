---
title: Apps
description: Deploy containerized services with flect.toml and flect deploy.
---

## What it is

A Flect app is a container (any Docker image) deployed to the cluster. Public
apps get an HTTPS hostname (`<name>-<shortid>.up.flect.run`) with an auto-issued
TLS certificate, and can bind to databases, KV stores, and object storage.

## Workflow

```
1. Build and push a Docker image to a registry (ghcr.io, Docker Hub, …)
2. Declare the app + its bindings in flect.toml
3. flect deploy
```

## flect.toml

Single app (top-level `name`):

```toml
name    = "my-app"
runtime = "node"
port    = 3000

[[databases]]
binding = "DB"
name    = "my-db"

[[kv]]
binding = "CACHE"
name    = "my-cache"
```

Or the app-group form for multiple apps / a custom domain:

```toml
[app]
name   = "my-app"
domain = "my-app.up.flect.run"

[[apps]]
name   = "api"
image  = "ghcr.io/you/api:1.0.0"
port   = 3000
public = true            # owns the domain (at most one)
```

See the [flect.toml reference](/platform/flect-toml/) for every field.

## Deploy

```bash
flect deploy
```

This provisions any missing resources, binds them, and submits the Nomad job.
Only `FLECT_TOKEN` and `FLECT_BROKER_URL` are injected — the SDK resolves
everything else at runtime.

Routing is chosen per app: `public = true` (owns a domain), `expose = "/path"`
(a path under a shared domain, prefix stripped), or private (default, no
ingress).

## Dockerfile requirements

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY dist/ ./dist/
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

The container must:

- Listen on the port declared in `flect.toml` (`port`).
- Respond to `GET /healthz` with `200` (Traefik health check).
- Bundle its dependencies (no workspace symlinks in the final image).

## Using @flect/sdk

```typescript
import { Hono } from 'hono'
import { createEnv } from '@flect/sdk'

const env = createEnv()
const db  = await env.db('DB')      // official @libsql/client
const app = new Hono()

app.get('/healthz', (c) => c.json({ ok: true }))

app.get('/users', async (c) => {
  const { rows } = await db.execute('SELECT * FROM users')
  return c.json({ users: rows })
})

export default app
```

## Custom domain

Set `[app].domain` to your hostname and point a CNAME at the generated
`<name>-<shortid>.up.flect.run`. The public app owns that domain.

## Management

```bash
flect apps                 # list apps in the active scope (name, status, URL)
flect deploy               # (re)deploy from flect.toml
```
