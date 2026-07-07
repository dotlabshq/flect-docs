# Flect — Agent Instructions

You are helping a user build and deploy a project on **Flect**, a developer
platform for apps, databases, KV stores, and object storage. The defining idea:
application code calls `createEnv()` and asks for a binding by name — it never
handles URLs, ports, or secrets. The platform provisions resources and resolves
each binding to a scoped, official client at runtime.

## Fetch current docs

Always fetch the latest docs before answering. Raw content is at:

```
https://raw.githubusercontent.com/dotlabshq/flect-docs/main/<path>
```

Key files:
- CLI: `cli/index.md`
- SDK: `sdk/index.md`
- Architecture: `platform/index.md`
- flect.toml: `platform/flect-toml.md`
- Quickstart: `guides/quickstart.md`

## Platform overview

Resources live in a hierarchy: **Org → Workspace → Project → Environment**. Each
environment can hold multiple **Apps** (containers), **Databases** (sqld/libsql),
**KV stores** (Valkey), and **Object stores** (Garage/S3).

At deploy time Flect injects only `FLECT_TOKEN` and `FLECT_BROKER_URL`. The SDK
uses them to resolve bindings through the broker into short-lived, scoped
connections. Nothing else — no `DB_URL`, no credentials — is ever put in the
image or the code.

## Typical workflow

```
1. flect login
2. flect use <workspace>/<project>/<environment>   # active scope
3. flect init                                       # scaffold flect.toml
4. Declare bindings in flect.toml
5. Write app code with createEnv()
6. flect dev        # optional: run locally with flect.local.json
7. flect deploy     # provisions resources, binds, deploys the container
```

Create resources explicitly with `flect db|kv|store create <name>` (each prints
the ref to paste into flect.toml), or let `flect deploy` provision anything named
in flect.toml that doesn't exist yet.

## flect.toml

Single-app form:

```toml
name    = "my-app"
runtime = "node"
port    = 3000

[[databases]]
binding        = "DB"           # the name code passes to env.db()
name           = "my-db"        # the Flect resource
migrations_dir = "migrations"   # optional

[[kv]]
binding = "CACHE"
name    = "my-cache"

[[stores]]
binding = "FILES"
name    = "my-uploads"
```

App-group form (multiple apps / shared domain) uses `[app]` + `[[apps]]` +
optional `[vars]` and `[[services]]`. See `platform/flect-toml.md`.

## SDK usage

Install: `npm install @flect/sdk`

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()

// Each accessor is async and returns the OFFICIAL client, already scoped.
const db    = await env.db('DB')       // @libsql/client  → db.execute(), db.batch()
const cache = await env.kv('CACHE')    // ioredis         → cache.get/set/del/...
const files = await env.store('FILES') // @aws-sdk/client-s3 S3Client → s3.send(cmd)
```

Rules to get right:
- `env.db/kv/store` are **async** — always `await` them.
- They return the **real** libsql / ioredis / S3 clients. Do NOT invent
  `db.query()`, `kv.getJson()`, or `store.put()` wrappers — use each library's
  own API.
- The binding string must match a `binding` in flect.toml.
- Install the client library your bindings need as a peer dependency:
  `@libsql/client`, `ioredis`, `@aws-sdk/client-s3`.

## Deploy requirements (apps)

- The container listens on the `port` in flect.toml.
- It responds to `GET /healthz` with 200 (Traefik health check).
- Build for `linux/amd64`; bundle dependencies into the image.
- Public apps get `https://<name>-<shortid>.up.flect.run`; set `[app].domain`
  for a custom domain and point a CNAME at it.

## Skills

For component detail, load the relevant skill:
- Database: `skills/db.md`
- KV store: `skills/kv.md`
- Object storage: `skills/store.md`
- App deploy: `skills/app.md`
