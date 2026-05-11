---
project: lpg-store
domain: specs
type: spec
status: done
depends-on: []
last-updated: 2026-05-07
---

# Spec: Backend v2 Skeleton (reset with v1 as reference)

## Problem Statement

The v1 backend (~22.5k LOC, 7 modules, custom DI, 5-strategy transaction pattern, 998-LOC workflow repository) became hard to maintain. The owner reported being unable to track the impact of changes across modules. The product is pre-production, so there are no users to migrate — the cost is in lost requirements knowledge if v1 is deleted outright.

## Proposed Solution

Build a clean v2 at the repo root with vertical modules. Archive v1 under `legacy/` as a read-only reference (not buildable, fully grep-able). Bootstrap a skeleton — framework + db + migrations + Docker + CI/CD + health endpoint — with **no business features**. Features port one at a time in later specs. Delete `legacy/` once v2 reaches functional parity.

## Acceptance Criteria

- [x] v1 archived under `lpg-backend/legacy/` via `git mv` (history preserved per file)
- [x] `fly.toml`, `.env.supabase.example`, and Supabase deploy script removed
- [x] Repo root has no v1 source files
- [x] v2 skeleton at `lpg-backend/src/` with `app.ts`, `server.ts`, `config/env.ts`, `db/{client,schema,migrate}`, `middleware/{errorHandler,requestLogger,notFound}`, `lib/{logger,errors,cache}`, empty `modules/`
- [x] `/health` endpoint returns 200 with `{ status, db }`; smoke test passes via Node's built-in test runner
- [x] `tsconfig.json` excludes `legacy/`; `tsc --noEmit` typechecks zero legacy files
- [x] Unified `docker-compose.yml` for dev (db+redis only) and prod (full stack with image from GHCR)
- [x] Multi-stage `Dockerfile` produces ~175MB distroless final image (runs as `nonroot:nonroot`, no shell)
- [x] `migrate` is a one-shot compose service that exits before `app` starts (`depends_on: condition: service_completed_successfully`)
- [x] `.github/workflows/main.yml` runs `ci` (typecheck/build/test) → `deploy` (build → GHCR push → SSH → docker compose up → health check)
- [x] `docs/DEPLOYMENT.md` has VPS provisioning runbook
- [x] Obsidian vault initialized for `lpg-store` covering all three repos

## Out of Scope

- Porting any business feature (auth, users, orders, inventory, customers, products, stores) — each becomes its own spec
- Frontend changes
- Deleting `legacy/` — only after v2 reaches parity
- Test coverage beyond the `/health` smoke test

## Technical Notes

- TypeScript path aliases (`@/...`) were intentionally **not** used. `tsc` doesn't rewrite them at compile time and the runtime resolver doesn't know about them, so the distroless image was failing to require modules. Relative imports only.
- The `:nonroot` distroless tag presets `USER nonroot:nonroot` (uid/gid 65532) and `ENTRYPOINT ["/nodejs/bin/node"]`. `CMD` is just the script.
- Compose `condition: service_completed_successfully` is the modern replacement for entrypoint scripts that wait-for-db then run migrations. No `pg_isready` inside the runtime image.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

- `src/app.ts` — express factory, mounts `/health`
- `src/server.ts` — http server boot
- `src/config/env.ts` — zod-validated env
- `src/db/{client,schema,migrate}.ts` — drizzle wiring
- `src/middleware/{errorHandler,requestLogger,notFound}.ts`
- `src/lib/{logger,errors,cache}.ts`
- `src/__tests__/health.test.ts` — smoke test
- `Dockerfile`, `docker-compose.yml`, `.env.example`, `drizzle.config.ts`
- `.github/workflows/main.yml`
- `docs/DEPLOYMENT.md`
- `legacy/` — v1 archive
- `legacy/README.md` — what `legacy/` is, when it gets deleted

## Implementation Notes

- [2026-05-07] [lpg-backend] Phase A: archived v1 to `legacy/` via `git mv` (single commit, 311 file renames at 100% similarity). Removed `fly.toml`, `.env.supabase.example`, `legacy/scripts/deploy-supabase.sh`. Wrote `legacy/README.md` explaining the archive's purpose.
- [2026-05-07] [lpg-backend] Phase B: fresh `package.json` (Express 5, Drizzle 0.45, Zod 4, Pino 10, postgres-js, ioredis), `tsconfig.json` (strict, ES2022, CJS, excludes `legacy/`), `.gitignore` cleaned up, `.dockerignore` added. Switched test runner from Jest to Node's built-in (`node --test`) per user request.
- [2026-05-07] [lpg-backend] Phase C: src/ skeleton landed. Refactored `loadEnv()` to throw instead of `process.exit(1)`, and made `logger.ts` read env vars directly so module-load doesn't fail in tests. `npm test` green (2/2).
- [2026-05-07] [lpg-backend] Phase D: dropped `@/` path aliases (caused MODULE_NOT_FOUND inside the distroless image since tsc doesn't rewrite them), replaced with relative imports. Single `docker-compose.yml` for dev and prod (services use `image: ${IMAGE:-lpg-backend:local}` + `build: .`). Distroless final stage = 175MB, runs as `nonroot:nonroot`. Compose owns ordering and healthchecks; no entrypoint script.
- [2026-05-07] [lpg-backend] Phase E: replaced v1's `ci.yml` + `release.yml` with a single `.github/workflows/main.yml`. Two jobs: `ci` (typecheck/build/test) → `deploy` (GHCR push, SSH, compose pull/up, /health poll). Wrote `docs/DEPLOYMENT.md`.
- [2026-05-07] [lpg-backend] Phase F: vault initialized for `lpg-store` (covers backend + frontend-vue + bot). Seeded `eng/{architecture,decisions,patterns/module-template}`, `product/overview`, `ops/{development,deployment}`, `specs/index` with the porting roadmap.
