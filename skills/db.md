# Flect Database Skill

## What It Is

Flect databases are SQLite-compatible, powered by **sqld** (libsql). Each database gets its own namespace on the shared sqld server, accessed via HTTP through `flect-proxy` for isolation and auth.

Compatible with: Drizzle ORM, Prisma (libsql adapter), raw libsql client.

## Create a Database

```bash
flect db create <name>          # e.g. flect db create my-db
flect db list                   # show databases in current env
flect db shell <name>           # interactive SQL shell
```

## Bind to an App

In `flect.toml`:

```toml
[[databases]]
binding = "DB"       # becomes DB_URL env var in the app
name    = "my-db"    # database slug from flect db list
```

Multiple databases:
```toml
[[databases]]
binding = "USERS_DB"
name    = "users-db"

[[databases]]
binding = "ANALYTICS_DB"
name    = "analytics-db"
```

## Use in Code

Install SDK: `npm install @flect/sdk`

```typescript
import { db } from '@flect/sdk'

// db() returns a libsql client, pre-configured from DB_URL + FLECT_TOKEN
const client = db()

// Execute queries
const result = await client.execute('SELECT * FROM users WHERE id = ?', [userId])
const rows = result.rows  // typed rows

// Transactions
await client.batch([
  'INSERT INTO users (name) VALUES ("Alice")',
  'INSERT INTO audit_log (action) VALUES ("user_created")',
])
```

### With Drizzle ORM

```typescript
import { drizzle }   from 'drizzle-orm/libsql'
import { db as flectDb } from '@flect/sdk'

const db = drizzle(flectDb())

// Use normally
const users = await db.select().from(usersTable)
```

### With Raw libsql

```typescript
import { createClient } from '@libsql/client'

const client = createClient({
  url:   process.env['DB_URL']!,
  authToken: process.env['FLECT_TOKEN'],
})
```

## Schema Migrations

Run migrations before app start. Example with Drizzle:

```typescript
// src/migrate.ts
import { migrate } from 'drizzle-orm/libsql/migrator'
import { drizzle } from 'drizzle-orm/libsql'
import { db }      from '@flect/sdk'

await migrate(drizzle(db()), { migrationsFolder: './drizzle' })
```

Or use `flect db shell` to run SQL directly:

```bash
flect db shell my-db
> CREATE TABLE IF NOT EXISTS users (id TEXT PRIMARY KEY, name TEXT);
```

## Local Development

For local dev without a deployed environment, use libsql local file:

```bash
# .env.local
DB_URL=file:./local.db
# FLECT_TOKEN not needed locally (no proxy)
```

```typescript
import { createClient } from '@libsql/client'

const client = createClient({
  url: process.env['DB_URL'] ?? 'file:./local.db',
  authToken: process.env['FLECT_TOKEN'],
})
```

## Limits

- SQLite semantics: no concurrent writes, best for read-heavy workloads
- Max DB size: determined by sqld instance storage
- Namespace isolation: each database is fully isolated from others
