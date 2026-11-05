# syntax=docker/dockerfile:1
# openwork-review — source-built, self-contained (stack-conventions §19).
# Empty build context: the pinned upstream commit is fetched in-image.
# Builds the Next.js review app (apps/review) from the monorepo at the same
# pinned commit as the other openwork images. No upstream source is vendored.
# Build: docker build -t localhost/openwork-review:<tag> app/images/openwork-review

ARG NODE_IMAGE=node:24.22.1-bookworm-slim

FROM ${NODE_IMAGE} AS builder
ARG OPENWORK_REPO=https://github.com/different-ai/openwork.git
ARG OPENWORK_COMMIT=ae67215d33a54128810e9d7aa49eca0e074272a2
ARG PNPM_VERSION=11.4.0
ENV NODE_OPTIONS=--max-old-space-size=3072 \
    CI=1
RUN apt-get update \
 && apt-get install -y --no-install-recommends git ca-certificates \
 && rm -rf /var/lib/apt/lists/* \
 && npm install -g "pnpm@${PNPM_VERSION}"

# Shallow-fetch the pinned commit and assert we got exactly it.
RUN git init /src \
 && git -C /src remote add origin "${OPENWORK_REPO}" \
 && git -C /src fetch --depth 1 origin "${OPENWORK_COMMIT}" \
 && git -C /src checkout FETCH_HEAD \
 && test "$(git -C /src rev-parse HEAD)" = "${OPENWORK_COMMIT}"
WORKDIR /src

# Filtered install: @openwork/review-app plus its workspace dependencies only.
RUN --mount=type=cache,id=openwork-pnpm-store,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --child-concurrency=2 --network-concurrency=4 --filter @openwork/review-app...

# Upstream build script: build workspace deps first, then `next build`.
RUN pnpm --filter @openwork/review-app build

FROM ${NODE_IMAGE}
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates git \
 && rm -rf /var/lib/apt/lists/*
# The full filtered tree is required at runtime: `next start` serves the
# non-standalone .next output against the installed node_modules, and
# outputFileTracingIncludes pulls freestyle builder sources from packages/.
COPY --from=builder /src /src
WORKDIR /src/apps/review

# Self-hosted storage: reports/assets persist in a local directory
# (upstream packages/review storage.ts reads OPENWORK_REVIEW_LOCAL_DIR;
# unset it would require a Vercel Blob token). Mount a volume here.
ENV OPENWORK_REVIEW_LOCAL_DIR=/data/reviews \
    REVIEW_PORT=3011
RUN mkdir -p /data/reviews

EXPOSE 3011
VOLUME ["/data/reviews"]

# Upstream binds 127.0.0.1 (Vercel preview assumption); inside a container the
# server must bind 0.0.0.0 so the compose port publish can reach it.
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.REVIEW_PORT||3011)+'/').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["/bin/sh", "-c", "exec ./node_modules/.bin/next start --hostname 0.0.0.0 --port \"${REVIEW_PORT:-3011}\""]
