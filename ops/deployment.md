# Deployment — lpg-store

---
project: lpg-store
domain: ops
last-updated: 2026-06-24
---

The whole product runs on a **single self-hosted VPS** as Docker Compose stacks
behind a **Traefik** edge proxy. The repo runbooks are the source of truth:
`lpg-backend/docs/VPS-SETUP.md` (one-time provisioning) and
`lpg-backend/docs/DEPLOYMENT.md` (per-release). This page is the index.

## Topology

Three **independently-deployed** pieces that meet only on a shared external
network `lpg-edge`:

- **edge** — `infra/traefik/` (separate compose): terminates TLS (**automatic
  Let's Encrypt**), redirects HTTP→HTTPS, applies security headers, routes by
  host (`api.<domain>` → backend `api:3000`, `app.<domain>` → frontend).
- **backend app** — `docker/docker-compose.yml`: `db` + one-shot `migrate` +
  profile-gated `seed` + `api`. `api` is on `lpg-edge` (Traefik) + `lpg-internal`
  (db); **`db` is on `lpg-internal` only — never on the edge**.
- **frontend** (Piloto) — its own nginx image + compose in the **lpg-frontend-vue**
  repo, attached to `lpg-edge` with its own Traefik labels. Not defined in
  lpg-backend.

**No Redis** in the MVP stack — logout degrades to a stateless client-side token
drop (no server-side blocklist; tokens expire in 24 h).

## Repository layout (centralized)

- `lpg-backend/docker/Dockerfile` — backend image
- `lpg-backend/docker/docker-compose.yml` — backend **app** stack
- `lpg-backend/docker/.env.prod.example` — app `.env` template
- `lpg-backend/infra/traefik/docker-compose.yml` — **Traefik edge** stack
- `lpg-backend/infra/traefik/dynamic.yml` — security-headers middleware
- `lpg-backend/infra/traefik/.env.example` — Traefik `.env` template (ACME email)

On the VPS, `/srv/lpg-backend/` is a **git clone** of this repo — config changes
are applied with `git pull`. Run each stack from its own directory (`docker/`,
`infra/traefik/`) so Compose auto-loads that directory's `.env` (named exactly
`.env`; `.env.prod` is not auto-loaded) — no `-f` / `--env-file` flags. The two
`.env` files live in the tree but are **gitignored**, so `git pull` never touches
them. Compose project names are pinned (`lpg-backend` / `lpg-traefik`) so
volumes/networks stay stable regardless of run directory. There is **no dev
compose** (local dev runs the app on the host).

## CI/CD

Push to `main` triggers `.github/workflows/main.yml`:

1. **`ci` job** — `check` → `typecheck` → `build` → `test`.
2. **`publish` job** (needs `ci`, main only) — builds `docker/Dockerfile`, tags
   `:latest` + `:sha-<short>`, pushes to GHCR. **CI never touches the VPS.**

**Deploys are manual** (owner decision): SSH to the VPS and
`docker compose pull && docker compose up -d` from `/srv/lpg-backend/docker`; config
changes (compose / Traefik) apply via `git pull` on the clone. `migrate`
runs automatically before `api`; the catalog `seed` is a manual one-shot
(`docker compose run --rm seed` — in production it seeds only the product
catalog, no demo users/customers). So CI needs **no VPS secrets** — only the
automatic `GITHUB_TOKEN` (`packages: write`).

## First-time provisioning (one-time)

`docs/VPS-SETUP.md`: non-root sudo user → SSH hardening → `ufw` (SSH + 80 + 443)
→ Docker + compose → DNS A-records → `docker network create lpg-edge` → **clone the
repo** (read-only deploy key if private) → create the two `.env` files (mode 600) → bring up Traefik then the backend app → seed the
catalog → **create the first developer user** via a one-time `BOOTSTRAP_TOKEN` +
`POST /api/v1/auth/bootstrap` (then remove/rotate the token) → verify HTTPS. All
further users / the real store / customers are created in-app.

## Rollback

SSH to the VPS, pin `IMAGE` in `docker/.env` to a previous `:sha-<short>` tag
(retained on GHCR), then from `/srv/lpg-backend/docker`: `docker compose pull && up -d`.

## Related Files

- `lpg-backend/docs/GITHUB-SETUP.md` — GitHub/GHCR one-time setup (Actions, package visibility, VPS pull token)
- `lpg-backend/docs/VPS-SETUP.md` — one-time provisioning runbook
- `lpg-backend/docs/DEPLOYMENT.md` — per-release runbook
- `lpg-backend/.github/workflows/main.yml` — CI + image publish
- `lpg-backend/docker/docker-compose.yml`, `lpg-backend/docker/Dockerfile`
- `lpg-backend/infra/traefik/docker-compose.yml`, `lpg-backend/infra/traefik/dynamic.yml`
- `lpg-frontend-vue/docs/DEPLOYMENT.md` — frontend deploy runbook
- `lpg-frontend-vue/docker/Dockerfile`, `lpg-frontend-vue/docker/nginx.conf`, `lpg-frontend-vue/docker/docker-compose.yml`
- `lpg-frontend-vue/.github/workflows/main.yml` — frontend CI + image publish

## Frontend (lpg-frontend-vue)

**Done.** Piloto ships as its own **`nginx:alpine` image behind Traefik** on the
same VPS (route `app.<domain>`), attached to `lpg-edge` — deployed **independently**
of the backend (its own image, compose, and GHCR pipeline). The frontend repo owns:

- `docker/Dockerfile` — multi-stage Vite build → nginx serving `dist/`. VITE_* (the
  API base + the publishable VAPID key) are **build args**, baked in CI.
- `docker/nginx.conf` — SPA history fallback, gzip, immutable cache on hashed
  `/assets/` + no-cache on `index.html`/service-workers, and a **same-origin `/api`
  reverse-proxy to `api:3000`** over `lpg-edge` (the SPA calls its own origin — **no
  CORS, no backend change**; chosen over the two-subdomain layout).
- `docker/docker-compose.yml` — the `frontend` service + Traefik labels
  (`Host(app.<domain>)`, resolver `le`, `secure-headers@file`, LB port 80) on the
  external `lpg-edge` **only** (never the db network).
- `.github/workflows/main.yml` — `ci` + GHCR `publish` (`:latest` + `:sha-<short>`),
  **no VPS secrets**; deploy is manual `docker compose pull && up -d` from the
  frontend stack dir.
- `docs/DEPLOYMENT.md` — the frontend deploy runbook (build args, nginx rationale,
  VPS sequence — slots in after the backend's `VPS-SETUP.md`).

## Bot (lpg-bot)

Deployment not yet documented. Can join the same `lpg-edge` edge later.
