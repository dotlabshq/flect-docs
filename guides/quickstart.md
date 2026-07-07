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
npm install -g @flect/cli
```

## 2. Log in

```bash
flect login
```

This opens your browser for SSO. On success the CLI stores an identity token in
`~/.flect/config.json`. Check where you're pointed anytime with:

```bash
flect whoami
```

## 3. Pick a scope

Flect organizes everything as a tree: **Org → Workspace → Project →
Environment**. Your login gives you an org; create the rest and select an active
scope. Resources and deploys always target the active scope.

```bash
flect scope create workspace web
flect scope create project site --parent <workspace-id>
flect scope create environment prod --parent <project-id>

flect scope list                 # shows the tree; the active scope is marked
flect use web/site/prod          # switch by slug path (or pass a scope id)
```

`flect ps` prints the active context (scope, token, broker) at a glance.

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

[[databases]]
binding = "DB"
name    = "myapp-db"

[[kv]]
binding = "CACHE"
name    = "myapp-cache"
```

- `binding` is the name your **code** uses: `env.db('DB')`.
- `name` is the Flect **resource** it maps to.

See the [flect.toml reference](/platform/flect-toml/) for every field.

## 5. Write your app

Your code only needs `@flect/sdk`. It never sees a URL or a secret.

```bash
npm install @flect/sdk
```

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()

const db    = await env.db('DB')       // an official @libsql/client
const cache = await env.kv('CACHE')    // an official ioredis client

await db.execute('CREATE TABLE IF NOT EXISTS hits (n INTEGER)')
await cache.set('ping', 'pong')
```

Try it locally first — see [Local development](/guides/local-dev/).

## 6. Deploy

Build and push your image, set it in `flect.toml` (or pass `--image` on the app
entry), then:

```bash
docker build -t ghcr.io/you/myapp:1.0.0 .
docker push ghcr.io/you/myapp:1.0.0

flect deploy
```

`flect deploy` reads `flect.toml`, **provisions** any missing resources,
**binds** them to your project, and **deploys** your container to the cluster
behind Traefik with an auto-issued TLS certificate.

```
✓ provisioned  database  myapp-db-a1b2c3d4
✓ provisioned  kv        myapp-cache-e5f6g7h8
✓ deployed     myapp     https://myapp-3f9k2a.up.flect.run   running
```

At deploy time Flect injects exactly two env vars into your container —
`FLECT_TOKEN` and `FLECT_BROKER_URL`. `createEnv()` uses them to resolve each
binding to a scoped, short-lived connection at runtime.

## Next steps

- [Local development](/guides/local-dev/) — `flect dev` + `createEnv()` offline.
- [Notes API example](/guides/notes-api/) — a full app with DB + cache.
- [CLI reference](/cli/) · [SDK reference](/sdk/) · [flect.toml](/platform/flect-toml/)
