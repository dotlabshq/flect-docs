---
title: Object storage
description: S3-compatible object storage via Garage, resolved as an official @aws-sdk/client-s3 client.
---

## What it is

Flect object stores are S3-compatible, powered by
[Garage](https://garagehq.deuxfleurs.fr/). Each store is its own bucket with its
own API key pair. `env.store(binding)` returns an official
[`@aws-sdk/client-s3`](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/)
`S3Client`, pre-configured with the endpoint, region, and credentials.

## Create

```bash
flect store create my-uploads
flect store list
flect store status <ref>
flect store delete <ref>
```

## Bind

```toml
[[stores]]
binding = "FILES"
name    = "my-uploads"
```

## Use in code

```typescript
import { createEnv } from '@flect/sdk'
import { PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3'

const env = createEnv()
const s3  = await env.store('FILES')          // configured S3Client
const Bucket = process.env.FILES_BUCKET!       // the resolved bucket name

// upload
await s3.send(new PutObjectCommand({
  Bucket, Key: 'hello.txt', Body: 'Hello, Flect!', ContentType: 'text/plain',
}))

// download
const { Body } = await s3.send(new GetObjectCommand({ Bucket, Key: 'hello.txt' }))
const text = await Body?.transformToString()

// delete
await s3.send(new DeleteObjectCommand({ Bucket, Key: 'hello.txt' }))
```

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

Each store has its own Garage API key, scoped to its own bucket. A key for store
A cannot touch store B — enforced by Garage. Garage requires path-style
addressing, which the resolved client already sets (`forcePathStyle: true`).

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
