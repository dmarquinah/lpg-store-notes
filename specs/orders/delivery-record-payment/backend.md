---
project: lpg-store
domain: specs
type: spec-track
spec: delivery-record-payment
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-15
---

# Delivery user records payment on own orders — lpg-backend track

Shared spec: [[index]]

## Technical Notes
- Endpoint already exists: `POST /:id/payments` → `OrdersService.recordPayment`.
  Route is currently guarded by `office = requireRole('operator','admin')` at
  `src/modules/orders/routes.ts:169`. Widen it to include delivery — either switch
  to the existing `anyStaff = requireRole('operator','admin','delivery')`
  (`routes.ts:58`) or a dedicated guard — then enforce ownership in the service.
- **Scoping is the crux.** Don't let the widened guard leak foreign orders to
  drivers. In `recordPayment` (`src/modules/orders/service.ts:526`), for a delivery
  caller apply the existing visibility check `assertVisible` (`service.ts:722`),
  which already limits a driver to orders whose `assignmentId` is among their own
  assignments (same rule as `GET /:id`). Operators/admins keep the current
  branch-scoped path; developer bypasses (`isGlobal`, `service.ts:678`). Mirror the
  role-branching pattern already used elsewhere in the service rather than inventing
  a new one.
- Preserve every existing invariant on the delivery path: order must be `delivered`,
  walk-ins (`customerId == null`) rejected, overpayment rejected, and settlement via
  `customers.settleOrderCharges` on full payment (`service.ts:558-559`). These are in
  `recordPayment` already — the scoping check goes **before** them so a foreign order
  yields 403, not a state error.
- No migration, no schema change. Follow the lifecycle-test convention: add cases to
  the orders `__tests__` — delivery success on own order, delivery 403 on a foreign
  order, operator path unchanged.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `src/modules/orders/routes.ts` — `office`/`anyStaff` guards (57-58); widen the `POST /:id/payments` guard (169).
- `src/modules/orders/service.ts` — `recordPayment` (526-562); `assertVisible` (722-735); `isGlobal` (678-680) — add the role-scoped ownership check.
- `src/modules/orders/__tests__/` — add the three lifecycle cases.
- `src/modules/auth/middleware.ts` — `createRequireRole` (71-81), for reference on the guard contract.

## Implementation Notes
<!-- [YYYY-MM-DD] [lpg-backend] description -->

[2026-07-15] [lpg-backend] Implemented. Turned out to be a one-line guard widening: `recordPayment` (`service.ts:526`) already calls `assertVisible` first, which already scopes a delivery caller to orders on its own assignments (404 on foreign, same boundary as `GET /:id`) and keeps operators branch-scoped / admin+developer global. So:
- `routes.ts` — `POST /:id/payments` guard `office` → `anyStaff` (operator/admin/delivery; developer bypasses via `createRequireRole`). Updated the guard-doc + route comments.
- `service.ts` — no logic change; updated the `recordPayment` scoping comment to cover the delivery path. All existing invariants (delivered-only, walk-in reject, overpayment reject, `settleOrderCharges` on full payment) run after `assertVisible`, so they hold for drivers unchanged.
- No migration, no schema change.
- Tests (`__tests__/orders.test.ts`, new describe block): `(cobro)` driver records final collection on its own delivered-on-credit order (201, paid, charge settled), operator partial in the same flow proves the office path unchanged, driver overpayment → 409, and the detail names the driver's payment `Carlos Reparto` (open-question confirmation — timeline reads role-safely). `(cobro-scope)` a second driver with no assignments → 404, no payment written.
- Resolved spec discrepancy: criterion #2 said 403 but the referenced `GET /:id` boundary (`assertVisible`) is 404; aligned criterion + tests to 404 (user-confirmed).

[2026-07-15] [lpg-backend] All criteria for this repo met — validated against the shared checklist (delivery-own success, foreign 404, operator path unchanged, invariants hold) and full gate green (typecheck, check, test 176/176, build). Track done; overall spec remains in-progress pending the frontend track.
