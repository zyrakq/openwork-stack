# 🤖 OpenWork Stack

Self-hosted deployment of OpenWork — the openwork-server agent runtime
(REST/WebSocket host managing an `opencode` CLI binary internally) plus the
selfhost web UI for AI coding-agent sessions, running in Docker. Server
state (embedded SQLite, tokens, sidecars) and the agent workspace live on
named volumes.

## 🧩 Components

### 🖥️ [OpenWork Server](app/)

Two services built from source (upstream publishes no MIT images — only EE
images are on ghcr): `openwork` on the npm-package image
(`openwork-server@0.18.4` + managed `opencode` CLI 1.17.11) and
`openwork-web` on the source-built selfhost UI image (Vite build from the
pinned upstream commit, resource-capped), with forwarding / letsencrypt /
step-ca environments behind the collection's nginx-proxy stacks. Auth is
the shared host token (`OPENWORK_TOKEN`); agent sessions run on the
`/workspace` tree with a manual approval flow.

### 🌐 [Let's Encrypt Manager](modules/ssl-automation/letsencrypt-manager/)

Automatic SSL certificate generation and renewal using Let's Encrypt for
production deployments with internet access.

### 🔒 [Step CA Manager](modules/ssl-automation/step-ca-manager/)

Self-signed trusted certificate authority for virtual Docker networks
without internet access. Automatically manages and distributes CA
certificates within isolated environments.

## 🚀 Quick Start

```bash
cd app
docker build -t localhost/openwork-host:0.18.4 images/openwork-host/
docker build -t localhost/openwork-web:0.18.4 images/openwork-web/
sb build
cd build/forwarding
cp .env.example .env  # set OPENWORK_TOKEN (+ at least one provider key)
docker compose up -d
```

`sb build` produces one flat directory per environment — `build/forwarding/`,
`build/letsencrypt/`, and `build/step-ca/` (the entire-trust combo) — pick the
one matching your deployment.

See [`app/README.md`](app/README.md) for the full runbook.

## 📋 Requirements

- Docker & Docker Compose
- A shared auth token (`OPENWORK_TOKEN`) between the web UI and the server
- A provider model key for agent sessions (`ANTHROPIC_API_KEY` /
  `OPENROUTER_API_KEY` / `OPENAI_API_KEY` — never committed to the repo)
- Domain name + email (for the Let's Encrypt environment)

## 📄 License

This project is dual-licensed under:

- [Apache License 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)
