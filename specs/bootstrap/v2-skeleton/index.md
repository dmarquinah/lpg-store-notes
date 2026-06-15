---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on: []
last-updated: 2026-05-08
---

# Spec: v2 Skeleton (reset both repos with v1 as reference)

## Problem Statement

Both the v1 backend (~22.5k LOC, custom DI, 5-strategy transaction pattern, 998-LOC workflow repository) and the v1 frontend (~32.7k LOC across 212 files for ~25 screens, role-as-folder duplication, i18n + FCM stacks for unused features) became hard to maintain. The owner reported being unable to track the impact of changes across modules. The product is pre-production, so there are no users to migrate — the only cost is lost requirements knowledge if v1 is deleted outright.

Both repos need the same treatment: archive v1 as a read-only reference, stand up a clean v2 skeleton with no business features, and port features one at a time in later specs.

## Proposed Solution

For **each** repo, mirror the same reset playbook:

1. Archive the current source tree to `legacy/` via `git mv` (history-preserving, fully grep-able, not buildable from the v2 entry point).
2. Bootstrap a fresh `src/` skeleton with **no business features** — framework + build wiring + a single thin vertical slice (health endpoint for backend; auth login shell for frontend).
3. Adopt vertical **module-by-domain** structure (`src/modules/<feature>/`) instead of layer- or role-by-folder.
4. Drop deferred dependencies (backend: Supabase/Fly deploy; frontend: i18n, Firebase FCM, datepicker, dual transaction API).
5. Delete `legacy/` only once v2 reaches functional parity.

The backend skeleton lands first (it owns the API the frontend hits); the frontend skeleton follows once backend `auth-foundation` is live.

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [x] v1 archived under `lpg-backend/legacy/` via `git mv` (history preserved per file)
- [x] `fly.toml`, `.env.supabase.example`, and the Supabase deploy script removed
- [x] Repo root has no v1 source files outside `legacy/`
- [x] v2 skeleton at `src/` with `app.ts`, `server.ts`, `config/env.ts`, `db/{client,schema,migrate}`, `middleware/{errorHandler,requestLogger,notFound}`, `lib/{logger,errors,cache}`, empty `modules/`
- [x] `/health` endpoint returns 200 with `{ status, db }`; smoke test passes via Node's built-in test runner
- [x] `tsconfig.json` excludes `legacy/`; `tsc --noEmit` typechecks zero legacy files
- [x] Unified `docker-compose.yml` for dev (db+redis only) and prod (full stack with image from GHCR)
- [x] Multi-stage `Dockerfile` produces ~175MB distroless final image (runs as `nonroot:nonroot`, no shell)
- [x] `migrate` is a one-shot compose service that exits before `app` starts
- [x] `.github/workflows/main.yml` runs `ci` (typecheck/build/test) → `deploy` (build → GHCR push → SSH → docker compose up → health check)
- [x] `docs/DEPLOYMENT.md` has the VPS provisioning runbook
- [x] Obsidian vault initialized for `lpg-store` covering all three repos

**Frontend (lpg-frontend-vue):**

- [x] v1 archived under `lpg-frontend-vue/legacy/` via `git mv` (history preserved per file); `legacy/README.md` explains the archive + deletion criteria
- [x] `tsconfig` excludes `legacy/`; `vue-tsc -b --noEmit` typechecks zero legacy files; repo root has no v1 source outside `legacy/`
- [x] Fresh `package.json` removes `vue-i18n`, `firebase`, `@vuepic/vue-datepicker`; keeps the core Vue/Vite/Pinia/Tailwind/PWA stack
- [x] `src/` skeleton landed: `main.ts`, `App.vue`, `router/` (single global guard + role redirects, no i18n loader), `layouts/AppLayout.vue` (one shell taking nav config as a prop), `lib/{apiClient,errors,types}.ts`, `modules/auth/` vertical slice, empty `modules/` peers
- [x] `vite.config.ts` keeps `vite-plugin-pwa`, drops the custom service-worker generator, parameterizes `manifest.start_url` via env var (default `/login`)
- [x] `npm run dev` boots, login screen renders, login flow against backend `/v1/auth/login` succeeds, role redirect lands on a stub home view
- [x] `npm run build` produces a working PWA bundle (38 KB gzipped main); `vue-tsc -b --noEmit` passes clean
- [x] [[../../../eng/architecture]] Frontend section updated; [[../../../eng/patterns/frontend-module-template]] added

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- Porting any business feature (auth, users, orders, inventory, customers, products, stores) — each becomes its own folder spec
- Deleting `legacy/` — only after v2 reaches parity
- Test coverage beyond the backend `/health` smoke test and a frontend manual login smoke test
- Reintroducing FCM, i18n, datepicker, or the legacy transaction API — each requires a fresh case if it returns

## Open Questions

None remaining.
