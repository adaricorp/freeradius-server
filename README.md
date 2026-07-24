# freeradius-server (multi-arch)

Multi-architecture (`linux/amd64` + `linux/arm64`) build of the official
[FreeRADIUS](https://www.freeradius.org/) server container image, for use by
[Adari](https://github.com/adaricorp/adarigod).

**Published image**: `ghcr.io/adaricorp/freeradius-server:<version>`

## Why this repo exists

The image Adari pins, `freeradius/freeradius-server`, is published
**`linux/amd64` only** — a single manifest, not a manifest list, verified across
`3.2.x`, `latest-3.2` and `latest`. On an arm64 host the RADIUS CNF therefore
can't start at all, which blocks arm64 as a deployment target for an
auth-critical service.

The FreeRADIUS maintainers *do* build multi-arch — but into two differently
named repositories, neither of which is usable as a fleet pin:

| Docker Hub repo | Arches | Freshness |
|---|---|---|
| `freeradius/freeradius-server` | amd64 only | current |
| `freeradius/freeradius-test` | amd64 + arm64 + armv7 | **stale at 3.2.8** |
| `freeradius/freeradius-dev` | amd64 + arm64 + armv7 + armv6 | rolling branch build |

`freeradius-test` is stale because upstream's `dockerhub-releases.yml` has
failed on every run since 2026-04-01 (the 3.2.8 rebuild, 3.2.9 and 3.2.10, on
*all* arches including amd64). Their matrix generator picks up the **peeled**
annotated-tag ref, so `^{}` leaks into the Docker tag:

```console
BUILD ubuntu22 release_3_2_10^{}
ERROR: failed to build: invalid tag
  "freeradius/freeradius-test:arm64v8-3.2.10^{}-ubuntu22": invalid reference format
```

See FreeRADIUS/freeradius-server#2780 (open since 2019) and adaricorp/adarigod
[#1001](https://github.com/adaricorp/adarigod/issues/1001).

## How it works

There is **no Dockerfile in this repo**, and nothing is forked. The build
checks out upstream at the pinned release tag and builds *their* unmodified
`scripts/docker/dists/ubuntu22/Dockerfile` — the same dist their own
`freeradius/freeradius-server:<version>` is built from
(`DOCKER_DEFAULT_UBUNTU := ubuntu22`; the published 3.2.10 image carries
`LABEL org.opencontainers.image.version=22.04`).

Staying on upstream's Dockerfile is the point, not an aesthetic choice: Adari's
`FreeradiusTask` hard-codes the layout that Dockerfile produces — `freerad`
at **uid/gid 101**, config at `/etc/freeradius`, and the `/etc/raddb` symlink.
A hand-rolled "thin image that apt-installs freeradius" would break all of it,
because the distro package puts config under `/etc/freeradius/3.0/` instead.

Each architecture is built on a **native runner** (`ubuntu-24.04` /
`ubuntu-24.04-arm`) and the two are merged into a manifest list. Unlike the Kea
sibling — where upstream's Dockerfile is just `apk add`, so a single
QEMU-emulated multi-platform build was fine — FreeRADIUS compiles from source
(`mk-build-deps` + `dpkg-buildpackage`), which is slow and flaky under
emulation. This mirrors upstream's own split-per-arch-then-merge job layout.

| File | Purpose |
|------|---------|
| `.freeradius-version` | The pinned release. Single source of truth: the workflow derives the upstream git tag, the published image tags, and the post-build version assertion from this one value. |
| `.github/workflows/build-and-publish.yml` | Per-arch native build of upstream's Dockerfile → smoke test → manifest merge → push to `ghcr.io/adaricorp/freeradius-server`. |
| `renovate.json` | Bumps `.freeradius-version` when upstream ships a new 3.2.x release. |

### Published tags

For `FREERADIUS_VERSION=3.2.10`: `3.2.10`, `3.2` , `latest`, and `sha-<short>`
— all multi-arch manifest lists — plus the `amd64-3.2.10` / `arm64-3.2.10`
inputs they were merged from.

### Build-time assertions

Every build fails loudly rather than publishing something subtly wrong:

- `freeradius -v` inside the image must report the **pinned version** (tag and
  binaries cannot drift);
- `freerad` must be **uid/gid 101** and `/etc/raddb` must be a symlink (the
  layout `freeradius.py` depends on);
- the merged manifest list must advertise **both** `linux/amd64` and
  `linux/arm64`.

## Versioning & updates

1. Renovate watches upstream's `freeradius/freeradius-server` Docker Hub tags —
   a tag there means the `release_X_Y_Z` git tag we build from exists — and
   opens a PR bumping `.freeradius-version`.
2. Merging that PR triggers this workflow, which publishes
   `ghcr.io/adaricorp/freeradius-server:<new version>` for both architectures.
3. Renovate on **adarigod** then sees the new tag and bumps
   `FREERADIUS_VERSION` in `src/adari/services/sdn/freeradius.py`.

Renovate is restricted to the **3.2 stable series**; moving to 4.x is a
deliberate, tested decision, not a bot PR.

The workflow also rebuilds weekly on a schedule. That republishes the *same*
version tag with a freshly patched `ubuntu:22.04` runtime base — upstream does
the same, for the same reason — so a pinned version keeps picking up base-OS
security fixes between FreeRADIUS releases. Immutable `sha-` tags are published
alongside if a specific build ever needs pinning.

If upstream ever publishes multi-arch under `freeradius/freeradius-server`
itself, point adarigod back at it and archive this repo.

## Licensing & provenance

This repo contains only build glue — no FreeRADIUS source and no modified
Dockerfile. The published image is built from unmodified
[FreeRADIUS](https://github.com/FreeRADIUS/freeradius-server) source at a public
release tag, and FreeRADIUS is licensed **GPL-2.0-or-later**. Every image
records exactly what it was built from:

```console
$ docker inspect ghcr.io/adaricorp/freeradius-server:3.2.10 \
    --format '{{json .Config.Labels}}' | jq
{
  "org.adari.upstream.repository": "https://github.com/FreeRADIUS/freeradius-server",
  "org.adari.upstream.ref": "release_3_2_10",
  "org.adari.upstream.dist": "ubuntu22",
  "org.opencontainers.image.licenses": "GPL-2.0-or-later",
  ...
}
```
