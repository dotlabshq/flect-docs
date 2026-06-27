# flect.toml Reference

`flect.toml` lives in the root of your project. The CLI reads it during `flect app deploy` to map binding names to the correct resources.

## Example

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
```

## Fields

### `name`

The app name. Must match the name used in `flect app create`.

```toml
name = "myapp"
```

### `port`

The port your app listens on inside the container. Default: `3000`.

```toml
port = 3000
```

### `runtime`

Informational. Not currently used by the platform.

```toml
runtime = "node"  # node | python | go | ...
```

---

## `[[databases]]`

Define a database binding. You can have multiple.

```toml
[[databases]]
binding        = "DB"
name           = "myapp-db"
migrations_dir = "migrations"
```

| Field | Required | Description |
|-------|----------|-------------|
| `binding` | yes | Env var prefix. `"DB"` → `DB_URL`, `DB_NAMESPACE` |
| `name` | yes | Name of the Flect database resource |
| `migrations_dir` | no | Directory for `flect db migrate` (default: `"migrations"`) |

The `binding` value becomes the prefix for env vars injected into your container:

```
DB_URL        = http://proxy:7001
DB_NAMESPACE  = <your-namespace>
FLECT_TOKEN   = <your-token>
```

---

## `[[kv]]`

Define a KV store binding. You can have multiple.

```toml
[[kv]]
binding = "CACHE"
name    = "myapp-cache"
```

| Field | Required | Description |
|-------|----------|-------------|
| `binding` | yes | Env var prefix. `"CACHE"` → `CACHE_URL` |
| `name` | yes | Name of the Flect KV resource |

The `binding` value becomes the prefix for env vars:

```
CACHE_URL = redis://:token:myapp-cache@proxy:7002
```

---

## Multiple bindings

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

[[kv]]
binding = "RATE_LIMIT"
name    = "rate-limit-store"
```

Each binding maps to its own isolated resource.
