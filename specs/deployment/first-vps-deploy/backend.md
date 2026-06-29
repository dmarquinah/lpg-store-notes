---
project: lpg-store
domain: specs
type: spec-track
spec: first-vps-deploy
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-24
---

# First Production VPS Deploy — lpg-backend track

Shared spec: [[index]]

## Technical Notes

This track owns the **VPS orchestration + the provisioning runbook** (the compose
file and `docs/DEPLOYMENT.md` live in this repo), plus the backend container's place
behind the proxy. It does **not** build the frontend image (see [[frontend]]) — it
only references it from the production compose.

**What already exists (extend, don't rewrite):**

- `docker-compose.yml` — single dev+prod file: `db` (postgres:16-alpine), `redis`
  (redis:7-alpine), one-shot `migrate`, `app` (distroless image from GHCR). `app`
  currently publishes `${APP_PORT:-3000}:3000`; `db`/`redis` publish 5432/6379.
- `.github/workflows/main.yml` — CI (typecheck/build/test) → build image → push GHCR
  `:latest` + `:sha-<short>` → SSH to VPS → `compose pull && up -d` → poll `HEALTH_URL`.
- `docs/DEPLOYMENT.md` — provisioning + secrets + rollback + troubleshooting runbook.
- `POST /auth/bootstrap` (auth module) gated by `BOOTSTRAP_TOKEN` — the first-user
  mechanism the runbook must document.

**Work in this track:**

- **Add a Traefik edge service** to the production composition: ports 80/443, the
  Let's Encrypt ACME resolver (HTTP or TLS challenge) with a persisted `acme.json`
  volume (mode 600), HTTP→HTTPS redirect middleware, and the internal docker network.
  Route the backend via labels on `app`: `Host(\`api.<domain>\`)`, TLS on, service
  port 3000.
- **Stop publishing host ports** in production: drop `app`'s `ports:` (reachable only
  via Traefik), and drop `db`/`redis` `ports:` (internal-only). Keep them published in
  the dev path — see the profile/overlay decision below.
- **Dev/prod separation.** The single-compose preference ([[../../../../MEMORY... ]] —
  `feedback_unified_compose`) says one file; gate Traefik + the frontend service (and
  the prod-only "no published ports" stance) behind a compose **profile** (e.g.
  `--profile prod`) so `docker compose up -d db redis` + `npm run dev` is untouched for
  daily backend work ([[../../../../ ]] `project_local_dev_postgres_no_redis`). If
  profiles can't cleanly express the published-ports difference, fall back to a
  `docker-compose.prod.yml` overlay — but try profiles first (Open Question in [[index]]).
- **Reference the frontend image** (GHCR) as a service in the prod composition with its
  own Traefik Host label; the image itself is produced by the [[frontend]] track.
- **Env-drive domains/secrets:** `<domain>`, the ACME contact email, and
  `BOOTSTRAP_TOKEN` come from the production `.env` (mode 600) — never committed.
- **CI:** the existing deploy job should still roll only the backend image; the
  `HEALTH_URL` check now hits `https://api.<domain>/health` **through Traefik** (verify
  TLS is up before the poll, or keep an internal health check). Confirm the SSH deploy
  step copies/refreshes any new compose + Traefik static config onto the VPS.

**Runbook additions (`docs/DEPLOYMENT.md`) — the documentation deliverable:**

1. **Fresh-box first-time section** (new, before the existing "One-time VPS
   provisioning"): create the owner's non-root **sudo login user** (`adduser` +
   `usermod -aG sudo`); SSH hardening (`PermitRootLogin no`, `PasswordAuthentication
   no`, key-only); firewall (`ufw` allow OpenSSH + 80 + 443, deny the rest); install
   Docker + compose; DNS A-records for `api.<domain>` + the app host → VPS IP.
2. **First developer user** (new): set a one-time `BOOTSTRAP_TOKEN` in `.env`, bring
   the stack up, `curl -X POST https://api.<domain>/auth/bootstrap` (document the exact
   body — token + the developer's name/phone/password) **once**, then **remove/rotate
   `BOOTSTRAP_TOKEN`** and `compose up -d` to drop it. State that all further users are
   created in-app from this developer account.
3. Update the `.env` template (add `BOOTSTRAP_TOKEN`, the domain(s), ACME email) and
   the "first deploy" command sequence to bring up the **whole** stack (Traefik +
   backend + frontend + db + redis).

Confirm the exact `POST /auth/bootstrap` request shape against the auth module before
writing the curl example (read `src/modules/auth/routes.ts` + `service.ts`).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `docker-compose.yml` — add Traefik + frontend services, prod profile, drop published ports in prod
- `docs/DEPLOYMENT.md` — the runbook to extend (fresh-box section + first-user bootstrap)
- `.github/workflows/main.yml` — deploy job; health check now via HTTPS through Traefik
- `Dockerfile` — backend distroless image (unchanged; referenced for context)
- `src/modules/auth/routes.ts`, `src/modules/auth/service.ts` — `POST /auth/bootstrap` + `BOOTSTRAP_TOKEN` gate (confirm the exact contract for the runbook)
- `src/app.ts` — CORS / trust-proxy config (review for running behind Traefik + cross-subdomain frontend)

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-24] [lpg-backend] **Backend track DONE.** Centralized the prod deploy under `docker/` + `infra/`:
- Moved `Dockerfile` → `docker/Dockerfile`; replaced the old root single compose with `docker/docker-compose.yml` — a **prod-only** stack (no dev compose; local dev runs the app on the host). Services: `traefik` v3 (Let's Encrypt HTTP-01, HTTP→HTTPS redirect, `secure-headers@file`, routes `Host(${API_HOST})`→`app:3000` / `Host(${APP_HOST})`→`frontend:80`), `db` (no published ports), one-shot `migrate` (reuses the same distroless app image), `app` (no published ports, Traefik labels), `frontend` (profile-gated `frontend`, `${FRONTEND_IMAGE}`, **off** until the FE track publishes its image). **No Redis** (2 GB box) — only Traefik binds 80/443.
- `infra/traefik/dynamic.yml` (security-headers middleware) + `infra/.env.prod.example` (prod env template). `.dockerignore` excludes `docker`/`infra`; `.gitignore` un-ignores `.env.prod.example`.
- Backend code: `app.set('trust proxy', 1)` (behind the edge); `AuthService.logout` degrades to a stateless **204** when no cache is configured instead of 503 (+1 lifecycle test → 103 tests).
- CI: a **real** `deploy` job in `main.yml` (needs `ci`, main-only) — build `docker/Dockerfile` → GHCR `:latest`+`:sha-<short>` → `scp` compose (flattened) + `infra/` to `/srv/lpg-backend` → `compose pull && up -d --remove-orphans` → health-check `${HEALTH_URL}` **through Traefik over HTTPS**. (Previously the documented pipeline did **not** exist — `main.yml` had only a `ci` job.)
- Runbook `docs/DEPLOYMENT.md` rewritten: fresh-box (sudo user → SSH hardening → `ufw` SSH/80/443 → Docker → DNS A-records → `.env` mode 600 → bring-up → end-to-end HTTPS) + first-developer bootstrap (`POST /api/v1/auth/bootstrap`, body `{token,email,name,phone?,password}`, set→call→remove the token, then all users in-app) + env-file guidance (the prod env MUST be named exactly `.env` — `.env.prod` is never auto-loaded; the gotcha the owner hit). README + vault `ops/deployment` updated; **ADR-019** recorded; the `feedback_unified_compose` memory updated.
- Env-driven (no committed hosts/emails). Gates green: typecheck + biome check + **103 tests** + build + `docker compose config` (both profiles, verified against a flattened VPS-like layout). Independent validation: **all 7 backend criteria MET**, no blocking gaps.

All criteria for this repo are met. **Remaining (separate repo `lpg-frontend-vue`, frontend track):** the multi-stage Vite→nginx image + `nginx.conf` (SPA history fallback, gzip, cache headers, SW/manifest), `VITE_API_URL`/`VITE_VAPID_PUBLIC_KEY` build args pointing at `https://api.<domain>`, and the GHCR image workflow. The compose `frontend` service is wired and waiting — enable `COMPOSE_PROFILES=frontend` once that image is published. The cross-cutting "whole stack over HTTPS" criterion closes when the FE track ships.

The first VPS deploy itself is manual (owner provisions the box + bootstraps the first user per the runbook); CI then rolls subsequent pushes. Owner has a domain + can get VPS access anytime.

[2026-06-24] [lpg-backend] **Follow-up refinements (owner review, track stays done).** Restructured the deploy for a cleaner, more decoupled layout:
- **Traefik split out of the app compose** into its own `infra/traefik/docker-compose.yml` (edge infra ≠ application). `docker/docker-compose.yml` is now the **backend app only**: `db` + one-shot `migrate` + profile-gated `seed` + `app`. The `frontend` service was **removed** — it ships from the lpg-frontend-vue repo and just attaches to the shared network.
- **Explicit networks:** `lpg-edge` (shared **external** — `docker network create lpg-edge` once; Traefik + app + the frontend) and `lpg-internal` (private backend↔db; **db never on the edge**). Traefik routes across the separate compose projects via the socket + `--providers.docker.network=lpg-edge` + a per-service `traefik.docker.network=lpg-edge` label. Verified with `docker compose config` on both files (flattened VPS-like layout).
- **CI build-only + manual deploy** (owner decision): the SSH `deploy` job was replaced by a `publish` job (build `docker/Dockerfile` → GHCR `:latest`+`:sha-<short>`). Deploys are by hand over SSH (`docker compose pull && up -d`); CI needs **no VPS secrets**, only `GITHUB_TOKEN`. (This amends the earlier criterion that CI rolls the VPS — CI now only publishes the image.)
- **Prod catalog seed:** `seed.ts` split so that under `NODE_ENV=production` it seeds **only** the real product catalog (tank types + items) — never the demo users/store/customers — run via a profile-gated one-shot `seed` service mirroring `migrate` (`docker compose run --rm seed`). 103 tests still green.
- **Two runbooks:** `docs/VPS-SETUP.md` (one-time provisioning incl. `docker network create lpg-edge`, bring up edge then app, seed, first-developer bootstrap) + `docs/DEPLOYMENT.md` (per-release manual SSH deploy + seed usage + rollback). Env split: `docker/.env.prod.example` (app) + `infra/traefik/.env.example` (just `ACME_EMAIL`), each auto-loaded from its own stack dir. README + vault `ops/deployment` + ADR-019 (amended) + the compose memory all updated.
- Gates green: typecheck + biome check + **103 tests** + build + `docker compose config` (app: db/migrate/app default, +seed under `--profile seed`; edge: valid).

[2026-06-24] [lpg-backend] **Polish (owner review).** (1) Renamed the backend compose service `app` → **`api`** (avoids clashing with the `app.<domain>` frontend host; reads naturally and sets up a future nginx `proxy_pass http://api:3000`) — updated compose labels/comments, both runbooks, the env template, vault `ops/deployment`, and ADR-019. (2) VPS-SETUP now uses the owner's user **`dmarquinah`** and opens with an optional **local-machine SSH-identity** step (dedicated `~/.ssh/lpg_vps` key + a `~/.ssh/config` `Host lpg-vps` alias with `IdentitiesOnly yes`) so this box's key never mixes with other VPS identities. (3) Clarified the network model in the compose comments: frontend↔backend would talk over the shared **`lpg-edge`** bridge (frontend reaches `api:3000` there), while **`db` stays isolated on `lpg-internal`** — the frontend is never on the db network. (4) Added **`docs/GITHUB-SETUP.md`** — the one-time GitHub/GHCR runbook (enable Actions, first publish creates the package, link-repo + visibility, optional read-only PAT for the VPS pull, no repo secrets needed). Recommended order documented across docs: GITHUB-SETUP → VPS-SETUP → DEPLOYMENT. `docker compose config` still validates (api/db/migrate default, +seed under the profile); no code change this round.

[2026-06-24] [lpg-backend] **VPS sync model → git clone (owner choice).** The VPS now holds a **git clone** at `/srv/lpg-backend` instead of hand-copied files: config changes apply with `git pull` (the two `.env` files live in the tree but are gitignored, so pulls never clobber them). Each stack runs from its own dir in the clone — `docker/` (backend) and `infra/traefik/` (edge). Pinned stable compose project names (`name: lpg-backend` / `name: lpg-traefik`) so volumes/networks don't get named after the run directory. VPS-SETUP rewritten (step 7 = clone + read-only **deploy key** for a private repo + GHCR login; step 8 = create the two `.env` from templates in-tree; bring-up paths now `…/docker` and `…/infra/traefik`). DEPLOYMENT rewritten (deploy from `…/docker`; **update config = `git pull` + re-up**; rollback/edge/troubleshooting paths). GITHUB-SETUP gained a deploy-key note. ADR-019 + vault `ops/deployment` + compose memory updated. `docker compose config` re-validated for both files.

**Cross-track note:** the vault shows the **frontend track shipped** and chose a **single-origin** layout — its nginx proxies `/api` → **`api:3000`** over `lpg-edge` (no CORS), "chosen over the two-subdomain layout." The backend's `app`→`api` rename lines up exactly with that proxy target. The backend is compatible as-is; its public `api.<domain>` Traefik route + `CORS_ORIGINS` are now **optional/unused** by the SPA (kept harmless) — a reconciliation decision flagged to the owner (keep `api.<domain>` for direct/bot access, or simplify to single-origin only).