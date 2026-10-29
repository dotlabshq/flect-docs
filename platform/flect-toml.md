---
title: flect.toml reference
description: One application in one scope — its services, the resources they use, and what each service gets.
sidebar:
  order: 1
---

`flect.toml` lives at the root of your project. **One file is one application
in one scope.** It names the scope it deploys to, the resources the application
uses (by id), and its services: what each one runs and what it gets. `flect
deploy` reads it. The SDK never does.

Another environment is another file: `flect.toml` for production,
`flect.dev.toml` for dev, each with its own scope and resource ids. Deploy the
other one with `flect deploy -c flect.dev.toml`. Nothing is shared or
inherited between them, so dev can run fewer services than production, or
different images.

```toml
[app]
name  = "shop"
scope = "web/shop/production"         # workspace/project/environment, in your org

# ── Resources: created apart, used by id ──────────────────────
[[databases]]
binding = "DB"                         # code: env.db("DB")
id      = "res_k7f3a2"                 # from `flect db create`

[[kv]]
binding = "CACHE"
id      = "res_m2b8td"

# ── For every service ─────────────────────────────────────────
[vars]
LOG_LEVEL = "info"

# ── Services ──────────────────────────────────────────────────
[services.api]
image   = "ghcr.io/acme/shop-api:1.4.0"
port    = 3000
uses    = ["DB", "CACHE"]              # the only bindings it may resolve
secrets = ["STRIPE_KEY"]               # values: flect secret set api STRIPE_KEY

[services.api.vars]
CURRENCY = "EUR"

[services.web]
image  = "ghcr.io/acme/shop-web:1.4.0"
port   = 3000
public = true                          # owns the domain
calls  = ["api"]                       # gets API_SERVICE_URL; nothing else
```

## The rule: a service gets only what it lists

| Key | What the service gets | Enforced by |
|---|---|---|
| `uses` | The bindings it may resolve. Any other gets `403 BINDING_NOT_USED`. | the broker |
| `calls` | The address of each listed service, as `<SERVICE>_SERVICE_URL`. | the deploy |
| `[vars]` + `[services.X.vars]` | Plain configuration. The service's own value wins. | the deploy |
| `secrets` | The listed secrets, read from the secret store. | the broker |

A service that lists nothing gets nothing: no bindings, no addresses, no
secrets. `web` above cannot open the database even though it is in the same
application.

## `[app]`

| Field | Required | Description |
|---|---|---|
| `name` | yes | The application's name. Services deploy as `<name>-<service>`. |
| `scope` | yes | `workspace/project/environment` (1–3 segments) inside the org you selected with `flect use <org>`. The file decides where it deploys; `--scope` is refused. |
| `domain` | no | A custom domain for the public service. Point a CNAME at its generated host. |
| `region` | no | Only `eu` exists today. |

## Resources: `[[databases]]`, `[[kv]]`, `[[stores]]`

A resource exists apart from any application. It belongs to the scope it was
created in and is used by writing its id. Deploying or deleting an application
never creates or removes one.

| Field | Required | Description |
|---|---|---|
| `binding` | yes | The name code asks for: `env.db("DB")`, `env.kv("CACHE")`, `env.store("FILES")`. |
| `id` | yes | `res_…` from `flect db\|kv\|store create`: a resource of this file's scope. |
| `migrations_dir` | no | Databases only: where your SQL migrations live. |

```bash
flect db create shop-db               # in the scope of ./flect.toml
# ✓ created database "shop-db" in acme/web/shop/production (provisioned)
#   flect.toml → id = "res_k7f3a2"
```

- **Scope-bound.** An id from another environment, project or workspace is
  refused: `res_p91xq4 is not a database in acme/web/shop/production`. A dev
  service can never be pointed at a production database, even by copying its
  id.
- **Sharing.** Two applications in the same scope share a resource by writing
  the same id.
- **Binding names are unique in a scope.** Two applications in one
  environment cannot both name different databases `DB`; the deploy stops and
  says which resource the name already points at.
- **Deleting.** `flect db delete <id>` is refused while a binding points at
  the resource; `--force` removes the bindings first.

## Services: `[services.<name>]`

`<name>` is lower-case letters, digits and `-`. It is what `calls` names and
what the env var is made from (`api` → `API_SERVICE_URL`).

| Field | Description |
|---|---|
| `image` | Container image, `registry/name:tag`. Build and push it first; the registry must be public. |
| `port` | The port the container listens on. |
| `public` | `true` to own the domain and get `https://<app>-<service>-<shortid>.up.flect.run`. At most one service. |
| `expose` | A path under the public service's host, e.g. `/v1/auth`. The prefix is stripped before the request reaches the service. |
| `replicas`, `cpuMhz`, `memoryMb` | Size. |
| `health` | Health check: a path, a table, or `false`. See below. |
| `uses` | Bindings it may resolve. |
| `calls` | Services of this file whose address it gets. Not itself. |
| `secrets` | Names of the secrets it gets. |
| `vars` | `[services.<name>.vars]`: its own plain vars, on top of `[vars]`. |
| `name` | The deployed app's name, when it is not `<app>-<service>`. Only to keep an app that already runs under another name. |

A service with no `public` and no `expose` is private: reachable only by the
services that `call` it.

### `health`

By default the platform probes `GET /healthz` every 15 s with a 3 s timeout.
**While it fails, no traffic reaches the service and the services that call it
get an empty address.**

```toml
[services.api]
health = "/api/health"         # another path

[services.slow]
health = { path = "/status", interval = "30s", timeout = 5 }

[services.worker]
health = false                 # no HTTP server at all
```

### `vars` and `secrets`

- `vars` are for configuration you could commit. A name that looks like a
  secret (`*_SECRET`, `*_KEY`, `*_TOKEN`, `*_PASSWORD`) gets a warning.
- `secrets` holds names only. Values are set per service and per scope from
  stdin, so they stay out of the file, the shell history and the event log:

  ```bash
  printf %s "$STRIPE_KEY" | flect secret set api STRIPE_KEY
  flect secret list            # names only; shows any listed one not set yet
  ```

- A name cannot be both a var and a secret.
- Setting a value in dev never sets it in production. A new value reaches the
  service on its next deploy.

## What `flect deploy` checks

Before it changes anything, `flect deploy` reads the target scope and checks
the whole file. Any problem stops the deploy with nothing changed:

| Found | Result |
|---|---|
| `uses`/`calls` naming something the file does not define | stop, with "did you mean" |
| A resource entry without `id` | stop, with the `flect db create` command |
| An `id` that is not a live resource of that kind in the scope | stop |
| A binding name the scope already points at another resource | stop |
| A listed secret with no value in the scope | stop, with the `flect secret set` command |
| Vars the last deploy kept that the file no longer has | stop, listing them; `--yes` drops them |
| A resource no service uses | warning |
| A secret-looking name in `vars` | warning |
| A service of this application running in the scope but no longer in the file | warning (it keeps running) |

Then it binds the resources, declares every service, and deploys each one with
only its own `uses`, `calls`, vars and secrets.

## `[page]` — a static site

A page is a git repository the platform clones, builds and serves. It has no
runtime and no bindings, and it sits in a file of its own.

```toml
[page]
name     = "docs"
repo     = "https://github.com/acme/docs"   # required, public
ref      = "main"                            # branch, tag or commit
template = "docs"                            # Starlight docs template
title    = "Acme Docs"
github   = "https://github.com/acme/docs"
# domain = "docs.acme.com"
```

Deploy it with `flect page deploy`; it goes to the active scope (`flect use`).
A repo with its own `astro.config.*` or a `[page.build]` command builds with
that instead of the template. Every build is published to its own directory
and swapped in atomically, so a failed build never touches the live site and
`flect page rollback` is a pointer switch.

## The old format

Files written before this format (`[[apps]]`, `[[services]]` entries,
resources by `name`) still deploy, with the old behaviour and a warning:

- they deploy to the active scope (`flect use`), and `scope` in the file is
  ignored;
- resources are created by `name` when missing;
- every app gets every var and every service address, and may resolve every
  binding of its scope.

A file is one format or the other: `[services.<name>]` together with
`[[apps]]` is an error. The old format goes away in v0.4.

Moving a file over:

1. `flect db list` (and `kv`, `store`) for each resource's `res_` id.
2. Turn each `[[apps]]` into a `[services.<name>]`. Set `name` when the app's
   current name is not `<app>-<service>`, so the running app is kept.
3. List `uses`, `calls` and `secrets` per service; keep only the vars each one
   reads.
4. Set the secrets with `flect secret set`.
5. `flect deploy` once: it stops and lists the vars that would be dropped.
   Check them, then `flect deploy --yes`.
