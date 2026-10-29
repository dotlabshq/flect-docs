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

## Object storage: a local MinIO, or the real store

`flect dev` gives each store binding the same names, so one app runs against
either.

**A local MinIO.** Start it (step 2), create the bucket `flect dev` named
(`<app>-<binding>`, e.g. `myapp-files`; or set `FILES_BUCKET`), and use the
`S3_*` values above:

```bash
docker run --rm --network host -e AWS_ACCESS_KEY_ID=minioadmin -e AWS_SECRET_ACCESS_KEY=minioadmin \
  amazon/aws-cli --endpoint-url http://localhost:9000 s3 mb s3://myapp-files
```

**The real store**, the one your services use. `flect store env` writes its
connection, its own key included, into a git-ignored `.env.flect.local`
(0600). Admins only:

```bash
flect store env res_wkw4s5 --binding FILES      # in the scope of ./flect.toml
# ✓ wrote .env.flect.local (0600): uploads in acme/web/shop/production
#   bucket   <org>-uploads-472bd970   (FILES_BUCKET)
#   endpoint https://s3.eu.flect.run
flect dev && node --env-file=.env.flect.local --import tsx/esm src/index.ts
```

The same file works with S3 tools:

```bash
set -a; . ./.env.flect.local; set +a
aws s3 ls "s3://$FILES_BUCKET"
```

The key is the store's own, limited to its bucket. Treat the file like a
password and delete it when you are done. A database or KV has no such command:
their addresses are inside the cluster.

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
