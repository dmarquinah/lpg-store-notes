---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on: []
last-updated: 2026-05-08
---

# Spec: Auth Foundation

## Problem Statement

V2 has no auth — no users, no login, no way to gate routes by role. Every other module (users, stores, inventory, customers, orders) presupposes "we know who's logged in and what they can do." Auth is the foundation that unblocks everything, in both repos: the backend issues and verifies sessions; the frontend needs a login slice and a token-aware API client before any feature UI can be built.

V1 had auth but with several over-builds we're not carrying forward: 4 user-type tables (admins / operators / delivery_personnel / superadmins), 4 login strategy classes that did the same thing modulo JWT payload, a `module:action` permission system with wildcards, and a separate pre-registration token flow distinct from user creation. We consolidate to a smaller surface that fits a 1-store, ~5-user business.

## Proposed Solution

- **Backend:** a single `users` table for all roles (delivery gets a 1:1 sub-table), admin-issued invite tokens, a `BOOTSTRAP_TOKEN`-gated developer-creation endpoint, HS256 JWTs, `requireAuth` / `requireRole` middleware, and a Redis logout blocklist. Role-based authorization only (no permission strings); resource scoping is per-handler. Single `AuthService` — no strategy pattern.
- **Frontend:** a minimal auth vertical slice (login view, auth store, token-aware `apiClient`, role redirect) delivered as the first module of the v2 skeleton. This track owns the client-side auth module going forward; richer auth screens (invite completion, change-password UI) are a separate future spec.

Detailed data model, endpoints, and module structure live in each track file.

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [x] `users`, `delivery_profiles`, `user_invitations` tables created via a generated migration in `src/db/migrations/`.
- [x] `POST /api/v1/auth/bootstrap` creates a `developer` user with `active=true` when `BOOTSTRAP_TOKEN` is set and the body matches; `404` when the env var is unset; `401` on token mismatch; `409` on duplicate email.
- [x] `POST /api/v1/auth/login` validates credentials (bcrypt compare), returns `{ token, user }`. Inactive users cannot log in.
- [x] Failed login returns `401` with a generic message; no info leak distinguishing unknown email vs wrong password.
- [x] `GET /api/v1/auth/me` requires `requireAuth` and returns the current user (no `password_hash`).
- [x] `POST /api/v1/auth/change-password` requires `requireAuth`; validates `currentPassword`; updates hash on success.
- [x] `POST /api/v1/auth/register` requires `requireRole('admin','developer')`; rejects duplicate email; creates `active=false` user + a `user_invitations` row (24h TTL); returns `{ user, invitation }`. Only `developer` callers may create another `developer`; admins requesting `role: 'developer'` get `403`.
- [x] `POST /api/v1/auth/invite/:token` accepts a `password` (min 12), validates the token (exists, not expired, not used), sets the hash, marks user `active=true`, stamps `used_at`.
- [x] `POST /api/v1/auth/logout` adds the token's `jti` to a Redis blocklist with TTL = remaining token lifetime. `503` if Redis is unavailable.
- [x] `requireAuth` checks the blocklist and rejects blocklisted tokens with `401`. `requireRole` allows the listed roles plus `developer`; `403` on mismatch.
- [x] Tests cover login success/failure, inactive-user block, missing/invalid/expired/blocklisted token, role mismatch, developer bypass, duplicate-email register, expired/used invite, wrong-current-password change, admin-cannot-create-developer.
- [x] All routes mounted at `/api/v1/auth` from `src/app.ts` via `createAuthModule({ db, cache })`.

**Frontend (lpg-frontend-vue):**

- [x] Login view authenticates against backend `/v1/auth/login`, stores the JWT (localStorage, read/written only inside `modules/auth/store.ts`).
- [x] `apiClient` attaches the token via a `getToken()` callback; the global router guard redirects by role and aligns `developer` with the admin bypass.
- [x] The auth module is a self-contained vertical slice under `src/modules/auth/`, ready for future auth screens to extend.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- Password reset (admin re-invites via `/register` → new invite link)
- Token refresh (24h tokens + login again is sufficient for an internal tool)
- Email / SMS delivery of invite URLs (admin shares the URL manually)
- Resource-level authorization in middleware (per-handler)
- Rate limiting, 2FA / MFA, OAuth / SSO
- **Frontend:** invite-completion and change-password screens — a separate future spec (`auth-frontend/` or extension of this folder's frontend track)

## Open Questions

None remaining (all resolved during the 2026-05-07 review).
