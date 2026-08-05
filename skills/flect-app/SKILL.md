---
name: flect-app
description: >-
  Deploy a containerized app on Flect — write flect.toml, build/push the image,
  flect deploy, and pick routing (public / expose / private). Use when shipping a
  service, wiring a Dockerfile for Flect, or fixing an app's flect.toml.
---

# Flect · Apps

Part of the [flect](../flect/SKILL.md) skill set. Deploy any Docker image as a
Flect app; it binds to databases ([flect-db](../flect-db/SKILL.md)), KV
([flect-kv](../flect-kv/SKILL.md)), and object storage
([flect-store](../flect-store/SKILL.md)).

## What it is

A Flect app is a container deployed to the cluster. Public apps get an HTTPS
hostname (`<name>-<shortid>.up.flect.run`) with an auto-issued TLS certificate.

## Workflow

```
0. flect login && flect config set gatewayUrl <origin>   (once per machine)
1. Build and push a Docker image to a registry (ghcr.io, Docker Hub, …)
2. Declare the app + its bindings in flect.toml
3. flect deploy
```

First time on a machine, point the CLI at the platform and confirm it's wired:

```bash
flect login
flect config set gatewayUrl https://flect.cloud   # self-hosted: your gateway origin
flect doctor                                       # all green before you deploy
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

App-group form — multiple apps sharing one domain (e.g. an app + a sibling auth):

```toml
[app]
name   = "my-app"
domain = "my-app.up.flect.run"   # optional custom/shared domain

[[apps]]
name   = "api"
image  = "ghcr.io/you/api:1.0.0"
port   = 3000
public = true            # owns the domain (at most one)

[[apps]]
name   = "auth"
image  = "ghcr.io/dotlabshq/auth-service:0.4.0"
expose = "/v1/auth"      # mounted at a path under the shared domain

[[services]]
binding = "iam"          # inject a sibling app's URL as <BINDING>_SERVICE_URL
app     = "iam-service"
```

`[[apps]]` fields: `name`, `image`, `port`, `public`, `expose`, `replicas`,
`cpuMhz`, `memoryMb`, `health`. See the [flect.toml reference](/platform/flect-toml/).

## Deploy

```bash
flect deploy
```

Provisions any missing resources, binds them, and submits the Nomad job. Only
`FLECT_TOKEN` and `FLECT_BROKER_URL` are injected — the SDK resolves everything
else at runtime.

If the app declares databases/KV/stores, the tenant needs substrate **providers**
registered (an operator does this once — `flect doctor` warns when they're
missing). An app with no resource bindings deploys without them. Routing is per app: `public = true` (owns a domain),
`expose = "/path"` (a path under a shared domain, prefix stripped), or private
(default, no ingress).

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
- Respond to `GET /healthz` with `200` — or declare `health = "/your/path"` in
  `flect.toml`. Until the check passes Traefik sends the app no traffic. A
  workload that serves no HTTP at all sets `health = false`.
- Be built for `linux/amd64` with dependencies bundled (no workspace symlinks).

## Using @getflect/sdk

```typescript
import { Hono } from 'hono'
import { createEnv } from '@getflect/sdk'

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
`<name>-<shortid>.up.flect.run`.

## Management

```bash
flect apps                 # list apps in the active scope (name, status, URL)
flect deploy               # (re)deploy from flect.toml
```

## Related skills

- [flect](../flect/SKILL.md) — the model, CLI, scopes, plan limits.
- [flect-db](../flect-db/SKILL.md) · [flect-kv](../flect-kv/SKILL.md) · [flect-store](../flect-store/SKILL.md) — bindings.
- [flect-pages](../flect-pages/SKILL.md) — static/docs sites instead of a container.
