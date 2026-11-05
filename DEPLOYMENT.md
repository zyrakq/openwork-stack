# DEPLOYMENT.md — openwork

Deployment model of the upstream product, per stack-conventions §20.
Produced by the `stack-analysis` skill (workflow session 2026-11-04).
Upstream: https://github.com/different-ai/openwork, branch `dev`,
commit `ae67215d33a54128810e9d7aa49eca0e074272a2` (2026-11-04).
Scope decisions confirmed with the user:

- **Product face**: Web UI + server (the recommended minimal configuration).
- **Den platform (`ee/`)**: excluded from the deployment model entirely —
  refused on licensing (OpenWork Enterprise Edition license).
- **Build resources**: the web client source build is resource-capped
  (bounded RAM/CPU during `pnpm`/`vite` builds) per an explicit user
  constraint — a full-fat build must not endanger the host.

## Product parts

Inventory from the upstream clone (branch `dev` @ `ae67215`):

| Part | Upstream evidence | Image source |
|------|------------------|--------------|
| `openwork-host` — openwork-server agent runtime: REST/WebSocket host on port 8787, manages an `opencode` CLI binary internally, persists host state in embedded SQLite (`bun:sqlite`), agent sessions run on a `/workspace` tree with an approval flow | `packaging/docker/Dockerfile` (npm `openwork-server@0.18.4` + opencode `1.17.11` download), `apps/server/package.json` (`build:npm-bundle`, `bun:sqlite`), CMD runs `openwork-server --workspace ... --port ${OPENWORK_PORT:-8787}` with `OPENWORK_MANAGE_OPENCODE=1` | npm-package build (`app/images/openwork-host/Dockerfile`, pins the same npm/binary versions; upstream publishes no MIT image — only EE images are on ghcr) |
| `web-ui` — `apps/app`, Vite React client built in selfhost mode and served separately; talks to the server origin via `VITE_OPENWORK_URL` | `apps/app/package.json` (`build:selfhost`), `apps/app/src/react-app/shell/providers.tsx:33`, `apps/app/src/react-app/kernel/server-provider.tsx:120` | source-built per §19 (`app/images/openwork-web/Dockerfile`, empty context, pinned repo `@` commit; build resource-capped) |

Not separate parts (no compose service created):

- **Electron desktop app** (`apps/desktop`) — client-side desktop program,
  not deployable as a compose service.
- **`apps/review`** — separate Next.js review UI (port 3011); optional
  companion part → extension catalog, not base (overreach guard: not
  exercised by the core gate).
- **OpenCode CLI** — not a part of its own: it is downloaded by the
  `openwork-host` image and managed/started by openwork-server
  (`OPENWORK_MANAGE_OPENCODE=1`, `OPENWORK_SIDECAR_DIR=/data/sidecars`
  for per-session sidecar processes).
- **Storage** — embedded SQLite inside openwork-server
  (`OPENWORK_DATA_DIR=/data/openwork-orchestrator`); no external DB part.

## Interaction map

- **browser → `web-ui`**: HTTPS static UI served by the `openwork-web`
  service.
- **browser (web-ui JS) → `openwork-host`**: REST/WebSocket against the
  server origin taken from `VITE_OPENWORK_URL`; auth is the shared
  host token (`OPENWORK_TOKEN` / `OPENWORK_HOST_TOKEN`); the server
  allows the browser origin via CORS (`--cors ${OPENWORK_CORS_ORIGINS:-*}`).
  Breaks the product end-to-end if absent — this link IS the product.
- **`openwork-host` → managed `opencode`**: server spawns/supervises the
  bundled opencode CLI per session (internal, same container).
- **`openwork-host` → LLM provider APIs** (outbound from opencode):
  model calls with provider keys from env/config. Breaks the product's
  core value if absent — provider credentials are a gate prerequisite.
- Grouping: `web-ui` and `openwork-host` form the single `core` group
  (the web client is useless alone and the server loses its primary UI
  without the web build — they ship and gate together). No inter-group
  links between collection stacks; everything above is intra-group or
  outbound to third-party endpoints (provider APIs, opencode release
  downloads). When outbound runs against internal endpoints over the
  custom CA (e.g. a local model provider), `step-ca-trust` applies.

## Functional base

| Item | Tier | Justification |
|------|------|---------------|
| `openwork` service (openwork-server agent runtime) | core | the product's core value; exercised end-to-end by the gate |
| managed `opencode` binary (downloaded into the image, supervised by the server) | core | sessions run ON opencode; without it no agent session can start |
| `openwork-web` service (source-built selfhost UI, resources capped) | core | the product's primary UI; the confirmed product face. No published MIT image exists → §19 source build; build must not exhaust host RAM/CPU (explicit user constraint) |
| `openwork-data` volume (server state at `/data`: SQLite, config, tokens, sidecars) | core | without it every restart loses host state and sessions |
| `openwork-workspace` volume (workspace tree at `/workspace`) | core | agent sessions act on files here; the gate provisions a throwaway project into it |
| Provider model keys (env fragment into `openwork`) | core (required-for-gate) | a session needs a working model; credentials are the gate prerequisite per stack-analysis — no menu confirmation |
| Token auth (`OPENWORK_TOKEN`, shared by web client and server) | core | the product's built-in auth in the MIT core; the wire between UI and server breaks without it |

Mandatory infra attempts (stack-analysis rule 4) — verdicts, never
silently dropped:

- **PostgreSQL (postgresql+pgrust)** — REFUSED: openwork-server persists
  via embedded SQLite (`bun:sqlite`; the npm bundle is built with
  `--external bun:sqlite` and no other DB driver is wired in `apps/server`);
  no upstream config surface for an external database exists in the MIT
  core. Nothing to wire; a DB container would be dead weight outside the
  gate scenario (overreach guard).
- **OIDC (kanidm)** — REFUSED: SSO/OIDC exists only in the Den platform
  (`ee/`), which is refused on licensing (see catalog). The MIT core
  authenticates with the shared host token (`OPENWORK_TOKEN`) only — no
  OIDC/OAuth surface on openwork-server. If a later upstream version adds
  SSO to the MIT core, re-run gap analysis.
- **S3 (s3-storage)** — REFUSED: no storage-object configuration surface
  observed in the MIT core; all product artifacts (session data, config,
  files) live in the `/data` and `/workspace` volumes. See Open
  hypotheses #1 for the residual check the gate performs.

## Base stage plan

Single group (`core`) → single stage.

- **Stage A (core)**: `openwork` + `openwork-web` + `openwork-data`
  + `openwork-workspace` + provider key env + token auth. Gate scenario
  (real run of the product's main work):
  1. `openwork` comes up healthy: openwork-server listening on its port
     published from `${OPENWORK_PORT:-8787}`; the managed opencode binary
     is present (`OPENWORK_OPENCODE_BIN`) and sidecar dir configured;
  2. browser opens the environment origin (forwarding: local ports of
     both services; TLS overlays: `https://<vhost>`): `openwork-web`
     serves the selfhost UI and connects to the server origin
     (`VITE_OPENWORK_URL`) without console errors;
  3. token auth accepts the `OPENWORK_TOKEN` value; the UI establishes a
     host connection (a rejected token must fail closed — negative check);
  4. provision a throwaway project (small git repo written into the
     `openwork-workspace` volume before/at the gate), start an agent
     session over it, send one prompt; the wire turns a provider key into
     a model response, and an applied file change is observable in
     `/workspace` and in the UI; the manual approval flow
     (`OPENWORK_APPROVAL_MODE=manual`) visibly gates the tool execution
     (this also validates the approval surface);
  5. restart the stack: the project/session survives
     (`openwork-data`/`openwork-workspace` persistence).
- Outer groups and cross-group links: none exist, so no other gates and
  no cross-group step-ca domain gate. `step-ca-trust` is used only by TLS
  environment overlays and applies to any outbound HTTPS from the stack
  over the collection's own domain (provider endpoints and opencode
  release downloads go to the public internet — no internal custom-CA
  link in the base).
- Env-var note: public URLs (`VITE_OPENWORK_URL`, `OPENWORK_CORS_ORIGINS`)
  are environment-dependent → declared ONLY in environment overlays
  (§7), never in base. Secrets (`OPENWORK_TOKEN`) are required vars in
  the owning fragment (`${VAR:?}`) with `.env.example` placeholders only.
- Build note for the §19 web image: cap the builder (bounded
  `NODE_OPTIONS`/pnpm parallelism or cgroup-equivalent limits inside the
  Dockerfile build step) so the `pnpm`/`vite` build cannot exhaust host
  RAM/CPU — an explicit user constraint carried into Phase 1.

## Extension catalog

Catalog extensions are implemented one at a time after the
extension-menu checkpoint (once the base gates pass), each gated by its
own stack-test stage.

| Extension | Type | Status | Dependencies / notes |
|-----------|------|--------|----------------------|
| Den platform (`ee/`: den-api `Dockerfile.den`, den-web `Dockerfile.den-web`, org/project layer, SSO, evaluation MySQL 8.4) | product | refused | OpenWork Enterprise License (`ee/LICENSE`): production use requires a commercial subscription; only ≤5 non-Enterprise users are free — unenforceable in the collection's deployment model, and the user confirmed exclusion. Takes the Den-required MySQL with it (`Dockerfile.eval`/mysql exists only under Den) |
| `headless-runner` + `workbot` (Den's headless execution and chat-bot services) | product | refused | EE + Den platform dependency (they are Den's services); no MIT deployment meaning |
| `openwork-inference` gateway (`Dockerfile.gateway`) | product | available | refused: EE-only image, not buildable from MIT tree — BYO-key via model-keys fragment only (upstream `Dockerfile.gateway` COPYs `ee/packages/{utils,den-db,telemetry-contracts,telemetry}` + `ee/apps/gateway`, all under the OpenWork Enterprise Edition license; published ghcr image is EE-targeted) |
| review app (`apps/review`, Next.js, port 3011) | product | implemented (gate: 2026-11-05) | standalone reviewer UI — verified upstream-first: it does NOT call `openwork-host`; reads reports from its own volume (`OPENWORK_REVIEW_LOCAL_DIR`, `packages/review` storage.ts) and launches sandbox previews via Freestyle/Daytona provider keys (optional, key-free gate serves reports only); source-built image `localhost/openwork-review:0.18.4`, combo `entire-review` |
| otel-lgtm (`docker-compose.otel-lgtm.yml` observability stack) | infra | implemented (gate: 2026-11-05) | all-in-one `grafana/otel-lgtm` (collector + Grafana + Tempo + Loki + Prometheus + Pyroscope), modeled on upstream `packaging/docker/docker-compose.otel-lgtm.yml` @ `ae67215d`; OTLP/HTTP-protobuf env on the `openwork` service; combo `entire-otel`. Note: upstream `apps/server` has no OTel instrumentation at this pin — the wired OTEL env is consumed by the in-process opencode runtime (service.name `opencode`), which is what appears in Tempo/Loki |
| TLS environment overlays (`forwarding` / `letsencrypt` / `step-ca`) | environment overlays | planned with the core scaffold | public URLs / CORS / TLS defaults are environment-dependent → overlays only per §7; `step-ca-trust` in the step-ca combo |

## Open hypotheses

| # | Hypothesis | Confidence | What the functional gate observes to settle it |
|---|-----------|------------|-----------------------------------------------|
| 1 | No hidden S3/object-store surface in the MIT core (uploads/artifacts handled only through `/workspace` + `/data`) | high | gate: run a full session incl. file output; inspect server env/config surface for any object-store variables (negative finding settles refusal) |
| 2 | `apps/app` `build:selfhost` produces a fully standalone static bundle that serves from any static host and only needs `VITE_OPENWORK_URL` at build time | medium | gate: serve the built bundle, connect to `openwork-host`, confirm end-to-end session works with zero dev-server artifacts |
| 3 | Token auth = the UI sends `OPENWORK_TOKEN` as a bearer/header credential and the server rejects wrong tokens cleanly | medium-high | gate: positive + negative auth checks during Stage A |
| 4 | `packaging/docker/Dockerfile` npm pins (`openwork-server@0.18.4` + opencode `1.17.11`) are current enough vs branch `dev` @ `ae67215` | medium | Phase 1: diff the npm package's changelog/behavior against the clone; bump the pin if the clone exposes behavior the pin lacks |
