# syntax=docker/dockerfile:1
# openwork web UI — source-built, self-contained (stack-conventions §19).
# Empty build context: the pinned upstream commit is fetched in-image.
# Build: docker build -t localhost/openwork-web:<tag> app/images/openwork-web

ARG NODE_IMAGE=node:24.22.1-bookworm-slim

FROM ${NODE_IMAGE} AS builder
ARG OPENWORK_REPO=https://github.com/different-ai/openwork.git
ARG OPENWORK_COMMIT=ae67215d33a54128810e9d7aa49eca0e074272a2
ARG PNPM_VERSION=11.4.0
ENV NODE_OPTIONS=--max-old-space-size=2048 \
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

# Expose the configured backend URL in the served document: Vite replaces
# %VITE_OPENWORK_URL% in HTML at build time, and the runtime entrypoint
# substitutes the placeholder in every served file at container start.
# Upstream index.html has no %VITE_OPENWORK_URL% reference (the app reads
# import.meta.env.VITE_OPENWORK_URL in JS chunks only), so inject one hook.
RUN sed -i 's#<title>OpenWork</title>#<title>OpenWork</title>\n    <script>window.__OPENWORK_URL__="%VITE_OPENWORK_URL%";</script>#' apps/app/index.html

# Filtered install: @openwork/app plus its workspace dependencies only.
RUN --mount=type=cache,id=openwork-pnpm-store,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --child-concurrency=2 --network-concurrency=4 --filter @openwork/app...

# Placeholder value: the runtime entrypoint replaces __OPENWORK_URL__ with
# ${OPENWORK_PUBLIC_URL} in every served file before nginx starts.
RUN VITE_OPENWORK_URL=__OPENWORK_URL__ pnpm --filter @openwork/app build:selfhost

FROM nginx:alpine
ENV OPENWORK_PUBLIC_URL=
COPY --from=builder /src/apps/app/dist/ /usr/share/nginx/html/
RUN <<'EOF'
set -eu
cat > /etc/nginx/conf.d/default.conf <<'NGINX'
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }
}
NGINX
cat > /entrypoint.sh <<'ENTRY'
#!/bin/sh
set -eu
url="${OPENWORK_PUBLIC_URL:-}"
escaped=$(printf '%s' "$url" | sed -e 's/[&|]/\\&/g')
grep -rl '__OPENWORK_URL__' /usr/share/nginx/html 2>/dev/null \
  | xargs -r sed -i "s|__OPENWORK_URL__|${escaped}|g"
exec nginx -g 'daemon off;'
ENTRY
chmod +x /entrypoint.sh
EOF
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD wget -q -O /dev/null http://127.0.0.1/ || exit 1
ENTRYPOINT ["/entrypoint.sh"]
