---
project: lpg-store
domain: specs
type: spec
status: done
depends-on: []
last-updated: 2026-05-08
---

# Spec: Auth Foundation

## Problem Statement

V2 has no auth — no users, no login, no way to gate routes by role. Every other module (users, stores, inventory, customers, orders) presupposes "we know who's logged in and what they can do." Auth is the foundation that unblocks everything.

V1 had auth but with several over-builds we're not carrying forward: 4 user-type tables (admins / operators / delivery_personnel / superadmins), 4 login strategy classes that did the same thing modulo JWT payload, a module:action permission system with wildcards (`users:*`, `*:read`, `manage`), and a separate pre-registration token flow distinct from the user creation flow. We're consolidating to a smaller surface that fits a 1-store, ~5-user business.

## Proposed Solution

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

A user created via `POST /auth/register` starts with `password_hash` set to a placeholder/random value and `active = false`; they activate by completing the invite. Once activated, `user_invitations.used_at` is stamped.

### Roles

- **developer** — bypasses every role check. Used by the dev (you) and any other systems-level operator. The first user is created via the `POST /api/v1/auth/bootstrap` endpoint (gated by `BOOTSTRAP_TOKEN`); subsequent developers are minted by an existing developer via `POST /api/v1/auth/register`.
- **admin** — full business access: manage users (create/disable, but not change passwords for others), catalog, all inventory, all orders.
- **operator** — daily order entry, customer registry, inventory read/update, delivery dispatch. Cannot manage users or catalog.
- **delivery** — sees only their own assignments and orders. Updates their own delivery state. Resource scoping is enforced per-handler (not in middleware).

### Endpoints (all under `/api/v1/auth`)

| Method | Path                    | Auth         | Body / Notes |
|--------|-------------------------|--------------|--------------|
| POST   | `/bootstrap`            | secret token | `{ token, email, name, phone?, password }`. Creates the first (or another) developer user. Disabled when `BOOTSTRAP_TOKEN` env var is unset → returns 404. See [[#Bootstrapping (first user)]]. |
| POST   | `/login`                | public       | `{ email, password }` → `{ token, user: { id, email, name, role, active } }` |
| GET    | `/me`                   | authenticated | returns the current user (no `password_hash`) |
| POST   | `/change-password`      | authenticated | `{ currentPassword, newPassword }` |
| POST   | `/register`             | admin or developer | `{ email, name, phone?, role }` → `{ user, invitation: { token, expiresAt } }`. Frontend builds the invite URL. **Only `developer` callers may create another `developer`**; admins can create admin/operator/delivery only. |
| POST   | `/invite/:token`        | public       | `{ password }`. Validates token, hashes password, sets `users.active = true`, stamps `used_at`. |
| POST   | `/logout`               | authenticated | adds the token's `jti` to a Redis blocklist with TTL = remaining token lifetime |

No token refresh, no password reset endpoint, no email/SMS for invites. Admin generates an invite, copies the URL, hands it off out-of-band.

### JWT

- Algorithm: HS256, secret from `JWT_SECRET` env var (already validated, ≥32 chars)
- Access token TTL: 24h
- Payload: `{ sub: userId, email, role, jti, iat, exp }` — `jti` is a uuid for the blocklist
- Bearer token in `Authorization: Bearer <token>` header

### Middleware

- `requireAuth` — verifies signature + expiry, checks Redis blocklist for `jti`, populates `req.user = { id, email, role }`.
- `requireRole(...roles)` — must be used after `requireAuth`. `developer` always passes. `403` on mismatch.
- Resource scoping (e.g. delivery user can only read their own orders) is **per-handler**: handlers consult `req.user.id` and `req.user.role` and scope their queries.

### Bootstrapping (first user)

There's no automatic seed. The first (and any subsequent) `developer` user is created via `POST /api/v1/auth/bootstrap`.

The endpoint requires a shared secret in the request body that must match the `BOOTSTRAP_TOKEN` env var. If `BOOTSTRAP_TOKEN` is unset (or empty), the endpoint returns `404 Not Found` — the bootstrap path effectively doesn't exist. To re-enable, set the env var, restart, call the endpoint, then unset and restart again.

Request:

```
POST /api/v1/auth/bootstrap
{
  "token": "<value of BOOTSTRAP_TOKEN env var>",
  "email": "diego@example.com",
  "name": "Diego",
  "phone": "+51...",
  "password": "<min-12-chars>"
}
```

Response (`201 Created`): `{ user: { id, email, name, role: "developer", active: true } }`.

Failure modes:
- `404` — `BOOTSTRAP_TOKEN` is unset
- `401` — token mismatch
- `409` — email already in use
- `400` — body validation error

Operational flow on first deploy:

1. Set `BOOTSTRAP_TOKEN=<random-32-chars>` in the VPS `.env`.
2. Run `docker compose up -d` — migrations run on container start.
3. `curl -X POST .../api/v1/auth/bootstrap -d '{...}'` to create the first developer.
4. Edit `.env` to remove `BOOTSTRAP_TOKEN` (or comment it out). Restart the app: `docker compose up -d --force-recreate app`.
5. From now on, only existing developers can mint new developers via `POST /register` (and the bootstrap endpoint is `404`).

A future spec will add a separate seeder for non-user reference data (catalog, store row, etc.). The bootstrap endpoint is **only** for the first developer.

### Module structure

```
src/modules/auth/
  index.ts           # createAuthModule({ db, cache }) → { router, requireAuth, requireRole }
  routes.ts          # express router for the 6 endpoints
  service.ts         # AuthService — login, register, completeInvite, changePassword, logout
  repository.ts      # drizzle queries: users + delivery_profiles + user_invitations
  schema.ts          # drizzle table defs (re-exported from src/db/schema.ts)
  types.ts           # zod request/response + role enum
  middleware.ts      # requireAuth, requireRole (exported via index.ts)
  __tests__/
    login.test.ts
    register.test.ts
    invite.test.ts
    middleware.test.ts
    logout.test.ts
```

`requireAuth` and `requireRole` are exported from the module's `index.ts` so other modules import them: `import { requireAuth, requireRole } from "../auth";`.

## Acceptance Criteria

- [x] `users`, `delivery_profiles`, `user_invitations` tables created via a generated migration in `src/db/migrations/`.
- [x] `POST /api/v1/auth/bootstrap` creates a `developer` user with `active=true` when `BOOTSTRAP_TOKEN` is set and the request body matches; returns `404` when the env var is unset; returns `401` on token mismatch; returns `409` on duplicate email.
- [x] `POST /api/v1/auth/login` validates credentials (bcrypt compare), returns `{ token, user }`. Inactive users (`active=false`) cannot log in.
- [x] Failed login (unknown email or wrong password) returns `401` with a generic message; no info leak distinguishing the two.
- [x] `GET /api/v1/auth/me` requires `requireAuth` and returns the current user (no `password_hash`).
- [x] `POST /api/v1/auth/change-password` requires `requireAuth`; validates `currentPassword` against stored hash; updates hash on success.
- [x] `POST /api/v1/auth/register` requires `requireRole('admin', 'developer')`; rejects duplicate email; creates a user with `active=false`, generates a `user_invitations` row with 24-hour TTL, returns `{ user, invitation: { token, expiresAt } }`. Only callers with role `developer` may set the new user's role to `developer`; admins requesting `role: 'developer'` get `403`.
- [x] `POST /api/v1/auth/invite/:token` accepts a `password` (min 12 chars), validates the token (exists, not expired, not used), sets the user's password hash, marks user `active=true`, stamps `used_at`.
- [x] `POST /api/v1/auth/logout` adds the token's `jti` to a Redis blocklist with TTL matching the token's remaining lifetime. If Redis is unavailable, returns 503.
- [x] `requireAuth` middleware checks the blocklist and rejects blocklisted tokens with `401`.
- [x] `requireRole` middleware allows the listed roles plus `developer` (which always passes); returns `403` on mismatch.
- [x] Tests cover: successful login, wrong password, inactive user blocked, missing/invalid/expired token, blocklisted token, role mismatch, developer role-bypass, duplicate-email register, expired/used invite token, password change with wrong current password, admin-cannot-create-developer. (Bootstrap is verified by smoke test only — operationally simple and exercised on first deploy.)
- [x] All routes mounted at `/api/v1/auth` from `src/app.ts` via `createAuthModule({ db, cache })`.

## Out of Scope

- Password reset (admin re-invites the user via `/register` → new invite link)
- Token refresh (24h tokens + login again is sufficient for an internal tool)
- Email or SMS delivery of invite URLs (admin shares the URL manually)
- Resource-level authorization in middleware (per-handler)
- Rate limiting (defer until production traffic justifies it)
- 2FA / MFA
- OAuth / SSO

## Technical Notes

- **No strategy pattern.** V1 had 4 login strategies that did the same thing modulo JWT payload. We use a single `AuthService.login()` and shape the user response with a serializer helper.
- **Permissions.** V1's `module:action` permission strings with wildcards are dropped. Authorization is *role-based only* in middleware. If a route needs finer access (e.g. delivery user can only see their own orders), the route handler scopes the query using `req.user.id` and `req.user.role`.
- **Invite token vs JWT.** Two different tokens. The invite token is opaque (32-byte random hex), one-time-use, in `user_invitations`. The session token is a signed JWT.
- **Bootstrap kill switch.** The bootstrap endpoint's only gate is the `BOOTSTRAP_TOKEN` env var. When unset, the endpoint returns `404` (the route is registered but checks the env var first; absent var = absent endpoint). To re-enable, set the env var and restart the app. This makes the on/off switch explicit and operator-controlled — no automatic "first user already exists, lock it forever" magic that could lock you out.
- **Redis dependency.** Logout requires Redis; we accept this. In dev without `REDIS_URL`, `/logout` returns 503. Live deployments must have Redis.
- **bcrypt 12 rounds** vs v1's 10 — slightly slower hash, slightly more secure. Login still well under 100ms.
- **Module exports.** `createAuthModule({ db, cache })` returns `{ router, requireAuth, requireRole }`. The middleware exports allow other modules to compose without re-importing helpers.
- **Developer role escalation.** Only callers with role `developer` may issue an invite for `role: 'developer'`. Admins are limited to creating admin/operator/delivery. The `requireRole` check on the route allows admin+developer; the *handler* enforces the developer-only-creates-developer rule on the body's `role` field.
- **Out-of-band password sharing.** The `bootstrap` endpoint is the one place a password is set directly; for all other users, the password is only set by the user via `/invite/:token`. The bootstrap caller (the deployer) sets their own password; nobody else ever knows it.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

Files to create:

- `src/modules/auth/{index,routes,service,repository,schema,types,middleware}.ts`
- `src/modules/auth/__tests__/{bootstrap,login,register,invite,middleware,logout}.test.ts`
- `src/db/migrations/0000_<auto>.sql` (generated by `drizzle-kit generate`)

Files to modify:

- `src/db/schema.ts` — `export * from "../modules/auth/schema"`
- `src/app.ts` — mount `createAuthModule({ db, cache }).router` at `/api/v1/auth`
- `src/server.ts` — wire `cache` into module construction
- `src/config/env.ts` — add optional `BOOTSTRAP_TOKEN` (z.string().min(16).optional())
- `.env.example` — add `BOOTSTRAP_TOKEN` (commented out, with usage note)

V1 reference (read-only):

- `legacy/src/services/authService.ts` — login + delegation to strategies
- `legacy/src/repositories/authRepository.ts` — bcrypt + token + user creation logic (esp. lines 281–334 for the registration transaction)
- `legacy/src/middlewares/authorization.ts` — JWT verification + role/permission middleware
- `legacy/src/db/schemas/user-management/users.ts` — base users table
- `legacy/src/db/schemas/user-management/pre-registration.ts` — invite token shape
- `legacy/src/dtos/request/authDTO.ts` — request schemas (zod)
- `legacy/src/dtos/response/authInterface.ts` — response shapes
- `legacy/src/routes/authRoutes.ts` — endpoint wiring (we drop pre-registration as separate step)

## Open Questions

None remaining (all resolved during the 2026-05-07 review).

## Implementation Notes

<!-- Claude appends progress here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-05-07] [lpg-backend] **Phase 1 — schema + types + errors.** Created `src/modules/auth/schema.ts` (drizzle: `userRoleEnum`, `users`, `delivery_profiles`, `user_invitations`); `src/modules/auth/types.ts` (zod request schemas + `PublicUser` serializer + `AuthUser`/`JwtPayload` types). Added `GoneError` (410) + `ServiceUnavailableError` (503) to `src/lib/errors.ts`. Re-exported auth schema from `src/db/schema.ts`. Generated migration `src/db/migrations/0000_auth_foundation.sql` via `npm run db:generate -- --name auth_foundation`. Removed `*.sql` blanket pattern from `.gitignore` so migration files are tracked.
- [2026-05-07] [lpg-backend] **Phase 2 — repository.** `AuthRepository` class implements `IAuthRepository` interface (extracted so tests can substitute a fake without nominal-private friction). Methods: `findUserByEmail/Id`, `createUser` (transactional when role=delivery, also inserts a `delivery_profiles` row), `updatePasswordHash`, `setUserActive`, `createInvitation`, `findInvitationByToken`, `findActiveInvitationByToken`, `completeInvitation` (transaction: hash + active=true + stamp used_at).
- [2026-05-07] [lpg-backend] **Phase 3 — service.** `AuthService` with bootstrap/login/getMe/changePassword/register/completeInvite/logout/isJtiBlocked. JWT signing via `jsonwebtoken` HS256 + `crypto.randomUUID()` jti, 24h expiry. Bcrypt 12 rounds. Invite tokens = `crypto.randomBytes(32).toString("hex")` with 24h TTL. Bootstrap throws `NotFoundError` when env-token unset (route translates to 404). Logout requires Redis (`BlocklistCache` interface — minimal subset of ioredis: `setex` + `exists`); returns 503 when cache is null.
- [2026-05-07] [lpg-backend] **Phase 4 — middleware.** `createRequireAuth` verifies bearer JWT, checks blocklist, attaches `req.user` and `req.authPayload` (raw payload retained so `/logout` can read jti+exp without re-verifying). `createRequireRole` is a pure factory; allows the listed roles + always allows `developer`. Express.Request augmented via `declare global` for `user` + `authPayload`.
- [2026-05-07] [lpg-backend] **Phase 5 — routes.** Express router with all 7 endpoints (`/bootstrap`, `/login`, `/me`, `/change-password`, `/register`, `/invite/:token`, `/logout`). Each handler: zod parse → call service → respond. `/bootstrap` returns 404 when `bootstrapToken` is falsy. `/register` enforces developer-only-creates-developer in the handler (after the role check passes admin or developer).
- [2026-05-07] [lpg-backend] **Phase 6 — composition.** `createAuthModule({ db, cache, jwtSecret, bootstrapToken, repo? })` returns `{ router, service, requireAuth, requireRole }`. `repo` override exists so tests can substitute `FakeAuthRepository` without standing up Postgres.
- [2026-05-07] [lpg-backend] **Phase 7 — env + app wiring.** Added optional `BOOTSTRAP_TOKEN` (≥16 chars) to `src/config/env.ts`. `createApp` now takes `{ db, cache, jwtSecret, bootstrapToken }`; mounts auth router at `/api/v1/auth`. `server.ts` wires `createCacheClient(env.REDIS_URL)` and quits cache on SIGTERM/SIGINT. Updated existing `health.test.ts` for the new `AppDeps` shape. Removed test exclusions from `tsconfig.json` so tests are typechecked too (build config still excludes them so `dist/` stays clean). Added commented `BOOTSTRAP_TOKEN` to `.env.example` with usage note.
- [2026-05-07] [lpg-backend] **Phase 8 — tests.** 30 tests across 5 suites under `src/modules/auth/__tests__/`: `login` (5), `register` (7), `invite` (5), `middleware` (8 — combines requireAuth + requireRole + change-password), `logout` (2). Each test spins up a fresh express app with `FakeAuthRepository` + `FakeCache` (in-memory; no Postgres or Redis needed). The bootstrap-endpoint tests were intentionally omitted (operationally simple; covered by smoke test). All 30 pass; existing /health smoke test (2) still passes; full suite green at 32 tests / 9 suites.
- [2026-05-07] [lpg-backend] **Quality gates.** `npm run typecheck` clean, `npm test` 32/32 green, `npm run build` produces `dist/` (auth module compiles to 7 JS files, no test files included).
