---
project: lpg-store
domain: specs
type: spec-track
spec: auth-foundation
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-05-08
---

# Auth Foundation — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

The client-side auth foundation (login + session + token-aware API client) was delivered as the first vertical slice of the v2 skeleton — see [[../../bootstrap/v2-skeleton/frontend]]. This track owns the `src/modules/auth/` module going forward.

- **Login flow**: `LoginView` posts to backend `/v1/auth/login`, receives `{ token, user }`, stores the JWT and routes the user to their role's home.
- **Token storage**: localStorage, read/written **only** inside `modules/auth/store.ts`. The shared `apiClient` never touches storage directly — it takes a `getToken()` callback wired in `main.ts`.
- **Role redirect**: a single global router guard reads `meta.roles`; `developer` is aliased to the admin home (matching backend "admin + override" semantics).
- **Type parity**: `USER_ROLES` const + `isUserRole` runtime guard mirror `lpg-backend/src/modules/auth/schema.ts`, so the role enum can't drift between repos.

Richer auth screens — invite-completion (`/invite/:token`) and a change-password UI — are **out of scope** here and tracked as a future spec (see the shared [[index]] Out of Scope).

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/auth/` — full vertical slice: `types.ts`, `service.ts`, `store.ts`, `routes.ts`, `views/LoginView.vue`
- `src/lib/apiClient.ts` — fetch wrapper taking a `getToken()` callback
- `src/router/index.ts` — single global guard with role redirects + `developer` bypass
- `src/main.ts` — wires `getToken()` from the auth store into the apiClient

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-05-08] [lpg-frontend-vue] Auth client slice built as the first module of the v2 skeleton: `modules/auth/` (types/service/store/routes/login view) with `USER_ROLES` + `isUserRole` mirroring the backend schema; localStorage JWT read/write isolated to `store.ts`; apiClient wired via `getToken()`; global guard role redirects with `developer` → admin home. Delivered and verified under [[../../bootstrap/v2-skeleton/frontend]] (login renders, transforms clean, build green). Backend round-trip not exercised locally during that session (backend not running) — visual + transform smoke test only.
- [2026-05-08] [lpg-frontend-vue] Auth foundation criteria for this repo met (login + session + token-aware client). Further auth screens deferred to a separate spec.
