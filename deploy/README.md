# Linux container

`deploy/Dockerfile` packages the standalone server so a Linux host can run LearnHarness as a long-lived web server. The image contains the curated `.next/standalone` tree on a Node 26 Debian base, plus the Claude Code CLI.

> [!WARNING]
> **Web mode has no authentication.** Anyone who can reach the port can read your learning records, change settings and API keys, and spend on your connected providers. Only make the image reachable through a private, access-controlled network or an authenticating proxy. Never publish the port to the public Internet.

The same-origin checks in the server are not user authentication. See [subscription connections in web/dev](../docs/development.md#subscription-connections-in-webdev).

## Build

Run from the repository root:

```sh
docker build -f deploy/Dockerfile -t learnharness:dev .
```

The image is built for the architecture of the machine that builds it, because `npm ci` installs that platform's native `@resvg/resvg-js` and `@napi-rs/canvas` prebuilds. Build on the host that will run the image, or on one of the same architecture.

| Image | Unpacked (`docker images`) | Compressed (`docker image inspect`) |
| --- | --- | --- |
| `linux/arm64` | 1.04 GB | 273 MB |

The other builds have not been measured. An `amd64` host builds its own image from the same Dockerfile. The build stage uses about 3.2 GB of disk.

The Dockerfile uses no BuildKit-only syntax, so it builds with or without `buildx`. The legacy builder prints a deprecation notice but works. The root `.dockerignore` keeps `data/`, key files and `.env*` files out of the build context.

| Build argument | Default | Purpose |
| --- | --- | --- |
| `CLAUDE_CODE_VERSION` | `2.1.311` | Pinned CLI version. LearnHarness requires at least 2.1.259. |
| `NODE_IMAGE` | `node:26.13.0-trixie-slim` | Base image for both stages. |

## Run

```sh
docker run -d --name learnharness --init --restart unless-stopped \
  -p 127.0.0.1:3000:3000 \
  -v learnharness-data:/data \
  -v learnharness-auth:/auth \
  --env-file /path/to/learnharness.env \
  learnharness:dev
```

This binds the port to loopback so that a private network or proxy on the host can front it. `--init` is recommended for a long-lived container because Node as process 1 does not reap orphaned processes. `--env-file` is optional; keep that file readable only by its owner.

The server runs as uid/gid 1000 and its tree is read-only to that user.

### Volumes

| Path | Holds |
| --- | --- |
| `/data` | Chats, jobs, graph, settings, and prompt overrides. |
| `/auth` | `auth.json` (saved keys and custom endpoints) and subscription credentials. These are plaintext files; treat the volume as secret. |

Back up both. Named volumes get the right owner automatically. A bind-mounted host directory must be writable by uid 1000.

Run one server per volume pair. Credential refresh is serialized only within a process.

### Environment

The image sets these:

| Variable | Value | Purpose |
| --- | --- | --- |
| `NODE_ENV` | `production` | |
| `HOSTNAME` | `0.0.0.0` | Listen address inside the container. |
| `PORT` | `3000` | Listen port inside the container. |
| `LH_DATA_DIR` | `/data` | Mutable records. |
| `LH_AUTH_FILE` | `/auth/auth.json` | Key store. Subscription credentials are written beside it. |
| `HOME` | `/home/node` | Writable home for the CLI. It is not a volume, so it is lost when the container is replaced. |
| `DISABLE_AUTOUPDATER` | `1` | Keeps the CLI at the pinned version. |
| `NEXT_TELEMETRY_DISABLED` | `1` | |

You can add these:

| Variable | Purpose |
| --- | --- |
| `CLAUDE_CODE_OAUTH_TOKEN` | Authenticates the Claude Code CLI. Use this rather than an interactive login, which would live in `HOME`. |
| Provider and search API keys | See [configuration](../docs/development.md#configuration). Environment keys take precedence over keys saved in Settings. |

Do **not** set `LH_DESKTOP_SERVER` or `LH_DESKTOP_API_SECRET`. They are for the desktop shell and lock browsers out of the API; they are not a way to add authentication.

### Behind a proxy

Same-origin routes, including the Claude Code status check and subscription sign-in, compare the browser's `Origin` with the request. A proxy must preserve the original `Host` header and, when it terminates TLS, send `X-Forwarded-Proto: https`. Otherwise these routes answer 403.

## Verify

```sh
docker image inspect learnharness:dev --format '{{.Os}}/{{.Architecture}} {{.Size}}'
docker run --rm -d --name lh-smoke -p 127.0.0.1:3100:3000 \
  -v lh-smoke-data:/data -v lh-smoke-auth:/auth learnharness:dev

for route in / /api/graph /api/models /api/settings; do
  curl -s -o /dev/null -w "$route %{http_code}\n" "http://127.0.0.1:3100$route"
done
docker exec lh-smoke id
docker exec lh-smoke claude --version

# Change one setting, replace the container, and confirm it survives.
curl -s -X PUT -H 'content-type: application/json' -d '{"enabled":true}' \
  http://127.0.0.1:3100/api/settings/claudeCode
docker stop lh-smoke
docker run --rm -d --name lh-smoke -p 127.0.0.1:3100:3000 \
  -v lh-smoke-data:/data -v lh-smoke-auth:/auth learnharness:dev
sleep 3
curl -s http://127.0.0.1:3100/api/settings

docker stop lh-smoke
docker volume rm lh-smoke-data lh-smoke-auth
```

Each route should answer 200, `id` should report uid 1000, and the final settings should show `claudeCode.enabled: true`. The container also has a `HEALTHCHECK` on `GET /api/settings`.

## Claude Code

The provider is off by default. After first boot, enable it in **Settings → Providers → External CLI → Claude Code** and select a model. The CLI is already installed and on `PATH`; only authentication is needed. See [Claude Code](../docs/claude-code.md).

To change the CLI version, rebuild with `--build-arg CLAUDE_CODE_VERSION=<version>`.

## Limitations

- **Demo verification is not included.** It needs Chromium, which the image does not contain, so it fails. The rest of the app is unaffected.
- **ChatGPT sign-in needs a browser on the server's machine.** Its callback listens on `127.0.0.1` inside the container. GitHub Copilot and Grok use device codes. See [subscription connections in web/dev](../docs/development.md#subscription-connections-in-webdev).
- **The server is single-user.** Everyone connected shares the same records, keys, and credentials.
