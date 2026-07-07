---
name: flect
description: >-
  Build and deploy an app plus its backing resources (database, KV, object
  storage) on the Flect platform. Use when the user wants to ship a service on
  Flect, wire up a Flect database/KV/store, write flect.toml, use @flect/sdk
  (createEnv), or run the flect CLI (login, scope, deploy).
---

# Flect

Flect is a developer platform for deploying containers and their backing
resources on your own infrastructure (Nomad + Traefik). The defining rule:
**application code calls `createEnv()` and asks for a binding by name — it never
handles a URL, port, or secret.** The platform provisions resources and resolves
each binding to a scoped, *official* client at runtime.

At deploy time Flect injects only `FLECT_TOKEN` and `FLECT_BROKER_URL`. The SDK
uses them to resolve bindings through the broker into short-lived, scoped
connections. No `DB_URL`, credential, or connection string ever appears in the
image or the code.

This skill is self-contained. For deeper detail, fetch the component skills
listed at the bottom.

## When to use

- Shipping a web service / API on Flect.
- Adding a database (sqld/libsql), KV (Valkey), or object store (Garage/S3).
- Writing or fixing a `flect.toml`.
- Using `@flect/sdk` (`createEnv`) or the `flect` CLI.

## Mental model

```
Org → Workspace → Project → Environment      (the scope tree)
                              ├── Apps        (containers)
                              ├── Databases   (env.db)
                              ├── KV stores   (env.kv)
                              └── Object stores (env.store)
```

- **Bindings** map a name your code uses (`env.db('DB')`) to a Flect resource.
- **Active scope** is where resources/deploys land — set it with `flect use`.
- **Resolution**: `createEnv().db('DB')` → broker returns a short-lived manifest
  → SDK builds the official client, scoped and isolated.

## End-to-end workflow

```bash
# 1. Install + authenticate
npm install -g @flect/cli
flect login                         # SSO, stored in ~/.flect/config.json
flect whoami                        # confirm identity + where you're pointed

# 2. Pick a scope (create the tree once, then just `flect use`)
flect scope create workspace web
flect scope create project site --parent <workspace-id>
flect scope create environment prod --parent <project-id>
flect scope list                    # shows the tree; active scope is marked
flect use web/site/prod             # slug path (or a scope id)

# 3. Scaffold + declare bindings
flect init                          # writes flect.toml, git-ignores local files
#   edit flect.toml — see below

# 4. Develop locally (no broker needed)
flect dev                           # writes flect.local.json; createEnv() resolves locally

# 5. Deploy
docker build -t ghcr.io/you/myapp:1.0.0 . && docker push ghcr.io/you/myapp:1.0.0
flect deploy                        # provisions missing resources, binds, deploys
```

`flect deploy` provisions any resource named in `flect.toml` that doesn't exist
yet — or create them explicitly with `flect db|kv|store create <name>` (each
prints the exact ref to paste into `flect.toml`).

## flect.toml

Single-app form (the common case):

```toml
name    = "myapp"
runtime = "node"
port    = 3000                      # the port your container listens on

[[databases]]
binding        = "DB"               # the name code passes to env.db()
name           = "myapp-db"         # the Flect resource
migrations_dir = "migrations"       # optional

[[kv]]
binding = "CACHE"
name    = "myapp-cache"

[[stores]]
binding = "FILES"
name    = "myapp-uploads"
```

App-group form (multiple apps sharing a domain):

```toml
[app]
name   = "clave"
domain = "clave-api.up.flect.run"   # optional custom/shared domain

[[apps]]
name   = "api"
image  = "ghcr.io/you/api:1.0.0"
port   = 3000
public = true                       # owns the domain (at most one)

[[apps]]
name   = "auth"
image  = "ghcr.io/you/auth:1.0.0"
expose = "/v1/auth"                 # mounted at a path under the shared domain

[vars]
OIDC_ISSUER = "https://nesskey.com"

[[services]]
binding = "AUTH_SERVICE"            # inject the sibling app's URL as this env var
app     = "auth"
```

`[[apps]]` fields: `name`, `image`, `port`, `public`, `expose`, `replicas`,
`cpuMhz`, `memoryMb`.

## SDK — the rules that matter

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()

// Each accessor is ASYNC and returns the OFFICIAL client, already scoped.
const db    = await env.db('DB')       // @libsql/client    → db.execute(), db.batch()
const cache = await env.kv('CACHE')    // ioredis           → cache.get/set/del/incr
const files = await env.store('FILES') // @aws-sdk/client-s3 → files.send(command)
```

Get these right — the most common agent mistakes:

- **Always `await`** `env.db/kv/store` — they return promises.
- They return the **real** libsql / ioredis / S3 clients. **Do NOT invent**
  `db.query()`, `kv.getJson()`, `kv.setJson()`, or `store.put()` — those don't
  exist. Use each library's own API (`db.execute`, `cache.set(k,v,'EX',ttl)`,
  `s3.send(new PutObjectCommand(...))`).
- The binding string must match a `binding` in `flect.toml`.
- Install the client libs your bindings need as peer deps: `@libsql/client`,
  `ioredis`, `@aws-sdk/client-s3`.
- For object storage, the resolved bucket name is available as `<BINDING>_BUCKET`
  (e.g. `process.env.FILES_BUCKET`).

Concrete examples:

```typescript
// database — parameterized query + batch
const { rows } = await db.execute('SELECT * FROM notes WHERE owner = ?', [userId])
await db.execute({ sql: 'INSERT INTO notes (id, title) VALUES (?, ?)', args: [id, title] })

// kv — TTL, counters
await cache.set(`session:${id}`, token, 'EX', 3600)
const n = await cache.incr('hits:today')

// storage — official S3 commands
import { PutObjectCommand } from '@aws-sdk/client-s3'
await files.send(new PutObjectCommand({ Bucket: process.env.FILES_BUCKET, Key: 'a.txt', Body: 'hi' }))
```

Drizzle works directly: `drizzle(await env.db('DB'))`.

## Deploy requirements (apps)

- Container listens on the `port` declared in `flect.toml`.
- Responds to `GET /healthz` with `200` (Traefik health check).
- Built for `linux/amd64`; dependencies bundled into the image.
- Public apps get `https://<name>-<shortid>.up.flect.run`. Set `[app].domain`
  for a custom domain and point a CNAME at the generated hostname.

## CLI cheat sheet

```bash
flect login | logout | whoami | ps
flect scope create <workspace|project|environment> <name> [--parent <id>] [--slug <s>]
flect scope list
flect use <slug-path|scopeId>
flect init [--name <n>]            # scaffold flect.toml
flect dev                          # flect.local.json for local createEnv()
flect bindings                     # list declared bindings/apps/services
flect deploy [--scope <id>]        # provision + bind + deploy
flect apps                         # list deployed apps
flect <db|kv|store> create <name> | list | status <ref> | delete <ref>
flect resolve <binding>            # inspect a binding manifest (secrets redacted)
flect config set <broker|token|scope|auth> <value>
```

Connection precedence: flags → env (`FLECT_BROKER_URL`, `FLECT_TOKEN`) →
`~/.flect/config.json` → defaults. Identity tokens aren't scope-bound; the active
scope travels in the `X-Flect-Scope` header, so `flect scope list` works before
you've picked one.

## Plan limits

Every org starts on **hobby** (3 apps / 1 db / 1 kv / 1 store). Exceeding a limit
returns HTTP `402` with a clear message; deprovisioned resources don't count.
Pro = 10/3/3/3, Team = 25/10/10/10.

## Component skills

Fetch for depth (raw): `https://raw.githubusercontent.com/dotlabshq/flect-docs/main/<path>`

| Task | Path |
|------|------|
| Databases | `skills/db.md` |
| KV stores | `skills/kv.md` |
| Object storage | `skills/store.md` |
| App deploy | `skills/app.md` |
| CLI reference | `cli/index.md` |
| SDK reference | `sdk/index.md` |
| Architecture | `platform/index.md` |
| flect.toml | `platform/flect-toml.md` |
