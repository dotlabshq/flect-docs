---
title: Platform architecture
description: How the Flect control plane provisions, binds, deploys, and resolves resources.
sidebar:
  order: 0
---

## The stack

| Layer | Technology | Role |
|-------|-----------|------|
| Control plane | **broker** (`flect-api`) | Event-sourced API: scopes, resources, bindings, deploys, resolution |
| Orchestration | [Nomad](https://www.nomadproject.io/) | Runs your containers |
| Routing | [Traefik](https://traefik.io/) | HTTPS termination, TLS certificates, reverse proxy |
| Service discovery | [Consul](https://www.consul.io/) | Nomad → Traefik wiring |
| Database | [sqld](https://github.com/tursodatabase/libsql) | SQLite over HTTP, multi-namespace |
| KV | [Valkey](https://valkey.io/) | Redis-compatible store |
| Object storage | [Garage](https://garagehq.deuxfleurs.fr/) | S3-compatible object store |

The **broker** is the only thing your app and CLI talk to. It's an
event-sourced control plane: every scope, resource, binding, and deployment is
an event in a shared log, and the current state is a projection of that log.

## Scopes

Everything is addressed within a tree:

```
Org
└── Workspace
    └── Project
        └── Environment
            ├── Apps
            ├── Databases
            ├── KV stores
            └── Object stores
```

An **org** is your tenant (you get one on login). You create the workspace /
project / environment scopes and select an **active scope** with `flect use`.
Resources and deploys target the active scope, so the same code and `flect.toml`
deploy to `dev`, `staging`, and `prod` with fully isolated resources.

## How resolution works

This is the core idea: **your app never holds a connection string or a secret.**

1. On deploy, Flect injects exactly two env vars into your container:
   `FLECT_TOKEN` (a scoped runtime token) and `FLECT_BROKER_URL`.
2. `createEnv().db('DB')` asks the broker to resolve the binding `DB`, presenting
   the token (and the active scope).
3. The broker returns a **manifest** — a temporary, scoped connection with an
   `expiresAt` lease (a URL + a sqld namespace, a redis URL + key prefix, or an
   S3 endpoint + bucket + keys).
4. The SDK builds the *official* client from the manifest and caches it until
   the lease expires, then re-resolves.

No credential is ever baked into your image, printed in your source, or reused
across scopes. Rotating access is a matter of the broker issuing a new manifest.

## Isolation

Each resource is isolated at the storage layer, not just by convention:

- **Database** — every request carries the resource's sqld namespace as an
  `x-namespace` header (RFC-0017 "direct mode"). sqld enforces namespace
  separation; a binding only ever sees its own data.
- **KV** — every key is prefixed with the resource's namespace, so two KV stores
  never collide even with identical key names.
- **Object storage** — each store is its own Garage bucket with its own API key
  pair. There is no cross-bucket access.

## Deploy and routing

`flect deploy` reads `flect.toml`, provisions missing resources, binds them, and
submits a Nomad `service` job. Traefik picks the service up from Consul and
issues a TLS certificate. An app declares one of three routing modes:

| Mode | `flect.toml` | Traefik rule |
|------|--------------|--------------|
| **public** | `public = true` | owns the domain: `Host(...)` |
| **expose** | `expose = "/path"` | a path under a shared domain: `Host(...) && PathPrefix(/path)` (prefix stripped) |
| **private** | *(default)* | no ingress; reachable only inside the cluster |

Public apps get a stable generated hostname:

```
https://<app-name>-<shortid>.up.flect.run
```

The `shortid` is generated once and stays with the app across redeploys. Bring
your own domain by setting `[app].domain` and pointing a CNAME at it.

## Plans and limits

Every org starts on the **hobby** plan. Limits are enforced org-wide at the
control plane; exceeding one returns HTTP `402` with a clear message.

| Plan | Apps | Databases | KV | Storage |
|------|------|-----------|----|---------|
| Hobby | 3 | 1 | 1 | 1 |
| Pro | 10 | 3 | 3 | 3 |
| Team | 25 | 10 | 10 | 10 |

Deprovisioned resources don't count toward the limit.
