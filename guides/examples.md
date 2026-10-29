---
title: Examples
description: Small, single-purpose apps that show what Flect code looks like.
sidebar:
  order: 3
---

Each example is a **complete, deployable app** whose entire data layer is
`createEnv()` — no URLs, no secrets. They're deliberately small: one Flect
capability each, so you can see exactly how the code looks.

## Notes — a database

The smallest `env.db` example: a public notes app. The whole data layer is three
lines.

```ts
const env = createEnv()
const db  = await env.db('DB')     // an official @libsql/client
await db.execute('SELECT ...')
```

`flect.toml`: one database, one service that uses it.

```toml
[app]
name  = "notes"
scope = "examples/notes/production"

[[databases]]
binding = "DB"
id      = "res_……"                 # flect db create notes-db

[services.web]
image  = "ghcr.io/getflect/flect-example-notes:0.2.0"
port   = 3000
public = true
uses   = ["DB"]
```

- **Live:** https://flect-example-notes-6b6e4f.up.flect.run
- **Source:** `projects/flect-example-notes` in the platform repo

## Counter — a KV store

The smallest `env.kv` example: a shared counter. Atomic `INCR`/`DECR`, no
read-modify-write.

```ts
const env = createEnv()
const kv  = await env.kv('KV')     // an official ioredis client
await kv.incr('counter')
```

```toml
[app]
name  = "counter"
scope = "examples/counter/production"

[[kv]]
binding = "KV"
id      = "res_……"                 # flect kv create counter-kv

[services.web]
image  = "ghcr.io/getflect/flect-example-counter:0.2.0"
port   = 3000
public = true
uses   = ["KV"]
```

- **Live:** https://flect-example-counter-5cdcd0.up.flect.run
- **Source:** `projects/flect-example-counter` in the platform repo

## The shape of every example

```bash
flect use <org>                                   # the file names its own scope
flect proj create --org <org> --ws <ws> --name app --slug app      # if the scope is missing
flect init                                        # scaffold flect.toml
flect db create app-db                            # paste the printed id = "res_…"
docker build --platform linux/amd64 -t ghcr.io/you/app:0.1.0 . && docker push ghcr.io/you/app:0.1.0
flect deploy                                      # checks, binds by id, deploys
```

Deploy injects only `FLECT_TOKEN` + `FLECT_BROKER_URL`; the SDK resolves each
binding the service lists in `uses`. For the low-level SDK snippets (db / kv / store / together),
see the `examples/` folder in the platform repo.
