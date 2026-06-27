# Flect App Skill

## What It Is

Flect apps are containerized services (Docker images) deployed to Nomad. Each app gets a public HTTPS hostname (`<name>-<id>.up.flect.run`) and can be bound to databases and KV stores via `flect.toml`.

## Workflow

```
1. Build Docker image
2. Push to a registry (ghcr.io, Docker Hub, etc.)
3. flect app deploy <name> --image <img>:<tag>
   OR
   configure flect.toml → flect app deploy
```

## Create & Deploy

```bash
# Create app record
flect app create my-app

# Deploy specific image
flect app deploy my-app --image ghcr.io/you/my-app:1.0.0

# With flect.toml in current directory
flect app deploy
```

## flect.toml

```toml
[app]
name    = "my-app"
image   = "ghcr.io/you/my-app:1.0.0"
port    = 3000       # container port to expose
memory  = 256        # MB
cpu     = 256        # MHz
region  = "eu"

[[databases]]
binding = "DB"
name    = "my-db"

[[kv]]
binding = "CACHE"
name    = "my-kv"
```

## Injected Environment Variables

Flect automatically injects these at deploy time:

| Variable | Description |
|----------|-------------|
| `DB_URL` | libsql HTTP endpoint (from `[[databases]]` binding) |
| `CACHE_URL` | redis:// URL (from `[[kv]]` binding) |
| `CACHE_PREFIX` | key prefix (empty — proxy handles isolation) |
| `FLECT_TOKEN` | proxy auth token — do not override |
| `PORT` | port to listen on (default: 3000) |

Multiple bindings use the binding name: `USERS_DB_URL`, `SESSION_STORE_URL`, etc.

## Dockerfile Requirements

```dockerfile
FROM node:22-alpine
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY dist/ ./dist/
EXPOSE 3000

CMD ["node", "dist/index.js"]
```

Must:
- Listen on `$PORT` (or 3000)
- Respond to `GET /healthz` with 200 (Traefik health check)
- Bundle all dependencies (no workspace symlinks in the image)

## Using @flect/sdk

```typescript
// src/index.ts
import { Hono } from 'hono'
import { db, kv } from '@flect/sdk'

const app = new Hono()

app.get('/healthz', (c) => c.json({ ok: true }))

app.get('/users', async (c) => {
  const rows = await db().execute('SELECT * FROM users')
  return c.json({ users: rows.rows })
})

app.get('/cache/:key', async (c) => {
  const val = await kv().get(c.req.param('key'))
  return c.json({ value: val })
})

export default app
```

## tsup Config for Bundled Build

```typescript
// tsup.config.ts
export default {
  entry:      ['src/index.ts'],
  format:     ['cjs'],
  platform:   'node',
  bundle:     true,
  noExternal: [/.*/],   // bundle all deps — required for ioredis (CJS)
}
```

## GitHub Actions CI/CD

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build & push image
        run: |
          echo ${{ secrets.GHCR_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker build --platform linux/amd64 -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Deploy to Flect
        run: |
          npm install -g @flect/cli
          flect app deploy my-app --image ghcr.io/${{ github.repository }}:${{ github.sha }}
        env:
          FLECT_TOKEN: ${{ secrets.FLECT_SERVICE_TOKEN }}
          FLECT_API:   https://api.flect.cloud
```

Generate a service token for CI:
```bash
flect token create ci-deploy --org my-org
```

## Management

```bash
flect app list                    # list all apps
flect app get <name>              # details + URL
flect app logs <name>             # tail logs
flect app restart <name>          # restart
flect app scale <name> --replicas 2
flect app delete <name>
```

## Custom Domain

```bash
flect app domain add <name> docs.mycompany.com
# Then add CNAME: docs.mycompany.com → <name>-<id>.up.flect.run
```
