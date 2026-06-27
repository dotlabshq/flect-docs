# Platform Concepts

## How Flect works

Flect is a deployment platform built on:

| Layer | Technology | Role |
|-------|-----------|------|
| Orchestration | [Nomad](https://www.nomadproject.io/) | Runs your containers |
| Routing | [Traefik](https://traefik.io/) | HTTPS, TLS, reverse proxy |
| Service discovery | [Consul](https://www.consul.io/) | Nomad → Traefik wiring |
| Database | [sqld](https://github.com/tursodatabase/libsql) | SQLite with HTTP API, multi-tenant namespaces |
| Cache | [Valkey](https://valkey.io/) | Redis-compatible KV store |
| Proxy | flect-proxy | Token auth + namespace enforcement |

## Resource hierarchy

```
Organization
└── Workspace
    └── Project
        └── Environment
            ├── Apps
            ├── Databases
            └── KV Stores
```

All resources in an environment share the same isolation boundary. An app can access any database or KV store in its environment — and only those.

## App lifecycle

1. `flect app create` — reserves a hostname, creates the DB record
2. `flect db create` + `flect kv create` — provision resources
3. `flect db migrate` — apply schema to the database
4. `flect app deploy --image <img>` — submits a Nomad job:
   - Injects `FLECT_TOKEN`, `DB_URL`, `DB_NAMESPACE`, `CACHE_URL` env vars
   - Registers service in Consul with Traefik tags
   - Traefik issues TLS cert via Let's Encrypt (Cloudflare DNS challenge)
5. App is live at `https://<name>-<shortid>.up.flect.run`

## Hostnames

Every app gets a permanent hostname at deploy time:

```
https://<app-name>-<shortid>.up.flect.run
```

The `shortid` is generated once and stays with the app. Redeploying the same app reuses the same hostname.

## Security model

### flect-proxy

Apps never connect directly to sqld or Valkey. All traffic goes through `flect-proxy`:

```
App → flect-proxy:7001 → sqld      (HTTP)
App → flect-proxy:7002 → Valkey    (RESP/TCP)
```

The proxy:
1. Validates `FLECT_TOKEN` against the database (60s TTL cache)
2. Verifies the requested namespace/prefix belongs to the token's environment
3. Forwards the request to the backend

This means:
- A compromised app cannot access another app's data even if it knows the namespace
- Tokens can be rotated by redeploying the app
- Credential exposure is limited to what the proxy allows

### Namespace isolation

Each database gets a unique namespace: `<org>-<db-name>`. sqld enforces namespace separation at the storage level — queries in namespace A cannot touch namespace B.

Each KV store gets a unique key prefix: `<org>:<ws>:<proj>:<env>:<name>:`. The proxy prepends this prefix to every key operation, so `GET mykey` from the app becomes `GET flect-examples:default:myapp:production:cache:mykey` on Valkey.

## Migrations

`flect db migrate` applies SQL files to a database. Migration state is tracked in a `_flect_migrations` table inside the database itself — so each database tracks its own history independently.

```bash
flect db migrate myapp-db --dir ./migrations
```

Files are applied in alphabetical order. Already-applied files are skipped. Migration is idempotent — safe to run multiple times.
