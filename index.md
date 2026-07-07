---
title: Flect
description: Ship apps, databases, KV, and object storage on your own infrastructure — your code only calls createEnv().
template: splash
hero:
  tagline: Write createEnv(). Deploy. Flect wires up the databases, caches, and buckets — no URLs, ports, or secrets in your code.
  actions:
    - text: Get started
      link: /guides/quickstart/
      icon: right-arrow
    - text: CLI reference
      link: /cli/
      icon: open-book
      variant: minimal
---

## What Flect is

Flect is a developer platform for deploying apps and their backing resources —
**databases, KV stores, and object storage** — on your own infrastructure. You
declare what your project needs in `flect.toml`, and Flect provisions it, binds
it, and deploys your containers behind TLS.

Your application code stays clean: it calls `createEnv()` and asks for a binding
by name. It never sees a connection string, a port, or a credential.

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()
const db  = await env.db('DB')        // an official @libsql/client
const rows = await db.execute('SELECT * FROM notes')
```

## The whole model in one screen

```bash
flect login                    # SSO — stored in ~/.flect/config.json
flect use acme/web/prod        # pick an org / workspace / project / environment
flect init                     # scaffold flect.toml
flect deploy                   # provision resources, bind them, deploy your app
```

At deploy time Flect injects exactly two environment variables into your
container — `FLECT_TOKEN` and `FLECT_BROKER_URL`. Everything else (the database
URL, the cache prefix, the bucket keys) is resolved at runtime through the
broker, scoped to your project, and never written into your image.

## What you get

- **Apps** — deploy any Docker image; get a public HTTPS URL with an
  auto-issued TLS certificate.
- **Databases** — SQLite-compatible via sqld/libsql, isolated per resource.
- **KV** — Redis-compatible via Valkey, key-prefixed per resource.
- **Object storage** — S3-compatible via Garage, one bucket + key pair per resource.
- **SDK** — `@flect/sdk` hands you the *official* client for each resource
  (`@libsql/client`, `ioredis`, `@aws-sdk/client-s3`), pre-wired and scoped.
- **Scopes** — an Org → Workspace → Project → Environment tree, so the same
  code deploys to `dev`, `staging`, and `prod` with isolated resources.

## Next steps

- [Quickstart](/guides/quickstart/) — from `flect login` to a live app.
- [Local development](/guides/local-dev/) — run `createEnv()` with no broker.
- [SDK reference](/sdk/) — `createEnv()` and the resource clients.
- [CLI reference](/cli/) — every command.
- [Platform architecture](/platform/) — how the control plane works.
