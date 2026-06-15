---
project: lpg-store
domain: specs
type: spec-track
spec: orders-multi-location
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-14
---

# Orders Multi-Location — lpg-backend track

Shared spec: [[index]]

## Technical Notes

### Data model (extend `orders/schema.ts`)

- **`orders.storeId`** int FK → `stores.id` **NOT NULL** — the order's owning
  branch (queue). Set at creation. Index `(storeId, status)` for the scoped queue.
- **`orders.ownerId`** int? FK → `users.id` — the operator currently managing the
  order; set = `createdBy` at creation, nullable after a transfer. `createdBy`
  stays the immutable audit author (do not repurpose it).
- Migration `<n>_orders_multi_location.sql`: add both columns + index. Back-fill
  existing rows (lean: the single seeded `stores.id`, `ownerId` null) — see Open
  Questions. `storeId` lands NOT NULL via add-nullable → back-fill → set-not-null.

### Store-scoping (reuse `store_assignments`)

`store_assignments` already links users↔stores (its comment names
"driver/operator"). Add an orders-repo read `storeIdsForUser(userId): number[]`
(active rows). `GET /orders` list filters `storeId IN (...)`; `GET /orders/:id`
returns **404** when the order's `storeId` is outside the caller's set.
Admin/developer bypass scoping (global). Delivery stays scoped to assigned orders
(as `orders-foundation` already does in `listOrders`). Keep the single-query,
no-N+1 shape.

### Ownership gating

`assign`, `cancel`, `transfer` require `caller.id === order.ownerId` OR
admin/developer → else `ForbiddenError` (403). Assigning an order with
`ownerId IS NULL` in the caller's branch **auto-claims** (`ownerId = caller`).
Field actions (`dispatch`/`deliver`/`fail`) are the driver's, not owner-gated.

### Transfer handoff — `POST /:id/transfer { storeId }`

Pre-dispatch only (`pending|assigned|failed`). Validate target store exists; set
`storeId = target`, `assignmentId = null`, status → `pending`, `ownerId = null`;
append `order_status_history` (reason `transferido a otra tienda`). Owner-or-admin
gated. The receiving branch's operators pick it up; their `assign` auto-claims.

### Atomic transitions (the real race fix)

Replace the read-status → check → write pattern with a **conditional update**
inside the unit-of-work:

```ts
// orders/repository.ts (sketch)
async transition(orderId, fromStatus, toStatus, patch?): Promise<Order | null> {
  const rows = await this.db.update(orders)
    .set({ status: toStatus, ...patch, updatedAt: new Date() })
    .where(and(eq(orders.id, orderId), eq(orders.status, fromStatus)))
    .returning();
  return rows[0] ?? null; // null → someone else already moved it
}
```

The service calls `transition(...)`; a `null` result throws `ConflictError`
("el pedido ya fue actualizado", 409). Apply to assign/dispatch/deliver/fail/
cancel/transfer. For **deliver**, run the conditional flip to `delivered`
(guarded on `in_transit`) as the FIRST write in the existing delivery
`withTransaction`, then the inventory/payment/debt writes — so a losing concurrent
deliver aborts the whole transaction. Ownership/assignment guards compose into the
same `WHERE` (e.g. assign also guards `assignment_id IS NULL OR status='failed'`
as needed). All of `orders-foundation`'s tests must stay green.

### Create

`POST /orders` sets `storeId` (operator's active store; admin may pass an explicit
`storeId`) and `ownerId = createdBy`. Validate the operator belongs to the chosen
store (admin exempt) → 403 otherwise.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/orders/schema.ts` — `storeId`/`ownerId` columns + `(storeId,status)` index
- `src/modules/orders/types.ts` — optional `storeId` on create; `transferOrderSchema`
- `src/modules/orders/repository.ts` — `storeIdsForUser`, `transition(...)`, store-filtered `listOrders`/`getOrderRow`, transfer write
- `src/modules/orders/service.ts` — scoping, owner gating, `transferOrder`, atomic transitions, create sets `storeId`/`ownerId`
- `src/modules/orders/routes.ts` — `POST /:id/transfer`; thread caller (id+role+stores) into list/detail; gating
- `src/modules/orders/__tests__/{helpers,orders.test.ts}` — seed a 2nd store + operator `store_assignments`; scoping/ownership/transfer/concurrent-assign tests
- `src/db/migrations/<n>_orders_multi_location.sql` — add columns + index + back-fill (`npm run db:generate -- orders_multi_location`)

Context (read; do not needlessly modify):

- `src/modules/catalog/schema.ts` — `stores`, `store_assignments` (the user↔store link)
- `src/modules/auth/middleware.ts` — `req.user` (id/role); `requireRole`
- `src/modules/orders/index.ts` — the `withTransaction` unit-of-work to reuse
- `src/lib/{errors,date}.ts`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

- [2026-06-14] [lpg-backend] **Backend track complete — all 8 backend acceptance criteria met.**
  - **Schema** (`orders/schema.ts`): `storeId` (NOT NULL FK→stores) + `ownerId` (nullable FK→users) + index `(storeId,status)`. Migration `0008_orders_multi_location.sql` lands `store_id` NOT NULL via add-nullable → back-fill `MIN(stores.id)` → set-not-null; `owner_id` nullable. Applied to dev DB (verified `store_id` NOT NULL, `owner_id` nullable, index present).
  - **Repository**: `storeIdsForUser` (active `store_assignments`), `storeExists`, and `transition(id, fromStatus, toStatus, patch?)` — a compare-and-swap conditional `UPDATE … WHERE id=? AND status=fromStatus RETURNING` (0 rows → null → 409). `listOrders` gained a `storeIds` filter (empty → no rows). Removed `setStatus`/`setAssignment` (fully replaced by `transition`).
  - **Service**: `createOrder` resolves `storeId` (operator: single store defaults, multi requires explicit ∈ set, 0 → 403; admin must pass an existing store) and sets `ownerId = caller`. `listOrders` scopes operators by `storeIds` (delivery by `assignmentIds`, admin/dev global). `getOrder` + `recordPayment` enforce `assertVisible` (out-of-branch → 404, no existence leak). `assign`/`cancel`/`transfer` owner-gated via `requireOwner` (assign `allowUnowned` → auto-claims `ownerId = caller`). Every lifecycle move now goes through `transition` (409 on lost-update). `deliver` runs the conditional flip to `delivered` as the **first** write in the tx, so a losing concurrent deliver aborts before any ledger/payment/debt write.
  - **New endpoint**: `POST /:id/transfer` (pre-dispatch only; validates target store; re-homes `storeId`, clears `assignmentId`, reverts to `pending`, releases `ownerId`; history reason `transferido a otra tienda`). `toSummary` now exposes `storeId`/`ownerId`.
  - **Decisions honored** (focus session): payments branch-scoped, not owner-gated; cross-branch read = 404; `403` (ownership/scope pre-check) vs `409` (CAS lost-update) kept as distinct error classes.
  - **Tests**: extended the fake repo (`transition`/`storeIdsForUser`/`storeExists`/store-assignment seeding/`storeIds`+`assignmentIds` filtering); 4 new tests — (a) operator branch scope + 404 cross-branch detail + admin-global, (b) non-owner assign 403 / owner success / unowned auto-claim, (c) transfer re-home, (d) two concurrent assigns → one 200 / one 409. **58 tests total, all green.** Gates: typecheck + lint + build green.
  - **Independent validation**: all 9 acceptance criteria + 4 design decisions confirmed MET.
  - **Deliberate non-goal**: field actions (`dispatch`/`deliver`/`fail`) stay un-scoped (`anyStaff`), preserving `orders-foundation` behaviour (criterion 8) per the track's "field actions are the driver's" note. Per-driver assignment scoping on those writes is a future hardening.
  - See **ADR-016** (optimistic concurrency via conditional status `UPDATE`).

- [2026-06-14] [lpg-backend] **Follow-on (during the frontend focus session, user-approved): `storeId` query param on `GET /orders`.** Added `storeId` to `listOrdersQuerySchema` and, in `service.listOrders`, intersect it with the caller's visible-store set: global (admin/dev) → narrow to the one store; scoped operator → only if it's already a branch they can see, else no rows (no cross-branch existence leak); not applied to `delivery` (scoped by assignments). The repo already supported a `storeIds` filter, so no repository change. Lets the frontend admin store switcher filter server-side instead of client-side. +1 test (admin `?storeId=` narrows; operator can only intersect its own) → **59 tests total, all green**; typecheck + lint + build green.
