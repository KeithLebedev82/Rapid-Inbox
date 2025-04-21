<div align="center">

**English** | [简体中文](README.zh-CN.md)

# Rapid Inbox

**A local-first, self-hosted temporary inbox for inbound email**

C++ SMTP ingress, optional public inboxes, an admin console, and an HTTP API<br/>
Mail, attachments, metadata, and audit records are stored on local disk and in SQLite

[![Python](https://img.shields.io/badge/python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org)
[![C++](https://img.shields.io/badge/C%2B%2B-20-00599C?logo=cplusplus&logoColor=white)](https://isocpp.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.136-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![SQLite](https://img.shields.io/badge/SQLite-003B57?logo=sqlite&logoColor=white)](https://www.sqlite.org)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-alpha-orange.svg)](CHANGELOG.md)
[![CI](https://github.com/wendaochangsheng/Rapid-Inbox/actions/workflows/release-ingestd.yml/badge.svg?branch=main)](https://github.com/wendaochangsheng/Rapid-Inbox/actions/workflows/release-ingestd.yml)
[![Release](https://img.shields.io/github/v/release/wendaochangsheng/Rapid-Inbox)](https://github.com/wendaochangsheng/Rapid-Inbox/releases)

[Quick Start](#quick-start) · [Demo](#demo) · [Project Scope](#project-scope) · [Roadmap](https://github.com/wendaochangsheng/Rapid-Inbox/issues/5) · [Implemented Features](#implemented-features) · [Configuration](#configuration) · [Usage](#basic-usage) · [Contributing](CONTRIBUTING.md) · [Security](SECURITY.md)

</div>

---

## Introduction

Rapid Inbox is built for open-source maintainers, developers, and test teams that need a
self-hosted email testing environment for **receiving verification codes**, **CI/E2E email
capture**, **internal-tool integration**, and **lightweight self-hosting**.

Its core goals are reliable ingestion, clear inspection, controlled access, and straightforward
recovery. Message bodies, attachments, and indexes are stored on local disk and in SQLite by
default, without depending on a third-party email SaaS or cloud database. Receiving mail from
the public Internet still requires your own domain, DNS/MX records, and a reachable SMTP ingress.

> The project is currently at an early Alpha stage. Interfaces and data structures may continue
> to change.

> **In short:** Rapid Inbox is an alpha-stage, inbound-only, self-hosted SMTP inbox for
> email testing, verification-code workflows, and internal integrations. It uses a
> single-host SQLite/local-filesystem architecture and is not an outbound mail server,
> hosted mailbox service, or multi-node mail platform.

## Project Scope

| Dimension | Current boundary |
| --- | --- |
| **Intended use** | Self-hosted inbound test mail, verification-code extraction, CI/E2E email capture, and internal-tool integration |
| **Deployment model** | Single-host local disk and SQLite. Docker Compose is the primary deployment path; the native systemd installer is secondary. Both run exactly one C++ ingestd and one Python HTTP process for one data directory |
| **HTTP paths** | The UI and API currently support deployment only at the site root `/`; reverse-proxy subpaths and ASGI `root_path` are not supported |
| **Explicit non-goals** | Outbound SMTP, IMAP/POP3, a full public MTA, hosted mailbox SaaS, advanced anti-spam or malicious-attachment sandboxing, and enterprise multi-node HA |
| **Operator responsibilities** | The project does not host or automatically configure DNS/MX, SMTP TLS termination, firewalls, backups, or global rate limiting across instances |
| **Public-access boundary** | Public Web/API access is disabled by default for new domains and the catch-all policy; anonymous Web browsing requires an explicit opt-in, and the public API still requires an API Key with `public.read` |
| **Data retention** | "Temporary mailbox" describes the use case, not default automatic destruction; mail is not deleted automatically when `retention_days` is unset or `0` |
| **Authorized use** | Operators must process only domains and mail they control or are explicitly authorized to handle; the project must not be used to intercept third-party mail, phish, collect credentials, send spam, or evade third-party rules |
| **Maturity** | `0.x` Alpha; there is no SLA or published throughput guarantee, and the database, configuration, and API may still receive incompatible changes |

See [Roadmap Issue #5](https://github.com/wendaochangsheng/Rapid-Inbox/issues/5) for the current plan and capabilities that are explicitly out of scope for now.

## Demo

![Rapid Inbox demo from a public inbox to message details](docs/assets/rapid-inbox-demo.gif)

<details>
<summary>View the static screenshot</summary>

![Static screenshot of Rapid Inbox message details](docs/assets/rapid-inbox-demo.png)

</details>

> The demo uses sanitized test data in an isolated environment and contains no real mailboxes,
> messages, or credentials. It was recorded with Python's embedded SMTP server. Current deployments
> persist committed public-mailbox delivery changes in a SQLite live-event outbox and tail them from
> the Python HTTP process, so the default C++ ingestd and standalone Python SMTP modes can update an
> open public inbox over WebSocket without a manual refresh. The administration console also uses
> an authenticated WebSocket: Python SMTP can publish process-local connection/command telemetry,
> while C++ and separate ingress surface committed deliveries as `delivery_committed` events through
> the SQLite outbox. Cursor-based reconnects can fall back to committed history.

## Implemented Features

| Category | Capability |
| --- | --- |
| **Email ingestion** | C++ `rapid-inbox-ingestd` provides durable ACKs, byte-level backpressure, group commit, multi-worker MIME parsing, and poison-task isolation; Python SMTP remains available as a development mode |
| **Domain modes** | Receive only configured domains, or enable `managed_plus_catchall` for any-domain SMTP deliveries that reach this service; longest-suffix rules take precedence and domain rules are hot-reloaded |
| **Inboxes** | Domains are private by default; the per-mailbox public flag defaults to enabled but takes effect only when the domain-level public switch is enabled, and it can still be disabled per mailbox; supports lists, details, raw EML, sandboxed HTML previews, and attachment downloads |
| **Live updates** | Python and C++ ingress persist committed public-mailbox delivery changes in a SQLite live-event outbox; each HTTP runtime tails the outbox and pushes matching mailbox updates over WebSocket, so open public inboxes do not require manual refreshes. The authenticated administration WebSocket uses cursor resume and gap reporting; committed C++ deliveries appear as `delivery_committed`, while connection/command telemetry remains process-local |
| **Verification-code detection** | A scored extraction algorithm with Chinese, English, Japanese, Korean, and Spanish context plus alphanumeric and separated-code patterns |
| **Access control** | `viewer` / `operator` / `superadmin` RBAC; API Keys are constrained by kind, scope, domain grant mode, mailbox glob, IP, rate limit, and expiry |
| **HTTP API** | `/api/v2` is recommended: Bearer-only, strict models, Problem Details, and stable cursors; public and admin `/api/v1` endpoints remain available |
| **Observability** | JSON/text structured logs, safe Request IDs, Prometheus metrics, live/ready probes, and a cached operations dashboard |
| **Persistence and recovery** | SQLite WAL stores indexes and the compact live-event outbox that bridges committed delivery notifications from ingress to HTTP; raw / text / html / attachments / manifests live on disk; manifests can rebuild metadata that was not committed |
| **Cleanup** | Delivery-retention cleanup in batches; file GC is registered inside the transaction and performed outside it with exponential-backoff retries; sessions, empty mailboxes, metrics, and audit records are cleaned independently |
| **Maintenance tools** | A cross-process `.maintenance.lock` coordinates clearing mail, pauses new ingestion, removes files, and compacts SQLite |

## Technology Stack

`Docker Compose` · `C++20` · `Python 3.10+` · `FastAPI` · `aiosmtpd` · `Jinja2` · `SQLite` · `Uvicorn` · `WebSocket`

## Quick Start

### Docker Compose (primary)

Docker Compose is the recommended deployment method. From a reviewed checkout, with Docker Engine
and Compose v2 installed, run:

```bash
./docker-deploy.sh
```

This one command builds the image, creates a mode-`0600` private configuration with random bootstrap,
cursor-signing, and metrics secrets, starts the Python control plane, waits for schema migration and
`/health/ready`, then starts the C++ SMTP ingress. The image runs as a non-root user. Both processes
are supervised in one container and one PID namespace because the cross-process maintenance protocol
records and verifies operating-system PIDs. A SQLite live-event outbox carries committed public-mailbox
delivery notifications across this process boundary. Public and administration WebSocket cursor
generations remain local to one HTTP process, so keep one HTTP worker per instance (as the supplied
runners do); a multi-worker or multi-replica HTTP tier requires sticky routing until cursor generations
are shared.

Default host bindings:

```text
HTTP: 127.0.0.1:8000
SMTP: 0.0.0.0:25
```

The first successful deployment prints the initial administrator password once. Open:

```text
http://127.0.0.1:8000/admin/login
```

> After the first bootstrap administrator login, the admin console **forces** a change of the
> initial password. The generated value remains in `.rapid-inbox-docker/rapid-inbox.env`; protect
> this file as a secret and replace the password in the administration UI immediately.

Mail, SQLite, manifests, and attachments are persisted in the Compose named volume
`rapid-inbox_rapid-inbox-data` by default. The deployment script never removes this volume.
Use the wrapper for lifecycle operations:

```bash
./docker-deploy.sh status
./docker-deploy.sh logs
./docker-deploy.sh credentials
./docker-deploy.sh update   # rebuild the current checkout with fresh base images
./docker-deploy.sh down     # retain configuration and the named data volume
```

Edit `.rapid-inbox-docker/rapid-inbox.env` to change published addresses, ports, or application
settings, then run `./docker-deploy.sh` again. For reproducible updates, check out a reviewed tag or
commit first; `update` does not fetch source code.

> [!CAUTION]
> Do not run `docker compose down -v` unless permanent deletion of the named volume is intended.
> Stop the deployment before making a consistent backup, and back up both the named volume and the
> private configuration. Do not place the SQLite volume on NFS or another filesystem without reliable
> local POSIX locking, and do not scale the Compose service beyond one replica.

> [!WARNING]
> Before the first Internet-facing deployment, confirm that the bootstrap password has been
> changed and keep the generated Metrics Token private. Compatibility
> `ADMIN_TOKEN` / `PUBLIC_API_KEY` credentials are disabled by default; new integrations should use
> API Keys issued from the admin console. HTTP is published only on host loopback by default. Put the
> admin plane behind a trusted HTTPS reverse proxy before changing that binding. The deployment does
> not configure DNS/MX, TLS termination, firewall rules, backups, or upstream anti-abuse controls.

See the [Docker deployment guide](deploy/docker/README.md) for backup, restore, rollback, port, and
troubleshooting procedures.

### Native systemd (secondary)

For Debian 12+ or Ubuntu 24.04+ hosts that require native services instead of Docker:

```bash
sudo bash deploy/system/install.sh install
```

The installer creates a dedicated `rapid-inbox` account, installs dependencies, stages a versioned
release under `/opt/rapid-inbox`, generates `/etc/rapid-inbox/rapid-inbox.env`, keeps data under
`/var/lib/rapid-inbox`, initializes and backs up SQLite at the write boundary, installs hardened
HTTP/SMTP units, and verifies HTTP plus SMTP protocol readiness.

```bash
sudo bash deploy/system/install.sh status
sudo bash deploy/system/install.sh update
sudo bash deploy/system/install.sh uninstall  # preserves configuration and data
```

See the [native systemd deployment guide](deploy/system/README.md) for the exact support boundary,
managed paths, update rollback, and uninstall behavior.

### Local source launcher

`quickstart.sh` is retained for local evaluation and development. It is a foreground launcher, not
a long-running production process manager:

```bash
bash quickstart.sh
```

It creates `.venv` and `.env`, initializes SQLite, and runs Python HTTP with C++ ingestd. By default,
it downloads the mutable `latest` ingestd release after SHA-256 verification; pin a reviewed release
with `--ingestd-version`, or build the current checkout with `--build-local`. Run
`bash quickstart.sh --help` for all options, including the Python SMTP compatibility mode.

## Local Development and Manual Startup

<details>
<summary><b>C++ SMTP ingestd + Python HTTP</b> (manual development topology)</summary>

```bash
# 1. Build the C++ SMTP ingress
cmake -S cpp/ingestd -B cpp/ingestd/build
cmake --build cpp/ingestd/build

# 2. Start Python HTTP without embedded SMTP
.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000

# 3. Start the C++ SMTP ingress
SMTP_HOST=0.0.0.0 SMTP_PORT=25 cpp/ingestd/build/rapid-inbox-ingestd --base-dir .
```

`INGEST_DURABLE_ACK=true` by default. Before SMTP returns `250 queued`, ingestd atomically writes
the raw EML and pending manifest. SQLite metadata can be committed later in a batch; if the process
exits in the meantime, the Python recovery worker rebuilds records from the manifest. Final parsed
manifests and the recovery worker share a 16 MiB per-file budget. If parsed metadata would exceed
that budget, ingestd retains a bounded pending manifest so recovery can parse the raw message again.
`INGEST_STORAGE_FSYNC=false` guarantees only process-crash recovery. Set it to `true` to cover host
power loss, at the cost of higher disk-sync latency. Disabling durable ACK returns to acknowledging
as soon as mail enters the in-memory queue; an abnormal process exit can then lose mail that already
received a `250` response.

Each SQLite batch commit rematches all recipients inside the transaction. A concurrent rename/delete
or fallback delivery to another tenant rolls back with `policy conflict`. If durable ACK was already
returned, the raw file and manifest are retained in an explicit quarantine forensic path, and the
recovery worker uses a persistent tombstone to prevent stale manifests from resurrecting renamed or
deleted domains.

</details>

<details>
<summary><b>HTTP + embedded Python SMTP in one process</b> (development/compatibility mode)</summary>

```bash
.venv/bin/rapid-inbox-http
```

</details>

<details>
<summary><b>Standalone Python SMTP listener</b> (compatibility mode)</summary>

```bash
.venv/bin/rapid-inbox-smtp
```

</details>

<details>
<summary><b>Development mode (module entry point)</b></summary>

```bash
.venv/bin/uvicorn app.main:app --reload
```

Running `uvicorn app.main:app` directly **does not** enable embedded SMTP. To receive SMTP mail,
run `rapid-inbox-ingestd` in another process. For development, use `rapid-inbox-http` or
`rapid-inbox-smtp`.

</details>

## CI and Release Binaries

The repository includes the GitHub Actions workflow `.github/workflows/release-ingestd.yml`:

- Regular pushes and pull requests run Python tests, validate both deployment paths, execute a real
  one-command Docker smoke deployment, and build/test C++ ingestd.
- Pushing a `v*` tag builds a Linux x86_64 release archive and publishes these files to a GitHub Release:
  - `rapid-inbox-ingestd-linux-x86_64.tar.gz`
  - `rapid-inbox-ingestd-linux-x86_64.tar.gz.sha256`

Example release commands:

```bash
git tag "$NEW_RELEASE_TAG"
git push origin "$NEW_RELEASE_TAG"
```

The publisher should set `NEW_RELEASE_TAG` explicitly according to the actual versioning policy.
Published versions are listed under
[GitHub Releases](https://github.com/wendaochangsheng/Rapid-Inbox/releases).

The Docker path builds a local image from the reviewed checkout; this workflow does not currently
publish a container image. After a Release is published, local-development `bash quickstart.sh`
downloads prebuilt ingestd from the mutable latest
release by default and prints a drift warning. Reproducible deployments should explicitly pass a
reviewed tag; use `--build-local` when a local build is required.

## Configuration

The application resolves variables in this order:

```text
Process environment  >  .env in the current working directory  >  defaults in app/config.py
```

Docker passes `.rapid-inbox-docker/rapid-inbox.env` as process environment; systemd uses
`/etc/rapid-inbox/rapid-inbox.env`. The repository `.env.example` is the local-source template.

<details>
<summary><b>Complete environment variable table</b></summary>

Python HTTP, compatibility SMTP, and shared configuration:

| Variable | `.env.example` | Description |
| --- | --- | --- |
| `STORAGE_ROOT` | `./storage` | Root directory for message files, attachments, manifests, and temporary files |
| `DATABASE_PATH` | `./storage/app.db` | SQLite database path; Python and ingestd must point to the same file |
| `BOOTSTRAP_ADMIN_USERNAME` | `admin` | Administrator username created automatically on the first start |
| `BOOTSTRAP_ADMIN_PASSWORD` | `change-me-now` (randomized by deployment scripts and quickstart) | The code fallback for manual startup is also `change-me-now` and must not be used with an Internet-facing bind |
| `SESSION_COOKIE_NAME` | `rapid_inbox_session` | Name of the HttpOnly administrator session cookie |
| `HOST` / `PORT` | `127.0.0.1` / `8000` | HTTP listen address and port; a non-loopback bind must be placed behind a trusted HTTPS reverse proxy |
| `HTTP_MAX_REQUEST_BODY_BYTES` | `1048576` | ASGI request-body limit for both `Content-Length` and streamed/chunked bodies; configurable up to 64 MiB |
| `HTTP_REQUEST_BODY_TIMEOUT_SECONDS` | `15` | Total time allowed to receive one complete HTTP request body, mitigating slow chunked uploads |
| `HTTP_BODY_MEMORY_BUDGET_BYTES` | `268435456` | Shared per-process byte budget for all buffered HTTP request bodies; must be at least the per-request limit |
| `HTTP_CONCURRENCY_LIMIT` | `1000` | Per-process admission limit across HTTP and WebSocket traffic; supported launchers also pass it to Uvicorn as `--limit-concurrency`, and application middleware enforces it |
| `HTTP_LIVE_CONNECTION_LIMIT` | `256` | Shared per-HTTP-process limit for administration and public-mailbox WebSockets; excess WebSockets close with 1013. The deprecated v1 SSE compatibility stream shares the same limit and receives 503 when full |
| `DATABASE_WRITE_QUEUE_CAPACITY` / `DATABASE_WRITE_MAX_WAITERS` | `256` / `1024` | Dual limits for requests accepted by the single SQLite write actor and requests waiting for it; excess load fails fast with 503 |
| `DATABASE_READ_POOL_SIZE` / `DATABASE_READ_QUEUE_CAPACITY` / `DATABASE_READ_MAX_WAITERS` / `DATABASE_READ_TIMEOUT_SECONDS` | `1` / `256` / `1024` / `5` | API v2 dedicated read-only actors, admitted-request and waiting-request limits, and end-to-end read timeout. A single actor is the conservative default; benchmark the actual workload before increasing connections. Maintenance first drains requests, then the owner thread closes connections. All values are calculated independently per HTTP process |
| `SMTP_HOST` / `SMTP_PORT` | `0.0.0.0` / `25` | SMTP listen address and port |
| `MAX_MESSAGE_SIZE_BYTES` | `52428800` | Maximum size of one message, shared by Python and C++ |
| `MAX_RECIPIENTS_PER_MESSAGE` | `20` | Maximum number of canonical recipients per 