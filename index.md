# Flect

Flect is a developer platform for deploying containerized applications with managed databases and caching — without touching infrastructure.

## What you get

- **Apps** — deploy any Docker image, get a public HTTPS URL instantly
- **Databases** — managed SQLite via sqld, isolated per app
- **KV stores** — managed Valkey (Redis-compatible), namespaced per app
- **SDK** — TypeScript SDK that connects to your resources with zero config
- **Proxy** — all traffic goes through `flect-proxy`, so credentials never leave the platform

## How it works

```
flect app deploy myapp --image ghcr.io/you/myapp:1.0.0
# → submits Nomad job
# → injects DB_URL, CACHE_URL, FLECT_TOKEN env vars
# → Traefik picks up the service, issues TLS cert
# → your app is live at https://myapp-abc123.up.flect.run
```

Your app talks to its database and cache through `flect-proxy`. The proxy validates your `FLECT_TOKEN` and enforces namespace/prefix isolation — no app can access another app's data.

## Quickstart

→ [Quickstart guide](./guides/quickstart.md)

## Sections

- [CLI Reference](./cli/index.md)
- [SDK Reference](./sdk/index.md)
- [Platform Concepts](./platform/index.md)
- [Guides](./guides/index.md)
