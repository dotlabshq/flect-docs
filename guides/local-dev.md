---
title: Local development
description: Run createEnv() with no broker by generating flect.local.json from flect.toml.
sidebar:
  order: 2
---

The same `createEnv()` code that runs in production runs locally — Flect just
resolves bindings from a **local config file** instead of the broker. No token,
no network, no broker required.

## 1. Generate `flect.local.json`

From the directory with your `flect.toml`:

```bash
flect dev
```

This reads your bindings and writes `flect.local.json` — a map of each binding
to a connection built from `${ENV}` placeholders. **No secrets are written**;
they come from your environment at runtime.

```json
{
  "_generatedBy": "flect dev — local binding map; secrets come from ${ENV}, never this file",
  "bindings": {
    "DB": {
      "kind": "database",
      "provider": "libsql",
      "connection": { "url": "${DB_URL:-:memory:}" }
    },
    "CACHE": {
      "kind": "kv",
      "provider": "redis",
      "connection": {
        "url": "${VALKEY_URL:-redis://localhost:6379}",
        "keyPrefix": "myapp-myapp-cache:"
      }
    },
    "FILES": {
      "kind": "storage",
      "provider": "s3",
      "connection": {
        "endpoint": "${S3_ENDPOINT:-http://localhost:9000}",
        "region": "${S3_REGION:-us-east-1}",
        "accessKeyId": "${S3_ACCESS_KEY_ID}",
        "secretAccessKey": "${S3_SECRET_ACCESS_KEY}",
        "bucket": "${FILES_BUCKET:-myapp-uploads}",
        "forcePathStyle": true
      }
    }
  }
}
```

`flect.local.json` is git-ignored by `flect init`. When no broker URL is
configured, `createEnv()` loads it automatically; you can also force it with
`createEnv({ local: true })`.

## 2. Run local backing services (optional)

The defaults above fall back to an in-memory database and localhost services.
To use real local engines:

```bash
# sqld (libsql) with namespaces
docker run -d --name sqld -p 8080:8080 \
  ghcr.io/tursodatabase/libsql-server:latest \
  /bin/sqld --enable-namespaces

# Valkey (Redis-compatible)
docker run -d --name valkey -p 6379:6379 valkey/valkey:8-alpine

# MinIO (S3-compatible) — only if you use object storage
docker run -d --name minio -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin -e MINIO_ROOT_PASSWORD=minioadmin \
  quay.io/minio/minio server /data --console-address ":9001"
```

## 3. Provide the env values

Only the values your bindings reference are needed. A database defaults to
`:memory:`, so the simplest setup is nothing at all. For persistent local data:

```bash
# .env.local
DB_URL=http://localhost:8080
VALKEY_URL=redis://localhost:6379
# storage bindings only:
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY_ID=minioadmin
S3_SECRET_ACCESS_KEY=minioadmin
```

## 4. Run your app

```bash
node --env-file=.env.local --import tsx/esm src/index.ts
```

Or as a script:

```json
{
  "scripts": {
    "dev": "node --env-file=.env.local --import tsx/esm src/index.ts"
  }
}
```

Because `createEnv()` returns the **official** client for each provider, the
same libsql / ioredis / S3 code works identically against your local engines and
against the resources Flect provisions in production.
