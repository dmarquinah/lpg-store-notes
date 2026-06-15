---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../../auth/auth-foundation/index]]"
last-updated: 2026-06-04
---

# Spec: Users CRUD (barebones — role assignment)

## Problem Statement

Everything downstream (inventory, orders) needs real users with the right roles to exist. Auth (`auth-foundation`, done) already creates users (`/register` + `/invite`) and lets a user change their own password — but there is **no way to manage users after creation**: no list, no lookup, and crucially **no way to change a user's role** or disable an account. The immediate driver is role assignment so test users can be seeded for the modules that follow.

This is intentionally barebones — the minimum needed to assign roles and edit basic profile fields on existing users, plus a thin admin UI to drive it.

## Proposed Solution

- **Backend (done):** a vertical module `src/modules/users/` operating on the **existing** auth-owned `users` table (no new tables, no migration). It reuses auth's `requireAuth` / `requireRole`, the `PublicUser` serializer, and `userRoleSchema`. Three endpoints (list with filters, get one, partial PATCH) with developer-escalation and self-lockout guards.
- **Frontend (not-started):** a users-management screen for admins/developers — list with role/active filters, view one, and an edit form for `name`/`phone`/`role`/`active` that surfaces the same guards (escalation, self-lockout) as friendly errors.

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [x] `GET /api/v1/users` requires `admin`/`developer`; returns all users without `password_hash`; `?role=` and `?active=` filter the result.
- [x] `GET /api/v1/users/:id` returns the user (no `password_hash`); `404` for an unknown id.
- [x] `PATCH /api/v1/users/:id` applies a partial update of `name`/`phone`/`role`/`active`; an empty body returns `400`; an unknown id returns `404`; `updatedAt` advances.
- [x] Assigning `role: 'developer'`, or modifying a user who is currently `developer`, requires the caller to be `developer` (else `403`).
- [x] A caller cannot change their own `role` or `active` (`403`); a `developer` caller can perform all of the above.
- [x] Non-admin roles (`operator`, `delivery`) get `403` on every endpoint; a missing/invalid token gets `401`.
- [x] Module mounted at `/api/v1/users` from `src/app.ts`, reusing the auth module's `requireAuth` / `requireRole`.
- [x] Lifecycle tests cover list+filter, view+404, role assignment, escalation + self-lockout guards, profile edit + empty-patch + authz. No Postgres/Redis (in-memory fake repo).

**Frontend (lpg-frontend-vue):**

- [x] A `users` module screen (admin/developer only) lists users with `role` / `active` filters; never shows `password_hash`.
- [x] An edit form patches `name`/`phone`/`role`/`active`; empty submit is prevented client-side.
- [x] Escalation and self-lockout rejections from the API surface as clear inline errors (caller can't demote/disable themselves; non-developers can't touch a developer).
- [ ] Manual smoke test (pending operator verification — requires a running backend): log in as admin, list, filter, change a test user's role, confirm persistence.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- User creation (auth's `/register` + `/invite` owns it).
- Password reset / change (auth owns it).
- Editing `email` (login identity; would need re-invite semantics).
- Hard delete (deactivate via `active: false` instead — keeps traceability).
- Delivery-profile fields (`license_number`, etc.) — separate spec when delivery lands.
- Pagination / full-text search (≤ ~5 users; defer until volume justifies it).

## Open Questions

None.
