---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../stores/store-management/index]]"
  - "[[../../users/users-crud/index]]"
  - "[[../../auth/auth-foundation/index]]"
last-updated: 2026-06-15
---

# Spec: Organization Management (unified admin console)

## Problem Statement

Admin management of **people and places** is scattered across three disconnected
surfaces: `/usuarios` (user list + edit), and the Catálogo page's **Tiendas** and
**Asignaciones** tabs (store CRUD + store↔user assignments, shipped in
[[../../stores/store-management/index|store-management]]). To answer a simple
question — *"which employees are assigned to each location?"* — an admin must hop
between pages and tabs and cross-reference by hand. There is no single overview of
locations and their staff, and there is **no UI to invite/create a new user at all**
(users only exist via the backend `POST /auth/register` invitation flow, which has
no frontend).

The owner wants one place to get a good overview of all locations and the
employees assigned to each, and to manage them with few clicks.

## Proposed Solution

A new admin/developer-only **Organización** view, organized as a
**location-centric board**: each store is a card/section listing its assigned
**operators** and **drivers**, with inline actions to assign a user, deactivate an
assignment, and edit the store; plus a control to create a new store. A separate
area lists **all users / unassigned users** with inline role/active edit and an
**Invitar usuario** action that wires the existing `POST /auth/register` and
surfaces a copyable invite link for the admin to share (no email infra in MVP).

This **replaces** the dispersed surfaces: the standalone `/usuarios` page and the
Catálogo **Tiendas** + **Asignaciones** tabs are folded into Organización; Catálogo
keeps only **Tipos de tanque** + **Artículos**. A new `organization` frontend
module composes the existing `catalog`, `users`, and `auth` pieces.

A new **aggregated backend endpoint** returns each store with its assigned users in
one call, so the board doesn't cross-join three lists on the client.

Detailed endpoints and file lists live in [[backend]] and [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] New aggregated read `GET /catalog/stores-with-assignments` (admin/developer):
  returns each store with its **active** assigned users (`{ id, name, role }`),
  grouped per store, in a single query (no N+1). Honours `?all=1` to include
  inactive stores, mirroring the other catalog reads. Returns a typed
  `StoreWithAssignments[]` envelope.
- [ ] Existing catalog + auth behaviour and tests stay green. No schema/migration
  change (the invite/create path reuses `POST /auth/register`; assignment writes
  reuse the `store-management` endpoints).
- [ ] Tests: the aggregate groups users under the right stores; reflects an
  assignment after create and drops a user after its assignment is deactivated;
  `?all=1` includes an inactive store.

**Frontend (lpg-frontend-vue):**

- [ ] New admin/developer-only **/organizacion** route + **Organización** nav entry.
  Location-centric board: one section per store (operators & drivers listed),
  with **assign user**, **deactivate assignment**, **edit store**, and **new store**
  — reusing `StoreFormDialog` + the assignment actions from `store-management`.
- [ ] A **users** area on the same view: list all users with inline **edit**
  (role/active, reusing `UserEditDialog`) and an **Invitar usuario** dialog that
  POSTs `/auth/register` and shows a **copyable invite link** (`/invite/:token`)
  for the admin to share. Surfaces global roles (admin/dev) and unassigned users
  clearly (they belong to no store section).
- [ ] **Replace** the old surfaces: remove the standalone **Usuarios** nav entry and
  the Catálogo **Tiendas** + **Asignaciones** tabs (Catálogo retains Tipos de
  tanque + Artículos). `/usuarios` redirects to `/organizacion` (no dead links).
- [ ] Consistent with [[../../eng/design-system]] (PageHeader, tokens, canonical
  dialog chrome, badges, ≥44px touch targets); both light/dark verified.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

> **Porting order:** the aggregated endpoint (backend) first, then the frontend
> board. The frontend can start against the existing separate list endpoints and
> swap to the aggregate when ready, so the tracks can overlap.

## Out of Scope

- **Operator/driver self-service** — Organización is admin/developer only. (The
  operator location feedback/defaults shipped separately under
  [[../../orders/orders-multi-location/index|orders-multi-location]].)
- **Changing the invitation mechanism** — reuse `POST /auth/register` +
  `POST /auth/invite/:token` as-is. No email/SMS delivery of invites in MVP (the
  admin copies the link). Password-reset / re-invite flows are a separate spec.
- **Bulk import** of users/stores/assignments.
- **Per-store configuration** beyond name/address/phone (hours, geo, etc.).
- **Customers** — unrelated domain, untouched.
- **Schema changes** — none; all tables exist.

## Open Questions (resolved 2026-06-15)

- **Invite delivery:** ✅ RESOLVED — show a copyable `/invite/:token` link + expiry
  in the UI for the admin to share manually (no email infra).
- **Deactivating a user:** ✅ RESOLVED — **independent**; setting a user
  `active:false` does NOT cascade to their store assignments. Surface both states
  separately; don't cascade, to preserve history.
- **Placement of global/unassigned users:** ✅ RESOLVED — a users panel/area with a
  filter, alongside the store sections.
- **Route name:** ✅ RESOLVED — `/organizacion`.
- **Drivers without a store:** ✅ RESOLVED — yes, drivers still appear in the users
  area even when unassigned, **but unassigned users are flagged** in the UI. The
  flag opens a product question for a later spec: *should an inventory/store
  assignment be a prerequisite before a driver can operate?* — surfaced, not
  enforced here.
