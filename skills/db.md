---
title: Databases
description: SQLite-compatible databases via sqld/libsql, resolved as an official @libsql/client.
---

## What it is

Flect databases are SQLite-compatible, powered by **sqld** (libsql). Each
database is an isolated sqld namespace. `env.db(binding)` hands you an official
[`@libsql/client`](https://github.com/tursodatabase/libsql-client-ts) already
scoped to that namespace — no URL or token in your code.

## Create

```bash
flect db create my-db          # prints the public ref for flect.toml
flect db list
flect db status <ref>
flect db delete <ref>
```

`flect deploy` also creates any database named in `flect.toml` that doesn't
exist yet.

## Bind

```toml
[[databases]]
binding        = "DB"
name           = "my-db"
migrations_dir = "migrations"   # optional: where your .sql files live
```

## Use in code

```typescript
import { createEnv } from '@flect/sdk'

const env = createEnv()
const db  = await env.db('DB')   // binding name from flect.toml

// SELECT
const { rows } = await db.execute('SELECT id, name FROM users WHERE active = ?', [1])

// INSERT / UPDATE / DELETE — parameterized
await db.execute({
  sql: 'INSERT INTO users (id, name) VALUES (?, ?)',
  args: [crypto.randomUUID(), 'Alice'],
})

// Atomic batch
await db.batch([
  { sql: 'INSERT INTO posts (id, title) VALUES (?, ?)', args: [id, title] },
  { sql: 'UPDATE users SET post_count = post_count + 1 WHERE id = ?', args: [userId] },
])
```

It's the standard client, so it works directly with Drizzle:

```typescript
import { drizzle } from 'drizzle-orm/libsql'
const orm = drizzle(await env.db('DB'))
```

## Isolation

Each request carries the resource's sqld namespace as an `x-namespace` header, so
a binding only ever reads and writes its own data — enforced by sqld, not by
convention.

## Local development

```bash
flect dev     # writes flect.local.json; createEnv() resolves DB locally
```

By default a local database is `:memory:`. For a persistent local sqld, set
`DB_URL` (see [Local development](/guides/local-dev/)):

```bash
docker run -d -p 8080:8080 ghcr.io/tursodatabase/libsql-server:latest \
  /bin/sqld --enable-namespaces
# DB_URL=http://localhost:8080
```

## Notes

- SQLite write semantics apply (a single writer at a time).
- Migrations: keep your `.sql` files in `migrations_dir` and apply them from your
  app or tooling using the same `@libsql/client` (`db.execute`/`db.batch`).
