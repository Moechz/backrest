# Changelog

All notable user-visible changes to the Backrest TOS application package.

## 1.14.1-3 — 2026-11-25

Rectified per review comments. Review reference: `[Backrest] 1.14.1-2
(x86_64)` on-hold notice, items **F5**, **F8**, and the security item closed by
resubmission; also addresses C2–C10 and T1.

### Fixed
- **F5 — backup, snapshot listing and restore all failed with HTTP 500.** The
  image's `docker-entrypoint` sets `XDG_CACHE_HOME=/cache`, so every restic
  command opens its cache before touching the repository. The package did not
  mount `/cache`, the image ships no `/cache`, and a uid-1000 process cannot
  rely on being allowed to create a directory in `/` (the review measured
  `unable to open cache: mkdir /cache: permission denied`). Backrest then
  reported `failed to get snapshots for plan "..."` and returned 500 within
  half a second. `/Volume1/DockerAppData/backrestdocker/cache` is now
  bind-mounted to `/cache`; the installer gives it 1000:1000 ownership, so the
  cache is always writable.
- **F8 — persistence layout did not cover the cache directory.** The compose
  file and `tosapp.json` now declare the third directory, `/cache`, next to
  `/config` and `/data`, and the persistence description in every language
  lists all three. `XDG_CACHE_HOME` is set explicitly so the mapping is visible
  in the compose file.

### Added
- `PRIVACY.md`: what is stored where, the third-party endpoint table (Docker
  Hub, the restic backend you configure, notification hooks, click-only help
  links, multihost peers), retention, user rights and security limits.
- `NOTICE.md`: upstream licence verified as **GPL-3.0**, image digests per
  architecture, components inside the image, trademarks, packaging licence.
- `README.md` sections: repository layout, runtime file manifest including
  `/cache`, privacy, notices, build, data and uninstall.
- `cap_drop: ALL` and `no-new-privileges` on the service.

### Changed
- `license` / `upstream_license`: MIT → **GPL-3.0** (the upstream project's
  real licence); `LICENSE` third-party notice corrected accordingly.
- `help` now points to the public `PRIVACY.md`; the stale `entrypoint_file`
  reference (a file that does not exist in this repository) is removed.
- `README` and the language files describe the three data directories and the
  fact that only mounted paths can be backed up.

### Notes
- Ports unchanged (host 8055 → container 9898), per the port-conflict rule.
- Upstream `v1.14.1` is still the newest release (T1: nothing newer to move
  to). The pinned image stays `garethgeorge/backrest:v1.14.1`.
- Both `x86_64` and `aarch64` packages are rebuilt from the same OCI index
  (`linux/amd64` + `linux/arm64` manifests both present).

## 1.14.1-2 — 2026-11-15

Packaging iteration: rebuilt both architectures from the same sources, no
functional change to the application.

## 1.14.1-1 — 2026-11-14

### Added
- First release. Backrest (`garethgeorge/backrest:v1.14.1`, Docker Hub official
  image, unmodified) packaged as a TOS 7 Docker application, published on port
  8055 and running as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/backrestdocker/`.
