# Linux container

`deploy/Dockerfile` packages the standalone server so a Linux host can run LearnHarness as a long-lived web server. The image contains the curated `.next/standalone` tree, including the four core plugins, on a Node 26 Debian base, plus the Claude Code CLI.

> [!WARNING]
> **Web mode has no authentication.** Anyone who can reach the port can read your learning records, change settings and API keys, and spend on your connected providers. Only make the image reachable through a private, access-controlled network or an authenticating proxy. Never publish the port to the public Internet.

The same-origin checks in the server are not user authentication. See [subscription connections in web/dev](../docs/development.md#subscription-connections-in-webdev).

## Build

Run from the repository root:

```sh
docker build -f deploy/Dockerfile -t learnharness:dev .
```

The build runs `npm run plugins:setup`, which installs each plugin project's locked dependencies, so it needs npm registry access. The image is built for the architecture of the machine that builds it, because `npm ci` installs that platform's native `@resvg/resvg-js` and `@napi-rs/canvas` prebuilds. Build on the host that will run the image, or on one of the same architecture.

| Image | `docker images` | `docker image inspect` `.Size` |
| --- | --- | --- |
| `linux/arm64` | 1.1 GB | 289 MB |

The first figure is the unpacked size. The second is the compressed size on the containerd image store; other image stores can report differently. Other architectures have not been measured. An `amd64` host builds its own image from the same Dockerfile. The build stage uses about 4.5 GB of disk.

The Dockerfile uses no BuildKit-only syntax, so it builds with or without `buildx`. The legacy builder prints a deprecation notice but works. The root `.dockerignore` keeps `data/`, key files, `.env*` and `*.env` files out of the build context.

| Build argument | Default | Purpose |
| --- | --- | --- |
| `CLAUDE_CODE_VERSION` | `2.1.311` | Pinned CLI version. LearnHarness requires at least 2.1.259. |
| `NODE_IMAGE` | `node:26.13.0-trixie-slim` | Base image for both stages. |

## Run

```sh
docker run -d --name learnharness --init --restart unless-stopped \
  --cap-drop ALL --security-opt no-new-privileges:true \
  --memory 2g --cpus 2 --pids-limit 512 \
  -p 127.0.0.1:3000:3000 \
  -v learnharness-data:/data \
  -v learnharness-auth:/auth \
  --env-file /path/to/learnharness.env \
  learnharness:dev
```

This binds the port to loopback so that a private network or proxy on the host can front it. `--init` is recommended for a long-lived container because Node as process 1 does not reap orphaned processes. `--env-file` is optional; keep that file readable only by its owner.

The server runs as uid/gid 1000 and its tree is read-only to that user. The server needs no Linux capabilities, so `--cap-drop ALL` and `no-new-privileges` cost nothing. The limits bound what runaway submitted code can consume; an idle server uses about 200 MiB and a dozen processes.

Anyone who can reach the port can run code as that user, for example through the answer to a Coding Questions problem, which its tests execute. That code can read the container's environment, including `CLAUDE_CODE_OAUTH_TOKEN`, and the files in `/auth`. Treat access to the port as access to those credentials.

### Volumes

| Path | Holds |
| --- | --- |
| `/data` | Chats, jobs, graph, settings, prompt overrides, themes, and plugin records (`core-plugin-data/`, `plugins/`). |
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
| `LH_ALLOWED_HOSTS` | Host names the API accepts besides `localhost` and IP addresses. Comma-separated, case-insensitive, port ignored, no wildcards. **Required** when browsers reach the server by a DNS name, such as a proxy or tailnet name. |
| Provider and search API keys | See [configuration](../docs/development.md#configuration). Environment keys take precedence over keys saved in Settings. |

Do **not** set `LH_DESKTOP_SERVER` or `LH_DESKTOP_API_SECRET`. They are for the desktop shell and lock browsers out of the API; they are not a way to add authentication.

### Behind a proxy

The API refuses any `Host` that is not `localhost`, a `*.localhost` name, or an IP address, unless the name is listed in `LH_ALLOWED_HOSTS`. This guards against DNS rebinding. A proxy at `https://learn.example.ts.net` therefore needs `LH_ALLOWED_HOSTS=learn.example.ts.net`. Without it the page loads, but `/api` requests answer 403. The `HEALTHCHECK` calls `127.0.0.1`, which is always allowed, so it stays healthy either way.

Same-origin routes, including the Claude Code status check and subscription sign-in, compare the browser's `Origin` with the request. A proxy must preserve the original `Host` header and, when it terminates TLS, send `X-Forwarded-Proto: https`. Otherwise these routes answer 403.

## Verify

```sh
docker image inspect learnharness:dev --format '{{.Os}}/{{.Architecture}} {{.Size}}'
docker run --rm -d --name lh-smoke -p 127.0.0.1:3100:3000 \
  -v lh-smoke-data:/data -v lh-smoke-auth:/auth learnharness:dev
sleep 3

asset=$(curl -s http://127.0.0.1:3100/ | grep -o '/_next/static/[^"]*\.js' | head -1)
[ -n "$asset" ] || echo "no static asset found on /"
for route in / /api/graph /api/models /api/settings /api/plugins "$asset"; do
  curl -s -o /dev/null -w "$route %{http_code}\n" "http://127.0.0.1:3100$route"
done
docker exec lh-smoke id
docker exec lh-smoke claude --version

# Enable Flashcards and print its status.
docker exec lh-smoke node -e "fetch('http://127.0.0.1:3000/api/plugins/flashcards', { method: 'PUT', headers: { 'content-type': 'application/json' }, body: '{\"enabled\":true}' }).then((r) => r.json()).then((j) => console.log(j.plugins.find((p) => p.id === 'flashcards').status))"

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

Each route should answer 200, `/api/plugins` should list the four core plugins, the Flashcards line should print `active`, `id` should report uid 1000, and the final settings should show `claudeCode.enabled: true`. The container also has a `HEALTHCHECK` on `GET /api/settings`.

## Upgrading

Back up `/data` before you move to a newer image. Some upgrades change record formats in one direction:

- **v0.5.0 → v0.6.x:** chat records move from format 1 to 2. Existing chats stay readable and are rewritten as format 2 the next time they change. A v0.5.0 image skips format-2 chats as if they were absent, so to roll back, restore the pre-upgrade backup. New directories (`themes/`, `core-plugin-data/`, `plugins/`) are ignored by older images.

## Core plugins

Flashcards, Coding Questions, Drawing Pad and Rich Code Blocks ship inside the image and start disabled. Enable them in **Settings → Core plugins**. Their records live under `/data/core-plugin-data/<id>/`; the code stays in the read-only image. Community plugins load from `/data/plugins/` and run as trusted, unsandboxed code. Install only code you trust.

## Claude Code

The provider is off by default. After first boot, enable it in **Settings → Providers → External CLI → Claude Code** and select a model. The CLI is already installed and on `PATH`; only authentication is needed. See [Claude Code](../docs/claude-code.md).

To change the CLI version, rebuild with `--build-arg CLAUDE_CODE_VERSION=<version>`.

## Limitations

- **Coding Questions can only use the interpreters in the image:** `node`, `sh`, `bash` and `perl`. Its tests execute programs found on the server's `PATH`, as the server user and without a sandbox. The image has no Python or compilers, so tests that need them fail.
- **Demo verification is not included.** It needs Chromium, which the image does not contain, so it fails. The rest of the app is unaffected.
- **ChatGPT sign-in needs a browser on the server's machine.** Its callback listens on `127.0.0.1` inside the container. GitHub Copilot and Grok use device codes. See [subscription connections in web/dev](../docs/development.md#subscription-connections-in-webdev).
- **Visual text is limited to DejaVu's coverage.** Latin, Greek, and Cyrillic render. Scripts such as CJK, Arabic, and Devanagari do not, because the image carries no fonts for them.
- **There is no system CA store.** Node and the CLI use their built-in roots, so a private CA or TLS-inspecting proxy needs extra configuration.
- **The server is single-user.** Everyone connected shares the same records, keys, and credentials.

## License

LearnHarness is licensed under [AGPL-3.0](../LICENSE). If other people use a modified version over a network, section 13 requires you to offer them its source.
