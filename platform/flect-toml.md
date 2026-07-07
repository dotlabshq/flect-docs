---
title: flect.toml reference
description: Declare your project's bindings, apps, and services.
sidebar:
  order: 1
---

`flect.toml` lives at the root of your project. It declares the **bindings**
your code resolves (`env.db('DB')`), the **apps** to deploy, and how they're
wired. `flect deploy` reads it to provision, bind, and deploy. The SDK never
reads this file — it's a CLI/deploy artifact.

Scaffold one with `flect init`.

## Single-app form

The common case: one app, a top-level `name`, and its bindings.

```toml
name    = "myapp"
runtime = "node"
port    = 3000

[[databases]]
binding        = "DB"
name           = "myapp-db"
migrations_dir = "migrations"

[[kv]]
binding = "CACHE"
name    = "myapp-cache"

[[stores]]
binding = "FILES"
name    = "myapp-uploads"
```

| Field | Description |
|-------|-------------|
| `name` | Project / app name (required unless using `[app]`). |
| `runtime` | Informational (e.g. `node`). |
| `port` | Port your container listens on. |

## Bindings

Bindings connect a **name your code uses** to a **Flect resource**. The three
kinds map to three code accessors:

| Section | Kind | Accessor |
|---------|------|----------|
| `[[databases]]` | database (sqld/libsql) | `env.db(binding)` |
| `[[kv]]` | KV (Valkey/redis) | `env.kv(binding)` |
| `[[stores]]` | object storage (Garage/S3) | `env.store(binding)` |

Each entry takes:

| Field | Required | Description |
|-------|----------|-------------|
| `binding` | yes | The name your code passes to `env.db()/kv()/store()`, e.g. `"DB"`. |
| `name` | yes | The Flect resource it maps to (the public ref from `flect <kind> create`). |
| `migrations_dir` | no | Databases only — directory holding your SQL migration files. |

Binding names must be unique across all kinds in a project.

```toml
[[databases]]
binding = "USERS_DB"
name    = "users-db"

[[databases]]
binding = "LOGS_DB"
name    = "logs-db"

[[kv]]
binding = "SESSION"
name    = "session-cache"
```

> Resource `name`s come from `flect db create` / `flect kv create` /
> `flect store create`, which print the exact ref to paste here. If a named
> resource doesn't exist yet, `flect deploy` provisions it.

## App-group form

To deploy several apps that share a domain (e.g. an API plus a sidecar service),
use `[app]` + `[[apps]]` instead of a top-level `name`.

```toml
[app]
name   = "clave"
domain = "clave-api.up.flect.run"   # optional: shared / custom domain

[[apps]]
name   = "clave-api"
image  = "ghcr.io/you/clave-api:1.0.0"
port   = 3000
public = true                       # owns the domain (at most one app)

[[apps]]
name   = "auth-service"
image  = "ghcr.io/you/auth:1.0.0"
expose = "/v1/auth"                 # mounted at a path under the shared domain

[vars]
OIDC_ISSUER = "https://nesskey.com"

[[databases]]
binding = "DB"
name    = "clave-db"

[[services]]
binding = "AUTH_SERVICE"            # inject the sibling app's URL as this env var
app     = "auth-service"
```

### `[[apps]]` fields

| Field | Description |
|-------|-------------|
| `name` | App name (required). |
| `image` | Container image (`registry/name:tag`). |
| `port` | Container port. |
| `public` | `true` to own the domain — only one app may be public. |
| `expose` | Mount at a path prefix under the shared domain (e.g. `/v1/auth`). |
| `replicas` | Instance count. |
| `cpuMhz` | CPU allocation, MHz. |
| `memoryMb` | Memory allocation, MB. |

### `[vars]`

Plain environment variables to set on the app(s) at deploy time. (Secrets should
come from your platform's secret store, not here.)

### `[[services]]`

Wire one app to another: inject the resolved URL of app `app` into the
environment as the variable named by `binding`.

| Field | Description |
|-------|-------------|
| `binding` | Env var name to inject. |
| `app` | The sibling app whose URL is injected. |
