# Build-deploy benchmark fixture: static-small (Vue docs, VitePress)

## Source

- Upstream: https://github.com/vuejs/docs at `40aa88af0094f7bab4aaf786e55c748a6a251d88` (2026-09-15). The text is CC BY 4.0 (`LICENSE`). The images are not under that license and keep their authors' copyright, so this repo stays private.
- This repo: https://github.com/build-deploy-bench/vuejs-docs-static (private, independent copy). The first commit is the unmodified upstream tree at the pinned commit, without its history.
- Output: `.vitepress/dist`, about 30 MB and 545 files (544 upstream files, plus `/version` and the touch page). Source 6 MB. The build makes no network requests.
- Two builds of the same commit are not byte-identical: the CSS hash changes, and about 113 files differ. This is accepted.

## Patches

1. Upstream GitHub automation removed (`.github/workflows/automerge.yml` and `.github/dependabot.yml`). Dependabot would open PRs in this copy.
2. `engines.node` changed from `>=20` to `24.x` in `package.json`. Vercel takes the Node version from `engines.node` and ignores the project setting: `>=20` gave Node 26. `.node-version` (`24.22.3`) pins Render, and the Dockerfile pins `node:24.22.3-slim` for Railway and Fly.
3. `vercel.json`: added `framework`, `installCommand`, `buildCommand`, `outputDirectory` and `git.deploymentEnabled: false` in front of the upstream headers and rewrites, which stay.
4. Added `bench/build.sh`, `bench/serve.sh`, `Caddyfile`, `Dockerfile`, `.dockerignore`, `fly.toml`, `.node-version` and the touch page `src/bench-touch.md`.

## Commands

| Step | Command | Markers |
|---|---|---|
| Toolchain | `node --version`, `corepack pnpm --version` (before the install marker) | |
| Install | `./bench/build.sh install` = `corepack pnpm install --frozen-lockfile` | `::bench::install_start` / `::bench::install_done` |
| Build | `./bench/build.sh build` = `corepack pnpm run build`, then write `.vitepress/dist/version` | `::bench::build_start` / `::bench::build_done` |
| Start (Railway, Fly) | `sh /bench/serve.sh` (Caddy) | `::bench::start` / `::bench::listening` |
| Health | `GET /` returns 200 | |

pnpm is 10.33.0 from `packageManager` through Corepack on every provider. On Render, the first `node` call installs Node and the first `pnpm` call downloads pnpm, so the build script makes both calls before `install_start`.

`/version` is a file with no extension in the output: `{"commit":"<sha>","run":"<BENCH_RUN_ID>"}`. Every provider serves it without a rewrite rule (Render and Vercel as `application/octet-stream`). The commit comes from the first value set in `BENCH_COMMIT`, `RENDER_GIT_COMMIT`, `RAILWAY_GIT_COMMIT_SHA` or `VERCEL_GIT_COMMIT_SHA`, then `git rev-parse HEAD`, then `unknown`. On Railway and Fly, `serve.sh` writes the file again at start with the runtime `BENCH_RUN_ID` and `RAILWAY_DEPLOYMENT_ID` and keeps the built commit.

Docker image: build stage `node:24.22.3-slim`, final stage `caddy:2.11.9-alpine` plus `coreutils` (BusyBox `date` has no milliseconds for the markers). 96 MB on Fly (linux/amd64, uncompressed).

## Providers

Render and Vercel serve the output from their CDNs, with no instance. Railway and Fly have no static hosting product, so Caddy serves the output from a container. Compare only `build_total` and `e2e` across the four providers.

### Render

- Workspace `tea-dbmbhbg8v5cc73e4m0jg` (Pro). Static site `srv-dbqu87u7bikc738ap83g` `bdb-vuejs-docs-static`, https://bdb-vuejs-docs-static.onrender.com. Static sites are free and have no region or instance plan. Build pipeline `starter`.
- Build command `./bench/build.sh all`, publish path `.vitepress/dist`. Env: `SKIP_INSTALL_DEPS=true` (Render would otherwise run its own `pnpm install` before the build command), `BENCH_RUN_ID` (set by the harness; it reaches the build). Auto-deploy off, PR previews off.
- Logs: `==> Uploading build...`, then `==> Your site is live 🎉`, with no end-of-upload line and no `==> Deploying...`. `build_ended` comes at the live line.
- The new `/version` answers about 20 s after `Your site is live`, also with a cache-busting query string, so the delay is Render's publish and not the CDN cache (`s-maxage=300`). Deploys `dep-dbqu8obrjlhs73ce94ug`, `dep-dbqu9eoae00c73deiua0`.

### Railway

- Workspace `d535606c-49a7-4a6c-9e29-ce73b5f01428`. Project `bdb-vuejs-docs-static` `6325af46-5034-45b4-98bf-798c29c8d245`, environment `production` `6bd2e816-ff6a-4937-adfe-b6ea12836ef9`, service `4db18587-50dd-4334-8e65-f27634559aaa`. https://bdb-vuejs-docs-static-production.up.railway.app
- Builds the root `Dockerfile` (Railway uses it when it exists). Health check `/`. No variables. One replica in the default region.
- Auto-deploy disabled with `serviceInstanceAutoDeployUpdate` and confirmed with `serviceInstanceAutoDeployStatus`.
- Setup deploy `49d3b8d3-6fbb-4d19-bfd6-8b102aa830e7` (the connect deploy of `d2dd278`): `/version` returned the commit and the deployment ID.

### Vercel

- Team `build-deploy-bench`. Project `bdb-vuejs-docs-static` `prj_FkcYg94EMQY6UzoSw7AQ9P8EKK7E`, https://bdb-vuejs-docs-static.vercel.app. No Git link; deploys upload a `git archive` of the commit with `vercel deploy --prod`.
- Framework preset `vitepress` (from `vercel.json`). Project Node version `24.x`. Build machine: team default, `standard` (4 cores, 8 GB) with `elastic` selection.
- The harness passes `--build-env BENCH_RUN_ID=<id> --build-env BENCH_COMMIT=<sha>`, so `/version` has the right commit and run.
- Setup deploy `dpl_2x4ShgJ4WsdhVXcjXoQ4i7kStvmK`. Above the fast-static threshold (10 HTML files, 5 MB), so Vercel runs the build.

### Fly.io

- Org `personal`. App `bdb-vuejs-docs-static`, https://bdb-vuejs-docs-static.fly.dev. One Machine, `iad`, `shared-cpu-1x` with 256 MB. Health check `/`.
- Deploy: `flyctl deploy --config fly.toml -a bdb-vuejs-docs-static --remote-only --depot=false --ha=false --yes --strategy bluegreen --build-arg BENCH_COMMIT=<sha> --env BENCH_RUN_ID=<id>` from a `git archive` of the commit. Setup deploy of `d2dd278` returned the right commit and run ID.
- The build step is slow on Fly's remote builder (`fly-builder-rosy-headland-8386`, `shared-cpu-8x`, 8 GB): `pnpm run build` took 297 s while another thread's Fly builds shared the builder, and 124 s alone (2026-11-12). It takes about 12 s on Render and 6 s on a laptop. Shared-CPU throttling of the builder is the likely cause; it applies to every fixture that uses this builder.

## Local checkout

The harness checkout in `build-deploy-bench-projects/` uses the HTTPS remote with `gh auth git-credential` (repo-local config), because the SSH agent refuses to sign for the harness when it runs in the background.
