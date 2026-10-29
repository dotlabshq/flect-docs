---
name: flect-store
description: >-
  Flect object storage — S3-compatible (Garage), resolved as an official
  @aws-sdk/client-s3 client via env.store(). Use for file uploads/downloads,
  presigned URLs, or binding [[stores]] in flect.toml.
---

# Flect · Object storage

Part of the [flect](../flect/SKILL.md) skill set.

## What it is

Flect object stores are S3-compatible, powered by
[Garage](https://garagehq.deuxfleurs.fr/). Each store is its own bucket with its
own API key pair. `env.store(binding)` returns an official
[`@aws-sdk/client-s3`](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/)
`S3Client`, pre-configured with the endpoint, region, and credentials.

## Create

```bash
flect store create my-uploads   # in the scope of ./flect.toml; prints id = "res_…"
flect store list            # id, name, status, and the services that use it
flect store status <id>
flect store delete <id>     # refused while bound; --force unbinds first
```

## Bind

```toml
[[stores]]
binding = "FILES"
id      = "res_q4wz7n"

[services.api]
uses = ["FILES"]
```

Then list the binding in the `uses` of every service whose code calls it.
A service that does not list it cannot resolve it (`403 BINDING_NOT_USED`).

## Use in code

```typescript
import { createEnv } from '@dotlabshq/flect-sdk'
import { PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3'

const env = createEnv()
const s3  = await env.store('FILES')          // configured S3Client
const Bucket = process.env.FILES_BUCKET!       // the resolved bucket name (<BINDING>_BUCKET)

await s3.send(new PutObjectCommand({ Bucket, Key: 'hello.txt', Body: 'Hello, Flect!', ContentType: 'text/plain' }))
const { Body } = await s3.send(new GetObjectCommand({ Bucket, Key: 'hello.txt' }))
const text = await Body?.transformToString()
await s3.send(new DeleteObjectCommand({ Bucket, Key: 'hello.txt' }))
```

The bucket name is **not** the binding — read it from `process.env.<BINDING>_BUCKET`.
It's the real `S3Client`, so use official commands, not `store.put()`.

### Presigned URLs (direct browser upload/download)

```typescript
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import { PutObjectCommand } from '@aws-sdk/client-s3'

const url = await getSignedUrl(
  s3,
  new PutObjectCommand({ Bucket, Key: `uploads/${filename}`, ContentType: type }),
  { expiresIn: 300 },
)
```

## Isolation

Each store has its own Garage API key, scoped to its own bucket — a key for store
A cannot touch store B. Garage requires path-style addressing, which the resolved
client already sets (`forcePathStyle: true`).

## Local development

```bash
flect dev     # createEnv() resolves the store locally
docker run -d -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin -e MINIO_ROOT_PASSWORD=minioadmin \
  quay.io/minio/minio server /data --console-address ":9001"
```

Set `S3_ENDPOINT`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` (see
[Local development](/guides/local-dev/)) and create the bucket in the MinIO
console at `http://localhost:9001`.

## Related skills

- [flect-app](../flect-app/SKILL.md) · [flect](../flect/SKILL.md).
- [flect-pages](../flect-pages/SKILL.md) — for serving a whole static site.
