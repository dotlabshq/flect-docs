---
name: flect
description: >-
  Build and deploy an app plus its backing resources (database, KV, object
  storage, static pages) on the Flect platform. Use when the user wants to ship a
  service on Flect, wire up a Flect database/KV/store, write flect.toml, use
  @getflect/sdk (createEnv), or run the flect CLI (login, scope, deploy). This is
  the hub — load a component skill for depth.
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

## Load a component skill for depth (lazy)

This hub covers the model and workflow. For a specific capability, load its skill
— each is self-contained:

| When you're… | Load |
|---|---|
| deploying a container / writing an app's `flect.toml` | [`flect-app`](../flect-app/SKILL.md) |
| adding a database, queries, migrations, Drizzle | [`flect-db`](../flect-db/SKILL.md) |
| adding a cache, counters, sessions, rate limits | [`flect-kv`](../flect-kv/SKILL.md) |
| storing files, presigned URLs, S3 | [`flect-store`](../flect-store/SKILL.md) |
| publishing a docs / static site from Markdown | [`flect-pages`](../flect-pages/SKILL.md) |

Raw fetch base (when a tool needs the URL):
`https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/<name>/SKILL.md`.
Deeper references: `cli/index.md`, `sdk/index.md`, `platform/index.md`,
`platform/flect-toml.md`.

## When to use

- Shipping a web service / API on Flect.
- Adding a database (sqld/libsql), KV (Valkey), object store (Garage/S3), or a
  static docs site (Pages).
- Writing or fixing a `flect.toml`.
- Using `@getflect/sdk` (`createEnv`) or the `flect` CLI.

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
# 1. Install + authenticate + point at the platform
npm install -g @getflect/cli
flect login                         # SSO, stored in ~/.flect/config.json
flect config set gatewayUrl https://flect.cloud   # one origin drives identity + broker
                                                  # (self-hosted: your gateway origin)
flect whoami                        # confirm identity + where you're pointed
flect doctor                        # green-check config/plane/broker/token/scope/providers

# 2. Scaffold, then pick where it deploys — the ACTIVE CONTEXT is the target
flect init                          # writes a starter flect.toml
flect proj create --org acme --ws web --name site --slug site   # create the scope
flect use acme/web/site             # THIS sets the deploy target (flect deploy uses it)
flect ls                            # inspect the org/ws/proj tree; active node marked
#   The toml `scope` field is reserved/ignored today — `flect use` (or
#   `flect deploy --scope <id>`) is what actually targets the deploy.

# 3. Develop locally (no broker needed)
flect dev                           # writes flect.local.json; createEnv() resolves locally

# 4. Deploy
docker build --platform linux/amd64 -t ghcr.io/you/myapp:1.0.0 . && docker push ghcr.io/you/myapp:1.0.0
flect deploy                        # provisions missing resources, binds, deploys
```

`flect deploy` provisions any resource named in `flect.toml` that doesn't exist
yet — or create them explicitly with `flect db|kv|store create <name>`.

> **Providers first (operator, once per tenant).** db/kv/store provisioning
> goes through substrate *providers* an operator registers for your tenant. If
> `flect doctor` warns `providers — none registered`, provisioning will fail
> until they're added — apps with no resource bindings still deploy fine.

## flect.toml (quick shape)

```toml
name    = "myapp"
runtime = "node"
port    = 3000
scope   = "myapp/main/prod"          # reserved/ignored — deploy target comes from `flect use`

[[databases]]
binding = "DB"                       # the name code passes to env.db()
name    = "myapp-db"                 # the Flect resource

[[kv]]
binding = "CACHE"
name    = "myapp-cache"
```

For multi-app groups (shared domain, sibling services, `[[services]]` bindings)
see [`flect-app`](../flect-app/SKILL.md) and the
[flect.toml reference](/platform/flect-toml/).

## SDK — the rules that matter

```typescript
import { createEnv } from '@getflect/sdk'

const env = createEnv()

// Each accessor is ASYNC and returns the OFFICIAL client, already scoped.
const db    = await env.db('DB')       // @libsql/client    → db.execute(), db.batch()
const cache = await env.kv('CACHE')    // ioredis           → cache.get/set/del/incr
const files = await env.store('FILES') // @aws-sdk/client-s3 → files.send(command)
```

The most common agent mistakes:

- **Always `await`** `env.db/kv/store` — they return promises.
- They return the **real** libsql / ioredis / S3 clients. **Do NOT invent**
  `db.query()`, `kv.getJson()`, `store.put()` — use each library's own API
  (`db.execute`, `cache.set(k,v,'EX',ttl)`, `s3.send(new PutObjectCommand(...))`).
- The binding string must match a `binding` in `flect.toml`.
- Install the client libs your bindings need: `@libsql/client`, `ioredis`,
  `@aws-sdk/client-s3`. For storage, the bucket is `process.env.<BINDING>_BUCKET`.
- The SDK has **no `runMigrations`** — apply schema yourself on boot (see
  [`flect-db`](../flect-db/SKILL.md)).

## Deploy requirements (apps)

- Container listens on the `port` declared in `flect.toml`.
- Responds to `GET /healthz` with `200`, or declares its own path with
  `health = "/your/path"` (`health = false` for a workload with no HTTP server).
  No traffic is routed until the check passes.
- Built for `linux/amd64`; dependencies bundled into the image.
- Public apps get `https://<name>-<shortid>.up.flect.run`. Set `[app].domain`
  for a custom domain and point a CNAME at the generated hostname.

## CLI cheat sheet

```bash
flect login | logout | whoami | ps | ls
flect org | ws | proj | env            # tenancy (each: bare=list, + create/get/update/delete)
flect use <org[/ws/proj/env]>          # set active context (interactive if no arg)
flect access role <scopeId>            # your effective role (PDP)
flect init [--name <n>]            # scaffold flect.toml
flect dev                          # flect.local.json for local createEnv()
flect bindings                     # list declared bindings/apps/services
flect deploy [--scope <scopeId>]   # provision + bind + deploy (targets the active context)
flect apps                         # list deployed apps
flect <db|kv|store> create <name> | list | status <ref> | delete <ref>
flect resolve <binding>            # inspect a binding manifest (secrets redacted)
flect doctor                       # diagnose config/plane/broker/token/context/providers
flect config set <gatewayUrl|broker|token> <value>
```

Connection precedence: flags → env (`FLECT_URL`, `FLECT_BROKER_URL`,
`FLECT_TOKEN`) → `~/.flect/config.json` → defaults. `gatewayUrl` (config) or
`FLECT_URL` (env) is the single origin the whole plane derives from — identity
at `/v1/*`, broker at `/`; set it once and skip `broker`. Tenancy is delegated
to org-service: the active org + scope travel as `X-Org-Id` + `X-Flect-Scope`,
so tenant-level reads (`flect ls`, `flect org`) work before you've picked a
scope.

## Plan limits

Limits are enforced org-wide at the control plane; exceeding one returns HTTP
`402` with a clear message. Until billing lands, **every org defaults to the
Team plan** (generous — real plan selection + downgrades come with billing).
Deprovisioned resources don't count.

| Plan | Apps | Databases | KV | Storage |
|------|------|-----------|----|---------|
| Hobby | 3 | 1 | 1 | 1 |
| Pro | 10 | 3 | 3 | 3 |
| **Team (current default)** | 25 | 10 | 10 | 10 |
