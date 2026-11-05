# openwork-host — self-contained source-built image (stack-conventions §19).
# Install method mirrors upstream packaging/docker/Dockerfile at pinned commit
# ae67215d33a54128810e9d7aa49eca0e074272a2 (npm bundle openwork-server +
# opencode release binary). No upstream source is vendored here.
FROM node:24.22.1-bookworm-slim

ARG OPENWORK_SERVER_VERSION=0.18.4
ARG OPENCODE_VERSION=1.17.11
ARG OPENCODE_DOWNLOAD_URL=

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    tar \
    unzip \
  && rm -rf /var/lib/apt/lists/*

RUN npm install -g "openwork-server@${OPENWORK_SERVER_VERSION}" \
  # The npm tarball ships the bundled ELF binary without the exec bit.
  && chmod +x "$(npm root -g)/openwork-server/dist/bin/openwork-server"

RUN set -eux; \
  arch="$(dpkg --print-architecture)"; \
  case "$arch" in \
    amd64) asset="opencode-linux-x64-baseline.tar.gz" ;; \
    arm64) asset="opencode-linux-arm64.tar.gz" ;; \
    *) echo "unsupported architecture: $arch" >&2; exit 1 ;; \
  esac; \
  url="$OPENCODE_DOWNLOAD_URL"; \
  if [ -z "$url" ]; then \
    url="https://github.com/anomalyco/opencode/releases/download/v${OPENCODE_VERSION}/${asset}"; \
  fi; \
  tmpdir="$(mktemp -d)"; \
  curl -fsSL "$url" -o "$tmpdir/$asset"; \
  tar -xzf "$tmpdir/$asset" -C "$tmpdir"; \
  binary="$(find "$tmpdir" -type f -name opencode | head -n 1)"; \
  test -n "$binary"; \
  install -m 0755 "$binary" /usr/local/bin/opencode; \
  rm -rf "$tmpdir"

# Persistent directories (mount volumes here).
RUN mkdir -p /data/openwork-orchestrator /data/sidecars /workspace

ENV OPENWORK_PORT=8787
ENV OPENWORK_DATA_DIR=/data/openwork-orchestrator
ENV OPENWORK_SIDECAR_DIR=/data/sidecars
ENV OPENWORK_MANAGE_OPENCODE=1
ENV OPENWORK_OPENCODE_BIN=/usr/local/bin/opencode
ENV OPENWORK_WORKSPACE=/workspace

EXPOSE 8787

VOLUME ["/workspace", "/data"]

# Defaults:
# - OpenWork server is published intentionally via --host 0.0.0.0
# - OpenCode is managed by openwork-server and stays internal
# Note: the 0.18.4 npm tarball ships only the compiled Bun binary
# (dist/bin/openwork-server); the OpenCode plugins are embedded in it, so
# OPENWORK_EXTENSIONS_PLUGIN_DIR must NOT be set (upstream's packaging
# Dockerfile still references dist/opencode-plugins, which no longer exists).
# ENTRYPOINT via /entrypoint.sh (not CMD): the step-ca-trust extension
# overrides the compose entrypoint with a trust wrapper that ends
# `exec /entrypoint.sh "$@"` — the image must bake its real command into
# /entrypoint.sh (same pattern as the openwork-web image). With CMD the
# wrapper would exec an empty argv and exit 0 in a restart loop.
RUN printf '%s\n' \
  '#!/bin/sh' \
  'set -eu' \
  'exec openwork-server --workspace "${OPENWORK_WORKSPACE:-/workspace}" --host 0.0.0.0 --port "${OPENWORK_PORT:-8787}" --cors "${OPENWORK_CORS_ORIGINS:-*}" --approval "${OPENWORK_APPROVAL_MODE:-manual}"' \
  > /entrypoint.sh && chmod 0755 /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
