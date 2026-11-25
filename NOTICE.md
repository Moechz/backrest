# Third-party notices — Backrest for TerraMaster TOS 7

Package: **Backrest** (App Center ID `backrestdocker`), version `1.14.1-3`,
published by **Moechz**. This repository is an independent community package,
not affiliated with the upstream project or with TerraMaster.

## 1. Upstream application

| | |
|---|---|
| Project | Backrest — web UI and orchestrator for restic |
| Source | <https://github.com/garethgeorge/backrest> |
| Version packaged | `v1.14.1` (newest upstream release at the time of this submission) |
| Licence | **GPL-3.0** — verified with the GitHub API (`/repos/garethgeorge/backrest` → `license.spdx_id = "GPL-3.0"`) and the `LICENSE` file in the source tree |
| Copyright | Copyright the Backrest authors |

The previous package metadata said MIT. That was wrong: Backrest is GPL-3.0,
and the value is corrected here and in `tosapp.json` (`license`,
`upstream_license`).

Backrest is distributed **inside the container image**; this repository contains
no upstream source code. The GPL-3.0 corresponding source for the exact build is
published by upstream at
<https://github.com/garethgeorge/backrest/tree/v1.14.1>. The image is used
unmodified and its binary is not statically or dynamically linked with any code
of this package, so the packaging files stay under MIT (see §6).

## 2. Container image

| | |
|---|---|
| Image | `garethgeorge/backrest:v1.14.1` (Docker Hub, used unmodified) |
| Index digest (multi-arch) | `sha256:b852979754281026230cc69fb11428e6d57c9a97784ab4a444ffc7934c53a215` |
| `linux/amd64` manifest | `sha256:8c717ba795263a1802696aacce7c4e05cfbee98af80b4806ff166e6e18bde944` |
| `linux/arm64` manifest | `sha256:dbf2118de71591f96c996acf5ac2292741d3f536829d0ca58c08558ab59f4e49` |

Registry evidence (queried 2026-11-25 through a reachable Docker Hub mirror,
`application/vnd.oci.image.index.v1+json`):

```
$ curl -sI -H 'Accept: application/vnd.oci.image.index.v1+json' \
    https://docker.1panel.live/v2/garethgeorge/backrest/manifests/v1.14.1
content-type: application/vnd.oci.image.index.v1+json
docker-content-digest: sha256:b852979754281026230cc69fb11428e6d57c9a97784ab4a444ffc7934c53a215

manifests: linux/amd64, linux/arm64, linux/arm/v6, linux/arm/v7, linux/386
```

Both architectures required by the store are present in the upstream index, so
this package ships real `x86_64` and `aarch64` builds of the same image and
nothing is cross-built or emulated.

The image entrypoint is `/sbin/tini -- /docker-entrypoint`, which only fills in
default environment variables and then runs `/backrest` (source:
`cmd/docker-entrypoint/main.go` in upstream `v1.14.1`). It sets
`XDG_CACHE_HOME=/cache`; the package therefore maps
`/Volume1/DockerAppData/backrestdocker/cache` to `/cache`.

## 3. Components inside the image

The image is based on Alpine Linux and additionally ships the restic binary and
rclone. Their licences are those of the upstream projects:

| Component | Licence |
|---|---|
| Backrest | GPL-3.0 |
| restic (`/bin/restic`, v0.19.1) | BSD-2-Clause |
| rclone | MIT |
| Alpine Linux base (musl, busybox, ca-certificates, curl, openssh, tini, tzdata, docker-cli) | MIT / GPL-2.0 / MPL-2.0 / BSD as documented by each Alpine package |

This package does not patch, rebuild or relicense any of them; they are
distributed exactly as they appear in the upstream image.

## 4. Trademarks

"Backrest" and the Backrest logo belong to the Backrest project and its
authors. "TerraMaster", "TOS" and the TerraMaster logo belong to TerraMaster.
All other product names are trademarks of their respective owners. This
package uses the names only to describe what it installs.

## 5. TerraMaster tooling

`TOSAppSelfTestingTool` in this repository is an official TerraMaster utility,
redistributed unmodified for local package validation. It is **not** part of
the installed application archive.

## 6. Licence of this package

Everything written by the packager — `tosapp.json`, `src/config.ini`,
`src/docker-compose.yml`, `src/backrestdocker.lang`, `scripts/`, `tools/`,
`test/`, `README.md`, `PRIVACY.md`, `NOTICE.md`, `CHANGELOG.md`, `LICENSE` — is
released under the MIT licence (see `LICENSE`). The container image and the
software inside it keep their own licences listed above.
