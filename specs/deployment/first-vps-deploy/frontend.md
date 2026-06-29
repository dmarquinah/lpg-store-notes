---
project: lpg-store
domain: specs
type: spec-track
spec: first-vps-deploy
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-24
---

# First Production VPS Deploy — lpg-frontend-vue (Piloto) track

Shared spec: [[index]]

## Technical Notes

This track makes **Piloto** deployable as its own container, served by its own nginx,
sitting behind the Traefik edge proxy (see [[backend]] for the proxy + orchestration).
The owner's chosen pattern: **nginx-in-front-of-the-build**, configured independently
of the edge.

**Work in this track:**

- **Production Dockerfile (multi-stage):**
  - Build stage: `node:22` (or the repo's pinned Node) → `npm ci` → `npm run build`
    (Vite) → emits `dist/`.
  - Serve stage: `nginx:alpine` copying `dist/` into the web root with a custom
    `nginx.conf`.
- **nginx config for an SPA:**
  - **History fallback** — `try_files $uri $uri/ /index.html;` so deep links
    (`/pedidos/:id`, `/mis-entregas/:id`, `/mi-dia`, `/invite/:token`, …) resolve.
  - `gzip` on for text assets; long-lived `Cache-Control: immutable` for Vite's
    hashed `assets/*`, but **`index.html` must not be cached** (so new deploys are
    picked up).
  - Serve the PWA bits correctly: `manifest.webmanifest`, the Workbox service worker,
    and `public/push-sw.js` (used by the push-notifications spec) at the right paths /
    content types — the SW must be served from the app origin, no aggressive cache on
    `sw.js`.
  - The container listens on plain HTTP internally; **TLS is terminated at Traefik**
    (no certs in this container).
- **Traefik routing:** the production compose ([[backend]] track) adds this image as a
  service with a Host label for the app host (e.g. `app.<domain>` or the apex) + TLS.
  This track just needs the image to serve `dist/` on a known internal port.
- **Build-time configuration:**
  - `VITE_API_URL` (or the existing equivalent) must point at `https://api.<domain>`
    in the production build — not the localhost dev default. Confirm the var name in
    the repo's `.env.example` + how the axios/fetch client reads it.
  - `VITE_VAPID_PUBLIC_KEY` must be present in the production build so Web Push keeps
    working behind the proxy (it's publishable — safe to bake in; see the
    push-notifications spec).
  - Decide build-arg vs. baked-`.env`: Vite inlines `VITE_*` at build time, so these
    are **build args** passed when the image is built in CI (not runtime env).
- **CI → GHCR:** add a workflow mirroring lpg-backend's (typecheck/build → build image
  → push GHCR `:latest` + `:sha-<short>`). Confirm whether the repo already has any CI
  to extend (Open Question in [[index]]).
- **CORS / cross-origin:** with the two-subdomain layout the SPA on `app.<domain>` calls
  the API on `api.<domain>` — cross-origin. Coordinate with the backend track's CORS +
  token strategy (the alternative is a single-host `/api` path prefix to stay
  same-origin — Open Question in [[index]]).

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)
- `Dockerfile` — **new**: multi-stage build → nginx serving `dist/`
- `nginx.conf` (or `docker/nginx.conf`) — **new**: SPA history fallback, gzip, cache headers, SW/manifest handling
- `vite.config.*` — PWA / build config; confirm SW + base path for production
- `.env.example` — the `VITE_API_URL` / `VITE_VAPID_PUBLIC_KEY` vars and their prod values
- `src/lib/` (the API/axios client) — confirm where the API base URL is read so the prod build targets `https://api.<domain>`
- `public/push-sw.js`, the Workbox-generated SW — must be served correctly by nginx (path + cache)
- `.github/workflows/` — **new or extended**: build + push the frontend image to GHCR

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-24] [lpg-frontend-vue] **Frontend track DONE.** Piloto now ships as its own nginx image behind the shared Traefik edge, deploying **independently** of the backend.
- **Image** `docker/Dockerfile` (multi-stage): `node:22-alpine` → `npm ci` + `npm run build` (vue-tsc + Vite) → `dist/`, served by **`nginx:alpine`**. Owner chose alpine over distroless for the MVP (official, debuggable, canonical SPA recipe; the serve stage is a one-line swap to distroless later). VITE_* are passed as **build args** (declared after `npm ci`, promoted to ENV before build, since Vite inlines them at build time): `VITE_API_URL=/api` (default) + `VITE_VAPID_PUBLIC_KEY` (publishable). Verified the args bake into the bundle.
- **Same-origin `/api` (owner decision — supersedes the spec's literal cross-origin AC).** `docker/nginx.conf` reverse-proxies `location /api/` → `http://api:3000` over `lpg-edge` via the Docker resolver (`127.0.0.11`) + a variable upstream (so nginx boots even if the backend is down; forwards `/api/v1/...` verbatim — confirmed the backend mounts every router under `/api/v1` in `src/app.ts`). The built app keeps `VITE_API_URL=/api` exactly as in dev — **no CORS, no mixed-content, zero backend changes**. Plus SPA history fallback (`try_files … /index.html`), gzip, immutable cache on hashed `/assets/`, and no-cache on `index.html` + `sw.js`/`push-sw.js`/`registerSW.js`/`manifest.webmanifest` (served `application/manifest+json`) so SW/PWA updates land.
- **Compose** `docker/docker-compose.yml`: one `frontend` service (GHCR image) on the external `lpg-edge` **only** (never the db/internal net), Traefik labels mirroring the backend exactly — `Host(${APP_HOST})`, `websecure`, `tls=true`, certresolver `le`, `secure-headers@file`, LB server port `80`. `docker/.env.prod.example` (`FRONTEND_IMAGE` + `APP_HOST`).
- **CI** `.github/workflows/main.yml` (net-new for this repo): `ci` (typecheck + build) + `publish` (main-only) → build `docker/Dockerfile` with the VITE_* build args (VAPID from a GitHub **variable**, not a secret) → GHCR `:latest` + `:sha-<short>`. **No VPS secrets** (only `GITHUB_TOKEN`); deploy stays manual SSH `compose pull && up -d`, mirroring the backend.
- **Docs** `docs/DEPLOYMENT.md` — topology, the image + build args, the nginx-config rationale, CI/GHCR, and the VPS deploy sequence (slots in after lpg-backend's `VPS-SETUP.md`). `.dockerignore` added (keeps `docker/` in build context). Removed the stale Netlify `public/_redirects`; `.env.example` `VITE_API_URL` comment → `/api`. **No source changes** — `apiClient`/`vite.config.ts` already did same-origin `/api`.
- Gates green: host typecheck + build; `docker compose config`; `docker build`; container smoke (`/` + deep-link `/mi-dia` → 200 index.html; index/SW/manifest `no-cache`; `/assets` immutable; manifest content-type; `/api/*` → 502 without a backend = proxy wired). Independent validation: **all 6 frontend + cross-cutting criteria MET**, no blocking bugs.

All criteria for this repo are met → frontend track **done**. The backend track was already done, so the overall **`first-vps-deploy` spec is now `done` (both tracks)**. The first VPS deploy itself remains a manual owner action per the runbooks; CI publishes the image, then the owner pulls + brings up the frontend stack alongside the backend/edge.

[2026-06-25] [lpg-frontend-vue] **Post-merge refinement (owner review — track stays done).** Moved build-time config out of the Dockerfile/CI into a committed **`.env.production`** that Vite auto-loads for `vite build`. Rationale: every `VITE_*` var is inlined into the client bundle and is therefore public by definition, so one committed file is safe and becomes the **single place to edit** when env changes — no per-var `ARG`/`ENV` in `docker/Dockerfile`, no `build-args` (and no `vars.VITE_VAPID_PUBLIC_KEY` GitHub-variable step) in CI. `.dockerignore` now keeps `.env.production` in the build context (still excludes `.env` + `*.local`). `docs/DEPLOYMENT.md` updated to match, and its VPS deploy steps now use **`scp`** over the `lpg-vps` SSH alias (mkdir → scp compose + `.env` → ssh → `chmod 600` → `pull` → `up -d`). Re-verified: image builds with **no build-args**; the VAPID key + `/api` bake in from `.env.production` (localhost fallback absent); serve smoke (`/`, deep-link `/mi-dia` → 200) green.

[2026-06-25] [lpg-frontend-vue] **Deploy-mechanism refinement (owner review — track stays done).** Switched the VPS deploy from hand-`scp`-ing files to **cloning the repo + `git pull`** to update, so a compose / Traefik-label change is a pull, not a re-copy. `docs/DEPLOYMENT.md` reworked accordingly: clone to `/srv/lpg-frontend`, create the gitignored `docker/.env` from `.env.prod.example`, run compose from the cloned `docker/` dir; routine release = `git pull` + `docker compose pull && up -d` (skip the pull for an image-only release). Confirmed `docker/nginx.conf` is **baked into the image** (the serve stage `COPY`s it), so it never lives on the VPS — only the compose file + `.env` do. Added a stable `name: lpg-frontend` to the compose so the project name doesn't default to the `docker/` subdir; `docker compose config` still validates.