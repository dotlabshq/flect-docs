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
commands. Select the org with `flect use`; each `flect.toml` names its own
workspace/project/environment.

```bash
flect org create --name Acme --slug acme
flect ws  create --org acme --name Web --slug web
flect proj create --org acme --ws web --name Site --slug site
flect env  create --org acme --ws web --proj site --name production
flect ls                         # inspect the tree (active node marked)
flect use acme/web/site/production
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

This writes a starter `flect.toml` for the active scope and adds
`flect.local.json` / `.flect/` to `.gitignore`. One file is one application in
one scope:

```toml
[app]
name  = "myapp"
scope = "web/site/production"        # workspace/project/environment, in your org

[[databases]]
binding = "DB"                        # the name your code uses: env.db('DB')
id      = "res_??????"                # step 5

[[kv]]
binding = "CACHE"
id      = "res_??????"

[services.web]
image  = "ghcr.io/you/myapp:1.0.0"
port   = 3000
public = true
uses   = ["DB", "CACHE"]              # the only bindings this service may resolve
```

See the [flect.toml reference](/platform/flect-toml/) for every field:
several services, `calls`, vars and secrets.

## 5. Create the resources

Resources are created apart from the app and used by id. Each command creates
the resource in the scope of `./flect.toml` and prints the line to paste:

```bash
flect db create myapp-db
# ✓ created database "myapp-db" in acme/web/site/production (provisioned)
#   flect.toml → id = "res_k7f3a2"
flect kv create myapp-cache
#   flect.toml → id = "res_m2b8td"
```

:::note[Substrate providers must exist for your tenant]
Databases, KV and stores are provisioned through **providers** an operator
registers once per tenant. If `flect doctor` warns `providers — none
registered`, ask your platform operator to register them; apps with no
resources deploy without them.
:::

## 6. Write your app

Your code only needs `@dotlabshq/flect-sdk`. It never sees a URL or a secret.

```bash
npm install @dotlabshq/flect-sdk@^0.2.0 @libsql/client ioredis
```

```typescript
import { createEnv } from '@dotlabshq/flect-sdk'

const env = createEnv()

const db    = await env.db('DB')       // an official @libsql/client
const cache = await env.kv('CACHE')    // an official ioredis client

await db.execute('CREATE TABLE IF NOT EXISTS hits (n INTEGER)')
await cache.set('ping', 'pong')
```

The container listens on `port` and answers `GET /healthz` with 200. Try it
locally first: see [Local development](/guides/local-dev/).

## 7. Deploy

Build and push the image named in `flect.toml` (Flect does not build
containers; the image must be pullable without credentials), then deploy:

```bash
docker build --platform linux/amd64 -t ghcr.io/you/myapp:1.0.0 .
docker push ghcr.io/you/myapp:1.0.0

flect deploy
```

`flect deploy` checks the whole file against its scope first (every id a
resource there, every listed secret set) and stops with nothing changed if
anything is off. Then it binds the resources and deploys each service with only
what it lists:

```
scope: acme/web/site/production  (flect.toml)
resources:
  ✓ DB           res_k7f3a2  bound now
  ✓ CACHE        res_m2b8td  bound now
services:
  ✓ web              deployed → https://myapp-web-3f9k2a.up.flect.run
deploy complete
```

The container gets `FLECT_TOKEN` and `FLECT_BROKER_URL`, and `createEnv()` uses
them to resolve each binding the service lists. No connection string is ever
in the image.

**Next environment:** copy the file to `flect.dev.toml`, set `scope` to
`web/site/dev`, create dev's own resources with `-c flect.dev.toml`, and
`flect deploy -c flect.dev.toml`.

## Next steps

- [Local development](/guides/local-dev/) — `flect dev` + `createEnv()` offline.
- [Examples](/guides/examples/) — small single-purpose apps (a database, a KV store).
- [CLI reference](/cli/) · [SDK reference](/sdk/) · [flect.toml](/platform/flect-toml/)
