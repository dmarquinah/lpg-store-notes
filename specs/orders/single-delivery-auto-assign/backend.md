---
project: lpg-store
domain: specs
type: spec-track
spec: single-delivery-auto-assign
repo: lpg-backend
kind: backend
track-status: not-started
last-updated: 2026-07-04
---

# Single-Delivery Auto-Assign — lpg-backend track

Shared spec: [[index]] · Related: [[../../inventory/quick-assignment/backend]]

## Technical Notes

Additive to the **orders** module. In the order-creation path, after the order
row is written, resolve the store's active delivery assignments via catalog's
`listStoreAssignmentDetails({ storeId, role: 'delivery', active: true })` (same
sole-driver resolution as [[../../inventory/quick-assignment/backend]]). Exactly
one → call the **existing** assign transition for that order/driver **in the same
transaction** so the CAS status change + `order_status_history` "Asignado" node
(with `target_user_id`, from [[../order-event-timeline/index]]) are produced by
the real path, not a bespoke one. 0 or >1 → leave unassigned (unchanged).

Keep the sole-driver rule in **one** shared place if practical (it's the same
predicate quick-assignment uses) to avoid two definitions of "single-delivery
store" drifting.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/orders/service.ts` — order creation: sole-driver lookup + reuse the
  assign transition when exactly one.
- `src/modules/orders/__tests__/*.ts` — auto-assign on single-delivery, no-op on
  0/>1, history node asserted.

Read-only context (no change):

- `src/modules/catalog/service.ts` — `listStoreAssignmentDetails` (sole-driver).
- `src/modules/orders/` — the existing assign transition + status-history writer.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

_None yet — backlog draft, track not started._
