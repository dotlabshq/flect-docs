# Flect — Agent Instructions

You are helping a user build, deploy or change a project on **Flect**: a
platform that runs containers and gives them databases, KV stores and object
storage. Two ideas carry everything:

1. **Code never holds a URL or a secret.** It calls `createEnv()` and asks for
   a binding by name (`env.db("DB")`). At runtime the SDK resolves it through
   the broker into the official client (`@libsql/client`, `ioredis`,
   `@aws-sdk/client-s3`), scoped to that one resource.
2. **One `flect.toml` is one application in one scope.** The file names its
   scope, the resources it uses (by id), and its services. Each service gets
   only what it lists. Another environment is another file
   (`flect.dev.toml`).

## Fetch current docs

These instructions are a summary. Before writing a `flect.toml` or running a
command you are unsure of, fetch the reference:

```
https://raw.githubusercontent.com/dotlabshq/flect-docs/main/<path>
```

| What | Path |
|---|---|
| flect.toml (every key, every check) | `platform/flect-toml.md` |
| CLI | `cli/index.md` |
| SDK | `sdk/index.md` |
| Architecture | `platform/index.md` |
| Skills (hub, then one per capability) | `skills/flect/SKILL.md`, `skills/flect-app/SKILL.md`, `skills/flect-db/SKILL.md`, `skills/flect-kv/SKILL.md`, `skills/flect-store/SKILL.md`, `skills/flect-pages/SKILL.md` |

## The model

```
Org → Workspace → Project → Environment        the scope tree
                               ├── resources    databases, KV, stores — each has an id: res_…
                               └── apps         one per service of a flect.toml
```

| In the file | Meaning |
|---|---|
| `[app]` | The application: a `name` and the `scope` it deploys to. |
| `[[databases]]` / `[[kv]]` / `[[stores]]` | A resource it uses: `binding` (the name in code) and `id`. |
| `[services.<name>]` | One microservice: one image. Deploys as the app `<app>-<name>`. |
| `uses` | The bindings that service may resolve. |
| `calls` | The services whose address it gets (`<NAME>_SERVICE_URL`). |
| `vars` / `secrets` | Its configuration, and the names of its secrets. |

## Workflow: a new application

```bash
# 0. Once per machine
npm install -g @dotlabshq/flect-cli          # >= 0.3.2
flect login                                   # SSO in the browser — the user does this
flect use <org>                               # the org; the file picks the rest
flect doctor                                  # every line green before going on

# 1. The scope must exist (workspace/project/environment)
flect ls                                      # see the tree
flect proj create --org <org> --ws <ws> --name Shop --slug shop              # if missing
flect env  create --org <org> --ws <ws> --proj shop --name production         # if missing

# 2. Write flect.toml with [app].scope, then create its resources IN that scope
flect db create shop-db                       # prints: flect.toml → id = "res_k7f3a2"
#    paste the id into the [[databases]] entry

# 3. Secrets: names in the file, values from stdin (never as an argument)
printf %s "$VALUE" | flect secret set api STRIPE_KEY
flect secret list                             # no "not set" lines may remain

# 4. Build and push every image (linux/amd64, public registry)
docker build --platform linux/amd64 -t ghcr.io/acme/shop-api:1.0.0 . && docker push ghcr.io/acme/shop-api:1.0.0

# 5. Deploy
flect deploy                                  # checks everything first; stops with nothing changed on any problem

# 6. Verify
flect apps                                    # every service deployed, the public one with its URL
flect db list                                 # each resource "used by" the services that list it
```

A minimal file:

```toml
[app]
name  = "shop"
scope = "web/shop/production"

[[databases]]
binding = "DB"
id      = "res_k7f3a2"

[services.api]
image   = "ghcr.io/acme/shop-api:1.0.0"
port    = 3000
uses    = ["DB"]
secrets = ["STRIPE_KEY"]

[services.web]
image  = "ghcr.io/acme/shop-web:1.0.0"
port   = 3000
public = true
calls  = ["api"]          # web reads process.env.API_SERVICE_URL
```

**Dev environment.** Copy the file to `flect.dev.toml` and change `scope` to
`…/dev`. Create dev's own resources with `flect db create shop-db -c
flect.dev.toml` and put *their* ids in it. Set its secrets with `-c
flect.dev.toml`, then `flect deploy -c flect.dev.toml`. Dev may run fewer
services. Never reuse a production id: the deploy refuses it.

## Workflow: a change to an existing application

| Change | Do |
|---|---|
| New image version | Change the tag in `image`, build and push it, `flect deploy`. |
| A service needs a database it did not have | Add the binding to its `uses`. If the resource is new: `flect db create`, then add the entry with its id. |
| A service needs another service | Add it to `calls`; read `<NAME>_SERVICE_URL` in code. |
| New secret | Add the name to `secrets`, `flect secret set <service> <NAME>`, `flect deploy`. |
| Rotate a secret | `flect secret set` again, then `flect deploy`: the value reaches the service on deploy. |
| Remove a service | Delete its block and `flect deploy`; it warns that the old app keeps running. Delete that app from the dashboard. |

## Rules that matter

**flect.toml**
- `scope` and every resource `id` are required. The file decides where it
  deploys; `flect deploy --scope` is refused for this format.
- A resource id must belong to the file's scope. Ids are never shared across
  environments.
- A service gets **only** what it lists. If its code calls `env.db("DB")`,
  its `uses` must contain `"DB"`, or the call fails with
  `403 BINDING_NOT_USED`.
- `calls` hands out an address. List only the services it really calls.
- Never put a secret in `vars`. Put its name in `secrets`.
- Binding names are unique per scope. If another application in the
  environment already uses `DB` for a different resource, pick another name.

**Code**
- Install `@dotlabshq/flect-sdk@^0.2.0`. Not `@getflect/sdk` (retired), and
  not `@dotlabshq/flect-sdk@0.1.x` (an unrelated old SDK).
- `env.db/kv/store` are **async**: always `await` them.
- They return the **real** clients. Do not invent `db.query()`,
  `kv.getJson()` or `store.put()`. Use `db.execute`, `cache.set(k, v, "EX",
  ttl)`, `s3.send(new PutObjectCommand(…))`.
- Install the client libraries the bindings need: `@libsql/client`,
  `ioredis`, `@aws-sdk/client-s3`. A store's bucket is
  `process.env.<BINDING>_BUCKET`.
- The SDK has no migrations. Apply your schema on boot (see `flect-db`).
- Read other services at `process.env.<NAME>_SERVICE_URL`; never hard-code a
  host.

**Container**
- Listen on `port`. Answer `GET /healthz` with 200, or set `health =
  "/path"`, or `health = false` for a worker with no HTTP. No traffic arrives
  until the check passes.
- `linux/amd64`, dependencies bundled. The image must be pullable without
  credentials.
- The environment holds only `FLECT_TOKEN`, `FLECT_BROKER_URL`, `PORT`,
  `NODE_ENV`, `APP_PUBLIC_URL` (public and exposed services), the service's
  vars and secrets, and its `*_SERVICE_URL`s. There is no `DATABASE_URL`.

## When the deploy stops

`flect deploy` checks the whole file against the scope before it changes
anything. Read the message; it says what to do.

| Message | Fix |
|---|---|
| `has no id: create one with flect db create …` | Run it and paste the printed `id = "res_…"`. |
| `res_… is not a database in <scope>` | The id is from another scope, or deleted. `flect db list` in this scope. |
| `already has a binding "DB" pointing at res_…` | Another app in the scope owns that name. Rename the binding, in the file and the code. |
| `needs secret X in <scope>: flect secret set …` | Ask the user to pipe the value in. |
| `the last deploy kept A, B, which … no longer has` | Check each name is really unused by that service, or add it back to `vars`/`secrets`. Then `--yes`. |
| `uses "DBB" … did you mean "DB"?` | A typo in the file. |

Never pass `--yes` without reading what it drops.

## What you must not do

- Do not type, print or commit secret values. Ask the user to pipe them into
  `flect secret set`.
- Do not run `flect login` for the user or enter passwords: it is a browser
  sign-in.
- Do not deploy to production, delete resources (`flect db delete`,
  `--force`) or drop vars (`--yes`) without the user's go-ahead.
- Do not edit `~/.flect/config.json`.

## The old format

A `flect.toml` with `[[apps]]` and resources by `name` is the old format. It
still deploys, to the active `flect use` scope, with every app getting every
var and binding, and it warns. Do not write new files in it. To move one,
follow "Moving a file over" in `platform/flect-toml.md`.
