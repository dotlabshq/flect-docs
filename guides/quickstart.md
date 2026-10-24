---
title: Quickstart
description: From flect login to a live app in a few minutes.
sidebar:
  order: 1
---

This walks you from a fresh login to a deployed app with a database and a cache.

## Prerequisites

- [Node.js](https://nodejs.org/) 20+ and the Flect CLI
- [Docker](https://docs.docker.com/get-docker/) and a container registry
  (GitHub Container Registry, Docker Hub, …) to build and push your image

## 1. Install the CLI

```bash
npm install -g @dotlabshq/flect-cli
```

## 2. Log in and point at the platform

```bash
flect login
flect config set gatewayUrl https://flect.cloud   # the hosted control plane
                                                  # (self-hosted: your gateway origin)
```

`flect login` opens your browser for SSO and stores an identity token in
`~/.flect/config.json`. `gatewayUrl` is the single origin the CLI talks to — the
identity plane and the broker both derive from it. Confirm everything is wired:

```bash
flect whoami        # your identity + where you're pointed
flect doctor        # green-checks config, plane, broker, token, and scope
```

## 3. Scopes — where things live

Flect organizes everything as a tree: **Org → Workspace → Project →
Environment**. Your login gives you an org; you create the rest with the noun
commands, then pick an active context with `flect use` — resource and deploy
commands target it.

```bash
flect org create --name Acme --slug acme
flect ws  create --org acme --name Web --slug web
flect proj create --org acme --ws web --name Site --slug site
flect ls                         # inspect the tree (active node marked)
flect use acme/web/site          # set the active context
```

`flect ps` prints the active context (scope, token, broker) at a glance.

### Invite your team

Bring others into the org by email — they don't need an account first. The
invite is pending until they log in and accept; roles rank
**owner > admin > member > viewer**.

```bash
flect org invite dev@acme.com --role member --org acme   # you (owner/admin) send it
# …the invited person, after `flect login`:
flect invites                     # list invitations addressed to you
flect invites accept <id>         # join the org at the invited role
flect org members --org acme      # everyone can now see the roster
```

## 4. Scaffold `flect.toml`

In your project directory:

```bash
flect init
```

This writes a starter `flect.toml` and adds `flect.local.json` / `.flect/` to
`.gitignore`:

```toml
name    = "myapp"
runtime = "node"
port    = 3000
scope   = "myapp/main/prod"      # reserved/ignored — deploy target comes from `flect use` (step 3)

[[databases]]
binding = "DB"
name    = "myapp-db"

[[kv]]
binding = "CACHE"
name    = "myapp-cache"
```

- `scope` is reserved today; the deploy target is your **active context** (`flect use`, step 3).
- `binding` is the name your **code** uses: `env.db('DB')`.
- `name` is the Flect **resource** it maps to.

See the [flect.toml reference](/platform/flect-toml/) for every field.

## 5. Write your app

Your code only needs `@getflect/sdk`. It never sees a URL or a secret.

```bash
npm install @getflect/sdk
```

```typescript
import { createEnv } from '@getflect/sdk'

const env = createEnv()

const db    = await env.db('DB')       // an official @libsql/client
const cache = await env.kv('CACHE')    // an official ioredis client

await db.execute('CREATE TABLE IF NOT EXISTS hits (n INTEGER)')
await cache.set('ping', 'pong')
```

Try it locally first — see [Local development](/guides/local-dev/).

## 6. Deploy

:::note[Substrate providers must exist for your tenant]
`flect deploy` provisions databases, KV, and stores through **providers** an
operator registers once per tenant (which sqld/valkey/garage substrate backs
each kind). If `flect doctor` warns `providers — none registered`, ask your
platform operator to register them before deploying a resource-backed app; apps
with no bindings deploy without providers.
:::

Build and push your image, set it in `flect.toml` (or pass `--image` on the app
entry), then:

```bash
docker build -t ghcr.io/you/myapp:1.0.0 .
docker push ghcr.io/you/myapp:1.0.0

flect deploy
```

`flect deploy` reads `flect.toml`, targets your **active context** (from `flect
use` in step 3), **provisions** any missing resources, **binds** them to your
project, and **deploys** your container to the cluster behind Traefik with an
auto-issued TLS certificate.

```
scope: acme/web/site (active context)
✓ provisioned  database  myapp-db-a1b2c3d4
✓ provisioned  kv        myapp-cache-e5f6g7h8
✓ deployed     myapp     https://myapp-3f9k2a.up.flect.run   running
```

At deploy time Flect injects exactly two env vars into your container —
`FLECT_TOKEN` and `FLECT_BROKER_URL`. `createEnv()` uses them to resolve each
binding to a scoped, short-lived connection at runtime.

## Next steps

- [Local development](/guides/local-dev/) — `flect dev` + `createEnv()` offline.
- [Examples](/guides/examples/) — small single-purpose apps (a database, a KV store).
- [CLI reference](/cli/) · [SDK reference](/sdk/) · [flect.toml](/platform/flect-toml/)
