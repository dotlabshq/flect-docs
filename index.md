---
title: Flect
description: Developer platform for deploying apps, databases, and KV stores on your own infrastructure.
hero:
  tagline: Deploy apps, databases, and KV stores — without touching infrastructure.
  actions:
    - text: Get Started
      link: /guides/quickstart/
      icon: right-arrow
---

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
