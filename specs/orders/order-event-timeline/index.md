---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../orders-foundation/index]]"
  - "[[../orders-multi-location/index]]"
last-updated: 2026-06-15
---

# Spec: Order Event Timeline (unified status + payment timeline with actor names)

## Problem Statement

The order detail (`/pedidos/:id` and the driver's `mis-entregas/:id`) shows the
order's history in **two separate cards** — a vertical "Historial" (status
changes) and a "Pagos" table — sitting side by side. They tell one story (what
happened to this order, and when) but are split by data source, so the reader has
to mentally interleave them by time. Worse, **neither shows _who_ did anything**:
the status events and payments carry only user IDs (`changedBy`, `recordedBy`),
and the frontend can't resolve names because `GET /users` is admin-only while
this view is used by operators and delivery drivers too.

There's also a missing dimension: the **`assigned`** event is inherently
*directed* — an operator assigns an order **to** a driver — but only the operator
(`changedBy`) is recorded; the assignee ("to whom") is nowhere in the history.

## Proposed Solution

Replace the two cards with a **single, full-width horizontal timeline** that
merges status events and payments into one time-ordered sequence, and make each
node name the people involved.

- **Backend enriches the detail response with names** (the only role-safe place,
  since `GET /users` is admin-only): each status event gains `changedByName`,
  each payment gains `recordedByName` (LEFT JOIN `users.name` in the detail
  queries).
- **Backend captures the assignment target.** A new nullable
  `order_status_history.target_user_id` (migration; old rows null, no back-fill)
  is populated on **assign** with the driver resolved from the bound inventory
  assignment → store-assignment → user. The view returns `targetUserId` +
  `targetUserName`. Generalizes to other directed events later.
- **Frontend renders one horizontal timeline** merging both sources by datetime.
  Each node shows its label + Lima-time stamp and the involved actor(s):
  most read `registrado por {name}`; the assignment node reads
  `asignado por {operator} → {driver}`; payment nodes read
  `pago · {método} · {monto} — registrado por {name}`.

No change to the order lifecycle, the payments ledger, or the status enum — this
is an additive read-enrichment + one nullable column + a UI consolidation.

Detailed backend design lives in [[backend]]; the timeline component in
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] The order-detail response includes a display name for the actor of every
  status event (`changedByName`) and every payment (`recordedByName`); works for
  operator/delivery/admin callers (no admin-only dependency).
- [ ] `order_status_history` has a nullable `target_user_id` (migration applied to
  the dev DB); existing rows are unaffected (null).
- [ ] The `assigned` transition stores `target_user_id` = the driver behind the
  bound inventory assignment; the detail view exposes `targetUserId` +
  `targetUserName`. Non-assign events leave it null.
- [ ] Other directed-or-not events are unchanged; `targetUserId`/name are null for
  them. No change to the lifecycle, status enum, or payments ledger.
- [ ] typecheck / lint / build green; existing orders tests stay green; new/updated
  tests cover the name enrichment + the assign target.

**Frontend (lpg-frontend-vue):**

- [ ] The order detail shows a **single horizontal timeline** (one row, full
  width) replacing the separate "Historial" and "Pagos" cards.
- [ ] The timeline merges status events **and** payments in datetime order; payment
  nodes show method + amount.
- [ ] Each node names the involved actor(s): `registrado por {name}` for
  single-actor events; the assignment node shows both the operator and the driver
  (`asignado por {X} → {Y}`); payment nodes show who recorded them.
- [ ] Mirrors the backend exactly (new fields, envelope keys); typecheck + build
  green. Works for both `/pedidos/:id` and `mis-entregas/:id` (operator + delivery).

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Name resolution for the inventory / other modules' views** — this spec only
  enriches the order-detail response. A general user-name lookup endpoint is a
  separate concern.
- **Editing/deleting timeline events** — the history stays append-only (ADR-009).
- **Target on non-assign events** (e.g. transfer's destination branch) — the
  column generalizes, but only `assigned` populates it here.
- **Back-filling `target_user_id`** for historical assigned rows — left null;
  the UI degrades to showing only the operator for old events.

## Open Questions

- _None._ Approach decided 2026-06-15: dedicated nullable `target_user_id` column
  (per-event, accurate across reassignments) over deriving from the order's
  current assignment.
