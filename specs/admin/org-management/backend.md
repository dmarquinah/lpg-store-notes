---
project: lpg-store
domain: specs
type: spec-track
spec: org-management
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-15
---

# Organization Management — lpg-backend track

Shared spec: [[index]]

## Technical Notes

Small backend slice: one **aggregated read** on the existing `catalog` module. The
invite/create-user path and the assignment writes already exist — **no new write
endpoints, no schema/migration change**.

### Aggregated endpoint

- `GET /catalog/stores-with-assignments` — `requireRole('admin')` (developer
  bypasses via the guard). Returns `StoreWithAssignments[]`: each store
  (`PublicStore` shape) plus its **active** assigned users
  (`{ id, name, role }[]`). Honour `?all=1` to include inactive stores (mirror
  `listQuerySchema`).
- Repository: a single query joining `stores ← store_assignments → users`
  (active assignments), grouped per store in code (no N+1). Stores with no active
  assignments still appear (left join), with an empty users array.
- Service: `listStoresWithAssignments(all)`; reuse the existing `PublicStore`
  mapping and the user public shape.

### Reused, not rebuilt

- **Invite/create user** = existing `POST /auth/register` (admin/developer →
  creates inactive user + invitation token, returns `{ user, invitation: { token,
  expiresAt } }`) and public `POST /auth/invite/:token`. The frontend wires these;
  backend unchanged.
- **Store CRUD + assignment create/deactivate** = the `store-management` endpoints
  (`POST`/`PATCH /catalog/stores`, `POST`/`PATCH /catalog/store-assignments`).
- **User role/active edit** = existing `PATCH /users/:id`.

Keep everything flowing through `src/lib/errors.ts`.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/catalog/repository.ts` — `storesWithAssignments(all)` aggregate (stores ⟕ active store_assignments ⟕ users)
- `src/modules/catalog/service.ts` — `listStoresWithAssignments(all)`
- `src/modules/catalog/routes.ts` — `GET /stores-with-assignments` (requireRole('admin'))
- `src/modules/catalog/types.ts` — `StoreWithAssignments` response type + envelope
- `src/modules/catalog/__tests__/*` — aggregate grouping; reflects assign + assignment-deactivate; `?all=1` inactive store

Context (read; do not needlessly modify):

- `src/modules/auth/routes.ts` / `service.ts` — `register` (invite/create) + `invite/:token` (reference; the contract the frontend wires)
- `src/modules/auth/schema.ts` — `users`, `user_invitations`
- `src/modules/catalog/schema.ts` — `stores`, `store_assignments`
- `src/modules/catalog/repository.ts` — existing `getStoreAssignments` join (shape to reuse)

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-backend] Backend track done. One aggregated read added to the existing `catalog` module — no schema/migration change, no new write endpoints (invite/create reuses `POST /auth/register`; assignment writes reuse the `store-management` endpoints).

- **types.ts** — `StoreWithAssignments extends PublicStore` with `users: { id, name, role }[]` (empty for an unassigned store).
- **repository.ts** — `StoreWithUserRow` flat-row type + `storesWithAssignments(includeInactive)`: a single `select` over `stores ⟕ store_assignments ⟕ users`. The **active-assignment filter lives in the JOIN condition** (`and(eq(storeAssignments.storeId, stores.id), eq(storeAssignments.active, true))`), not WHERE, so stores with zero active assignments survive the left join (one all-null-user row). Ordered `asc(stores.id), asc(users.id)`. `?all=1` toggles the `where(eq(stores.active, true))` on the active-only path only.
- **service.ts** — `listStoresWithAssignments(includeInactive)` groups the flat rows per store in a `Map` (insertion order = store id), null-guarding user columns → empty `users` for unassigned stores. No N+1.
- **routes.ts** — `GET /stores-with-assignments`, `requireRole('admin')` (developer bypasses via the guard), envelope `{ stores: StoreWithAssignments[] }`, `?all=1` via the shared `listQuerySchema`.
- **__tests__** — `FakeCatalogRepository.storesWithAssignments` mirrors the real left-join semantics; 2 new lifecycle tests (grouping under the right stores incl. an empty group; deactivation drops a user; `?all=1` includes an inactive store with its still-active link; non-admin → 403). Catalog suite 7 → 9; **project 73 → 75 tests**.

Gates green: typecheck ✓ · biome lint ✓ · 75 tests ✓ · build ✓. Independent validation confirmed all 3 backend criteria, no gaps. Frontend track remains.
[2026-06-15] [lpg-backend] User-approved cross-repo follow-on (track stays `done`). Added a **public read** `GET /auth/invite/:token` so the invite-completion screen can confirm whose account it is. New `AuthService.getInvitation(token)` → `InvitationInfo { name, email, expiresAt }`, with the same validity gates as completing it (404 unknown, 410 used/expired); reuses existing `findInvitationByToken` + `findUserById` (no repo/schema change). New route mirrors the existing `POST /auth/invite/:token`. Extended the invite-flow test to cover the read (200 with name/email, 404 unknown, 410 after use). Gates green: typecheck ✓ · biome lint ✓ · 75 tests ✓ · build ✓.