---
name: flect
description: >-
  Build and deploy an application and its backing resources (database, KV,
  object storage, static pages) on the Flect platform. Use when the user wants
  to ship services on Flect, write or fix a flect.toml, wire up a Flect
  database/KV/store, set secrets, use @dotlabshq/flect-sdk (createEnv), or run
  the flect CLI (login, deploy, db, secret). This is the hub — load a component
  skill for depth.
---

# Flect

Flect runs containers and gives them databases, KV stores and object storage.
Two rules carry everything:

1. **Code never holds a URL or a secret.** It calls `createEnv()` and asks for a
   binding by name (`env.db("DB")`). The SDK resolves it through the broker
   into the official client, scoped to that one resource.
2. **One `flect.toml` is one application in one scope.** It names its scope,
   the resources it uses by id, and its services. **Each service gets only what
   it lists.** Another environment is another file (`flect.dev.toml`).

## Load a component skill for depth (lazy)

| When you're… | Load |
|---|---|
| writing an application's `flect.toml`, deploying services, routing, secrets | [`flect-app`](../flect-app/SKILL.md) |
| adding a database, queries, migrations, Drizzle | [`flect-db`](../flect-db/SKILL.md) |
| adding a cache, counters, sessions, rate limits | [`flect-kv`](../flect-kv/SKILL.md) |
| storing files, presigned URLs, S3 | [`flect-store`](../flect-store/SKILL.md) |
| publishing a docs / static site from Markdown | [`flect-pages`](../flect-pages/SKILL.md) |

Raw fetch base:
`https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/<name>/SKILL.md`.
Full references: `platform/flect-toml.md` (every key and every deploy check),
`cli/index.md`, `sdk/index.md`, `platform/index.md`.

## Mental model

```
Org → Workspace → Project → Environment          the scope tree
                               ├── resources      database / KV / store, each with an id res_…
                               └── apps           one per service of a flect.toml
```

| In flect.toml | Meaning |
|---|---|
| `[app] name, scope` | The application and the scope (`workspace/project/environment`) it deploys to. |
| `[[databases]]`, `[[kv]]`, `[[stores]]` | A resource it uses: `binding` (name in code) + `id` (`res_…`). |
| `[services.<name>]` | One microservice: one image, deployed as the app `<app>-<name>`. |
| `uses` | Bindings that service may resolve. The broker refuses any other. |
| `calls` | Services whose address it gets, as `<NAME>_SERVICE_URL`. |
| `[vars]`, `[services.X.vars]` | Plain configuration: shared, then per service. |
| `secrets` | Names of its secrets; values live in the platform, set from stdin. |

- A resource belongs to the scope it was created in. A dev service cannot use a
  production resource, even with its id.
- Resolution: `createEnv().db("DB")` → the broker checks the service lists
  `DB` → returns a short-lived manifest → the SDK builds `@libsql/client`.

## End-to-end workflow

```bash
# Once per machine
npm install -g @dotlabshq/flect-cli       # >= 0.3.2
flect login                                # browser SSO — the user does this
flect use <org>                            # the org; each file names its own scope
flect doctor                               # all green before going on

# The scope must exist
flect ls
flect proj create --org <org> --ws <ws> --name Shop --slug shop        # if missing
flect env  create --org <org> --ws <ws> --proj shop --name production   # if missing

# flect.toml with [app].scope = "<ws>/shop/production"; then, in that scope:
flect db create shop-db                    # → flect.toml: id = "res_k7f3a2"
printf %s "$VALUE" | flect secret set api STRIPE_KEY
flect secret list                          # nothing "not set"

# Images: build and push first (linux/amd64, public)
docker build --platform linux/amd64 -t ghcr.io/acme/shop-api:1.0.0 . && docker push ghcr.io/acme/shop-api:1.0.0

flect deploy                               # checks the whole file; stops with nothing changed on a problem
flect apps                                 # verify
flect db list                              # "used by" = the services whose uses list it
```

`flect dev` writes `flect.local.json` so `createEnv()` resolves bindings
locally (sqlite, Valkey, MinIO) with no broker.

> **Providers first (operator, once per tenant).** db/kv/store are provisioned
> through substrate providers an operator registers. If `flect doctor` warns
> `providers — none registered`, `flect db create` fails until they exist.

## flect.toml (shape)

```toml
[app]
name  = "shop"
scope = "web/shop/production"

[[databases]]
binding = "DB"
id      = "res_k7f3a2"

[vars]
LOG_LEVEL = "info"

[services.api]
image   = "ghcr.io/acme/shop-api:1.0.0"
port    = 3000
uses    = ["DB"]
secrets = ["STRIPE_KEY"]

[services.web]
image  = "ghcr.io/acme/shop-web:1.0.0"
port   = 3000
public = true
calls  = ["api"]
```

Routing, health, sizing, vars vs secrets and the dev file: see
[`flect-app`](../flect-app/SKILL.md).

## SDK — the rules that matter

```typescript
import { createEnv } from '@dotlabshq/flect-sdk'   // ^0.2.0

const env = createEnv()

// ASYNC, and each returns the OFFICIAL client, already scoped.
const db    = await env.db('DB')       // @libsql/client     → db.execute(), db.batch()
const cache = await env.kv('CACHE')    // ioredis            → cache.get/set/del/incr
const files = await env.store('FILES') // @aws-sdk/client-s3 → files.send(command)
```

The most common agent mistakes:

- **Wrong package.** `@dotlabshq/flect-sdk@^0.2.0` only. `@getflect/sdk` is
  retired; `@dotlabshq/flect-sdk@0.1.x` is an unrelated old SDK.
- **Missing `uses`.** The service calling `env.db('DB')` must list `"DB"` in its
  `uses`, or it gets `403 BINDING_NOT_USED`.
- **No `await`.** `env.db/kv/store` return promises.
- **Invented methods.** Use each library's own API (`db.execute`,
  `cache.set(k, v, 'EX', ttl)`, `s3.send(new PutObjectCommand(...))`), never
  `db.query()`, `kv.getJson()`, `store.put()`.
- **Missing peers.** Install `@libsql/client`, `ioredis`,
  `@aws-sdk/client-s3` as needed. A store's bucket is
  `await env.bucket('<BINDING>')`.
- **Migrations.** The SDK has none; apply schema on boot (see
  [`flect-db`](../flect-db/SKILL.md)).
- **Secrets in vars.** Put the name in `secrets` and set the value with
  `flect secret set`.

## CLI cheat sheet

```bash
flect login | logout | whoami | ps | ls | doctor
flect org | ws | proj | env                 # tenancy (bare = list; create/get/update/delete)
flect use <org[/ws/proj/env]>               # active org (the file names its own scope)
flect init [--name <n>]                     # scaffold flect.toml
flect bindings [-c <file>]                  # what the file declares, per service
flect dev [-c <file>]                       # flect.local.json for local createEnv()
flect deploy [-c <file>] [--yes]            # check, bind, deploy each service
flect apps                                  # deployed apps of the active scope
flect <db|kv|store> create <name> | list | status <id> | delete <id> [--force]   [-c <file>]
flect store env <id> [--binding FILES]      # a store's S3 connection → .env.flect.local (admins)
flect secret set <service> <NAME> | list [service] | unset <service> <NAME>      [-c <file>]
flect resolve <binding>                     # a binding's manifest, secrets redacted
flect page deploy | ls | status | logs | rollback | stop
flect org members | invite | invites        # membership
flect config set <gatewayUrl|broker|token> <value>
```

`-c <file>` points a command at another file, and so at that file's scope
(`flect deploy -c flect.dev.toml`).

## Safety

- Never type, print or commit a secret value; the user pipes it into
  `flect secret set`.
- `flect login` is a browser sign-in; do not do it for the user.
- Ask before deploying to production, deleting a resource, or passing `--yes`
  (it drops kept vars — read the list first).

## Plan limits

Enforced org-wide; exceeding one returns HTTP `402`. Until billing lands every
org is on the Team plan.

| Plan | Apps | Databases | KV | Storage |
|------|------|-----------|----|---------|
| Hobby | 3 | 1 | 1 | 1 |
| Pro | 10 | 3 | 3 | 3 |
| **Team (current default)** | 25 | 10 | 10 | 10 |
