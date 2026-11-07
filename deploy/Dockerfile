# Linux container build for the LearnHarness standalone web server.
# Build from the repository root:
#   docker build -f deploy/Dockerfile -t learnharness:dev .
# See deploy/README.md. Web mode has no authentication: never publish this
# image's port without a private network or an authenticating proxy.
#
# The file stays free of BuildKit-only syntax so it also builds on the legacy
# builder. That is why the ignore file is the root .dockerignore: the legacy
# builder does not read a per-Dockerfile <name>.dockerignore.

ARG NODE_IMAGE=node:26.13.0-trixie-slim

FROM ${NODE_IMAGE} AS build
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1
# The desktop shell's runtime is a devDependency the server never uses.
ENV ELECTRON_SKIP_BINARY_DOWNLOAD=1
# The lockfile carries every platform's prebuilds; npm ci selects this
# container's @resvg/resvg-js and @napi-rs/canvas binaries.
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build && node scripts/build-server.mjs

FROM ${NODE_IMAGE} AS runtime
# npm links the claude executable into /usr/local/bin, a location the Claude
# Code provider searches. The minimum supported version is in
# src/core/contracts/claude-code.ts.
ARG CLAUDE_CODE_VERSION=2.1.311
ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    HOSTNAME=0.0.0.0 \
    PORT=3000 \
    LH_DATA_DIR=/data \
    LH_AUTH_FILE=/auth/auth.json \
    DISABLE_AUTOUPDATER=1
# resvg rasterizes visuals with system fonts and finds them through
# /etc/fonts/fonts.conf; without both packages, SVG text renders blank.
RUN apt-get update \
    && apt-get install -y --no-install-recommends fontconfig-config fonts-dejavu-core \
    && rm -rf /var/lib/apt/lists/* \
    && npm install -g "@anthropic-ai/claude-code@${CLAUDE_CODE_VERSION}" \
    && npm cache clean --force \
    && install -d -o 1000 -g 1000 -m 0700 /data /auth
# Set after the root install above so that it leaves nothing in this home.
ENV HOME=/home/node
WORKDIR /app
# The server tree stays root-owned: only /data, /auth and HOME are writable.
COPY --from=build /app/.next/standalone ./
USER 1000:1000
VOLUME ["/data", "/auth"]
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD ["node", "-e", "fetch(`http://127.0.0.1:${process.env.PORT}/api/settings`).then((r) => process.exit(r.ok ? 0 : 1), () => process.exit(1))"]
ENTRYPOINT ["node", "server.js"]
