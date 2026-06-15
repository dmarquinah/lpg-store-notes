---
project: lpg-store
domain: specs
type: spec-track
spec: auth-foundation
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-04
---

# Auth Foundation — lpg-backend track

Shared spec: [[index]]

## Technical Notes

### Data model

Single `users` table for all roles. Delivery has a 1:1 sub-table for the only role-specific fields v1 actually used.

```
users
  id              serial PK
  email           varchar 255 unique not null
  password_hash   varchar 255 not null    -- bcrypt, 12 salt rounds
  name            varchar 255 not null
  phone           varchar 32  null
  role            user_role enum not null  -- developer | admin | operator | delivery
  active          boolean not null default true
  created_at      timestamptz not null default now()
  updated_at      timestamptz not null default now()

delivery_profiles  (1:1 with users where role = 'delivery')
  user_id              integer PK references users(id) on delete cascade
  license_number       varchar 64  null
  vehicle_type_pref    varchar 32  null
  total_deliveries     integer not null default 0
  average_rating       numeric(3,2) null

user_invitations
  id                serial PK
  user_id           integer references users(id) on delete cascade
  token             varchar 128 unique not null   -- 32-byte random hex
  expires_at        timestamptz not null
  used_at           timestamptz null
  created_by        integer references users(id)  -- which admin issued it
  created_at        timestamptz not null default now()
```

A user created via `POST /auth/register` starts with a placeholder `password_hash` and `active = false`; they activate by completing the invite, which stamps `user_invitations.used_at`.

### Roles

- **developer** — bypasses every role check. First one created via `POST /api/v1/auth/bootstrap` (gated by `BOOTSTRAP_TOKEN`); subsequent developers minted by an existing developer via `/register`.
- **admin** — full business access: manage users (create/disable, not change others' passwords), catalog, all inventory, all orders.
- **operator** — daily order entry, customer registry, inventory read/update, dispatch. No user or catalog management.
- **delivery** — sees only their own assignments/orders; updates their own delivery state. Resource scoping per-handler.

### Endpoints (all under `/api/v1/auth`)

| Method | Path | Auth | Body / Notes |
|--------|------|------|--------------|
| POST | `/bootstrap` | secret token | `{ token, email, name, phone?, password }`. Creates a developer. `404` when `BOOTSTRAP_TOKEN` unset. |
| POST | `/login` | public | `{ email, password }` → `{ token, user }` |
| GET | `/me` | authenticated | current user (no `password_hash`) |
| POST | `/change-password` | authenticated | `{ currentPassword, newPassword }` |
| POST | `/register` | admin or developer | `{ email, name, phone?, role }` → `{ user, invitation }`. Only `developer` may create `developer`. |
| POST | `/invite/:token` | public | `{ password }`. Validates token, hashes password, sets `active=true`, stamps `used_at`. |
| POST | `/logout` | authenticated | adds the token's `jti` to a Redis blocklist (TTL = remaining lifetime) |

No token refresh, no password reset, no email/SMS for invites.

### JWT

HS256, secret from `JWT_SECRET` (≥32 chars). Access TTL 24h. Payload `{ sub, email, role, jti, iat, exp }` (`jti` is a uuid for the blocklist). Bearer header.

### Middleware

- `requireAuth` — verifies signature + expiry, checks the Redis blocklist for `jti`, populates `req.user`.
- `requireRole(...roles)` — used after `requireAuth`. `developer` always passes; `403` on mismatch.
- Resource scoping is per-handler (handlers consult `req.user.id` / `req.user.role`).

### Bootstrapping (first user)

No automatic seed. The first (and any subsequent) `developer` is created via `POST /api/v1/auth/bootstrap`, gated by a shared secret matching `BOOTSTRAP_TOKEN`. If the env var is unset/empty the endpoint returns `404` — the bootstrap path effectively doesn't exist. To re-enable: set the env var, restart, call, unset, restart again. Operational flow on first deploy: set `BOOTSTRAP_TOKEN`, `docker compose up -d` (migrations run on start), `curl` the endpoint, then remove the var and `--force-recreate app`.

### Design decisions

- **No strategy pattern** — single `AuthService.login()` + a serializer helper, vs v1's 4 strategies.
- **Role-based authorization only** — v1's `module:action` permission strings dropped; finer access is per-handler scoping.
- **Invite token vs JWT** — two different tokens: opaque one-time hex in `user_invitations` vs signed session JWT.
- **bcrypt 12 rounds** (v1 used 10); login still < 100ms.
- **Redis dependency** — logout requires Redis; `/logout` returns 503 in dev without `REDIS_URL`.
- **Developer escalation** — `requireRole` allows admin+developer on `/register`; the handler enforces developer-only-creates-developer on the body's `role`.

### Module structure

```
src/modules/auth/
  index.ts        # createAuthModule({ db, cache }) → { router, requireAuth, requireRole }
  routes.ts       # express router for the 7 endpoints
  service.ts      # AuthService
  repository.ts   # drizzle queries: users + delivery_profiles + user_invitations
  schema.ts       # drizzle table defs (re-exported from src/db/schema.ts)
  types.ts        # zod request/response + role enum
  middleware.ts   # requireAuth, requireRole
  __tests__/
```

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

Created:

- `src/modules/auth/{index,routes,service,repository,schema,types,middleware}.ts`
- `src/modules/auth/__tests__/{login,register,invite,middleware,logout}.test.ts`
- `src/db/migrations/0000_auth_foundation.sql`
- `src/db/seed.ts` — dev user seeder (added 2026-06-04)

Modified:

- `src/db/schema.ts` — `export * from "../modules/auth/schema"`
- `src/app.ts` — mount `createAuthModule({ db, cache }).router` at `/api/v1/auth`
- `src/server.ts` — wire `cache` into module construction
- `src/config/env.ts` — optional `BOOTSTRAP_TOKEN`, `SEED_PASSWORD`
- `.env.example` — `BOOTSTRAP_TOKEN` + `SEED_PASSWORD` (commented)

V1 reference (read-only): `legacy/src/services/authService.ts`, `legacy/src/repositories/authRepository.ts` (esp. 281–334), `legacy/src/middlewares/authorization.ts`, `legacy/src/db/schemas/user-management/{users,pre-registration}.ts`, `legacy/src/dtos/{request/authDTO,response/authInterface}.ts`, `legacy/src/routes/authRoutes.ts`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-05-07] [lpg-backend] **Phase 1 — schema + types + errors.** `schema.ts` (`userRoleEnum`, `users`, `delivery_profiles`, `user_invitations`); `types.ts` (zod requests + `PublicUser` serializer + `AuthUser`/`JwtPayload`). Added `GoneError` (410) + `ServiceUnavailableError` (503) to `src/lib/errors.ts`. Generated migration `0000_auth_foundation.sql`. Removed `*.sql` from `.gitignore`.
- [2026-05-07] [lpg-backend] **Phase 2 — repository.** `AuthRepository` implements `IAuthRepository` (extracted so tests substitute a fake). Methods for find/create users (transactional for delivery), password/active updates, invitations.
- [2026-05-07] [lpg-backend] **Phase 3 — service.** `AuthService` with bootstrap/login/getMe/changePassword/register/completeInvite/logout/isJtiBlocked. JWT HS256 + `randomUUID()` jti, 24h. Bcrypt 12. Invite tokens = `randomBytes(32).hex` 24h TTL. Logout requires Redis; 503 when cache null.
- [2026-05-07] [lpg-backend] **Phase 4 — middleware.** `createRequireAuth` verifies bearer JWT, checks blocklist, attaches `req.user` + `req.authPayload`. `createRequireRole` allows listed roles + always `developer`.
- [2026-05-07] [lpg-backend] **Phase 5 — routes.** Router with 7 endpoints; `/bootstrap` 404 when token falsy; `/register` enforces developer-only-creates-developer in the handler.
- [2026-05-07] [lpg-backend] **Phase 6 — composition.** `createAuthModule({ db, cache, jwtSecret, bootstrapToken, repo? })` → `{ router, service, requireAuth, requireRole }`.
- [2026-05-07] [lpg-backend] **Phase 7 — env + app wiring.** Optional `BOOTSTRAP_TOKEN` (≥16) in env. `createApp` takes `{ db, cache, jwtSecret, bootstrapToken }`; mounts at `/api/v1/auth`. `server.ts` wires `createCacheClient(env.REDIS_URL)`.
- [2026-05-07] [lpg-backend] **Phase 8 — tests.** 30 tests across 5 suites with `FakeAuthRepository` + `FakeCache` (no Postgres/Redis). Full suite green at 32/9.
- [2026-05-07] [lpg-backend] **Quality gates.** typecheck clean, 32/32 green, build produces `dist/` (no test files).
- [2026-06-04] [lpg-backend] **Dev user seeder** (`npm run db:seed` → `src/db/seed.ts`). Creates `operator@lpg.local` + `delivery@lpg.local`; goes through the same creation path via a shared `AuthService.createActiveUser` + idempotent `AuthService.seedUser` (skips existing emails). Password from `SEED_PASSWORD` (min 6, dev default); refuses to run when `NODE_ENV=production`. +1 lifecycle test; full suite 11/11 green.
- [2026-06-04] [lpg-backend] All backend acceptance criteria for this repo met.
