---
title: "Walkthrough: notes"
description: Build and deploy a real app on Flect — a signed-in database + cache demo — from flect init to a live URL.
sidebar:
  order: 3
---

A complete, copy-paste walkthrough: scaffold **notes** with `flect init`, give it
a database and a cache, add **sign-in** with a bundled auth service, and deploy
the whole thing to a live HTTPS URL. Your code only ever calls `createEnv()` —
no connection strings, no secrets, and (by design) **no platform endpoint ever
appears in your config or reaches the browser**.

By the end you'll have a signed-in, per-user notes app where every read tells you
whether it came from the **database** or the **cache** — so you can watch the
cache warm up, serve hits, expire, and get busted on writes.

## 1. Sign in and pick a scope

```bash
flect login                                    # SSO → ~/.flect/config.json
flect scope create workspace acme
flect scope create project notes --parent <workspace-id>
flect scope create environment prod --parent <project-id>
flect use acme/notes/prod                      # everything below lands here
```

`flect ps` shows the active context; `flect doctor` sanity-checks your setup
(token, broker, scope, providers) if anything looks off.

## 2. Scaffold

```bash
mkdir notes && cd notes
flect init --name notes
```

This writes `flect.toml` with a database and a cache binding, plus a commented
app block. We'll fill it out as we go.

- `binding` is the name your **code** uses (`env.db('DB')`).
- `name` is the Flect **resource** it maps to — provisioned on `flect deploy`.

## 3. Write the app

```bash
npm init -y && npm pkg set type=module
npm install @getflect/sdk @libsql/client ioredis hono @hono/node-server
```

The app has three parts: resolve the database + cache, verify the signed-in
user, and serve an instrumented cache-aside read. `src/index.ts`:

```typescript
import { createHmac, timingSafeEqual } from "node:crypto"
import { Hono } from "hono"
import { getCookie } from "hono/cookie"
import { serve } from "@hono/node-server"
import { createEnv } from "@getflect/sdk"
import type { Client } from "@libsql/client"
import type { Redis } from "ioredis"

const JWT_SECRET = process.env.JWT_SECRET ?? ""
const TTL = 30 // seconds the cached list lives

const env = createEnv()
const db = await env.db<Client>("DB")        // official @libsql/client → notes-db
const cache = await env.kv<Redis>("CACHE")   // official ioredis → notes-cache

await db.execute(
  "CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY AUTOINCREMENT, owner TEXT NOT NULL, text TEXT NOT NULL, created_at INTEGER NOT NULL)",
)

// Verify the auth service's `oidc_token` cookie (HS256) → the signed-in user id.
function ownerFrom(c: import("hono").Context): string | null {
  const token = getCookie(c, "oidc_token")
  if (!token || !JWT_SECRET) return null
  const [header, payload, sig] = token.split(".")
  if (!header || !payload || !sig) return null
  const expected = createHmac("sha256", JWT_SECRET).update(`${header}.${payload}`).digest("base64url")
  const a = Buffer.from(sig), b = Buffer.from(expected)
  if (a.length !== b.length || !timingSafeEqual(a, b)) return null
  const claims = JSON.parse(Buffer.from(payload, "base64url").toString()) as { sub?: string; exp?: number }
  if (claims.exp && claims.exp * 1000 <= Date.now()) return null
  return claims.sub ?? null
}

const app = new Hono()
app.get("/healthz", (c) => c.json({ ok: true, service: "notes" }))
app.get("/me", (c) => c.json({ user: ownerFrom(c) }))

// cache-aside read — fully instrumented so you can SEE where the data came from
app.get("/notes", async (c) => {
  const owner = ownerFrom(c)
  if (!owner) return c.json({ error: "unauthorized" }, 401)
  const listKey = `notes:${owner}`

  const t0 = performance.now()
  const cached = await cache.get(listKey)
  if (cached) {
    const hits = await cache.incr(`hits:${owner}`)   // a KV counter
    const ttl = await cache.ttl(listKey)             // seconds until this copy expires
    const { notes, cachedAt } = JSON.parse(cached)
    return c.json({ notes, source: "cache", ms: +(performance.now() - t0).toFixed(2),
      ageSec: Math.round((Date.now() - cachedAt) / 1000), expiresInSec: ttl, hits })
  }

  const { rows } = await db.execute({
    sql: "SELECT id, text, created_at FROM notes WHERE owner = ? ORDER BY id DESC",
    args: [owner],
  })
  const notes = rows.map((r) => ({ id: Number(r.id), text: String(r.text), created_at: Number(r.created_at) }))
  await cache.set(listKey, JSON.stringify({ notes, cachedAt: Date.now() }), "EX", TTL)
  return c.json({ notes, source: "database", ms: +(performance.now() - t0).toFixed(2), ttlSec: TTL })
})

app.post("/notes", async (c) => {
  const owner = ownerFrom(c)
  if (!owner) return c.json({ error: "unauthorized" }, 401)
  const { text } = await c.req.json<{ text?: string }>()
  if (!text) return c.json({ error: "text required" }, 400)
  await db.execute({ sql: "INSERT INTO notes (owner, text, created_at) VALUES (?, ?, ?)", args: [owner, text, Date.now()] })
  await cache.del(`notes:${owner}`)                  // write busts the cache
  return c.json({ ok: true }, 201)
})

app.delete("/notes/:id", async (c) => {
  const owner = ownerFrom(c)
  if (!owner) return c.json({ error: "unauthorized" }, 401)
  await db.execute({ sql: "DELETE FROM notes WHERE id = ? AND owner = ?", args: [Number(c.req.param("id")), owner] })
  await cache.del(`notes:${owner}`)                  // delete busts the cache too
  return c.json({ ok: true })
})

// bounce sign-in / sign-out to the bundled auth service on the SAME origin
app.get("/login", (c) => c.redirect("/v1/auth/login?redirectTo=/"))
app.get("/logout", (c) => c.redirect("/v1/auth/logout?redirectTo=/"))

serve({ fetch: app.fetch, port: Number(process.env.PORT ?? 3000) })
```

The app never sees a URL or a secret — at deploy time Flect injects only
`FLECT_TOKEN` + `FLECT_BROKER_URL`, and the SDK resolves `DB` and `CACHE` to the
official clients. (A small browser UI that renders the badge, add form, and
delete buttons ships with the full example; it's plain `fetch()` against the
routes above.)

Want to run it locally first, with no broker? See
[Local development](/guides/local-dev/) — `flect dev` writes a `flect.local.json`
that points the same code at a local sqlite/Valkey.

## 4. Add sign-in with a bundled auth service

`notes` scopes every note to a user, so it needs a login. Rather than run a
separate identity service on another domain (and fight cross-site cookies),
**bundle an auth service into the same app** at `/v1/auth`. Because it shares the
origin, the `oidc_token` cookie it sets is same-site — `notes` just reads it.

The auth service is itself a Flect app, and by default it **shares notes'
database and cache** — it resolves the same `DB` / `CACHE` bindings (bindings are
scope-level, so it reaches the very same resources). Auth's tables (`users`,
`organizations`, …) are namespaced by name and never collide with notes' `notes`
table, so a second database would be wasteful. Want a dedicated store instead?
Set `AUTH_DB_BINDING` / `AUTH_KV_BINDING` in `[vars]` and declare those bindings.

Either way, no substrate address ends up in your `flect.toml`. That rule is
**endpoint opacity** — the client side must never know or leak a platform
endpoint; every resource is reached through the SDK + `FLECT_TOKEN`. See
[how resolution works](/platform/#how-resolution-works).

You'll need one thing from your identity provider: an **OIDC (PKCE) client id**
whose redirect URI is `https://notes-<shortid>.up.flect.run/v1/auth/oidc/callback`.
You get the `<shortid>` after the first deploy, so it's fine to deploy once, read
the URL, register the callback, and set the id below.

## 5. Containerize

`Dockerfile`:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install --omit=dev
COPY src ./src
ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000
CMD ["node", "--experimental-strip-types", "src/index.ts"]
```

```bash
docker build --platform linux/amd64 -t ghcr.io/you/notes:0.1.0 .
docker push ghcr.io/you/notes:0.1.0
```

> New GitHub Container Registry packages are **private** by default — make
> `notes` public (or grant the cluster pull access) or the deploy can't pull it.

## 6. Fill out `flect.toml`

Two apps on one origin (notes owns the domain; auth is exposed under `/v1/auth`),
and just **one database + one cache** — the bundled auth shares them:

```toml
name    = "notes"
runtime = "node"
port    = 3000

# The notes app — owns the domain, serves the UI + owner-scoped API.
[[apps]]
name   = "notes"
image  = "ghcr.io/you/notes:0.1.0"
port   = 3000
public = true                       # → https://notes-<shortid>.up.flect.run

# A sibling auth service on the SAME origin at /v1/auth, so its sign-in cookie
# is same-site. It shares notes' DB / CACHE by default — no store of its own to
# declare, and no substrate URL appears here (ADR-0004).
[[apps]]
name   = "auth"
image  = "ghcr.io/dotlabshq/auth-service:0.3.0"
port   = 3000
expose = "/v1/auth"

[vars]
JWT_SECRET     = "<shared-realm-secret>"   # auth signs the cookie, notes verifies it
OIDC_ISSUER    = "https://your-idp.example"
OIDC_SCOPE     = "openid profile email offline_access"
OIDC_CLIENT_ID = "<your-pkce-client-id>"
APP_PUBLIC_URL = "https://notes-<shortid>.up.flect.run"

# one database + one cache, used by both notes and the bundled auth
[[databases]]
binding = "DB"
name    = "notes-db"
[[kv]]
binding = "CACHE"
name    = "notes-cache"
```

## 7. Deploy

```bash
flect deploy
```

```
bindings:
  ✓ DB           bound
  ✓ CACHE        bound
apps:
  ✓ notes            deployed → https://notes-5a7ac3.up.flect.run
  ✓ auth             deployed
deploy complete — apps resolve their bindings with createEnv()
```

`flect deploy` provisioned one database (sqld) and one cache (Valkey), bound them
to your project, and rolled both containers out behind Traefik on one TLS
domain — `notes` on `/`, auth on `/v1/auth`, sharing the same store.

> First time through, deploy once to learn your `notes-<shortid>` hostname, then
> register `https://notes-<shortid>.up.flect.run/v1/auth/oidc/callback` as the
> redirect URI on your OIDC client, set `OIDC_CLIENT_ID` + `APP_PUBLIC_URL`, and
> deploy again.

## 8. See it work

Open `https://notes-<shortid>.up.flect.run` and **Log in** — you'll bounce
through your IdP and land back signed in. Add a note, then hit **Reload** a few
times and watch the badge:

- The first read after a write says **DATABASE** — a fresh query against sqld.
- The next reads (for `TTL` = 30s) say **CACHE**, with the age, time-to-expire,
  latency, and a cache-hit counter — served from Valkey.
- Adding or deleting a note, or hitting **Flush cache**, busts the cached list —
  the next read is **DATABASE** again.

The same flow on the command line (once you have a cookie from the browser):

```bash
H=https://notes-5a7ac3.up.flect.run
C='oidc_token=<paste from your browser cookies>'

curl -s -H "cookie: $C" -X POST $H/notes -H 'content-type: application/json' -d '{"text":"first note"}'
# {"ok":true}

curl -s -H "cookie: $C" $H/notes
# {"notes":[{"id":1,...}],"source":"database","ms":6.2,"ttlSec":30}

curl -s -H "cookie: $C" $H/notes
# {"notes":[...],"source":"cache","ms":0.5,"ageSec":1,"expiresInSec":29,"hits":1}

curl -s -H "cookie: $C" -X DELETE $H/notes/1
# {"ok":true}   ← delete busts the cache; the next read is "database" again
```

The `source` flips from `database` to `cache` and back — every write and delete
invalidates the cached list, and reads warm it again, all resolved through
`createEnv()`.

> Reusing a database from an older version of this app? A schema that changed
> (say, a new `owner` column) won't be applied by `CREATE TABLE IF NOT EXISTS`
> over an existing table. The example self-heals: on boot it checks the columns
> and recreates the table if the shape is stale (the demo data is throwaway).

## What you built

- A signed-in, per-user app whose code only knows `createEnv()` — portable,
  secret-free, and with **no platform endpoint** in its config or its client.
- One Flect-managed database and one cache, provisioned and wired by
  `flect deploy` and shared by both apps.
- A bundled auth service sharing the app's origin *and* its store, so sign-in is
  one same-site cookie away — no separate domain, no cross-site cookie dance, no
  second database to run.
- A live HTTPS endpoint on a generated hostname (set `[app].domain` for your own).

Next: [CLI reference](/cli/) · [SDK reference](/sdk/) · [flect.toml](/platform/flect-toml/)
