---
title: CLI Reference
description: Every flect command — login, tenancy, resources, deploy, pages, and config.
---

Install the CLI:

```bash
npm install -g @dotlabshq/flect-cli
```

The CLI is a client of the Flect control plane (the *broker*). It reads its
connection from, in order of precedence: command flags → environment
(`FLECT_URL`, `FLECT_BROKER_URL`, `FLECT_TOKEN`) → `~/.flect/config.json` →
defaults.

**Point the CLI at your platform once.** A single gateway origin fronts the
whole plane — identity at `/v1/*`, the broker at `/`. Set it after logging in
and every command (identity *and* broker) derives from it:

```bash
flect config set gatewayUrl https://flect.cloud   # the hosted control plane
                                                  # (self-hosted: your gateway origin)
flect doctor                                       # verify plane + broker + token + scope
```

---

## Auth

### `flect login`

Sign in via SSO and store an identity token in `~/.flect/config.json`.

```bash
flect login
flect login --auth https://api.example.com   # override the auth base
```

### `flect logout`

Clear the stored token.

```bash
flect logout
```

### `flect whoami`

Show who you are and where the CLI is pointed (tenant, identity, broker, active
scope).

```bash
flect whoami
```

---

## Tenancy — org, workspace, project, environment

Everything lives in a tree: **Org → Workspace → Project → Environment**. Tenancy
is managed by the identity plane (org-service); the CLI has one noun command per
level. Your login gives you an org; you create the rest. A `flect.toml` names
its own scope inside the active org, and commands run next to one act on that
scope. Without a file, they act on the **active context** you set with
`flect use`.

### `flect org` · `flect ws` · `flect proj` · `flect env`

Each noun command lists on its own and takes `create` / `get` / `update` /
`delete`. Parents are named, not passed as ids.

```bash
flect org  create --name Acme --slug acme         # (bare `flect org` lists)
flect ws   create --org acme --name Web --slug web
flect proj create --org acme --ws web --name Site --slug site
flect env  create --org acme --ws web --proj site --name prod   # `use` auto-creates `production`
```

### `flect use <path>`

Set the active org / workspace / project / environment. A services-format
`flect.toml` only needs the org; it names its own scope. Commands with no such
file (and old-format files) target the whole path, sent to the broker as
`X-Org-Id` + `X-Flect-Scope`.

```bash
flect use acme               # just the org
flect use acme/web/site/prod # down to an environment
flect use                    # interactive picker
```

### `flect ls`

Print the whole org → workspace → project tree, active node marked.

```bash
flect ls
```

### `flect ps`

Show the active context (org path, token, broker, config path) as a compact card.

### `flect access`

Inspect the PDP that authorizes your actions (roles resolved by org-service).

```bash
flect access role <scopeId>          # your effective role at a scope
flect access check <app> <action> <scopeId>
```

### `flect org members`

Manage who belongs to an org and at what role. Roles rank
**owner > admin > member > viewer**; the PDP inherits an org role down the whole
scope subtree. Adding an existing member re-assigns their role (upsert), so this
is also how you promote or demote. Owner/admin only.

```bash
flect org members                                   # (bare) list members of the active org
flect org members --org acme                        # list a specific org
flect org members add <userId> --role admin --org acme   # add or re-role by user id
flect org members rm  <userId> --org acme           # remove a member
```

### `flect org invite` · `flect org invites`

Invite someone by **email** — they don't need an account yet. The invite stays
pending until that person logs in and accepts; matching is by the verified
`email` claim in their token, case-insensitive. One pending invite per
(org, email) — re-inviting returns the existing one. Owner/admin only.

```bash
flect org invite <email> --role member --org acme   # send a pending invitation
flect org invites --org acme                        # (bare) list pending invites for the org
flect org invites revoke <id> --org acme            # revoke a pending invite (alias: rm)
```

### `flect invites` — invitations addressed to you

The invitee side. After `flect login`, list and accept invitations sent to your
email; accepting joins you to the org at the invited role (no re-login needed —
org-service reads your role from membership on each request).

```bash
flect invites                # (bare) list invitations addressed to you
flect invites accept <id>    # accept and join the org
```

---

## Project files

Every command that reads `flect.toml` takes `-c, --config <file>` for another
one. One file is one scope, so `-c flect.dev.toml` is how you act on dev.

### `flect init`

Scaffold a `flect.toml` in the current directory (and git-ignore the local dev
files).

```bash
flect init
flect init --name myapp        # default: the directory name
```

### `flect bindings`

What the file declares: its scope, resources, and per service what it uses,
calls and which secrets it gets.

```bash
flect bindings
flect bindings -c flect.dev.toml
```

### `flect dev`

Generate `flect.local.json` from `flect.toml` so `createEnv()` resolves bindings
locally with no broker. See [Local development](/guides/local-dev/).

```bash
flect dev
```

---

## Resources

Each resource kind has the same four subcommands. They act on the scope of the
`flect.toml` in the current directory (or `-c <file>`), and on the active
context when there is none. `<id>` is the resource's `res_…` id (its name and
full `resource-…` id work too).

```bash
flect db    create <name> | list | status <id> | delete <id> [--force]   # sqld / libsql
flect kv    create <name> | list | status <id> | delete <id> [--force]   # Valkey / redis
flect store create <name> | list | status <id> | delete <id> [--force]   # Garage / S3
```

- **`create`** provisions a resource in that scope and prints the line to put
  in `flect.toml`.
- **`list`** shows the scope's resources: id, name, status, and the services
  whose `uses` name them.
- **`status <id>`** prints the full resource record as JSON.
- **`delete <id>`** deprovisions it. Refused while a binding points at it;
  `--force` removes the bindings first. Its data is gone.
- **`store env <id>`** (storage only, admins) writes the store's S3
  connection, its key included, to `.env.flect.local` (0600, git-ignored):
  the `S3_*` names `flect dev` reads, `<BINDING>_BUCKET`, and `AWS_*` for the
  aws CLI. `--binding <name>` picks the bucket var, `--out <file>` another
  file (`-` prints it). See [Local development](/guides/local-dev/).

```bash
$ flect db create shop-db
✓ created database "shop-db" in acme/web/shop/production (provisioned)
  flect.toml → id = "res_k7f3a2"

$ flect db list
scope: acme/web/shop/production
  res_k7f3a2   shop-db                  provisioned    used by: shop-api
```

A resource belongs to the scope it was created in: dev and production each
have their own.

---

## Secrets

The file lists secret names per service (`secrets = ["STRIPE_KEY"]`); the
values live in the platform, per service and per scope. A value is read from
stdin, never from an argument, so it stays out of shell history. No command
prints a value.

```bash
printf %s "$STRIPE_KEY" | flect secret set api STRIPE_KEY      # piped
flect secret set api STRIPE_KEY                                 # typed, hidden
flect secret list            # every service of the file: names set, and listed ones not set
flect secret list api
flect secret unset api STRIPE_KEY
flect secret set api STRIPE_KEY -c flect.dev.toml               # dev's own value
```

`<service>` is the key in `[services.<service>]`. A new value reaches the
service on its next `flect deploy`.

---

## Deploy

### `flect deploy`

```bash
flect deploy                       # ./flect.toml → the scope it names
flect deploy -c flect.dev.toml     # another file, another scope
flect deploy --yes                 # also drop vars the last deploy kept that the file no longer has
```

For a services-format file, `flect deploy`:

1. resolves `[app].scope` in the active org (`--scope` is refused: the file
   decides);
2. **checks the whole file against that scope** and stops, with nothing
   changed, on any problem: an id that is not a resource there, a binding name
   that points elsewhere, a listed secret not set, kept vars that would be
   dropped (unless `--yes`);
3. binds the resources by id, declares every service, and deploys each one
   with only its own `uses`, `calls`, vars and secrets.

Warnings (it still deploys): a resource no service uses, a secret-looking name
in `vars`, a service of the application still running but no longer in the
file. Every check is listed in the [flect.toml reference](/platform/flect-toml/).

An old-format file (`[[apps]]`) deploys as before, to the active context or
`--scope <scopeId>`, creating resources by name, with a warning.

| Flag | Description |
|------|-------------|
| `-c, --config <file>` | The file to deploy (default `flect.toml`) |
| `--yes` | Drop kept vars the file no longer has |
| `--scope <scopeId>` | Old format only: target an explicit scope id |
| `--broker <url>` / `--token <token>` | Connection overrides |

The container gets only `FLECT_TOKEN`, `FLECT_BROKER_URL`, its vars and
secrets, and the addresses of the services it calls; the SDK resolves
everything else at runtime.

### `flect apps`

List the apps of the active scope: name, status, URL, image.

```bash
flect apps
```

---

## Inspection

### `flect resolve <name>`

Inspect the manifest the broker returns for a binding — the same resolution the
SDK performs — with secrets redacted. Useful for debugging bindings.

```bash
flect resolve DB
```

---

## Pages

A static site built from git by the platform. Declare it in `flect.toml`:

```toml
[page]
name     = "docs"
repo     = "https://github.com/acme/docs"
ref      = "main"
template = "docs"
# domain = "docs.acme.com"   # optional custom domain
```

```bash
flect page deploy                 # build the [page] repo and publish it
flect page ls                     # pages in the active scope
flect page status [page]          # status + build history (* = the build served)
flect page logs [page]            # the last build's log (--build <id> for another)
flect page rollback [page]        # serve a previous build — no rebuild
flect page stop [page]            # stop serving (builds are kept)
```

`deploy` waits for the build and exits non-zero if it fails. Builds are
immutable and swapped in atomically, so a failed build leaves the live site
untouched.

---

## Config

Config lives in `~/.flect/config.json`.

```bash
flect config set <key> <value>    # key: gatewayUrl | broker | token
flect config get [key]            # print config (token masked)
flect config path                 # print the config file path
```

```bash
flect config set gatewayUrl https://flect.cloud   # one origin drives the whole plane
flect config set broker https://api.example.com   # override just the broker (rarely needed)
```

`gatewayUrl` is the recommended single knob: the identity-plane bases
(`/v1/auth`, `/v1/iam`, `/v1/org`) and the broker URL all derive from it. Set
`broker` only to point the control plane somewhere different from the gateway.

---

## Connection flags

Commands that talk to the broker accept:

| Flag | Description |
|------|-------------|
| `--broker <url>` | Broker base URL |
| `--token <token>` | Runtime / identity token |
| `--scope <scopeId>` | Override the active scope for this call (refused next to a services-format file, which names its own) |

The active scope is sent to the broker as the `X-Flect-Scope` header; identity
tokens are not scope-bound, so tenant-level reads (like `flect ls`) work
without one.
