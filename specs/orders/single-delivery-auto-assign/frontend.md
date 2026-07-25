---
project: lpg-store
domain: specs
type: spec-track
spec: single-delivery-auto-assign
repo: lpg-frontend-vue
kind: frontend
track-status: not-started
last-updated: 2026-07-04
---

# Single-Delivery Auto-Assign — lpg-frontend-vue track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

Minimal frontend impact — auto-assign is a backend behavior. After creating an
order for a single-delivery store, the returned order is already assigned, so the
detail/queue reflect the driver + "Asignado" timeline node with no extra step.
Confirm the order-entry flow and detail view render the already-assigned state
cleanly (no "assign" prompt shown for an already-assigned order); the manual
assign affordance stays for 0/>1-delivery stores.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To review / adjust:

- `src/modules/orders/` — order-entry wizard + detail/queue: verify the
  already-assigned state renders without a manual-assign prompt.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

_None yet — backlog draft, track not started._
