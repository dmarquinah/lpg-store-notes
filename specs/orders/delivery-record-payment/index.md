---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on: []
last-updated: '"2026-07-15"'
---

# Spec: Delivery user records payment on own orders

## Problem Statement
When an order is delivered on credit, a debt is recorded against the customer.
The customer often pays later — and the person who collects that money is the
**delivery driver** who goes back to the customer. Today only operators/admins can
register a collection: `POST /:id/payments` is guarded by `office` and delivery is
blocked. So a driver who collects cash for a previously-unpaid order can't record
it himself; it has to be relayed to the office, which is friction and a
traceability gap. Drivers should be able to record a collection **just like
operators/admins — but only for orders assigned to them.**

## Proposed Solution
Allow the `delivery` role to call the existing record-payment endpoint, scoped so a
driver can only record payments on orders on **their own** assignments. The
ownership boundary already exists (`assertVisible` limits a driver to orders on his
assignments); we reuse it rather than inventing new scoping. Operators/admins keep
their current branch-scoped behavior unchanged. Frontend surfaces the existing
"record payment / cobro" affordance to delivery users on the order detail for their
own delivered-on-credit orders.

No new endpoint, no schema change — a guard widening plus a role-conditional
ownership check, and a frontend affordance.

## Acceptance Criteria
<!-- THE single shared checklist — source of truth across all tracks. -->
- [x] A delivery user can `POST /orders/:id/payments` for an order that is on **one of their own assignments**, recording a payment against its pending debt.
- [x] A delivery user attempting to record a payment on an order **not** on their assignments is rejected (**404** — same visibility boundary as `GET /:id` via `assertVisible`, which deliberately doesn't leak cross-scope existence), not silently allowed.
- [x] Operator/admin behavior on the endpoint is unchanged (still branch-scoped, not owner-gated); developer still bypasses.
- [x] All existing record-payment invariants still hold for the delivery path: order must be `delivered`, walk-ins (no `customerId`) rejected, overpayment rejected, and full payment flips the charge to settled via `customers.settleOrderCharges`.
- [x] Frontend: on the order detail, a delivery user viewing their own delivered order with outstanding debt sees and can use the record-payment ("cobro") affordance; it's hidden for orders they can't act on.
- [x] Backend tests cover: delivery records on own order (success), delivery blocked on a foreign order (403), and the unchanged operator path. Full gate green (typecheck, check, test, build).

## Tracks
<!-- Overall status becomes `done` only when EVERY track is done. -->

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope
- Any change to how the debt **charge** is created (still auto-created at delivery on credit).
- Letting delivery record payments on orders they aren't assigned to.
- New payment methods or changes to the payment ledger / settlement logic.
- Partial-payment UX changes beyond exposing the existing affordance to drivers.

## Open Questions
_None — all resolved._

- ~~Should the driver's recorded payment name the driver as `recordedBy` in the
  timeline? Confirm the operator console reads fine when the actor is a driver.~~
  **Resolved (2026-07-15): Yes, no code needed.** `recordPayment` stamps
  `recordedBy: caller.id`; `paymentsForOrder` resolves the name via a role-agnostic
  `innerJoin(users, …)`, so a driver's `recordedByName` renders identically to an
  operator's/admin's. The operator console reads fine — name resolution never
  branches on role.
- ~~Foreign-order rejection status.~~ **Resolved (2026-07-15): 404**, reusing
  `assertVisible` unchanged (same boundary as `GET /:id`). Criterion #2 updated.
