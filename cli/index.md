# CLI Reference

Install the Flect CLI:

```bash
npm install -g @flect/cli
```

---

## Authentication

### `flect login`

Authenticate with your Flect account.

```bash
flect login
```

Opens a browser window. After sign-in, the token is saved to `~/.config/flect/config.json`.

### `flect logout`

Remove saved credentials.

```bash
flect logout
```

---

## Context

The CLI remembers your active org, workspace, project, and environment. Most commands use these automatically — you can override any of them with `--org`, `--ws`, `--proj`, `--env` flags.

```bash
flect org use my-org
flect ws use default
flect proj use myproject
flect env use production
```

---

## Organizations

### `flect org create`

```bash
flect org create --name <name>
```

### `flect org list`

```bash
flect org list
```

### `flect org use`

Set the active organization.

```bash
flect org use <name-or-id>
```

---

## Workspaces

Workspaces live inside an org. Use them to group projects (e.g. by team or product line).

### `flect ws create`

```bash
flect ws create --name <name>
```

### `flect ws list`

```bash
flect ws list
```

### `flect ws use`

```bash
flect ws use <name-or-id>
```

---

## Projects

### `flect proj create`

```bash
flect proj create --name <name>
```

### `flect proj list`

```bash
flect proj list
```

### `flect proj use`

```bash
flect proj use <name-or-id>
```

---

## Environments

Environments live inside a project. Typical names: `production`, `staging`, `preview`.

### `flect env create`

```bash
flect env create --name <name>
```

### `flect env list`

```bash
flect env list
```

### `flect env use`

```bash
flect env use <name-or-id>
```

---

## Databases

Flect databases are isolated SQLite instances powered by [sqld](https://github.com/tursodatabase/libsql). Each database gets its own namespace — no data leaks between apps.

### `flect db create`

```bash
flect db create --name <name>
```

### `flect db list`

```bash
flect db list
```

### `flect db migrate`

Run SQL migration files against a database. Migration names are tracked — already-applied files are skipped.

```bash
flect db migrate <name-or-id> --dir ./migrations
```

Migration files must be `.sql` files. They are applied in alphabetical order.

```
migrations/
  0001_create_users.sql
  0002_add_posts.sql
```

### `flect db delete`

```bash
flect db delete <name-or-id>
```

---

## KV Stores

Flect KV stores are isolated namespaces on a shared Valkey (Redis-compatible) cluster. Each store has a unique key prefix — keys from one app are invisible to others.

### `flect kv create`

```bash
flect kv create --name <name>
```

### `flect kv list`

```bash
flect kv list
```

### `flect kv delete`

```bash
flect kv delete <name-or-id>
```

---

## Apps

### `flect app create`

```bash
flect app create --name <name>
```

Creates the app record and reserves a hostname: `<name>-<shortid>.up.flect.run`.

Options:
| Flag | Default | Description |
|------|---------|-------------|
| `--cpu <mhz>` | 256 | CPU allocation in MHz |
| `--mem <mb>` | 256 | Memory allocation in MB |
| `--replicas <n>` | 1 | Number of instances |

### `flect app list`

```bash
flect app list
```

### `flect app get`

```bash
flect app get <name-or-id>
```

### `flect app deploy`

Deploy a container image. Run this from the directory containing `flect.toml` — the CLI reads binding names from it and maps them to the correct env vars.

```bash
flect app deploy <name-or-id> --image <image:tag>
```

What happens:
1. CLI reads `flect.toml` for `[[databases]]` and `[[kv]]` binding names
2. API generates a `FLECT_TOKEN` for the app (or reuses the existing one)
3. A Nomad job is submitted with all env vars injected
4. Traefik picks up the new service from Consul, issues a TLS cert if needed
5. Your app is live at `https://<name>-<shortid>.up.flect.run`

### `flect app delete`

```bash
flect app delete <name-or-id>
```

---

## Deployments

### `flect deployment list` / `flect app deployment list`

Show deployment history for an app.

```bash
flect deployment --app <name-or-id>
# or
flect app deployment --app <name-or-id>
```

### `flect deployment get`

Show details of a specific deployment.

```bash
flect app deployment get <deployment-id> --app <name-or-id>
```

---

## Global flags

Most commands accept:

| Flag | Description |
|------|-------------|
| `--org <slug>` | Override active org |
| `--ws <slug>` | Override active workspace |
| `--proj <slug>` | Override active project |
| `--env <slug>` | Override active environment |
| `--output json` | Output as JSON (default: table) |
| `--wide` | Show extra columns in table output |
