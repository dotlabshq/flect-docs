---
name: flect-db
description: >-
  Flect database — SQLite-compatible (sqld/libsql), resolved as an official
  @libsql/client via env.db(). Use when adding a database, writing queries,
  batches, or migrations, using Drizzle, or binding [[databases]] in flect.toml.
---

# Flect · Databases

Part of the [flect](../flect/SKILL.md) skill set.

## What it is

Flect databases are SQLite-compatible, powered by **sqld** (libsql). Each
database is an isolated sqld namespace. `env.db(binding)` hands you an official
[`@libsql/client`](https://github.com/tursodatabase/libsql-client-ts) already
scoped to that namespace — no URL or token in your code.

## Create

```bash
flect db create my-db          # in the scope of ./flect.toml (-c <file> for another)
# ✓ created database "my-db" in acme/web/shop/production (provisioned)
#   flect.toml → id = "res_k7f3a2"
flect db list                  # id, name, status, and the services that use it
flect db status <id>
flect db delete <id>           # refused while bound; --force unbinds first
```

A database belongs to the scope it was created in. Dev and production each get
their own, with their own ids. `flect deploy` never creates one.

## Bind

```toml
[[databases]]
binding        = "DB"            # the name code passes to env.db()
id             = "res_k7f3a2"    # from flect db create
migrations_dir = "migrations"    # optional: where your .sql files live

[services.api]
uses = ["DB"]                    # only services that list it may resolve it
```

Then list the binding in the `uses` of every service whose code calls it.
A service that does not list it cannot resolve it (`403 BINDING_NOT_USED`).

## Use in code

```typescript
import { createEnv } from '@dotlabshq/flect-sdk'

const env = createEnv()
const db  = await env.db('DB')   // binding name from flect.toml

// SELECT (parameterized)
const { rows } = await db.execute('SELECT id, name FROM users WHERE active = ?', [1])

// INSERT / UPDATE / DELETE
await db.execute({ sql: 'INSERT INTO users (id, name) VALUES (?, ?)', args: [crypto.randomUUID(), 'Alice'] })

// Atomic batch
await db.batch([
  { sql: 'INSERT INTO posts (id, title) VALUES (?, ?)', args: [id, title] },
  { sql: 'UPDATE users SET post_count = post_count + 1 WHERE id = ?', args: [userId] },
])
```

Standard client → Drizzle works directly:

```typescript
import { drizzle } from 'drizzle-orm/libsql'
const orm = drizzle(await env.db('DB'))
```

**Don't invent methods** — it's the real `@libsql/client`. Use `db.execute` /
`db.batch`, not `db.query()`.

## Migrations

The new SDK has **no `runMigrations`** — apply schema yourself. Keep `.sql` files
in `migrations_dir` and run them on boot with the same client (idempotent DDL):

```typescript
import { readFileSync, readdirSync } from 'node:fs'
for (const f of readdirSync('./migrations').filter(f => f.endsWith('.sql')).sort()) {
  await db.executeMultiple(readFileSync(`./migrations/${f}`, 'utf8'))
}
```

Use `CREATE TABLE IF NOT EXISTS`. If a database can be reused across app versions,
self-heal a changed shape (check `PRAGMA table_info(...)`, recreate if stale).

## Isolation

Each request carries the resource's sqld namespace as an `x-namespace` header, so
a binding only ever reads/writes its own data — enforced by sqld, not convention.

## Local development

```bash
flect dev     # writes flect.local.json; createEnv() resolves DB locally
```

Local default is `:memory:`. For persistent local sqld, run
`ghcr.io/tursodatabase/libsql-server` with `--enable-namespaces` and set `DB_URL`
(see [Local development](/guides/local-dev/)).

## Notes

- SQLite write semantics apply (a single writer at a time).

## Related skills

- [flect-app](../flect-app/SKILL.md) — deploy the app that uses this DB.
- [flect-kv](../flect-kv/SKILL.md) — pair a cache with the DB (cache-aside).
- [flect](../flect/SKILL.md) — the model, CLI, scopes.
