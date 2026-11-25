# Privacy Policy — Backrest for TerraMaster TOS 7

This policy covers the TOS 7 application package **Backrest** (App Center ID
`backrestdocker`, published by **Moechz**). It describes what the packaged
application stores on your NAS, what it sends over the network, how long it is
kept, and how you can access, correct or delete it.

The application is the open-source **Backrest** project
(<https://github.com/garethgeorge/backrest>, GPL-3.0). The container image is
the unmodified upstream build `garethgeorge/backrest:v1.14.1` from Docker Hub.
This package only adds the TOS metadata, the data-directory mapping and the
service definition.

Last updated: 2026-11-25 (package version 1.14.1-3).

Contact for any privacy question or request: open an issue at
<https://github.com/Moechz/backrest/issues> (public repository).

---

## 1. What the application does

Backrest is a web UI and orchestrator for **restic**, running entirely on your
own NAS:

- it reads the folders that are mapped into the container and stores them as
  encrypted snapshots in a **repository that you create** — a local disk, an
  S3-compatible bucket, Backblaze B2, Azure Blob, Google Cloud Storage, an SFTP
  server, a REST server, an rclone remote, or another restic backend;
- it keeps a schedule, a retention policy and, optionally, notification hooks
  per plan, and lets you browse, diff and restore snapshots in the browser;
- restic encrypts and deduplicates the data **on the NAS before it is
  written**, with the repository password that you choose.

There is no Backrest account, no vendor cloud and no subscription. Neither the
package author nor TerraMaster operates any service that this application
talks to.

**Data reach:** by default the container sees only
`/Volume1/DockerAppData/backrestdocker/{config,data,cache}`. A plan can back up
those three paths, or a remote backend that you configure. To back up another
NAS folder, add a bind mount for it to `src/docker-compose.yml` and rebuild.

## 2. Data stored on your NAS

All persistent data is kept below `/Volume1/DockerAppData/backrestdocker/`:

| What | Where | Why |
|---|---|---|
| Repositories, plans, schedules, hooks, **restic repository passwords in clear text** | `config/config.json`, mode `0600` (Backrest enforces it) | restic needs the password to open a repository unattended; the file is readable only by uid 1000 |
| Operation history (what ran, when, exit status, short log lines) | `data/oplog.sqlite` | the dashboard, the operation list and the log viewer read from it |
| Signing key for UI session cookies (`jwt-secret`) | `data/jwt-secret`, mode `0600`, random, generated on first start | signs the local session cookie; deleting it only logs browsers out |
| Per-operation log files | `data/processlogs/` | the log viewer in the UI |
| restic cache (repository index and pack metadata; **not** the backup itself) | `cache/restic/` | makes snapshot listing, diff and restore fast; safe to delete while the app is stopped, the next run rebuilds it |
| The backups themselves | wherever **you** pointed the repository — a folder on the NAS or a remote bucket/server | this is the encrypted backup |
| Container log | `docker logs backrestdocker` (stdout/stderr) | troubleshooting |

Nothing else on the NAS is written; `/tmp` inside the container is discarded
with the container. The `TOSAppSelfTestingTool` in the source repository is a
local validation utility and is not part of the installed package.

**No telemetry, no analytics, no crash reporting, no update check.** Backrest
has no usage reporting and does not contact GitHub or any vendor endpoint on
its own; the only external hosts it reaches are the ones you configure for a
repository or a notification hook, plus the Docker registry when the App Center
pulls the image.

## 3. Data sent to third parties (API disclosure)

Nothing is uploaded until **you** create a repository, a notification hook or
a multihost peer. "Region" means the region of the provider **you** chose. The
package does not proxy, pin or route data through the packager.

| Endpoint (host) | When | Data sent | Destination / region |
|---|---|---|---|
| `registry-1.docker.io`, `auth.docker.io` | App Center pulls or updates the image | image layers, no user data | Docker Hub (US) |
| none — `local` repository | you pick a path below a mounted folder | nothing leaves the NAS | local disk |
| `s3.<region>.amazonaws.com` or any S3-compatible host you type (MinIO, Wasabi, Cloudflare R2, Hetzner, …) | you create an `s3` repository | encrypted repository data and indexes, bucket name, access key id | the bucket's region / the host you entered |
| `api.backblazeb2.com` and `s3.<region>.backblazeb2.com` | `b2` repository (native or S3-compatible) | encrypted data, key id | Backblaze region you chose |
| `<account>.blob.core.windows.net` | `azure` repository | encrypted data, account name | Azure region of the storage account |
| `storage.googleapis.com`, `oauth2.googleapis.com` | `gs` repository | encrypted data, OAuth token requests | Google Cloud region |
| your SFTP host | `sftp` repository | encrypted data, user name, SSH host key | the server you entered |
| your REST server URL | `rest` repository | encrypted data, credentials | the server you entered |
| your rclone remote (any provider rclone supports: WebDAV, SMB, Dropbox, OneDrive, Google Drive, …) | `rclone` repository | encrypted data, the rclone config you supplied | whatever that remote is |
| `discord.com/api/webhooks/…` | only if you add a Discord hook | notification text (plan, snapshot, error summary) | Discord (US) |
| `hooks.slack.com` | only if you add a Slack hook | notification text | Slack (US) |
| your Gotify server | only if you add a Gotify hook | notification title/text, app token | the server you entered |
| `api.telegram.org` | only if you add a Telegram hook | notification text, bot token, chat id | Telegram |
| `hc-ping.com` / your Healthchecks URL | only if you add a Healthchecks hook | a ping with the plan status | the server you entered |
| `ntfy.sh` or your ntfy server, `api.pushover.net`, your SMTP server, … | only if you add a Shoutrrr hook with that URL | notification text and the credentials in the URL | the service you entered |
| any webhook URL you enter | only if you add a webhook hook | notification text, GET or POST | that URL |
| a command you enter | only if you add a command hook | runs a local command inside the container, nothing is sent by Backrest itself | the container |
| `https://garethgeorge.github.io/backrest…`, `https://restic.readthedocs.io…` | only when **you** click a help link in the UI | your browser's normal HTTP request | GitHub Pages / Read the Docs |
| a peer instance you explicitly pair with (multihost sync) | only if you configure a peer | config, operation events, snapshot metadata, signed with an ed25519 identity generated on the NAS | the host you paired with |

The **repository password never leaves the NAS** in clear text: restic derives
the encryption key locally and only ciphertext is uploaded. Notification
payloads contain the plan name, the operation result and, if you include it in
the template, a short log excerpt.

## 4. Retention

- **Snapshots** stay in your repository until you forget them, the retention
  policy you configured expires them, or you delete the repository.
- **Operation history and per-operation logs** are trimmed by Backrest's
  built-in garbage collection: most operation types after 30 days (100 kept per
  repository/plan/type), prune/check/stats entries after 365 days (12 kept).
  The task runs once a day shortly after start.
- **`config/config.json`, `data/jwt-secret` and the cache** stay until you
  delete the files or the application data directory.
- **Container logs** are kept by the Docker daemon's logging driver (TOS
  default `json-file`) and disappear with the container.

## 5. Your rights: access, correction, deletion

You are the only party with access to this data (NAS administrator); there is
no account with the packager, so every request is handled locally.

- **Access / export** — browse every snapshot and operation in the web UI
  (`http://<NAS-IP>:8055/`), run `docker exec -it backrestdocker restic
  snapshots`, or copy `DockerAppData/backrestdocker/`. The repository
  passwords are in `config/config.json` (mode `0600`).
- **Correction** — edit or delete plans and repositories in the UI, fix a wrong
  schedule, or restore an earlier snapshot over a file (`Restore`).
- **Deletion** — delete a snapshot in the UI (or `restic forget`), remove the
  operation history (`Clear history` in the UI, or delete `data/oplog.sqlite`
  while the app is stopped), remove the whole application data directory after
  uninstalling. Deleting `cache/restic` only costs a re-index.
- **Uninstall** — uninstall the app in App Center, then delete
  `/Volume1/DockerAppData/backrestdocker/`. The backups in your repository are
  **not** touched by uninstalling — delete them where you stored them.

## 6. Security measures and limits

- The container runs as uid/gid `1000:1000`, with all capabilities dropped
  (`cap_drop: ALL`) and `no-new-privileges`; no privileged mode, no host
  network, no Docker socket.
- `config/config.json` and `data/jwt-secret` are mode `0600` inside the data
  folder.
- **The web UI has no built-in login.** Backrest ships without user accounts;
  anyone who can reach port 8055 can read your plans and start a restore. Keep
  the app on the LAN, or put a TLS reverse proxy with authentication in front
  of it. The UI is plain HTTP.
- The repository passwords are stored in clear text in `config.json` (that is
  how upstream works, so unattended backups can run). Protect the data folder
  and any backup of it; if you lose the password, the snapshots cannot be
  restored by anyone — including you.

## 7. Changes and contact

This policy is versioned with the package; the current text is the one shipped
in the repository at tag `v1.14.1-3`. Material changes are listed in
`CHANGELOG.md` and in the App Center release note. Questions or requests:
open an issue at <https://github.com/Moechz/backrest/issues>.
