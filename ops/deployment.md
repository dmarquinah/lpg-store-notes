# Deployment — lpg-store

---
project: lpg-store
domain: ops
last-updated: 2026-05-07
---

## Backend (lpg-backend)

The full runbook lives in the repo at `lpg-backend/docs/DEPLOYMENT.md`. This page is the index.

### Pipeline

Push to `main` triggers `.github/workflows/main.yml`:

1. **`ci` job** — typecheck → build → test
2. **`deploy` job** (needs `ci`) — builds the Docker image, tags `:latest` + `:sha-<short>`, pushes to GHCR, SSHes to the VPS and runs `docker compose pull && up -d --remove-orphans`. Polls `HEALTH_URL` for up to 30s and fails on non-200.

### Stack (on VPS)

`/srv/lpg-backend/` holds:

- `docker-compose.yml` — copied from the repo
- `.env` — production env (mode 600)
- named docker volumes for `db-data` and `redis-data`

Compose services: `db` (postgres:16-alpine), `redis` (redis:7-alpine), `migrate` (one-shot, exits 0 before app starts), `app` (distroless image from GHCR).

### Required secrets (GitHub repo)

- `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`
- `HEALTH_URL` — public URL to `/health` (e.g. `https://api.example.com/health`)
- `GITHUB_TOKEN` is automatic; the workflow uses it for GHCR pushes via `packages: write`

### Rollback

SSH to VPS, pin `IMAGE` in `.env` to a previous `:sha-<short>` tag (always retained on GHCR), then `docker compose pull && up -d`.

### Related Files

- `lpg-backend/docs/DEPLOYMENT.md` — full runbook
- `lpg-backend/.github/workflows/main.yml`
- `lpg-backend/docker-compose.yml`
- `lpg-backend/Dockerfile`

## Frontend (lpg-frontend-vue)

Deployment not yet documented. Likely a static-host target (Cloudflare Pages, Vercel, or VPS-served static).

## Bot (lpg-bot)

Deployment not yet documented.
