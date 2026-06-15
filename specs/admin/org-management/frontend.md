---
project: lpg-store
domain: specs
type: spec-track
spec: org-management
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-15
---

# Organization Management — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

The bulk of the work. A new **`organization` vertical module** that *composes*
existing modules rather than duplicating them — the cross-cutting admin surface
the v2 conventions allow (the module root is the only public boundary).

### The view — location-centric board

- `views/OrganizationView.vue` (route `/organizacion`, admin/developer): a board
  of **store sections**. Each section shows the store header (name/address/phone +
  active badge) with **Editar** (reuse `catalog/components/StoreFormDialog.vue`),
  and two grouped lists — **Operadores** and **Repartidores** — of assigned users
  with a **Quitar** (deactivate assignment) action, plus **Asignar usuario**
  (reuse the `StoreAssignmentDialog` logic, pre-scoped to that store). A
  **Nueva tienda** control creates a store.
- A **users area** (panel/section beside or below the board): every user with role
  + active badges, inline **Editar** (reuse `users/components/UserEditDialog.vue`),
  and **Invitar usuario**. Global roles (admin/dev) and unassigned users live here
  (they belong to no store section).
- Data: prefer the new aggregate `GET /catalog/stores-with-assignments`; the users
  area uses the `users` store. Reuses `catalog` store actions
  (`createStore`/`updateStore`/`createStoreAssignment`/`deactivateStoreAssignment`)
  already shipped in `store-management`.

### Invite/create user

- `components/InviteUserDialog.vue`: name/email/phone/role form (validation
  mirroring backend `registerRequestSchema`), POSTs `/auth/register`, then shows
  the **invite link** (`/invite/:token`) + expiry with a copy button. Needs new
  `auth` service wiring: `register(payload)` over `POST /auth/register` (the
  frontend `auth` service has login/me only today).

### Replace the old surfaces (don't leave duplicates)

- `layouts/AppLayout.vue` `ROLE_NAV`: drop the **Usuarios** entry for admin +
  developer, add **Organización** (`/organizacion`). (Catálogo entry stays.)
- `modules/catalog/views/CatalogView.vue`: remove the **Tiendas** and
  **Asignaciones** tabs (and their now-unused wiring); keep Tipos de tanque +
  Artículos. The store/assignment dialogs move to (or are shared with) the
  `organization` module.
- Routing: register `/organizacion`; redirect `/usuarios` → `/organizacion` (no
  dead links/bookmarks). Decide whether to retire the `users` *view* while keeping
  the `users` store/service (still used for the users area + edit dialog).
- `main.ts`: wire `createOrganizationModule({ apiClient })`.

### Reuse map (avoid rebuilding)

- `StoreFormDialog.vue`, `StoreAssignmentDialog.vue`, catalog store actions — from `store-management`.
- `UserEditDialog.vue`, `users` store/service — from `users-crud`.
- `auth` store/service — extend with `register`.
- Shared chrome: `PageHeader`, `Spinner`, `EmptyState`, `formatMoney`, badge variants per [[../../eng/design-system]].

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To create:

- `src/modules/organization/views/OrganizationView.vue` — the board
- `src/modules/organization/components/InviteUserDialog.vue` — invite/create + copyable link
- `src/modules/organization/{service,store,routes,types,index}.ts` — module (stores-with-assignments read; composes catalog/users/auth)

To modify:

- `src/modules/auth/{service,types}.ts` — add `register(payload)` (`POST /auth/register`) + payload/response types
- `src/layouts/AppLayout.vue` — `ROLE_NAV`: replace Usuarios with Organización (admin + developer)
- `src/modules/catalog/views/CatalogView.vue` — remove Tiendas + Asignaciones tabs
- `src/router/index.ts` (and/or module routes) — `/organizacion` route + `/usuarios` redirect
- `src/main.ts` — wire the new module

Context (read; reuse):

- `src/modules/catalog/components/{StoreFormDialog,StoreAssignmentDialog}.vue`, `src/modules/catalog/store.ts`
- `src/modules/users/components/UserEditDialog.vue`, `src/modules/users/{store,service,types}.ts`
- `src/modules/auth/store.ts`
- `src/components/ui/*` (tabs/card/table/dialog/select), `src/components/app/*`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-frontend-vue] Frontend track done. New `organization` vertical module that *composes* `catalog` + `users` + `auth` (no duplication); old surfaces replaced. typecheck + build green; independent validation confirmed all 4 frontend criteria.

- **New module `src/modules/organization/`** — `types.ts` (`StoreWithAssignments extends` catalog `PublicStore` + `users[]`; mirrors backend), `service.ts` (`listStoresWithAssignments(all)` → `GET /catalog/stores-with-assignments[?all=1]`), `store.ts` (`useOrganizationStore`, service-injection pattern), `routes.ts` (`/organizacion` admin/developer **+** `/usuarios` → `/organizacion` redirect), `index.ts` (`createOrganizationModule`), wired in `main.ts`.
- **`views/OrganizationView.vue`** — location-centric board: one `Card` per store (active badge, address/phone, **Editar** → `StoreFormDialog`, **Asignar usuario** → `StoreAssignmentDialog` pre-scoped, grouped **Operadores**/**Repartidores** with **Quitar**), "Mostrar inactivas" switch + **Nueva tienda**. Plus a **Usuarios** card: all users with role/active badges, **Global** badge for admin/dev, **Sin tienda** warning for unassigned operator/delivery, inline **Editar** (`UserEditDialog`) + **Invitar usuario**.
- **Quitar id-resolution wrinkle:** the aggregate returns assigned users as `{id,name,role}` with no assignment id, so deactivation resolves the `store_assignment` id from the catalog's active `storeAssignments` list (keyed `storeId:userId`); the board itself renders from the aggregate per "prefer the aggregate".
- **`components/InviteUserDialog.vue`** — name/email/phone/role form (validation mirrors `registerRequestSchema`; hides `developer` for non-developers), POSTs `/auth/register`, then shows a **copyable** `${origin}/invite/:token` link + Lima-time expiry (`isoToDateTimeDisplay`). (The `/invite/:token` acceptance screen itself is a separate spec — out of scope.)
- **Auth extended** — `auth/types.ts` (`RegisterPayload`/`RegisterResponse`), `auth/service.ts` (`register()`), `auth/store.ts` (`register()` action; does NOT touch the session token).
- **Reused dialogs extended (additive, backward-compatible):** `StoreAssignmentDialog` gained an optional `lockedStore` prop (pre-select + hide the store picker); `StoreFormDialog`/`StoreAssignmentDialog`/`UserEditDialog` each emit `saved` so the board refreshes the aggregate after a write.
- **Old surfaces replaced:** `AppLayout` `ROLE_NAV` Usuarios → **Organización** (`Building2`) for admin+developer; `CatalogView` lost the **Tiendas**+**Asignaciones** tabs (keeps Tipos de tanque + Artículos); `users/routes.ts` emptied + `UsersListView.vue` deleted (store/service/types retained); `AdminHome` shortcut Usuarios → Organización; router spreads `organizationRoutes`.
[2026-06-15] [lpg-frontend-vue] Two user-approved follow-ons (spec stays `done`). typecheck + build green; independent validation PASS on both.

- **Invite-acceptance page wired up (was deferred as out-of-scope):** new **public** route `/invitacion/:token` (Spanish wording, per the owner) → `auth/views/AcceptInviteView.vue` (login-style public chrome): password + confirm (mirrors backend `passwordSchema` min-6), calls new `auth` `completeInvite(token, password)` service + store action over `POST /auth/invite/:token`, success → "Iniciar sesión". Does NOT log the user in. `InviteUserDialog` link updated `/invite/:token` → `/invitacion/:token` to match.
- **OrganizationView redesigned to a two-pane master/detail** (owner chose this over an accordion or drag-and-drop board, to stop cross-referencing a separate users table against location cards): LEFT = selectable list of stores (+ people count) plus **Sin tienda** worklist and **Todos los usuarios** entries; RIGHT = context detail. Store detail shows assigned **Operadores**/**Repartidores** (rows + **Quitar**) and a **Disponibles para asignar** pool with **one-tap Asignar** (inline `createStoreAssignment`, no dialog). **Sin tienda** lists unassigned active operators/drivers with **Asignar a una tienda** → `StoreAssignmentDialog` (now also takes an optional `lockedUser` prop, mirroring `lockedStore`). **Todos los usuarios** keeps the full table + inline **Editar** + **Invitar usuario**. All four original acceptance criteria still hold (re-validated). Detail pane shows a loading state until the first aggregate fetch resolves.
[2026-06-15] [lpg-frontend-vue] Round-3 polish (user-approved; track stays `done`). typecheck + build green.

- **Invite page now confirms the account + saves credentials:** on mount it reads `GET /auth/invite/:token` (new `auth` `getInvitation` service/store action → `InvitationInfo`) and shows "Activando la cuenta de {nombre}". The email renders as a read-only `autocomplete="username"` field paired with the `autocomplete="new-password"` password, so the browser offers to **save the email + password as login credentials**. Invalid/used/expired token → error state with "Ir a iniciar sesión".
- **OrganizationView right panel widened** `md:grid-cols-3` → `md:grid-cols-4` (left 1/4, detail 3/4) so the full user table reads comfortably.
- **Multiple assignments per location clarified:** this already worked (backend `uq_store_assignments_active` is per `(store,user)`, no per-role cap; the UI lists *all* assigned staff and the pool only excludes those already on that store). Added copy under "Disponibles para asignar" — "Puede asignar tantos operadores y repartidores como necesite" — and a clearer empty-pool message so it doesn't read as a one-each limit.