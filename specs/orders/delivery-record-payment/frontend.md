---
project: lpg-store
domain: specs
type: spec-track
spec: delivery-record-payment
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: '"2026-07-15"'
---

# Delivery user records payment on own orders — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes
- The record-payment ("cobro") UI already exists for operators/admins on the order
  detail. This track exposes that same affordance to delivery users for their own
  delivered-on-credit orders — reuse the existing dialog/action, don't build a new one.
- Gate visibility on: role is `delivery` **and** the order is `delivered` with
  outstanding debt **and** it's one of the driver's own orders (the backend already
  scopes the list/detail to the driver, so a driver only ever sees his own orders —
  the client-side guard is mostly about showing the button for the right order state).
- The driver-facing order screens live in the delivery views (e.g. `mis-entregas`
  / delivery order detail). Place the button where a driver already views a finished
  delivery, consistent with the phone-first delivery UI.
- On success, re-fetch the order so the payment/timeline and remaining-debt update;
  reuse existing toast + error handling. No new store endpoint — the orders store
  already wraps `POST /:id/payments`.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)
- `src/modules/orders/views/OrderDetailView.vue` — existing record-payment affordance; extend visibility to delivery.
- `src/modules/orders/views/` — delivery order/delivery detail view (`mis-entregas`), if drivers use a distinct screen.
- `src/modules/orders/store` (Pinia) — existing `recordPayment` action wrapping `POST /:id/payments`.
- `src/modules/orders/components/` — the existing payment dialog/form to reuse.

## Implementation Notes
<!-- [YYYY-MM-DD] [lpg-frontend-vue] description -->

[2026-07-15] [lpg-frontend-vue] Implemented — a one-guard widening, exactly as the track scoped it. In `views/OrderDetailView.vue`, `canRecordPayment` gate `isOffice` → `isField` (office + delivery; `isField` already includes `delivery`). The existing "Registrar pago" button, `PaymentDialog`, `confirmPayment` (→ role-agnostic `store.recordPayment` wrapping `POST /:id/payments`), success re-fetch, timeline/saldo update, toast + error handling all already apply to the delivery path — no new component, dialog, store action, or route. The delivery role reaches the same `OrderDetailView` via the `delivery-order-detail` route.

Client scoping left at order-state only (`delivered` + has customerId + `outstanding > 0`) — safe because the backend scopes a driver's detail via `assertVisible`, so a driver only ever loads orders on his own assignments, and the payment endpoint enforces the same boundary (foreign → 404). Updated the guard comment to record this.

Validation (agent vs. shared checklist): frontend criterion MET; office roles unregressed (`isField` still contains operator/admin/developer). Gate green: typecheck ✓, build ✓.

[2026-07-15] [lpg-frontend-vue] All criteria for this repo met.
