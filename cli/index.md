---
title: CLI Reference
description: Every flect command — login, scopes, resources, deploy, and config.
---

Install the CLI:

```bash
npm install -g @flect/cli
```

The CLI is a client of the Flect control plane (the *broker*). It reads its
connection from, in order of precedence: command flags → environment
(`FLECT_BROKER_URL`, `FLECT_TOKEN`) → `~/.flect/config.json` → defaults.

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

## Scopes

Everything lives in a tree: **Org → Workspace → Project → Environment**. Your
login gives you the org; you create the rest. Resource and deploy commands act
on the **active scope**.

### `flect scope create <kind> <name>`

`kind` is `workspace`, `project`, or `environment`. Nest with `--parent`.

```bash
flect scope create workspace web
flect scope create project site --parent <workspace-id>
flect scope create environment prod --parent <project-id> --slug prod
```

| Flag | Description |
|------|-------------|
| `--parent <scopeId>` | Parent scope (a project belongs to a workspace, etc.) |
| `--slug <slug>` | Slug used in path addressing (default: the name) |

### `flect scope list`

Print the scope tree with the active scope marked.

```bash
flect scope list
```

### `flect use <scope>`

Switch the active scope — by slug path or by scope id.

```bash
flect use web/site/prod        # slug path: workspace/project/environment
flect use scope-6f1c…          # or an explicit scope id
```

### `flect ps`

Show the active context (scope, token, broker, config path) as a compact card.

```bash
flect ps
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

Resource commands accept the connection flags below (`--broker`, `--token`,
`--scope`).

---

## Deploy

### `flect deploy`

Realize `flect.toml` against the broker: provision any missing resources, bind
them to the project, and deploy the app(s) to the cluster behind Traefik.

```bash
flect deploy
flect deploy --scope <scopeId>          # deploy to a specific scope
```

| Flag | Description |
|------|-------------|
| `--broker <url>` | Broker URL (default: `FLECT_BROKER_URL` / config) |
| `--token <token>` | Runtime token (default: `FLECT_TOKEN` / config) |
| `--scope <scopeId>` | Target scope (default: the active scope) |

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

## Config

Config lives in `~/.flect/config.json`.

```bash
flect config set <key> <value>    # key: broker | token | scope | auth
flect config get [key]            # print config (token masked)
flect config path                 # print the config file path
```

```bash
flect config set broker https://api.example.com
```

---

## Connection flags

Commands that talk to the broker accept:

| Flag | Description |
|------|-------------|
| `--broker <url>` | Broker base URL |
| `--token <token>` | Runtime / identity token |
| `--scope <scopeId>` | Override the active scope for this call |

The active scope is sent to the broker as the `X-Flect-Scope` header; identity
tokens are not scope-bound, so tenant-level reads (like `scope list`) work
without one.
