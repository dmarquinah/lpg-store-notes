---
project: lpg-store
domain: specs
type: spec-track
spec: v2-skeleton
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-05-07
---

# v2 Skeleton — lpg-backend track

Shared spec: [[index]]

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

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-05-07] [lpg-backend] Phase A: archived v1 to `legacy/` via `git mv` (single commit, 311 file renames at 100% similarity). Removed `fly.toml`, `.env.supabase.example`, `legacy/scripts/deploy-supabase.sh`. Wrote `legacy/README.md` explaining the archive's purpose.
- [2026-05-07] [lpg-backend] Phase B: fresh `package.json` (Express 5, Drizzle 0.45, Zod 4, Pino 10, postgres-js, ioredis), `tsconfig.json` (strict, ES2022, CJS, excludes `legacy/`), `.gitignore` cleaned up, `.dockerignore` added. Switched test runner from Jest to Node's built-in (`node --test`) per user request.
- [2026-05-07] [lpg-backend] Phase C: src/ skeleton landed. Refactored `loadEnv()` to throw instead of `process.exit(1)`, and made `logger.ts` read env vars directly so module-load doesn't fail in tests. `npm test` green (2/2).
- [2026-05-07] [lpg-backend] Phase D: dropped `@/` path aliases (caused MODULE_NOT_FOUND inside the distroless image since tsc doesn't rewrite them), replaced with relative imports. Single `docker-compose.yml` for dev and prod (services use `image: ${IMAGE:-lpg-backend:local}` + `build: .`). Distroless final stage = 175MB, runs as `nonroot:nonroot`. Compose owns ordering and healthchecks; no entrypoint script.
- [2026-05-07] [lpg-backend] Phase E: replaced v1's `ci.yml` + `release.yml` with a single `.github/workflows/main.yml`. Two jobs: `ci` (typecheck/build/test) → `deploy` (GHCR push, SSH, compose pull/up, /health poll). Wrote `docs/DEPLOYMENT.md`.
- [2026-05-07] [lpg-backend] Phase F: vault initialized for `lpg-store` (covers backend + frontend-vue + bot). Seeded `eng/{architecture,decisions,patterns/module-template}`, `product/overview`, `ops/{development,deployment}`, `specs/index` with the porting roadmap.
- [2026-05-07] [lpg-backend] All backend acceptance criteria for this repo met.
