---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: draft
depends-on:
  - "[[../orders-multi-location/index]]"
  - "[[../../inventory/quick-assignment/index]]"
last-updated: 2026-07-04
---

# Spec: Single-Delivery Auto-Assign (auto-assign the order's driver when a store has one)

## Problem Statement

While the business runs **one delivery driver per store** (see
[[../../inventory/quick-assignment/index|quick-assignment]]), assigning an order
to a driver is a redundant manual step: there's only one possible driver. The
owner floated it as a "maybe expand" of the quick-assignment work — *"if a store
has only 1 delivery, when an order is created for a store, automatically assign
the delivery so the product delivery becomes effective."* Deferred here so
quick-assignment can ship focused; picked up after.

## Proposed Solution

When an order is created for a store that has **exactly one active delivery**,
**auto-assign** that driver as part of order creation — moving the order straight
to its assigned state instead of leaving it unassigned for a manual pick. Reuse
the existing assign transition (the atomic CAS assign from
[[../orders-multi-location/index]] + the `order_status_history.target_user_id`
"Asignado" node from [[../order-event-timeline/index]]) — do **not** invent a new
path. When the store has 0 or >1 active delivery, creation is unchanged
(unassigned, manual pick).

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Backend (lpg-backend):**

- [ ] On order creation for a store with **exactly one** active delivery
      assignment, the order is **auto-assigned** to that driver via the existing
      assign transition (same status-history "Asignado" node + `target_user_id`,
      same ownership rules), in the creation transaction.
- [ ] Stores with **0 or >1** active delivery → **unchanged** (order created
      unassigned; manual assignment as today).
- [ ] Reuses the existing assign logic (no duplicate transition path); the
      auto-assign is attributable to the creating actor. Tests: auto-assign on
      single-delivery store, no auto-assign on 0/>1, history node present.
      Existing tests green; typecheck / lint / build green.

**Frontend (lpg-frontend-vue):**

- [ ] Order creation for a single-delivery store reflects the **already-assigned**
      state (driver shown, "Asignado" in the timeline) without a manual step; the
      manual assign affordance remains for 0/>1-delivery stores. `npm run
      typecheck` + `npm run build` green.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | not-started |
| [[frontend]] | lpg-frontend-vue | frontend | not-started |

## Out of Scope

- **Inventory auto-load** (quick-open, purchase auto-load) — sibling
  [[../../inventory/quick-assignment/index]].
- **Auto-fulfilling / delivering the order.** Auto-assign only sets the driver; the
  delivery + sale recording stay explicit.
- **Multi-driver assignment logic** — only the unambiguous single-driver case is
  automated.

## Open Questions

- **Does auto-assign also auto-open the driver's day if none is open?** Proposed:
  **no** — assignment and the inventory day are separate; day-open stays the
  quick-open button. Confirm when this leaves the backlog.
- **Should a later transfer/cancel be unaffected?** Proposed: yes — auto-assign
  only changes the *initial* state; all later transitions behave as today.
