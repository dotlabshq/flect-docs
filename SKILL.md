---
title: Flect Platform Skill
---

## Purpose

This skill helps you build and deploy projects on Flect — a developer platform
for apps, databases, KV stores, and object storage. Apps call `createEnv()`; the
platform resolves each binding to a scoped, official client at runtime.

## Setup

Fetch the current instructions at session start:

```
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/AGENTS.md
```

Load component skills as needed:

```
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/db.md
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/kv.md
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/store.md
GET https://raw.githubusercontent.com/dotlabshq/flect-docs/main/skills/app.md
```

## Architecture

```
Flect Platform
├── Control plane — broker (flect-api), event-sourced
│   ├── Auth: SSO identity tokens (per user); runtime tokens (per deploy, scoped)
│   ├── Resources: apps, databases, KV, object storage
│   ├── Resolution: binding → short-lived, scoped manifest (has expiresAt)
│   └── Deploy: Nomad job submission
│
├── Substrate
│   ├── sqld (libsql, multi-namespace) — x-namespace header per request
│   ├── Valkey (Redis-compatible) — key-prefix per store
│   └── Garage (S3-compatible) — one bucket + key pair per store
│
├── Routing — Traefik (TLS) + Consul discovery; *.up.flect.run
│
└── SDK (@flect/sdk)
    ├── createEnv()          → reads FLECT_BROKER_URL + FLECT_TOKEN
    ├── await env.db(name)    → official @libsql/client (scoped)
    ├── await env.kv(name)    → official ioredis (prefixed)
    └── await env.store(name) → official @aws-sdk/client-s3 (bucket-scoped)
```

## Security model

- **Identity token** — from `flect login` (SSO), stored in `~/.flect/config.json`;
  identifies a user across scopes. Not scope-bound; the active scope travels in
  the `X-Flect-Scope` header.
- **Runtime token** — injected into a deployed app as `FLECT_TOKEN`; scoped, and
  used by `createEnv()` to resolve bindings through the broker.
- Apps never receive raw DB/KV/S3 credentials. Each resolution returns a
  short-lived, scoped connection; isolation is enforced at the substrate
  (sqld namespace, KV prefix, Garage bucket key).

## Scope hierarchy

```
Org (your tenant, from login)
└── Workspace
    └── Project
        └── Environment
            ├── Apps
            ├── Databases
            ├── KV stores
            └── Object stores
```

Select the active scope with `flect use <workspace>/<project>/<environment>`.

## CLI quick reference

```bash
# Auth & context
flect login
flect use web/site/prod        # set active scope (slug path or scope id)
flect ps                        # show active context
flect whoami

# Scopes
flect scope create workspace <name>
flect scope create project <name> --parent <workspace-id>
flect scope create environment <name> --parent <project-id>
flect scope list

# Project
flect init                      # scaffold flect.toml
flect dev                       # generate flect.local.json (local createEnv)
flect bindings                  # list declared bindings/apps/services
flect deploy                    # provision + bind + deploy

# Resources  (kind = db | kv | store)
flect <kind> create <name>
flect <kind> list
flect <kind> status <ref>
flect <kind> delete <ref>

# Apps & inspection
flect apps                      # list deployed apps
flect resolve <binding>         # inspect a binding manifest (secrets redacted)

# Config (~/.flect/config.json)
flect config set <broker|token|scope|auth> <value>
flect config get [key]
```

## Routing to component skills

| Task | Load skill |
|------|-----------|
| Add a database | `skills/db.md` |
| Add a KV store | `skills/kv.md` |
| Add object storage | `skills/store.md` |
| Deploy an app | `skills/app.md` |
