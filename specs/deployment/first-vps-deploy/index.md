---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../bootstrap/v2-skeleton/index]]"
  - "[[../../auth/auth-foundation/index]]"
last-updated: 2026-06-24
---

# Spec: First Production VPS Deploy (provisioning runbook + Traefik edge proxy + TLS)

## Problem Statement

The product is about to go live on a single self-hosted VPS, but the path from a
**fresh box** to a **running, HTTPS-reachable system** has gaps:

1. **No from-scratch provisioning guide.** `lpg-backend/docs/DEPLOYMENT.md` assumes
   the VPS is already a usable host — Docker installed, a deploy user present, a
   directory writable. It never covers the OS-level first steps: creating the
   owner's own login user, locking down SSH, a firewall, pointing DNS. The owner
   needs a copy-pasteable "I just rented a VPS, now what?" runbook.
2. **No reverse proxy and no TLS.** Compose publishes the backend on port `3000`
   directly. There is no edge proxy, no certificate management, and the **frontend
   (Piloto) isn't deployed at all** — it lives only as a dev build. There is no
   single front door serving both apps over HTTPS on real domains.
3. **No documented way to create the first user.** v2 gates the initial
   developer-role account behind a `BOOTSTRAP_TOKEN` + `POST /auth/bootstrap`, but
   that flow is undocumented in the deploy runbook. On a brand-new database there
   is no admin, so nobody can sign in to create anyone — a chicken-and-egg the
   runbook must resolve explicitly.

The owner wants one cohesive first-deploy spec that closes all three: a fresh-box
runbook (including creating his own developer account), plus a Traefik edge proxy
with automatic Let's Encrypt TLS fronting both containerized apps.

## Proposed Solution

A single first-deploy effort with two repo tracks plus a documentation deliverable.

**Topology (owner decision 2026-06-18) — edge proxy → two independent services:**

```
        Internet (443/80)
              │
        ┌─────▼─────┐   automatic Let's Encrypt TLS, HTTP→HTTPS redirect
        │  Traefik  │   (edge reverse proxy, label/Host routing)
        └──┬─────┬──┘
   api.<domain> │ │ app.<domain>
        ┌───────▼ ▼────────┐
   ┌────▼────┐        ┌─────▼──────┐
   │ backend │        │  frontend  │   each independently configurable
   │  (Node  │        │ (nginx     │   — the pattern the owner has used
   │  image, │        │  serving   │     before: per-service container,
   │  :3000) │        │  the build)│     own config, own image
   └────┬────┘        └────────────┘
        │
   db + redis (internal network only, not published)
```

- **Traefik** is the edge: it terminates TLS (automatic Let's Encrypt issuance +
  renewal via the ACME resolver), redirects HTTP→HTTPS, and routes by host —
  `api.<domain>` → backend, `app.<domain>` (or the apex) → frontend. Chosen over a
  hand-written nginx edge for auto-TLS and label-based config (owner preference).
- **Backend** stays the existing Node/distroless image, no longer publishing
  `3000` to the host — only reachable through Traefik on the internal docker
  network. `db`/`redis` stop publishing ports entirely in production.
- **Frontend** is built and served by **its own nginx** container (multi-stage:
  `npm run build` → static `dist/` served by nginx with SPA history fallback,
  gzip, and cache headers). It is configured independently of the edge, matching
  the owner's preferred "nginx-in-front-of-the-build behind a top proxy" pattern.

**First-user bootstrap (owner ask 2026-06-18).** The runbook documents creating
the owner's own **developer-role** account on a fresh DB via the existing
`BOOTSTRAP_TOKEN` + `POST /auth/bootstrap` flow: set a one-time `BOOTSTRAP_TOKEN`
in the production env, call the bootstrap endpoint once (through Traefik over
HTTPS) to mint the developer account, then **remove/rotate the token** so it can't
be reused. From that account the owner creates every other user in-app — no other
account is ever created out-of-band.

**Provisioning runbook (the documentation deliverable).** Extends
`lpg-backend/docs/DEPLOYMENT.md` (and its vault index [[../../ops/deployment]])
with a top-to-bottom **first-time** section: create a non-root sudo login user for
the owner; SSH hardening (key-only, disable root login); a firewall (allow 22/80/
443 only); install Docker + compose; DNS A-records for `api`/`app`; obtain the
production env + secrets; bring the stack up; bootstrap the first developer user;
verify HTTPS end-to-end. Routine deploys, rollback, and troubleshooting stay as
they are.

Backend-track detail (compose orchestration, Traefik service + labels, env, the
runbook) in [[backend]]; the frontend production image, nginx config, build-time
API base URL, and CI image push in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Provisioning runbook (documentation):**

- [ ] A fresh-VPS, top-to-bottom first-time runbook exists (extending
      `lpg-backend/docs/DEPLOYMENT.md`) covering, in order: create the owner's
      non-root **sudo login user**; SSH hardening (key-only auth, root login
      disabled); host firewall allowing only SSH + 80 + 443; Docker + compose
      install; DNS A-records for the `api`/`app` hostnames; placing the production
      `.env` + secrets (mode 600); bringing the stack up; and an end-to-end HTTPS
      verification step. Every step is copy-pasteable.
- [ ] The runbook documents **creating the first developer-role user** on a fresh
      database via `BOOTSTRAP_TOKEN` + `POST /auth/bootstrap` (set token → call
      once over HTTPS → **remove/rotate the token**), and states plainly that the
      owner then creates all other users in-app from that account.
- [ ] The vault [[../../ops/deployment]] index is updated to mention the edge
      proxy, TLS, and the frontend service (currently it says frontend deploy is
      "not yet documented").

**Edge proxy + TLS (backend track / infra):**

- [ ] A **Traefik** edge service fronts the stack: terminates TLS with
      **automatic Let's Encrypt** issuance + renewal, redirects HTTP→HTTPS, and
      routes by host — `api.<domain>` → backend, the frontend host → frontend.
- [ ] The **backend** no longer publishes `3000` to the host; it is reachable only
      via Traefik on the internal docker network. `db` and `redis` publish **no**
      host ports in the production configuration.
- [ ] Production secrets/domains are env-driven (no hardcoded hostnames or emails
      in committed files); the existing CI → GHCR → VPS deploy still rolls the
      backend image and stays green (health check passes **through** Traefik over
      HTTPS).
- [ ] The local-dev workflow is **not broken** by the proxy: daily backend dev
      (`docker compose up -d db redis` + `npm run dev`) still works without
      Traefik/the frontend container. (Resolve via compose profiles / a separate
      prod overlay — see Open Questions.)

**Frontend service (frontend track):**

- [ ] `lpg-frontend-vue` has a production **Dockerfile** (multi-stage: build →
      nginx serving `dist/`) and an nginx config with **SPA history fallback**,
      gzip, and sane static cache headers; the image is built + pushed (CI → GHCR,
      mirroring the backend) and consumed by the production compose.
- [ ] The frontend's **API base URL** and `VITE_VAPID_PUBLIC_KEY` are build/deploy
      configurable so the built app calls `https://api.<domain>` (not a localhost
      dev default), and Web Push keeps working behind the proxy.
- [ ] Routed behind Traefik on its host, the deployed SPA loads over HTTPS, deep
      links resolve (history fallback), and it successfully reaches the backend API
      across origins (CORS reviewed for the two-subdomain setup).

**Cross-cutting:**

- [ ] One documented command sequence brings the **whole** stack up on the VPS
      (Traefik + backend + frontend + db + redis), and a fresh visitor reaches both
      apps over HTTPS with valid certificates.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **The lpg-bot deployment** — the third repo isn't part of this first deploy;
  it can join the same Traefik later.
- **Multi-VPS / horizontal scaling / orchestration (k8s, swarm)** — single box,
  docker compose, per the project's deliberate simplicity.
- **CDN / external static hosting for the frontend** — the owner chose a
  self-hosted nginx container behind Traefik on the same VPS (not Cloudflare
  Pages/Vercel).
- **Automated backups / monitoring / log aggregation** — valuable, but a separate
  ops spec; this one gets the system live over HTTPS.
- **Switching the edge to hand-written nginx** — explicitly rejected in favour of
  Traefik (auto-TLS + label routing).
- **Zero-downtime / blue-green deploys** — the existing `compose up -d` roll is
  acceptable for MVP.

## Open Questions

- **Routing scheme:** two subdomains (`api.<domain>` + `app.<domain>`) vs. apex
  for the app + `api.` for the backend vs. a single host with a `/api` path
  prefix. Subdomains are assumed above (cleanest with Traefik Host rules and avoids
  SPA base-path issues); confirm the actual domain(s) at implementation time.
- **Dev/prod compose split:** keep the single `docker-compose.yml` and gate
  Traefik + frontend behind a compose **profile** (e.g. `--profile prod`), or
  introduce a `docker-compose.prod.yml` overlay? The single-file preference says
  profiles; verify it cleanly keeps Traefik/frontend out of daily backend dev.
- **Where does the production compose live** now that it orchestrates both repos'
  images? Stays in `lpg-backend` (current deploy home) referencing the frontend
  image from GHCR, or moves to a small infra location? Assumed: stays in
  lpg-backend.
- **CORS vs same-site:** two subdomains means cross-origin API calls (CORS +
  credentials). Confirm the backend CORS config and cookie/token strategy work
  across `app.` → `api.`, or reconsider a single-host `/api` path to stay
  same-origin.
- **Frontend CI:** does lpg-frontend-vue already have a CI workflow to extend, or
  is the GHCR image-build pipeline net-new for that repo?
- **Traefik dashboard:** expose it (secured) or keep it off in production?
