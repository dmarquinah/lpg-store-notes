---
project: lpg-store
domain: specs
type: spec-track
spec: order-event-timeline
repo: lpg-backend
kind: backend
track-status: '"done"'
last-updated: 2026-06-15
---

# Order Event Timeline — lpg-backend track

Shared spec: [[index]] · Frontend track: [[frontend]] · Foundation: [[../orders-foundation/index]]

## Technical Notes

Additive read-enrichment + one nullable column. No lifecycle, enum, or payments
ledger change.

### 1. Actor names in the detail views

`buildDetail` ([service.ts](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/orders/service.ts)) maps `statusHistoryForOrder` /
`paymentsForOrder` to the views, exposing only `changedBy` / `recordedBy` ids.
Add the display name:

- Repo: `statusHistoryForOrder` and `paymentsForOrder` LEFT JOIN `users` to also
  return the actor's `name` (and, for status, the target's name — see §3).
- Views (`OrderStatusEventView`, `OrderPaymentView` in `types.ts`): add
  `changedByName: string` and `recordedByName: string`. Keep the ids.
- This is the role-safe path: `GET /users` is admin-only, but the order detail is
  served to operator/delivery/admin, so names must be embedded here.

### 2. Migration: `target_user_id`

`order_status_history` ([schema.ts:147](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/orders/schema.ts)) gains a nullable
`target_user_id integer references users(id)`. Migration only adds the column
(old rows null, no back-fill). Add `targetUserId` to the Drizzle table + the
`appendStatusHistory` input.

### 3. Populate the assign target

`assignOrder` ([service.ts:205](../../../../../dev/personal/freelance/lpg-store/lpg-backend/src/modules/orders/service.ts)) already loads the bound
`assignment` (`this.inventory.getAssignment(input.assignmentId)`). Resolve the
driver: `assignment.storeAssignmentId` → `inventory.getStoreAssignment(...).userId`
(add a thin accessor if not already exposed to orders). Pass it as
`targetUserId` to `appendStatusHistory` in the assign transaction. All other
`appendStatusHistory` callers leave it null.

- View: add `targetUserId: number | null` + `targetUserName: string | null` to
  `OrderStatusEventView`; the repo join resolves the name.

### Notes

- Keep the existing `reason`/`notes` text as-is (the frontend can still show
  `reason`). The target is structured data, not free text.
- No new endpoint; everything rides the existing `GET /orders/:id` detail.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/orders/schema.ts` — add `targetUserId` to `orderStatusHistory`.
- `src/modules/orders/types.ts` — `changedByName`/`targetUserId`/`targetUserName`
  on `OrderStatusEventView`; `recordedByName` on `OrderPaymentView`.
- `src/modules/orders/repository.ts` — JOIN `users` in `statusHistoryForOrder`
  (actor + target) and `paymentsForOrder` (actor); `appendStatusHistory` accepts
  `targetUserId`.
- `src/modules/orders/service.ts` — resolve + pass the driver as `targetUserId` in
  `assignOrder`; map the new fields in `buildDetail`.
- `src/db/migrations/<n>_*.sql` — add the nullable column.
- `src/modules/orders/__tests__/` — detail name enrichment + assign-target test.

Reference: [[../orders-foundation/backend|orders-foundation backend track]] for the
status-history + payments shapes, and the inventory module for `getStoreAssignment`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-backend] Backend track done. **Name enrichment:** `OrdersRepository.statusHistoryForOrder` now joins `users` twice — `innerJoin` on the actor (`changedBy`, NOT NULL) + `leftJoin` on the new target — returning `changedByName` + `targetUserName`; `paymentsForOrder` `innerJoin`s `users` for `recordedByName`. New enriched return types `OrderStatusHistoryRowEnriched` / `OrderPaymentRowEnriched`; views (`OrderStatusEventView`/`OrderPaymentView`) gained `changedByName`/`recordedByName` (+ `targetUserId`/`targetUserName`); `buildDetail` maps them. Role-safe — rides the existing `GET /orders/:id`, no admin-only `/users` dependency. **Assign target:** new nullable `order_status_history.target_user_id` (FK users) via migration **0010_order_status_target_user** (generated + applied to dev DB). `assignOrder` resolves the driver through a new `InventoryService.getStoreAssignment(storeAssignmentId)` (delegates to the repo's `StoreAssignmentRef`) and passes it as `targetUserId` to `appendStatusHistory`; all other transitions leave it null. **Tests:** fake repo gained a `userNames` map + `seedUserName`, enriched the two read methods, stores `targetUserId`; new `(timeline)` test asserts actor names on every event, the `assigned` event's operator→driver target, null target on non-assign events, and payment `recordedByName`. Gates green: typecheck + lint + **81 tests** (was 80) + build. Frontend track remains.