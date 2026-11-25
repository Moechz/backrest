# backrest-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Backrest](https://github.com/garethgeorge/backrest).

## What this is

A four-file TOS Docker application archive (`backrestdocker.tar.gz`) that installs
Backrest on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `backrestdocker`)
- `backrestdocker.lang` — 23-language superset (TOS requires 14)
- `backrestdocker.svg` — application icon (8235 bytes, 15 SVG elements)
- `docker-compose.yml` — one service, image `garethgeorge/backrest:v1.14.1`

## Repository layout

| Path | What it is |
|---|---|
| `tosapp.json` | Application spec: app id, image, ports, volumes, licence, descriptions, translations. The build reads everything from here. |
| `src/config.ini` | TOS metadata template (`@@VERSION@@` / `@@PLATFORM@@` are substituted at build time). |
| `src/docker-compose.yml` | The service definition, `x-app-meta` web port, volumes and security options. |
| `src/backrestdocker.lang` | 23-language text file rendered from `tosapp.json` → `translations`. |
| `src/backrestdocker.svg` | Icon (512×512 viewBox, 15 elements, no filters). |
| `scripts/build.sh` | Build one architecture: stage → substitute → `tools/verify.py` → deterministic tar.gz + sha256. |
| `scripts/build-dual.sh` | Build `x86_64` and `aarch64` and run the official self-test on both. |
| `tools/verify.py` | Review-standards self-check (hard gate, see below). |
| `test/install-and-verify.sh` | On-device end-to-end check (needs root on a TOS NAS). |
| `PRIVACY.md` | Privacy policy and third-party endpoint disclosure. |
| `NOTICE.md` | Upstream licence, image digests, component licences, trademarks. |
| `CHANGELOG.md` | User-visible history. |

## How it works

| | |
|---|---|
| Container image | `garethgeorge/backrest:v1.14.1` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8055/` (container port 9898) |
| Runs as | non-root, `user: "1000:1000"`, `cap_drop: ALL`, `no-new-privileges` |
| Persistent data | `/config`, `/data`, `/cache` → `/Volume1/DockerAppData/backrestdocker/{config,data,cache}` |
| Logs | `docker logs backrestdocker` (stdout/stderr) |

`/config` holds `config.json` (repositories, plans, schedules, hooks, and the
restic repository passwords); `/data` holds the operation log (`oplog.sqlite`),
the UI session signing key (`jwt-secret`) and per-operation logs; `/cache` holds
the restic cache (repository index and pack metadata). The restic repositories
themselves live wherever you point them.

### Why `/cache` is mapped

The image's `docker-entrypoint` sets `XDG_CACHE_HOME=/cache`; restic opens its
cache **before** it touches the repository. The image ships no `/cache`, and a
uid-1000 process cannot rely on being allowed to create a directory in `/`
(the review measured `unable to open cache: mkdir /cache: permission denied`),
so without the mount every restic call dies with:

```
unable to open cache: mkdir /cache: permission denied
```

and Backrest reports `failed to get snapshots for plan "..."` and returns
HTTP 500 for backup, snapshot listing and restore. The bind mount gives the
installer a directory to chown to 1000:1000, which is what makes the cache
writable. This was review finding **F5/F8**; see `CHANGELOG.md`.

### Backup sources

Only `/config`, `/data` and `/cache` are mapped into the container, so a plan
can back up those paths or a remote restic backend (S3, B2, SFTP, REST, rclone
remote…). To back up another NAS folder, add a bind mount for it to
`src/docker-compose.yml` (for example `/Volume1/Documents:/sources/Documents`),
rebuild, and reinstall — then use `/sources/Documents` as the plan path.

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container
  reports uid 1000). All capabilities are dropped and `no-new-privileges` is on.
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no
  `network_mode: host`, no Docker socket, no `cap_add`, no `group_add`.
- **Health check** probing the service on container port 9898 (`curl` ships in
  the image).
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip
  without a timestamp, plus the review-standards self-check below.
- **No database server.** Backrest keeps its state in a JSON config plus an
  operation log in `/data`.

## Data and uninstall

Everything the application writes on the NAS lives in its own data directory;
the container-internal `/tmp` is discarded with the container.

| Path | Purpose | Created | Lifecycle |
|---|---|---|---|
| `/Volume1/DockerAppData/backrestdocker/config` | `config.json`: repositories, plans, schedules, hooks, restic passwords (mode 0600) | first save in the UI | persistent; survives updates |
| `/Volume1/DockerAppData/backrestdocker/data` | `oplog.sqlite`, `jwt-secret` (0600), `processlogs/` | first start | persistent; survives updates |
| `/Volume1/DockerAppData/backrestdocker/cache` | restic cache (repository index, pack metadata) | first restic command | persistent; safe to delete while the app is stopped, rebuilt on the next run |

**Uninstall:** remove the app in App Center, then delete
`/Volume1/DockerAppData/backrestdocker/` if you also want to drop the
configuration and history. Your **backups are not deleted** by uninstalling —
they are in the repository you configured (a folder on the NAS or a remote
bucket/server); delete them there. There is no uninstall hook in a Docker
package, so this manual step is the documented way (F11).

## Privacy and notices

- [`PRIVACY.md`](PRIVACY.md) — what is stored where, which third-party
  endpoints the app can contact (Docker Hub on install, the restic backend you
  configure, the notification hooks you add, click-only help links, optional
  multihost peers), retention, your access/deletion rights and the security
  limits (the UI has no built-in login; the repository passwords are stored in
  `config.json` so unattended backups work).
- [`NOTICE.md`](NOTICE.md) — upstream licence **GPL-3.0** (verified, the old
  metadata said MIT), the Docker Hub image digests for `linux/amd64` and
  `linux/arm64`, licences of the components inside the image, trademarks and
  the MIT licence of the packaging files.
- The `help` link in `config.ini` points at this repository's `PRIVACY.md`
  (`https://github.com/Moechz/backrest/blob/main/PRIVACY.md`).

## Build

```bash
scripts/build.sh                  # -> out/backrestdocker.tar.gz + .sha256
scripts/build.sh 1.14.1-3         # packaging iteration bump
scripts/build.sh 1.14.1-3 aarch64
scripts/build-dual.sh 1.14.1-3    # both architectures + official -c check
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs `tools/verify.py`, a review-standards
self-check (archive layout, JSON validity, required fields, 14+ language
coverage, version consistency, Docker-Hub-only images, fixed tag,
reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/backrestdocker_x86_64.tar.gz    # official spec check
./TOSAppSelfTestingTool -c out/backrestdocker_aarch64.tar.gz
bash test/install-and-verify.sh                                # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

The F5 fix was verified on real hardware against the same image as uid 1000:
with the `/cache` mount, `SetConfig` → `Backup` → `ListSnapshots` all return
200 and a snapshot is listed; without it the same flow returns HTTP 500 with
`unable to open cache: mkdir /cache: …`.

## Store submission checklist

1. Public GitHub repo (code + README + PRIVACY + NOTICE).
2. Release **tag `v1.14.1-3`**, assets `backrestdocker_x86_64.tar.gz` /
   `backrestdocker_aarch64.tar.gz` (and the generic `backrestdocker.tar.gz`),
   each with a `.sha256`.
3. Developer platform → app `backrestdocker`, package type Docker, repository
   URL, architectures `x86_64` and `aarch64`.
4. Version Management → Add Version `1.14.1-3` → automated validation → review.

## Licence / branding

The upstream application **Backrest is GPL-3.0** (verified against
<https://github.com/garethgeorge/backrest>; the earlier package metadata said
MIT and was wrong). The container image is the official Docker Hub image
`garethgeorge/backrest:v1.14.1`, used unmodified; no upstream source code is
redistributed here. All packaging files in this repository are MIT (see
`LICENSE`). This is an independent community submission by Moechz and is not
affiliated with the upstream project or TerraMaster.

## Limitations

- The restic repository password is required to restore; it is kept in
  `config.json` (mode 0600) so unattended backups can run — protect that file
  and any copy of it.
- The UI has no built-in login and is plain HTTP; keep it on the LAN or front
  it with a TLS reverse proxy that authenticates.
- Only the three mapped directories are visible to the container; add bind
  mounts for other NAS folders (see *Backup sources*).
- Backup destinations must be reachable from the container (S3/SFTP/WebDAV
  credentials are configured in the UI).
- The restic cache in `/cache` grows with the repository size; it is safe to
  delete while the app is stopped.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
