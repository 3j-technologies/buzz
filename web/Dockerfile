# syntax=docker/dockerfile:1.7
#
# Standalone Buzz web UI image — published as ghcr.io/<owner>/buzz-web:<tag>.
#
# The relay's own Dockerfile (../Dockerfile) already bundles this same web/
# app and serves it same-origin via BUZZ_WEB_DIR — that path needs no
# VITE_RELAY_URL because relay-url.ts falls back to window.location when the
# app is served from the relay itself. This image is for the OTHER case: a
# web UI deployed as its own container against a relay reachable at a
# different origin. Because Vite inlines import.meta.env.VITE_* at build
# time, the relay URL must be supplied as a build ARG, not a runtime env var.
#
# Build:
#   docker build -f web/Dockerfile --build-arg VITE_RELAY_URL=wss://relay.example.com .
# (build context must be the repo root — see pnpm-workspace.yaml filtering.)

ARG NODE_VERSION=24
ARG DEBIAN_VERSION=bookworm
ARG NGINX_VERSION=1.29-alpine

# Optional extra CA bundle for builds behind a TLS-intercepting corporate
# proxy. Empty by default, so public CI builds are unaffected. Mirrors the
# same knob in the relay Dockerfile.
ARG EXTRA_CA_CERTS=

# Optional npm registry mirror. Empty default = public npmjs. Mirrors the
# same knob in the relay Dockerfile.
ARG NPM_REGISTRY=

# The relay URL to bake into the static bundle. Required — there is no
# fallback baked in here, matching the "no silent default" rule for anything
# that points at a specific backend. Leaving it empty degrades gracefully at
# runtime to relay-url.ts's same-origin fallback, which is almost never what
# a standalone deployment wants, so we fail the build instead of guessing.
ARG VITE_RELAY_URL

# ─── Stage 1: build the static bundle (pnpm + vite) ─────────────────────────
FROM node:${NODE_VERSION}-${DEBIAN_VERSION}-slim AS builder
WORKDIR /build

ARG EXTRA_CA_CERTS
COPY --chmod=0644 ${EXTRA_CA_CERTS:-web/Dockerfile} /tmp/extra-ca/src
RUN if [ -n "${EXTRA_CA_CERTS}" ]; then \
        apt-get update && apt-get install -y --no-install-recommends ca-certificates \
        && cp /tmp/extra-ca/src /usr/local/share/ca-certificates/extra-proxy-ca.crt \
        && update-ca-certificates \
        && rm -rf /var/lib/apt/lists/*; \
    fi
ENV NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt

ARG NPM_REGISTRY
ENV COREPACK_NPM_REGISTRY=${NPM_REGISTRY}
RUN if [ -n "${NPM_REGISTRY}" ]; then \
        echo "registry=${NPM_REGISTRY}" > /build/.npmrc \
        && echo "COREPACK_INTEGRITY_KEYS=0" >> /etc/environment; \
    fi
ENV COREPACK_INTEGRITY_KEYS=${NPM_REGISTRY:+0}
RUN corepack enable

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY patches/ patches/
COPY web/package.json web/
RUN pnpm install --frozen-lockfile --filter buzz-web
COPY web/ web/

ARG VITE_RELAY_URL
RUN test -n "${VITE_RELAY_URL}" || (echo "VITE_RELAY_URL build-arg is required" >&2 && exit 1)
ENV VITE_RELAY_URL=${VITE_RELAY_URL}
RUN pnpm -C web build

# ─── Stage 2: serve the static bundle ───────────────────────────────────────
FROM nginx:${NGINX_VERSION} AS runtime

LABEL org.opencontainers.image.title="Buzz Web" \
      org.opencontainers.image.description="Standalone web UI for the Buzz communications platform" \
      org.opencontainers.image.source="https://github.com/block/buzz" \
      org.opencontainers.image.url="https://github.com/block/buzz" \
      org.opencontainers.image.documentation="https://github.com/block/buzz#readme" \
      org.opencontainers.image.licenses="Apache-2.0"

COPY web/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /build/web/dist /usr/share/nginx/html

EXPOSE 8080
