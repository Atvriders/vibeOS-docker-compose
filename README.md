# vibeOS-docker-compose

A containerized Next.js 15.5 application with Claude Code integration via a type-safe tRPC API. The recommended way to run vibeOS is **Docker Compose using the prebuilt, published image** — no local toolchain required beyond Docker.

The published image bundles everything the app spawns at runtime: the Next.js server, the global `claude` CLI, `python3` + `uv` (for the browser-use and Dedalus Python runners), and a Chrome install for Puppeteer/browser-use.

## Features

- **Next.js app** — built with Bun, TypeScript, Tailwind CSS; runs on port 3000, listening on `0.0.0.0`.
- **tRPC API** — type-safe API surface (crypto, files, gcp, terminal, kernel, dedalus, browser-use).
- **Claude Code integration** — the Claude Code SDK/CLI is invoked server-side to drive code modification and agent workflows.
- **Docker-first** — pull-and-run via the published GHCR image, or build locally for hot-reload development.

## Prerequisites

- Docker (with the `docker compose` plugin).
- An Anthropic API key (`ANTHROPIC_API_KEY`) for Claude Code.

That's it — you do **not** need Bun, Node, Python, or Chrome installed on the host. They all live inside the image.

## Install with Docker Compose

This is the primary, fully-working install path. It pulls the prebuilt image
`ghcr.io/atvriders/vibeos-docker-compose:latest` from GHCR — no local build.

1. **Clone the repo** (you only need the `docker-compose.yml`, but cloning is simplest):

   ```bash
   git clone https://github.com/Atvriders/vibeOS-docker-compose.git
   cd vibeOS-docker-compose
   ```

2. **Paste your Anthropic API key.** The environment now lives **inside the
   compose file** — there is no separate mandatory `.env` step. Open
   `docker-compose.yml`, find the `environment:` block, and replace the
   placeholder for `ANTHROPIC_API_KEY` with your real key:

   ```yaml
   environment:
     # PASTE YOUR ANTHROPIC API KEY HERE (replace the placeholder).
     # A host env var of the same name, if set, overrides this value.
     ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY:-sk-ant-REPLACE_WITH_YOUR_KEY}"
   ```

   With the `${VAR:-default}` form, an exported host env var
   (`export ANTHROPIC_API_KEY=sk-ant-...`) wins if present; otherwise the value
   you pasted inline is used. **Never commit a real key** — keep the obvious
   placeholder in any commit.

3. **Start it** (this pulls `ghcr.io/atvriders/vibeos-docker-compose:latest`):

   ```bash
   docker compose up
   ```

4. **Open the app:** [http://localhost:3000](http://localhost:3000)

To run detached, use `docker compose up -d`; stop with `docker compose down`.

### What runs inside the container

In production the container runs the Next.js production server:

```sh
bun run start
```

`bun run start` is `next start`, which serves the regular Next.js production
build using the full `node_modules` tree shipped in the image. The image is a
deliberately single-stage build: the running app needs the complete
`node_modules` at runtime (the Claude Code SDK/CLI and bundled binaries), plus
the Python runners under `/app/src`, so no multi-stage "standalone" copy is
used. (`next.config.ts` sets
`output: 'standalone'`, but that self-contained `server.js` artifact is **not**
what this image runs — `next start` serves the normal build.) The app listens
on `0.0.0.0` and honors the `PORT` env var (default `3000`). It runs as the
non-root user `nextjs` (uid 502, gid 20) with `HOME=/home/nextjs`.

## Local development (hot reload)

To work on the source with live reload, build the dev image locally (it mounts
your working tree and runs `next dev`):

```bash
docker compose -f docker-compose.dev.yml up --build
```

Then open [http://localhost:3000](http://localhost:3000). Edits on the host are
reflected in the running container.

> The dev image installs `devDependencies` (TypeScript, Tailwind, `@types/*`,
> `eslint-config-next`) because they are required to build. Do **not** set
> `NODE_ENV=production` before `bun install` / `bun run build`, or the build
> will fail; `NODE_ENV=production` is set only for the production runtime.

## Building & publishing the image

The production image is built and published automatically by a GitHub Action.

- **Triggers:** push to `main`, pushing a `v*` tag, or a manual
  `workflow_dispatch`.
- **Registry:** the image is published to GHCR at the hardcoded path
  **`ghcr.io/atvriders/vibeos-docker-compose`** (already lowercase) — the exact
  path the production `docker-compose.yml` pulls as `:latest`. The workflow uses
  `docker/metadata-action` to attach tags. (Pushes to `main` also publish
  additional `:main` and `:sha-xxxxxxx` tags; `:latest` is the one compose
  consumes.)
- **Visibility:** the published package is **public**, so `docker compose up`
  can pull it with no GHCR login.
- **Forks:** if you fork this repo, the **first** build may need to be kicked
  off manually via **Actions → Build and Push Docker Image → Run workflow**
  (`workflow_dispatch`) before pushes to `main` build automatically. A fork
  under a different owner must point both the workflow's `images:` value and
  `docker-compose.yml`'s `image:` at its own GHCR path (the built-in
  `GITHUB_TOKEN` can only publish to its own owner's namespace).

### Building the image by hand

If you want to build the production image locally instead of pulling it:

```bash
docker build -t ghcr.io/atvriders/vibeos-docker-compose:latest .
```

The build runs `bun install --frozen-lockfile` then `bun run build`
(`next build`) with `devDependencies` present, installs Chrome for
Puppeteer/browser-use, switches to the non-root `nextjs` user, and sets
`NODE_ENV=production` for the runtime only (never before the build). It is a
single-stage image that keeps the full `node_modules` tree for runtime.

## Environment variables

Set these in the `environment:` block of `docker-compose.yml` (see **Install with Docker Compose** above).

| Variable | Required | Notes |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | **Yes** | Read from the environment by the Claude Code SDK/CLI. The app does not work without it. The compose placeholder (`sk-ant-REPLACE_WITH_YOUR_KEY`) is intentionally invalid — the container will start, but Claude calls fail at runtime until you paste a real key. |
| `OPENAI_API_KEY` | No | Optional server-side features. Defaults to empty. |
| `DEDALUS_API_KEY` | No | Optional server-side (Dedalus) features. Defaults to empty. |
| `PORT` | No | Server port (default `3000`). |
| `NODE_ENV` | No | `production` at runtime in the published image. Do not set to `production` for a from-source build before install/build. |

### Build-time-only variables (`NEXT_PUBLIC_*`)

`NEXT_PUBLIC_OPENAI_API_KEY` and `NEXT_PUBLIC_KERNEL_API_KEY` are **build-time
only**. `next build` inlines `NEXT_PUBLIC_*` values into the client bundle, so
setting them in the compose `environment:` block of a **prebuilt** image has
**no effect** — runtime env cannot change them. To use these, set them as build
args/env when building the image yourself and rebuild.

## API usage

The app exposes a tRPC endpoint at `/api/trpc`. The available routers are
`crypto`, `files`, `gcp`, `terminal`, `kernel`, `dedalus`, and `browserUse`.
A typical mutation accepts a prompt plus optional context:

```typescript
{
  prompt: string,      // the prompt to send to claude code
  context?: any        // optional context data
}
```

Response format:
```typescript
{
  success: boolean,
  results: Array<{
    type: 'result' | 'text' | 'tool_use',
    content: any,
    timestamp: string
  }>,
  request: object
}
```

## Project structure

```
.
├── src/
│   ├── app/                    # next.js app router
│   ├── server/                 # trpc server & api (routers/)
│   └── utils/                  # utility functions
├── public/                     # static assets
├── Dockerfile                  # production image (single-stage, full node_modules)
├── Dockerfile.dev              # development image (hot reload)
├── docker-compose.yml          # production compose (pulls the published GHCR image)
└── docker-compose.dev.yml      # development compose (builds locally, hot reload)
```

## Troubleshooting

### App not loading
- Ensure port 3000 is free, or change the published port mapping / `PORT`.
- Check logs: `docker compose logs -f`.

### Claude Code errors
- Verify `ANTHROPIC_API_KEY` is set correctly in `docker-compose.yml` (or exported on the host).
- The shipped placeholder `sk-ant-REPLACE_WITH_YOUR_KEY` is invalid on purpose; an auth error here usually means you haven't replaced it yet.
- Confirm the key has the proper permissions.

### Image won't pull
- The GHCR package is public; if a pull fails, retry, or confirm the tag is `ghcr.io/atvriders/vibeos-docker-compose:latest`.
- If you forked the repo, the image publishes under your fork's GHCR path; point `docker-compose.yml`'s `image:` at the path your fork's workflow actually pushed.

### Local dev build fails
- Ensure `bun.lock` is committed (it is) and that `NODE_ENV` is not forced to `production` before the build.

## License

MIT
