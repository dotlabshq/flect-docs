# Flect Platform Skill

## Purpose

This skill helps you build and deploy projects on Flect — a developer platform for apps, databases, KV stores, and static pages.

## Setup

Fetch current docs at session start:
```
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/AGENTS.md
```

Load component-specific skills as needed:
```
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/db.md
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/kv.md
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/app.md
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/page.md
```

## Architecture

```
Flect Platform
├── Control Plane (flect-api)
│   ├── Auth: user tokens (multi-org) + service tokens (org-scoped)
│   ├── Resources: apps, databases, kvs, pages
│   └── Deploy: Nomad job submission
│
├── Data Plane
│   ├── flect-proxy (port 7001: HTTP→sqld, port 7002: RESP→Valkey)
│   ├── sqld (libsql-compatible SQLite server, multi-namespace)
│   └── Valkey (Redis-compatible, key-prefix isolation per KV store)
│
├── Routing
│   ├── Traefik (TLS termination, Consul service discovery)
│   └── Cloudflare DNS (*.up.flect.run wildcard)
│
└── SDK (@flect/sdk)
    ├── db() → libsql client pre-configured with DB_URL + FLECT_TOKEN
    └── kv() → ioredis client pre-configured with CACHE_URL
```

## Security Model

- **User token**: one token per user, works across all orgs the user is a member of
- **Service token**: org-scoped, for CI/CD pipelines (`flect token create`)
- **App proxy token**: per-app token injected as `FLECT_TOKEN`, enforces namespace/prefix isolation at the proxy layer

Apps never get direct DB/KV credentials — all access goes through `flect-proxy` which validates the app's `FLECT_TOKEN` and enforces isolation.

## Resource Hierarchy

```
Org
└── Workspace (default: "default")
    └── Project (default: "main")
        └── Environment (default: "production")
            ├── Apps
            ├── Databases
            ├── KV Stores
            └── Pages
```

## CLI Quick Reference

```bash
# Context
flect login
flect use <org>/<ws>/<proj>    # set active context
flect ps                        # show current context

# Apps
flect app create <name>
flect app deploy <name> --image <img>:<tag>
flect app list
flect app logs <name>

# Databases
flect db create <name>
flect db list
flect db shell <name>           # interactive SQL shell

# KV Stores
flect kv create <name>
flect kv list

# Pages
flect page create <name>
flect page deploy <name> --repo github.com/<owner>/<repo>
flect page list

# Service Tokens
flect token create <name>
flect token list
flect token revoke <id>
```

## Routing to Component Skills

| Task | Load skill |
|------|-----------|
| Add a database | `skills/db.md` |
| Add a KV store | `skills/kv.md` |
| Deploy an app | `skills/app.md` |
| Deploy a static site | `skills/page.md` |
