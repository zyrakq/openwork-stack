# 🤖 OpenWork Server

A modular Docker Compose configuration system for self-hosting OpenWork —
the openwork-server agent runtime (REST/WebSocket host on port 8787 that
manages an `opencode` CLI binary internally) plus the selfhost web UI, with
server state (embedded SQLite, tokens, sidecars) and the agent workspace on
named volumes. Two services: `openwork` (the server) and `openwork-web`
(the static UI build that talks to the server origin).

## 🚀 Quick Start

### 0. Build the Images

The stack ships **source-built images, not published ones** — build them
once before the first `docker compose up`:

```bash
docker build -t localhost/openwork-host:0.18.4 images/openwork-host/
docker build -t localhost/openwork-web:0.18.4 images/openwork-web/
```

(`openwork-host` installs `openwork-server@0.18.4` from npm plus the
`opencode` CLI 1.17.11; `openwork-web` builds the Vite selfhost UI from the
pinned upstream commit — resource-capped so the build cannot exhaust the
host.)

### 1. Build Configurations

```bash
sb build
```

This creates one flat build directory per environment under `build/` —
`build/forwarding/`, `build/letsencrypt/`, `build/step-ca/` — each containing
that environment's single combo (see [`stackbuilder.toml`](stackbuilder.toml)).

### 2. Choose Your Configuration

```bash
# For development with port forwarding
cd build/forwarding/

# For production with Let's Encrypt SSL
cd build/letsencrypt/

# For production with Step CA SSL (entire-trust combo)
cd build/step-ca/
```

### 3. Configure Environment

```bash
cp .env.example .env
# Edit .env: set OPENWORK_TOKEN (required — the shared auth token between
# the web UI and the server).
# Provider keys: set at least one of ANTHROPIC_API_KEY / OPENROUTER_API_KEY /
# OPENAI_API_KEY — agent sessions cannot run without a provider credential.
```

### 4. Deploy

```bash
docker compose up -d
```

Access: `http://localhost:8787` for the API and `http://localhost:8080` for
the UI (forwarding mode, both ports configurable), or `https://<VIRTUAL_HOST>`
(proxy modes). The UI connects to the server origin (`OPENWORK_PUBLIC_URL`)
and authenticates with the shared `OPENWORK_TOKEN`. Agent sessions act on
the `openwork-workspace` volume (mounted at `/workspace` in the container);
the server supervises the managed `opencode` binary per session and gates
tool execution with the approval flow (`OPENWORK_APPROVAL_MODE=manual` by
default).

## 📁 Project Structure

- [`components/`](components/) — Source compose components
  - [`base/`](components/base/) — Base two-service configuration
  - [`environments/`](components/environments/) — Environment-specific configurations
  - [`extensions/`](components/extensions/) — Extension fragments
    (`model-keys`, `repo-sync`, `repo-local`, `step-ca-trust`)
- [`images/`](images/) — Source-built images (`openwork-host`, `openwork-web`)
- [`build/`](build/) — Generated configurations (auto-generated, ready to deploy)
- [`stackbuilder.toml`](stackbuilder.toml) — Build configuration for stackbuilder

## 🔧 Available Configurations

### Environments

- **forwarding**: publishes the API port directly to the host
  (`OPENWORK_PUBLISH_PORT` → container `OPENWORK_PORT`) and the UI on
  `OPENWORK_WEB_PUBLISH_PORT` → 80; `OPENWORK_PUBLIC_URL` defaults to
  `http://localhost:$OPENWORK_PUBLISH_PORT` so the zero-config dev path works
- **letsencrypt**: joins `letsencrypt-network`; nginx-proxy terminates TLS
  and routes to both services; the UI vhost (`VIRTUAL_HOST`) and the API
  vhost (`OPENWORK_API_VHOST`) each get their own cert entry, CORS is
  narrowed to the UI origin, and `OPENWORK_PUBLIC_URL` points at the API
  vhost
- **step-ca**: joins `step-ca-network`; same two-vhost wiring with the
  step-ca stack

### Extensions

Each environment builds exactly one combo: **entire** (`forwarding`,
`letsencrypt`) or **entire-trust** (`step-ca`). The combo selects from the
extension menu in [`stackbuilder.toml`](stackbuilder.toml):

- **model-keys**: passes the provider API keys
  (`ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`, `OPENAI_API_KEY`) through to
  the `openwork` service. At least one working key is a prerequisite for
  agent sessions.
- **repo-sync**: sidecar (`alpine/git`) clones a git repo into the
  `openwork-workspace` volume and fast-forward-pulls it every
  `REPO_SYNC_INTERVAL` seconds. Configure via `REPO_SYNC_URL` (required),
  `REPO_SYNC_BRANCH`, `REPO_SYNC_DIR_NAME`, `REPO_SYNC_TOKEN` /
  `REPO_SYNC_USER` (https auth), `REPO_SYNC_SSH_KEY` (ssh URLs).
- **repo-local**: swap-in alternative to repo-sync (both own `/workspace` —
  never both): bind-mounts a host path (`REPOS_PATH`, required) at
  `/workspace` instead of the named volume — use existing working copies in
  place. To switch, replace `repo-sync` with `repo-local` in the combo in
  `stackbuilder.toml` and rebuild. This is a deliberate exception to the
  collection's no-bind-mount policy: the working copies live on the host
  and must be used in place (edits visible both ways).
- **step-ca-trust**: mandatory in the step-ca combo — mounts the step-ca
  root CA read-only (external `step-ca-data` volume, `certs/` subpath),
  installs it into the container trust store and sets `NODE_EXTRA_CA_CERTS`
  so outbound HTTPS to step-ca-signed services works. Applied to both
  services.

Extension statuses per the catalog in
[`DEPLOYMENT.md`](../DEPLOYMENT.md) (stack-conventions §20): the four
extensions above are **implemented**; **available** but not implemented —
`openwork-inference` gateway, `review` app, `otel-lgtm` observability
stack; **refused** — the Den platform and its `headless-runner`/`workbot`
services (Enterprise license), and the mandatory-infra attempts
PostgreSQL (server persists via embedded SQLite), OIDC (the MIT core
authenticates with `OPENWORK_TOKEN` only), and S3 (no object-store
surface in the MIT core).

`sb build` generates one flat directory per environment under `build/`
(single combo per environment, no combo subdirectories).

## 🔐 Persistence

- `openwork-data` → `/data` (server state: embedded SQLite at
  `/data/openwork-orchestrator`, per-session sidecars at `/data/sidecars`)
- `openwork-workspace` → `/workspace` (agent workspace tree; provisioned by
  the `repo-sync` extension or replaced by the `repo-local` bind mount)
