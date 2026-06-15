---
project: lpg-store
domain: specs
type: spec-track
spec: orders-foundation
repo: lpg-backend
kind: backend
track-status: '"done"'
last-updated: '"2026-06-08"'
---

# Orders Foundation — lpg-backend track

Shared spec: [[index]]

## Technical Notes

### Conventions (carry the v2 module pattern)

`serial` int PKs (shared/referenced tables — ADR pattern), snake_case columns,
`numeric(10,2)` money, `timestamptz`. Module layout per
[[../../../eng/patterns/module-template]]:
`src/modules/orders/{index,routes,service,repository,schema,types}.ts` +
`__tests__/orders.test.ts`. Only `repository.ts` imports `schema.ts`. Types are
Zod + `z.infer<>` — no `dtos/`, no `I*` interfaces, no DI container. Thin routes:
validate → service → respond; errors via `src/lib/errors.ts`.

### Cross-module composition (ADR-012) — the central design constraint

The orders **service** receives the **inventory** and **customers** services as
injected `deps` (factory `createOrdersModule({ db, inventory, customers, requireAuth, requireRole })`,
wired in `src/app.ts`). Delivery must be **atomic across modules in one
`db.transaction()`**. Today this is blocked because
`inventory.service.recordSale` opens its **own** `repo.transaction(...)`
internally — you cannot nest it.

**Required inventory refactor (this track):** give inventory's write methods an
**external-transaction seam**. Recommended shape — make each write method
accept an optional transactional repo/handle, e.g.:

```ts
// inventory/service.ts (sketch)
async recordSale(assignmentId, input, userId, tx?: IInventoryRepository): Promise<TankTransaction> {
  const run = (r: IInventoryRepository) => { /* existing body, using r */ };
  return tx ? run(tx) : this.repo.transaction(run);   // standalone keeps its own tx
}
```

Then orders owns the outer transaction and threads the same handle to every
inventory call + its own writes:

```ts
// orders/service.ts deliver() (sketch)
return this.db.transaction(async (dbtx) => {
  const invRepo = this.inventory.repoFor(dbtx);          // inventory exposes a repo-binder
  for (const line of tankLines)
    await this.inventory.recordSale(assignmentId, {…, orderId, customerId}, userId, invRepo);
  for (const line of itemLines)
    await this.inventory.recordItemSale(assignmentId, {…, orderId, customerId}, userId, invRepo);
  if (initialPayment) await ordersRepo(dbtx).insertPayment({…});
  if (outstanding > 0 && customerId) await ordersRepo(dbtx).insertCustomerDebt({…});  // writes customers.customer_debts
  await ordersRepo(dbtx).appendStatusHistory({…});
  await ordersRepo(dbtx).setStatus(orderId, 'delivered');
});
```

Exact seam (method param vs a `repoFor(tx)` binder vs exposing
`inventory.repo.transaction`) to be finalized at `/focus` time — keep inventory's
standalone routes working unchanged. **Do not** reintroduce a strategy/event bus
(ADR-006/012).

### `inventory.recordItemSale` (new, this track)

Mirror `recordSale` for the **item** ledger: locate/create the assignment
item-holder for `itemId`, write one `item_transactions` row with `−qty`,
`kind='sale'`, `refOrderId`/`refCustomerId` soft-refs. Items have **no
empty-debt** dimension — no `customer_empty_debts` write. Add to the inventory
service's public surface (and a route if a standalone item-sale UI is ever
wanted — not required by this spec). Keep `item_transactions`/`item_balance`
shapes unchanged (append a `sale` arm to the item `kindToDelta`).

### Data model (new tables in `orders/schema.ts`)

- **`orders`** — `id` serial PK; `customerId` int? FK → `customers.id` (null =
  walk-in); `customerNameSnapshot`/`customerPhoneSnapshot` varchar? (walk-in
  fallback); `deliveryAddress` text; `assignmentId` int? FK →
  `inventory_assignments.id` (set at `assigned`); `status`
  `orderStatusEnum(pending|assigned|in_transit|delivered|failed|cancelled)`;
  `paymentStatus` `paymentStatusEnum(unpaid|partial|paid)`; `totalAmount`
  numeric(10,2); `createdBy` FK → `users.id`; `notes` text?; `createdAt`/`updatedAt`.
  Index `(status)`, `(customerId)`, `(assignmentId)`.
- **`order_items`** — `id` serial PK; `orderId` FK → `orders.id`; `lineType`
  `lineTypeEnum(tank|item)`; `tankTypeId` int? FK → `tank_types.id`; `itemId` int?
  FK → `inventory_items.id`; CHECK exactly one of tank/item set and matching
  `lineType`; `qty` int `> 0`; `emptyReceived` int default 0 (tank lines —
  exchange count; ≤ qty); `unitPrice` numeric(10,2); `lineTotal` numeric(10,2).
  Index `(orderId)`.
- **`order_payments`** (append-only) — `id` serial PK; `orderId` FK → `orders.id`;
  `customerId` int? FK → `customers.id`; `amount` numeric(10,2) CHECK `> 0`;
  `method` `paymentMethodEnum(cash|yape|plin|transfer)`; `occurredAt` timestamptz
  default now; `recordedBy` FK → `users.id`. Index `(orderId)`, `(customerId)`.
  **No UPDATE/DELETE in the repo.**
- **`order_status_history`** (append-only, ADR-009 — orders' audit trail) — `id`
  serial PK; `orderId` FK → `orders.id`; `fromStatus` enum?; `toStatus` enum
  notNull; `changedBy` FK → `users.id`; `reason` varchar notNull; `notes` text?;
  `createdAt` timestamptz. Index `(orderId, createdAt)`.

### Changes to existing modules

- **`customers` module** (revise — orders is now the debt writer):
  - Harden `customer_debts.order_id` to a real FK → `orders.id` (was a soft int
    ref; mirror how customers hardened inventory's soft `customerId`).
  - **Redefine the monetary balance to net payments:** `customer_debt_balance`
    (or a new `customer_balance` view) = `SUM(customer_debts.amount) −
    SUM(order_payments.amount)` per customer. Update the customers `GET /:id`
    detail + the list-row `outstandingBalance` aggregate to read the netted value
    (keep the single-query, no-N+1 shape it already has).
  - Add a debt-write seam usable inside an external transaction (orders inserts
    the charge row on credit delivery). Either a `createDebt`/`settleDebt` on the
    customers service, or orders writes `customer_debts` via its own repo using
    the re-exported schema — finalize at `/focus` (lean: a thin customers-service
    method so the table stays owned by its module).
  - Optional: keep `isPaid`/`paidAt` as derived convenience flags flipped when an
    order's payments fully cover its charge (the view stays source of truth).
- **`inventory` module** — the `recordItemSale` addition + the external-tx seam
  (above). No ledger schema changes; `ref_order_id` already exists.
- **`src/db/schema.ts`** — re-export `../modules/orders/schema`.
- **`src/app.ts`** — mount `createOrdersModule({ db, inventory: inventory.service, customers: customers.service, requireAuth, requireRole })`. This is the **first** cross-module service injection in v2 — establishes the ADR-012 wiring pattern for later specs.
- **`src/db/seed.ts`** — optional: a sample order or two (phone/idempotency-guarded) so the frontend list is non-empty in dev. Not required for acceptance.

### Routes (`/api/v1/orders`)

| Method + path | Role | Effect |
|---|---|---|
| `POST /` | operator, admin | Create (`pending`); catalog-default prices, override-able; validate customer exists. |
| `GET /` | operator, admin | List + filters (`status`, `customerId`, `assignmentId`, date range) + pagination; rows carry `totalAmount`/`paymentStatus`/`outstanding`. |
| `GET /:id` | operator, admin, delivery | Detail: lines, history, payments, outstanding, linked inventory tx. |
| `POST /:id/assign` | operator, admin | `pending → assigned`; bind open `assignmentId`; optional oversell check. |
| `POST /:id/dispatch` | delivery, operator, admin | `assigned → in_transit`. |
| `POST /:id/deliver` | delivery, operator, admin | `in_transit → delivered`; **atomic** inventory + payments + debt + status. |
| `POST /:id/fail` | delivery, operator, admin | `in_transit → failed` (reason); no ledger effect. |
| `POST /:id/cancel` | operator, admin | pre-delivery → `cancelled` (reason); no ledger effect. |
| `POST /:id/payments` | operator, admin | Record a payment (at/after delivery); recompute `paymentStatus`; flip the charge's settled flag when covered. |

All guarded via `src/modules/auth/middleware.ts` (developer bypass). Lima business
dates via `src/lib/date.ts`.

### State machine (enforce in the service, not the DB)

Allowed transitions: `pending → {assigned, cancelled}`; `assigned → {in_transit,
cancelled}`; `in_transit → {delivered, failed}`; `failed → {assigned, cancelled}`;
`delivered`/`cancelled` terminal. Illegal transition → `ConflictError` (409).
Deliver requires a bound **open** assignment (reject if `closed`/`carried`).

### Tests (`__tests__/orders.test.ts`) — lifecycle, not edge enumeration

Each test walks a full path (per [[../../../eng/decisions]]). Substitute the
injected inventory/customers services with the real ones over a test DB (the
in-memory-class substitution pattern is fine where it keeps the test honest).
Cover the five flows listed in the shared Acceptance Criteria — at minimum the
paid-in-full delivery, the credit + partial-then-final payment (balance nets;
`paymentStatus` unpaid→partial→paid), the empty-short-return (E1) mixed
tank+item order, the walk-in paid order, and a pre-delivery cancel.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To create:

- `src/modules/orders/{index,routes,service,repository,schema,types}.ts`
- `src/modules/orders/__tests__/orders.test.ts`
- `src/db/migrations/<n>_orders.sql` (`npm run db:generate -- orders`) — `orders`, `order_items`, `order_payments`, `order_status_history`; the `customer_debts.order_id → orders.id` FK; the revised customer monetary-balance view

To modify:

- `src/db/schema.ts` — re-export `../modules/orders/schema`
- `src/app.ts` — mount orders with injected `inventory.service` + `customers.service` (first ADR-012 cross-module wiring)
- `src/modules/inventory/service.ts` — add `recordItemSale`; add external-tx seam to write methods
- `src/modules/inventory/index.ts` — expose the repo-binder/seam if needed; export `InventoryService` type for orders' deps
- `src/modules/inventory/repository.ts` — item-sale write; ensure `transaction`/repo-binding is reusable from outside
- `src/modules/customers/service.ts` — debt-write seam; netted-balance reads
- `src/modules/customers/repository.ts` — insert `customer_debts`; net `order_payments` in the balance aggregate
- `src/modules/customers/schema.ts` — harden `order_id` FK; revise the balance view definition
- `src/db/seed.ts` — (optional) sample order

Context (read; do not needlessly modify):

- `src/modules/inventory/{service,repository,schema}.ts` — `recordSale`/`recordReturn` shapes, holder model, `kindToDelta`, `tank_transactions.ref_order_id`
- `src/modules/customers/{schema,service,repository}.ts` — `customers`, `customer_debts`, balance view, `findById` seam
- `src/modules/catalog/schema.ts` — `tank_types`/`inventory_items` `sellPrice` (pricing defaults), `store_assignments`
- `src/modules/auth/middleware.ts` — `requireAuth`/`requireRole`
- `src/lib/{errors,date}.ts` — error classes; Lima business date
- `src/db/client.ts` — `Database` type / `db.transaction`

v1 reference (shapes + business rules only; **do not** port reservations, the
998-LOC `PgOrderWorkflowRepository`, invoices, the `fulfilled` state, or the
strategy pattern):

- `legacy/src/db/schemas/orders/{orders,order-items,order-status-history,order-deliveries}.ts`
- `legacy/src/services/orders/{OrderService,OrderWorkflowService}.ts`
- `legacy/src/routes/orders/*.ts`
- `legacy/docs/orders/PRD - Orders.md`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->


[2026-06-08] [lpg-backend] **Backend track complete.** Resolved all 4 open
questions at `/focus` (overpayment→reject 4xx; oversell→soft warning + reuse
`inventory.recordLoad`; keep `isPaid`/`paidAt` derived; deliver rejects non-open
assignment with a re-assign 409). Shipped the `orders` vertical module
(`schema`/`types`/`repository`/`service`/`routes`/`index` + `__tests__`):
4 tables (`orders`, `order_items`, `order_payments`, `order_status_history`) with
CHECKs (line target exactly-one, qty>0, emptyReceived≤qty, payment amount>0) in
migration `0006_orders.sql`. 9 routes with per-route role guards (office =
operator/admin for create/list/assign/cancel/payments; field =
delivery/operator/admin for dispatch/deliver/fail; detail = any staff;
developer bypass). State machine in the service (illegal → 409).

- **ADR-012 cross-module composition (first in v2):** an injected unit-of-work
  `withTransaction` opens ONE `db.transaction` binding the orders + inventory +
  customers repos; `deliver` writes the inventory `recordSale`/`recordItemSale`
  rows, `order_payments`, the `customer_debts` charge, and the order status/history
  all atomically. Wired in `app.ts` with `inventory.service` + `customers.service`.
  Tests inject a fake `withTransaction` returning the three shared in-memory fakes,
  so the orchestration is verified without a test Postgres (matches the module's
  fake-repo test pattern).
- **Inventory seam:** `recordSale`/`recordItemSale` gained an optional
  `tx?: IInventoryRepository` (runs on the caller's tx when passed; standalone
  routes unchanged). New `recordItemSale` (reuses the existing `kindToItemDelta`
  `sale` arm — no enum/delta change) + `recordSale` state-check moved onto the
  supplied repo via `requireStateOn`. Added `tank/itemTransactionsByOrder` reads
  for the detail's `ref_order_id` traceability link.
- **Customers seam:** `createCharge`/`setOrderChargesPaid`/`getOutstandingBalance`
  on the repo; tx-aware `createCharge`/`settleOrderCharges` + `findById` on the
  service; `getCustomer` headline balance now reads the **netted** view. The
  `customer_debt_balance` view was redefined (in the orders migration) to
  `SUM(customer_debts.amount) − SUM(order_payments.amount WHERE customer_id NOT
  NULL)`; the list aggregate auto-nets (no change). `customer_debts.order_id`
  hardened to a real FK (SQL-only, to avoid a customers↔orders import cycle).
- **Payment/debt model:** paid-in-full or walk-in → no charge, payment recorded
  with `customer_id = NULL` (cash sale, excluded from the netted balance); on
  credit → charge = full total, payments carry `customer_id`, `is_paid` flips when
  covered. `order.paymentStatus` is per-order (Σpayments vs total); the customer
  balance is per-customer (netted view). Overpayment rejected at BOTH `/payments`
  and `/deliver` (4xx). Walk-in with any outstanding → 400.
- **Tests:** 6 lifecycle tests (a paid-in-full + E2 swap + no debt; b credit +
  partial→final settling with unpaid→partial→paid + overpay 409; c E1 short-return
  + mixed tank+item; d walk-in paid + unpaid-walk-in 400; e cancel pre-delivery).
- **Gates:** typecheck + lint + build green; **54 tests pass** (6 new). Migration
  applied to the dev DB and **smoke-verified**: 4 order tables present, the
  `customer_debts.order_id` FK present, and the netted `customer_debt_balance`
  view confirmed (100 charge − 60 = 40, − 40 = 0 in a rolled-back tx). Independent
  validation agent confirmed all 22 backend criteria met, no correctness bugs.
- **Note for the frontend track:** API is live at `/api/v1/orders`. `assign`
  returns `{ order, warnings }` where `warnings: {tankTypeId, required, available,
  shortfall}[]` — surface the shortfall and offer a "load more tanks" button that
  calls the existing `POST /api/v1/inventory/assignments/:id/loads`. `deliver`
  takes `{ emptiesReceived?: [{orderItemId, emptyReceived}], payment?: {amount,
  method} }`. Money fields are numbers in JSON.


[2026-06-10] [lpg-backend] Follow-on additions made during the frontend track
(user-approved): (1) **opened `GET /orders` to the delivery role**, scoped to the
driver's own assignments — route guard `office`→`anyStaff`, `listOrders(query,
caller)` derives the assignment set via `inventory.listAssignments({userId})`,
repo `ListOrdersFilter.assignmentIds` (inArray; empty → no rows); also
consolidated the duplicate `field`/`anyStaff` role guards into `anyStaff`.
(2) **nullable `orders.payment_method`** (the method agreed at registry) —
migration `0007_orders_payment_method.sql`, optional in `createOrderSchema`,
persisted + returned on `OrderSummary`/`OrderDetail`. Gates: typecheck + lint +
54 tests + build green; migration applied to the dev DB.

[2026-06-08] [lpg-backend] **Fix: `POST /orders` create returned 404 "Pedido no
encontrado" against a real DB.** `createOrder` built the response detail *inside*
the `withTransaction` callback, and `buildDetail` reads through the pool repo
(`this.repo`), which can't see the uncommitted order on a separate connection →
read-back returned null → NotFoundError (and the tx rolled back, so nothing
persisted). Moved the detail build to *after* the transaction commits. The fake
unit-of-work shares one repo instance with `this.repo`, so the in-memory tests
can't model connection isolation and didn't expose this — a known limitation of
the fake-repo pattern for cross-connection read-your-writes.
