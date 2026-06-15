---
project: lpg-store
domain: specs
type: spec-track
spec: users-crud
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-04
---

# Users CRUD — lpg-backend track

Shared spec: [[index]]

## Technical Notes

A new vertical module `src/modules/users/` that operates on the **existing** `users` table (owned by auth — no new tables, no migration). It reuses auth's `requireAuth` / `requireRole` middleware, the `PublicUser` serializer, and `userRoleSchema`.

### Endpoints (all under `/api/v1/users`; all require `admin` or `developer`)

| Method | Path | Body / Query | Notes |
|--------|------|--------------|-------|
| GET | `/api/v1/users` | `?role=&active=` (optional) | List; never returns `password_hash`. |
| GET | `/api/v1/users/:id` | — | One user; `404` if absent. |
| PATCH | `/api/v1/users/:id` | `{ name?, phone?, role?, active? }` | Partial update; ≥1 field required. |

`email` is not editable (login identity). Password is out of scope — auth owns it.

### Authorization & guards

Route gate: `requireAuth` + `requireRole('admin')` (middleware always lets `developer` through). Per-field rules in `UsersService.updateUser`, mirroring auth's developer-escalation rule:

- Target not found → `404`.
- Empty patch → `400`.
- Setting `role` to `developer`, or modifying a user whose current role is `developer`: requires caller role `developer`, else `403`.
- A caller may not change **their own** `role` or `active` (self-lockout guard) → `403`. Editing one's own `name`/`phone` is allowed.

### Module structure

```
src/modules/users/
  index.ts        # createUsersModule({ db, requireAuth, requireRole, repo? }) → { router }
  routes.ts       # GET /, GET /:id, PATCH /:id
  service.ts      # UsersService — listUsers, getUser, updateUser (guards above)
  repository.ts   # IUsersRepository + UsersRepository (queries the shared users table)
  types.ts        # zod listQuerySchema + updateUserSchema; reuses PublicUser from ../auth/types
  __tests__/
```

No `schema.ts` — `repository.ts` imports the `users` table and `User` / `UserRole` types from `../auth/schema` (single source of truth). `createUsersModule` takes `requireAuth` / `requireRole` as deps so `src/app.ts` passes the auth module's instances.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

Created:

- `src/modules/users/{index,routes,service,repository,types}.ts`
- `src/modules/users/__tests__/{helpers,users.test}.ts`

Modified:

- `src/app.ts` — mount `createUsersModule({ db, requireAuth, requireRole }).router` at `/api/v1/users`

Reused (read-only): `src/modules/auth/schema.ts` (`users`, `User`, `UserRole`), `src/modules/auth/types.ts` (`PublicUser`, `toPublicUser`, `userRoleSchema`), `src/modules/auth/index.ts` (`requireAuth`, `requireRole`), `src/lib/errors.ts`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-06-04] [lpg-backend] Implemented barebones `users` module (`{index,routes,service,repository,types}.ts`) on the auth-owned `users` table — no new schema, no migration. Endpoints: `GET /` (with `?role=`/`?active=`), `GET /:id`, `PATCH /:id`. All gated by `requireAuth` + `requireRole('admin')`, reusing the auth middleware injected via `createUsersModule`. Guards in `UsersService.updateUser`: 404 missing target, 400 empty patch (zod `.refine`), 403 on developer-escalation and on self role/active changes. Reuses `PublicUser`/`toPublicUser`/`userRoleSchema` from auth so `password_hash` never serializes. Mounted at `/api/v1/users`.
- [2026-06-04] [lpg-backend] Tests: 5 lifecycle integration tests over an in-memory `FakeUsersRepository` + minted HS256 JWTs — no Postgres/Redis. Covers list+filter, view+404, role assignment + profile edit + empty-patch, escalation + self-lockout, authz (operator 403, no-token 401). Gates green: `tsc --noEmit` clean, `biome check` clean, `node --test` 10/10, build emits 5 JS files. Independent validation agent confirmed all 8 acceptance criteria met, no bugs.
- [2026-06-04] [lpg-backend] PK note: integer PKs for tables referenced across the schema; UUID reserved for leaf tables. `users` already uses a serial key, so this module is consistent.
- [2026-06-04] [lpg-backend] All backend acceptance criteria for this repo met.
