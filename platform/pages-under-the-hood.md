---
title: What happens when you deploy a page
description: The full path of `flect page deploy` — from flect.toml to a build job, an atomic publish, and Traefik.
sidebar:
  order: 3
---

`flect page deploy` looks like one command. Underneath it is a control-plane
command, a one-off build job, an atomic symlink swap on a shared volume, and a
long-lived Caddy job that Traefik routes to. This page walks the whole path with
the real values from `docs.flect.cloud`.

The short version:

```
flect page deploy
   │
   ├─ POST /v1/pages           declare-or-update the page        → PageDeclared
   ├─ POST /v1/pages/:id/deploy  broker submits a batch job      → PageBuildStarted
   │
   │   Nomad: flect-pagebuild-<page>-<build>
   │     build   (main)      clone → build → /alloc/dist   [no volume, no Nomad]
   │     publish (poststop)  copy → builds/<id> → swap `current`  [volume, no repo code]
   │
   ├─ broker reads the allocation's task states               → PageBuildSucceeded
   ├─ broker ensures the Caddy job exists                     → PageDeployed
   └─ Traefik picks it up from Consul and serves it over TLS
```

Nothing in that chain gives the repository's own build commands access to the
cluster. That is the point, and it shapes everything below.

## 0. The declaration

A page is declared in `flect.toml`, like an app:

```toml
[page]
name     = "flect-docs"
repo     = "https://github.com/dotlabshq/flect-docs"
ref      = "main"
template = "docs"
title    = "Flect"
github   = "https://github.com/dotlabshq/flect-docs"
domain   = "docs.flect.cloud"
```

The source is a **git repository**, not a local directory. Your machine never
builds the site and never uploads it — it only says where the content lives.
That is why a page can be rebuilt from CI, from another laptop, or from a
webhook without anything on your machine being involved.

## 1. The CLI declares the page

`flect page deploy` reads `flect.toml` and `POST`s the `[page]` block to
`/v1/pages`. The route is **idempotent by name within the active scope**: an
existing page is updated, a new one is declared. The active scope comes from
`flect use` — the `scope` key in `flect.toml` is ignored.

This writes a `PageDeclared` (or `PageUpdated`) event to the event log and
returns the page's id:

```
page-59117479-13da-4aac-a86d-5532c1447812
```

Nothing has been built yet. A page in this state is `declared`.

## 2. The broker submits a build job

`POST /v1/pages/:id/deploy` hands off to the page deployer, which mints a build
id and submits **one throwaway Nomad batch job per build**:

```
build id  ccc2c43999e2
job id    flect-pagebuild-59117479-ccc2c43999e2
```

The broker holds the only Nomad credential in this path. The CLI never talks to
Nomad; neither does the builder image. The call returns as soon as Nomad accepts
the job — the page is now `building`, and a `PageBuildStarted` event records the
build id, the ref and the job id.

The job has one task group with **two tasks that share the allocation
directory** (`/alloc`):

| | `build` | `publish` |
|---|---|---|
| Lifecycle | main | `poststop` |
| Runs | the repo's build commands | a fixed copy-and-switch |
| `pages-data` volume | **no** | yes, read-write |
| Nomad credential | **no** | no |
| Can write to | `/alloc` only | `/alloc`, the pages volume |

`build` runs untrusted, repo-controlled code, so it is given nothing but its own
allocation directory. `publish` touches the shared volume but runs no code from
the repository. Restarts are disabled (`Attempts: 0`) — re-running someone
else's build commands buys nothing, and the outcome is reported either way.

## 3. Inside the build task

The builder image runs with `MODE=build` and the page's parameters as
environment (`REPO_URL`, `PAGE_REF`, `PAGE_TEMPLATE`, `PAGE_TITLE`,
`PAGE_GITHUB`). It:

1. Clones the repo into `/tmp/flect-build/repo` — `--depth 1 --branch <ref>` for
   a branch or tag, a blobless clone plus `checkout` for a commit sha.
2. Reads the **repo's own** `flect.toml`, if it has one, so the site's settings
   travel with the content.
3. Picks a path:
   - **Custom build** — the repo has `astro.config.mjs`/`.ts` or a
     `[page].build.command`. Its own `npm install` and build command run, and the
     output is taken from `[page].build.out` (default `dist/`).
   - **Template** — no build config, so the content is markdown. The image's
     bundled Starlight template is copied to `/tmp/flect-build/stage`, its sample
     pages are cleared, and the repo's `.md`/`.mdx` files are copied in.
4. Writes the result to the shared allocation directory:

```
/alloc/dist      the built site
/alloc/commit    the exact commit that was built
/alloc/ok        the success marker — written LAST, on purpose
```

`/alloc/ok` is the handshake. It exists only if every previous step succeeded,
so `publish` can decide what happened without trusting an exit code it cannot
see.

### How markdown becomes pages

In the template path, the repo's tree maps onto Starlight's content collection:

- Directories are kept, so `guides/quickstart.md` serves at `/guides/quickstart/`.
- Repository meta — `README.md`, `AGENTS.md`, `CHANGELOG.md`, `LICENSE.md`,
  `CONTRIBUTING.md`, `SKILL.md` — is skipped **at the repo root only**.
- A `SKILL.md` one level down is that folder's landing page: it becomes
  `index.md`, so `skills/flect-db/SKILL.md` serves at `/skills/flect-db/`. An
  explicit `index.md` always wins.
- Starlight requires a `title` in frontmatter. A file without one gets a title
  derived from its filename (or its folder name, for a folder page).

## 4. Inside the publish task

`publish` is a `poststop` hook, so Nomad runs it after `build` exits — **whether
the build passed or failed**. That is what makes a failed deploy harmless.

If `/alloc/ok` is missing, it logs and stops. `current` is untouched and the live
site keeps serving the previous revision.

If it is there, the build becomes immutable content on the shared volume:

```
/opt/flect/data/pages/                     ← the `pages-data` host volume
└── page-59117479-…/
    ├── builds/
    │   ├── ccc2c43999e2/                  ← this build
    │   ├── 8f21a0b4c7de/
    │   └── …
    └── current -> builds/ccc2c43999e2     ← the only moving part
```

The swap is a `symlink` to a temporary name followed by `rename` over `current`.
On POSIX that rename is atomic, so a request in flight never sees a missing or
half-written root. Builds are never modified after they are written — a deploy
only ever adds a directory and moves a pointer.

Then the oldest builds are pruned, keeping the newest 5 plus whatever `current`
points at.

## 5. The broker records what happened

The build task cannot report its own result — it has no credential to report it
with. So the broker reads Nomad's allocation **task states** instead:

- `build` failed → `PageBuildFailed` with the reason.
- `build` succeeded but `publish` failed → `PageBuildFailed`.
- both `dead` and neither failed → `PageBuildSucceeded`.

This happens two ways, deliberately. A background watch polls every 5s while the
build runs (with a 20-minute timeout), and `GET /v1/pages/:id` settles a
`building` page on read. The second is why a broker restart mid-build never
leaves a page stuck: the next read reconciles it.

`flect page deploy` is just polling that read until the status changes.

## 6. The serve job

`PageBuildSucceeded` is separate from `PageDeployed` on purpose: content and
serving have different lifetimes.

Serving is one long-lived Nomad job per page:

```
flect-page-flect-docs-59117479
  caddy:2-alpine
  caddy file-server --root /srv/page-59117479-…/current --listen :80
  pages-data mounted at /srv, READ ONLY
```

Caddy's root is the `current` symlink, not a build directory. **This is why
rebuilds don't restart anything** — the publish task moves the pointer, and the
running Caddy serves the new content on the next request. The serve job is only
submitted when it doesn't exist or its hostnames changed. A hundred rebuilds
touch it zero times.

The health check is TCP rather than HTTP: a page that has been declared but never
built answers 404 on `/`, which would keep an HTTP check critical forever.

## 7. Routing

The Nomad service registers in Consul with Traefik tags. Every page gets a
generated hostname derived from its id, plus any custom domain:

```
flect-docs-591174.up.flect.run        generated — deterministic, survives redeploys
docs.flect.cloud                      custom — CNAME to the generated host
```

which becomes the router rule:

```
Host(`flect-docs-591174.up.flect.run`) || Host(`docs.flect.cloud`)
traefik.http.routers.….tls.certresolver=letsencrypt
```

Traefik watches Consul, so the route appears without anyone editing Traefik's
config, and it terminates TLS with a Let's Encrypt certificate for both names.
A custom domain needs one DNS record — a `CNAME` to the generated host — and
nothing else.

Request path end to end:

```
browser → DNS → Traefik (TLS) → Consul-discovered alloc port
        → Caddy → /srv/<pageId>/current → builds/<buildId>/index.html
```

## Rollback is a pointer move

Because builds are immutable and `current` is the only moving part, rolling back
doesn't rebuild anything:

```bash
flect page rollback flect-docs --build 8f21a0b4c7de
```

The broker submits the **publish half alone** (`MODE=switch`): no clone, no repo
code, no npm. It swaps the symlink and exits. Seconds, not minutes. Without
`--build` it picks the newest succeeded build that isn't already current.

## Reading a build

```bash
flect page status flect-docs      # status, source, url, build history (* = served)
flect page logs   flect-docs      # the build task's stdout+stderr
flect page logs   flect-docs --build 8f21a0b4c7de
```

Logs come from the build allocation, so they are subject to Nomad's garbage
collection: an old build's log is eventually gone even though its content is
still on the volume and still rollback-able.

## Stopping

```bash
flect page stop flect-docs
```

Stops the Caddy job. The builds stay on the volume — a later deploy or rollback
brings the same content back.

## What is guaranteed

- **A failed build cannot take the site down.** `publish` only moves `current`
  when `/alloc/ok` exists.
- **A repository cannot reach the cluster.** The task that runs repo code has no
  volume mount and no Nomad address; the task with the volume runs no repo code.
- **One page cannot write to another.** Only `publish` mounts the volume, and it
  only ever writes under its own `PAGE_ID`.
- **Rebuilds never restart the server.** The root is a symlink; the job is
  untouched.
- **Every state change is an event.** `PageDeclared`, `PageBuildStarted`,
  `PageBuildSucceeded`/`PageBuildFailed`, `PageDeployed`, `PageRolledBack`,
  `PageStopped` — the page's status is a fold over that stream, not a field
  someone remembered to update.

## Operator notes

The volume is a Nomad host volume declared on the client:

```hcl
host_volume "pages-data" {
  path = "/opt/flect/data/pages"
}
```

The builder image the broker submits is configured on the broker, not per page:

```
FLECT_PAGE_BUILDER_IMAGE = ghcr.io/getflect/flect-page-builder:<tag>
```

It must be pullable **without credentials** — Nomad pulls it anonymously, and a
private image fails with `unauthorized` before the build task ever starts.
