---
title: CLI Reference
description: Every flect command — login, tenancy, resources, deploy, pages, and config.
---

Install the CLI:

```bash
npm install -g @getflect/cli
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
level. Your login gives you an org; you create the rest. Resource and deploy
commands act on the **active context** you set with `flect use`.

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

Set the active org / workspace / project / environment. Resource and deploy
commands target it (sent to the broker as `X-Org-Id` + `X-Flect-Scope`).

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

### `flect init`

Scaffold a `flect.toml` in the current directory (and git-ignore the local dev
files).

```bash
flect init
flect init --name myapp        # default: the directory name
```

### `flect bindings`

List the bindings, apps, and services declared in `flect.toml`.

```bash
flect bindings
```

### `flect dev`

Generate `flect.local.json` from `flect.toml` so `createEnv()` resolves bindings
locally with no broker. See [Local development](/guides/local-dev/).

```bash
flect dev
```

---

## Resources

Each resource kind has the same four subcommands. `<ref>` is either the resource
id or its public reference (e.g. `myapp-db-a1b2c3d4`).

```bash
flect db    create <name> | list | status <ref> | delete <ref>   # sqld / libsql
flect kv    create <name> | list | status <ref> | delete <ref>   # Valkey / redis
flect store create <name> | list | status <ref> | delete <ref>   # Garage / S3
```

- **`create`** provisions a resource in the active scope and prints the public
  ref to put in `flect.toml` (`name = "…"`).
- **`list`** shows the resources of that kind in the scope.
- **`status <ref>`** prints the full resource record as JSON.
- **`delete <ref>`** deprovisions the resource.

```bash
$ flect db create notes-db
✓ created database "notes-db-a1b2c3d4" (provisioning)
  flect.toml → name = "notes-db-a1b2c3d4"
```

Resource commands act on the **active context** (`flect use`) and accept the
connection flags below (`--broker`, `--token`, `--scope`).

---

## Deploy

### `flect deploy`

Realize `flect.toml` against the broker: provision any missing resources, bind
them to the project, and deploy the app(s) to the cluster behind Traefik. It
targets your **active context** (set with `flect use`).

```bash
flect use acme/web/site/prod            # pick the target context first
flect deploy
flect deploy --scope <scopeId>          # or target an explicit org-service scope id
```

The deploy target is the active org-service scope (env → project → …) from
`flect use`; `--scope` overrides it with an explicit scope id. `region` (only
`eu` today) is reserved in the toml for future multi-region.

| Flag | Description |
|------|-------------|
| `--broker <url>` | Broker URL (default: `FLECT_BROKER_URL` / config) |
| `--token <token>` | Runtime token (default: `FLECT_TOKEN` / config) |
| `--scope <scopeId>` | Target org-service scope id (default: the active context) |

At deploy time only `FLECT_TOKEN` and `FLECT_BROKER_URL` are injected into your
container; the SDK resolves everything else at runtime.

### `flect apps`

List the apps deployed in the active scope (name, status, URL).

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
| `--scope <scopeId>` | Override the active scope for this call |

The active scope is sent to the broker as the `X-Flect-Scope` header; identity
tokens are not scope-bound, so tenant-level reads (like `flect ls`) work
without one.
