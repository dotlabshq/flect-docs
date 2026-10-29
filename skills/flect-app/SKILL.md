---
name: flect-app
description: >-
  Deploy an application's services on Flect — write flect.toml ([app],
  resources by id, [services.<name>] with uses/calls/vars/secrets), build and
  push the images, flect deploy, routing (public / expose / private), a dev
  environment in flect.dev.toml. Use when shipping services, wiring a
  Dockerfile for Flect, or fixing or migrating a flect.toml.
---

# Flect · Applications

Part of the [flect](../flect/SKILL.md) skill set. A Flect application is one
`flect.toml`: its services are containers, and they use databases
([flect-db](../flect-db/SKILL.md)), KV ([flect-kv](../flect-kv/SKILL.md)) and
object storage ([flect-store](../flect-store/SKILL.md)) by id.

## The file

```toml
[app]
name  = "shop"
scope = "web/shop/production"        # workspace/project/environment in the active org
# domain = "shop.example.com"        # custom domain for the public service

[[databases]]
binding = "DB"
id      = "res_k7f3a2"               # flect db create shop-db

[vars]                               # every service
LOG_LEVEL = "info"

[services.api]
image   = "ghcr.io/acme/shop-api:1.4.0"
port    = 3000
uses    = ["DB"]                     # may resolve DB; nothing else
secrets = ["STRIPE_KEY"]

[services.api.vars]                  # api only; wins over [vars]
CURRENCY = "EUR"

[services.auth]
image  = "ghcr.io/dotlabshq/auth-service:1.0.1"
port   = 3000
expose = "/v1/auth"                  # under web's host, prefix stripped

[services.web]
image    = "ghcr.io/acme/shop-web:1.4.0"
port     = 3000
public   = true                      # owns the domain
memoryMb = 512
calls    = ["api", "auth"]           # API_SERVICE_URL, AUTH_SERVICE_URL
```

Service fields: `image`, `port`, `public`, `expose`, `replicas`, `cpuMhz`,
`memoryMb`, `health`, `uses`, `calls`, `secrets`, `vars`, and `name` (only to
keep an app already running under a name other than `<app>-<service>`). Every
key is in the [flect.toml reference](/platform/flect-toml/).

## Deciding what each service lists

Go through the code of each service:

| The code… | List |
|---|---|
| calls `env.db('DB')` / `env.kv(…)` / `env.store(…)` | the binding in `uses` |
| reads `process.env.API_SERVICE_URL` | `"api"` in `calls` |
| reads a config value | the name in that service's `vars` (or `[vars]` if every service reads it) |
| reads a key, token, password or signing secret | the name in `secrets` |

List nothing a service does not read. That is the point: a leaked front end
holds no database access and no other service's secrets.

## Routing

| Setting | Reachable at |
|---|---|
| `public = true` | `https://<app>-<service>-<shortid>.up.flect.run`, plus `[app].domain` via a CNAME. One per file. |
| `expose = "/v1/auth"` | That path under the public service's host. The prefix is stripped. |
| neither | Private: only the services that `call` it, through `<NAME>_SERVICE_URL`. |

## Deploy

```bash
# images first (Flect does not build containers; the registry must be public)
docker build --platform linux/amd64 -t ghcr.io/acme/shop-api:1.4.0 . && docker push ghcr.io/acme/shop-api:1.4.0

printf %s "$STRIPE_KEY" | flect secret set api STRIPE_KEY    # each listed secret, once per scope
flect deploy
```

`flect deploy` first checks the whole file against the scope: every id is a
live resource of this scope, no binding name points elsewhere, every listed
secret is set, and no var kept by the last deploy is silently dropped. On any
problem it stops with nothing changed and says what to run. Then it binds the
resources, declares every service, and deploys each one with only what it
lists.

A changed image tag, var, secret, `uses` or `calls` takes effect on the next
`flect deploy`. A service restarts when its own settings change, or when the
address of a service it `calls` changes. It does not restart for its other
siblings.

## A dev environment

```bash
cp flect.toml flect.dev.toml
# edit: [app].scope = "web/shop/dev"; drop services dev does not need
flect db create shop-db -c flect.dev.toml          # dev's own database → its id into flect.dev.toml
printf %s "$DEV_KEY" | flect secret set api STRIPE_KEY -c flect.dev.toml
flect deploy -c flect.dev.toml
```

The two files are independent: no inheritance, no shared ids, no shared
secret values.

## Container requirements

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY dist/ ./dist/
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

- Listen on `port`.
- Answer `GET /healthz` with `200`, or set `health = "/your/path"`, or
  `health = false` for a worker with no HTTP. Until the check passes no traffic
  arrives, and services that call it get an empty address.
- `linux/amd64`, dependencies bundled (no workspace symlinks).
- The environment holds `FLECT_TOKEN`, `FLECT_BROKER_URL`, `PORT`, `NODE_ENV`,
  `APP_PUBLIC_URL` (public and exposed services), its vars and secrets, and its
  `*_SERVICE_URL`s. Nothing else: no database URL.

## Using @dotlabshq/flect-sdk

```typescript
import { Hono } from 'hono'
import { serve } from '@hono/node-server'
import { createEnv } from '@dotlabshq/flect-sdk'

const env = createEnv()
const db  = await env.db('DB')      // official @libsql/client; 'DB' is in this service's uses
const app = new Hono()

app.get('/healthz', (c) => c.json({ ok: true }))
app.get('/users', async (c) => {
  const { rows } = await db.execute('SELECT * FROM users')
  return c.json({ users: rows })
})

serve({ fetch: app.fetch, port: Number(process.env.PORT ?? 3000) })
```

## Moving an old-format file

A file with `[[apps]]`, `[[services]]` entries and resources by `name` is the
old format. It still deploys, with a warning, but every app gets every var and
binding.

1. `flect db list` (and `kv`, `store`) for each resource's `res_` id.
2. One `[services.<name>]` per `[[apps]]`. If the app's current name is not
   `<app>-<service>`, set `name` to it so the running app is kept.
3. Add `[app].scope`, and per service `uses`, `calls`, `secrets` and only the
   vars it reads.
4. `flect secret set` each secret.
5. `flect deploy`: it stops and lists the kept vars that would be dropped.
   Check the list, then `flect deploy --yes`.

## Management

```bash
flect apps                  # the scope's apps: status, URL, image
flect bindings              # what the file gives each service
flect db list               # resources and the services using them
flect secret list           # secret names per service; any missing
```

## Related skills

- [flect](../flect/SKILL.md) — the model, CLI, scopes, plan limits.
- [flect-db](../flect-db/SKILL.md) · [flect-kv](../flect-kv/SKILL.md) · [flect-store](../flect-store/SKILL.md) — resources.
- [flect-pages](../flect-pages/SKILL.md) — static/docs sites instead of a container.
