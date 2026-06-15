---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../auth/auth-foundation/index]]"
  - "[[../../stores/stores-and-catalog/index]]"
  - "[[../../inventory/inventory-foundation/index]]"
  - "[[../../customers/customers-crud/index]]"
last-updated: 2026-06-10
---

# Spec: Orders Foundation (the money-recording workflow)

## Problem Statement

Orders is **where the business records its money**. Everything shipped so far —
auth, catalog, the ledger-first inventory module, the customer registry — is
scaffolding for this moment: a customer calls, the operator takes down what they
want and at what price, a driver delivers it, and the system records **what was
sold, to whom, for how much, whether they paid, and whether they returned their
empties**. There is no `orders` module in v2 yet (`src/modules/orders/` does not
exist).

Two modules are already built **anticipating orders as their consumer/writer**:

1. **Inventory is ready to record the sale.** `inventory.recordSale(...)` already
   writes the `tank_transactions` row *and* the balancing `customer_empty_debts`
   row atomically (E1–E5), and `tank_transactions.ref_order_id` already exists
   (indexed) as a soft link back to an order. Orders does **not** re-implement
   any of this — at delivery it calls into inventory.
2. **Customers is waiting for its debt writer.** `customer_debts` + the
   `customer_debt_balance` view exist but **read 0 until orders writes rows**;
   `findById` is the documented seam to validate a customer exists.

The product's value is **traceability, not accounting** — so orders is a faithful
ledger of each sale, not a full invoicing/finance engine (invoices, the
`fulfilled` state, and daily cash reconciliation were deliberately dropped from
v2). v1's order module was the worst offender for bloat (a 998-LOC
`PgOrderWorkflowRepository` mixing CRUD + analytics + workflow + querying; a
reservations table with a 24h `expires_at` **no job ever expired**; manually
keyed prices that drifted; a `tank_returned` flag the sale logic ignored). v2
must record the same business reality with a fraction of the surface by leaning
on the modules above.

## Proposed Solution

A slim `orders` vertical module on the backend plus an `orders` frontend module,
composed with the existing `inventory` and `customers` services per **ADR-012**
(injected as `deps`, called inside one `db.transaction()`; no event bus).

**Lifecycle — five states** (down from v1's seven; no `fulfilled`, since invoices
are out of scope):

```
pending ──assign──> assigned ──dispatch──> in_transit ──deliver──> delivered
   │                   │                       │
   └──> cancelled  <───┘                       └──> failed ──(re-assign)──> assigned
```

- **pending** — operator created it from the call (customer + lines + price). No
  inventory effect.
- **assigned** — routed to a driver's open inventory assignment (the bank-teller
  queue, ADR-013). Optional read-time oversell check against the driver's current
  balance. **No ledger movement** — there are **no reservations** (decided
  2026-06-08; see Open Questions / the inventory spec's deferral).
- **in_transit** — driver is out with it.
- **delivered** — *the money moment.* In one `db.transaction()`: per **tank**
  line → `inventory.recordSale(assignmentId, {…, orderId})`; per **item** line →
  `inventory.recordItemSale(...)` (**new — added by this spec**); record any
  payment(s) into the new `order_payments` ledger; if not fully paid, write the
  monetary `customer_debts` charge; set `paymentStatus`; append `order_status_history`.
- **failed** — delivery didn't happen; nothing was written pre-delivery, so
  nothing to reverse; can be re-assigned.
- **cancelled** — terminal before delivery; nothing to reverse.

**Pricing:** each line's `unitPrice` **pre-fills from the catalog `sellPrice`**
(tank type / item) and is **operator-overridable** per line on the call (decided
2026-06-08). `lineTotal` and order `totalAmount` are derived from the stored line
prices.

**Payments & monetary debt (partial payments supported, decided 2026-06-08):**

- **New `order_payments` ledger** (append-only): `orderId`, `customerId`,
  `amount > 0`, `method` (cash/yape/plin/transfer), `occurredAt`, `recordedBy`.
  Payments can be recorded **at delivery and afterward** (customer pays off a debt
  later).
- `customer_debts` keeps **charge rows** (one per credit delivery, `amount` =
  total owed). The customer monetary balance is **redefined to net payments**:
  `outstanding(customer) = SUM(customer_debts.amount) − SUM(order_payments.amount)`.
  The binary `isPaid` becomes a derived convenience (debt fully covered), not the
  source of truth. This mirrors the empty-debt signed ledger (ADR-010) and the
  balance-view pattern (ADR-007). **This spec revises the customers module's
  `customer_debt_balance` view + detail read path accordingly** (customers-crud
  explicitly named orders as the writer; the balance has read 0 until now).

**Empty-tank debt** is *not* re-modelled here — it falls out of `recordSale`
(E1/E3/E5) on the customer-empty-debt ledger inventory already owns.

**Two inventory prerequisites this spec carries** (decided 2026-06-08 — in this
spec's backend track, since they have no consumer outside orders):

1. **`inventory.recordItemSale(...)`** — there is no path to sell accessory
   *items* today (inventory has tank `recordSale`/`recordReturn` and item
   *purchase* only). Items on an order need an item-sale ledger write.
2. **External-transaction seam** — `recordSale` currently opens its *own*
   `db.transaction()`, so orders cannot nest it to make the whole delivery atomic
   (ADR-012). Inventory's write methods must accept an injected transaction/repo
   so the delivery's inventory writes + payment + debt + status change commit
   together.

**Walk-in / no-customer orders (E4)** are supported: `customerId` null → inventory
records the honest delta, no empty-debt, no monetary debt; a walk-in must be paid
(no debt without a customer to attribute it to).

Detailed data model, transaction flow, routes, and file lists live in [[backend]]
and [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `orders` table exists: `id` serial PK; `customerId` int FK → `customers.id` **nullable** (walk-in); optional `customerNameSnapshot`/`customerPhoneSnapshot` for walk-ins; `deliveryAddress` text; `assignmentId` int FK → `inventory_assignments.id` nullable (set at `assigned`); `status` enum (`pending|assigned|in_transit|delivered|failed|cancelled`); `paymentStatus` enum (`unpaid|partial|paid`); `totalAmount` numeric(10,2); `createdBy` FK → `users.id`; `notes?`; timestamps. Drizzle migration in `src/db/migrations/` (no `db:push`); re-exported from `src/db/schema.ts`.
- [ ] `order_items` table exists: `id` serial PK; `orderId` FK → `orders.id`; `lineType` enum (`tank|item`); `tankTypeId?`/`itemId?` (CHECK: exactly one set, matching `lineType`); `qty` int `> 0`; `emptyReceived` int (tank lines; default = qty); `unitPrice` numeric(10,2); `lineTotal` numeric(10,2). Append/replace only while order is `pending`.
- [ ] `order_payments` table exists (append-only, no UPDATE/DELETE in repo): `id` serial PK; `orderId` FK → `orders.id`; `customerId` int? FK → `customers.id`; `amount` numeric(10,2) `> 0`; `method` enum (`cash|yape|plin|transfer`); `occurredAt` timestamptz; `recordedBy` FK → `users.id`. Index `(orderId)` and `(customerId)`.
- [ ] `order_status_history` table exists (ADR-009 — this is the order audit trail): `orderId`, `fromStatus?`, `toStatus`, `changedBy` FK → `users.id`, `reason`, `notes?`, `createdAt`. Append-only.
- [ ] Customer monetary balance redefined to net payments: `customer_debt_balance` (or a new view) returns `SUM(customer_debts.amount) − SUM(order_payments.amount)` per customer (0/absent tolerated). The customers `GET /:id` detail reflects the netted outstanding. The `customer_debts.order_id` soft ref is hardened to a real FK → `orders.id` now that orders exists.
- [ ] `src/modules/orders/` follows [[../../../eng/patterns/module-template]]: only `repository.ts` imports `schema.ts`; types are Zod + `z.infer<>` (no `dtos/`, no `I*` interfaces, no DI container). Exposes `createOrdersModule({ db, inventory, customers, requireAuth, requireRole })`; mounted in `src/app.ts`.
- [ ] Cross-module writes are atomic per **ADR-012**: order delivery commits the inventory ledger row(s), the `order_payments` row(s), any `customer_debts` charge, and the `orders`/`order_status_history` change in **one DB transaction**. No partial commits on failure.
- [ ] **`inventory.recordItemSale(...)`** added: writes one `item_transactions` row (`−qty`) against the driver's assignment item-holder, accepting `orderId`/`customerId` soft-refs; available to orders via the injected service.
- [ ] Inventory write methods accept an **external transaction seam** so orders can compose `recordSale`/`recordItemSale` inside its own `db.transaction()` (no nested-transaction error). Inventory's own routes still work standalone.
- [ ] `POST /api/v1/orders` — create from `{ customerId? | walk-in name/phone, deliveryAddress, items[], notes? }`. Each line's `unitPrice` defaults to the catalog `sellPrice` and may be overridden; `lineTotal`/`totalAmount` computed server-side. Validates customer exists (when `customerId` given) via the injected customers service. Status starts `pending`.
- [ ] `POST /api/v1/orders/:id/assign` — `pending → assigned`; binds an `assignmentId` (a driver's **open** assignment). Optional oversell guard: rejects (409) if the order's tank lines exceed the driver's current available balance. Writes `order_status_history`.
- [ ] `POST /api/v1/orders/:id/dispatch` — `assigned → in_transit`. History row.
- [ ] `POST /api/v1/orders/:id/deliver` — `in_transit → delivered`; the atomic delivery transaction above. Accepts per-line `emptyReceived` (tank exchange) and an optional initial payment. Sets `paymentStatus` from `SUM(payments)` vs `totalAmount`.
- [ ] `POST /api/v1/orders/:id/fail` — `in_transit → failed` with reason; no ledger effect; re-assignable.
- [ ] `POST /api/v1/orders/:id/cancel` — any pre-delivery state → `cancelled` with reason; no ledger effect.
- [ ] `POST /api/v1/orders/:id/payments` — record a payment against an order (usable **after** delivery to pay down debt); appends `order_payments`, recomputes `paymentStatus`, and flips the order's `customer_debts` charge `isPaid` when fully covered. 4xx if it would overpay beyond `totalAmount` (configurable: clamp vs reject — see Open Questions).
- [ ] `GET /api/v1/orders` — list with filters (`status`, `customerId`, `assignmentId`, date range) + basic pagination; rows carry `totalAmount`, `paymentStatus`, `outstanding`.
- [ ] `GET /api/v1/orders/:id` — detail: lines, status history, payments, outstanding balance, linked inventory transactions (via `ref_order_id`).
- [ ] Routes guarded by role via `src/modules/auth/middleware.ts`: operator + admin create/assign/cancel/record-payment; delivery (+ operator/admin) dispatch/deliver/fail; developer bypass. Errors flow through `src/lib/errors.ts`.
- [ ] Order/business dates use `src/lib/date.ts` (Lima business date), never raw UTC.
- [ ] Lifecycle tests (per [[../../../eng/decisions]] guidance — 2–5 integration tests, each a full happy path), covering at least: (a) create → assign → dispatch → deliver **paid in full** (inventory `sale` row written, empty swap E2, no debt); (b) deliver **on credit** with **partial payment**, then a later payment settling the rest (monetary `customer_debts` + `order_payments`, balance nets correctly, `paymentStatus` transitions unpaid→partial→paid); (c) deliver with an **empty short-return** (E1 empty-debt accrues) and a mixed tank+item order; (d) **walk-in** paid order (no customer/debt rows); (e) **cancel** before delivery (no ledger effect).

**Frontend (lpg-frontend-vue):**

- [ ] `src/modules/orders/` vertical module (`types`/`service`/`store`/`routes`/`index` + `views/` + `components/`) mirroring the established slices, wired in `src/main.ts` and the router.
- [ ] **Order entry (operator)** — phone-driven create form: customer search/select by name or phone (reusing the `customers` module) *or* quick walk-in name/phone; add tank/item lines from the `catalog` with `unitPrice` pre-filled and editable; live `totalAmount`. Inline "new customer on the call" path (the `customers` spec deferred this to orders).
- [ ] **Order queue / list** — filter by status; show payment status + outstanding; actions to assign / dispatch / cancel per role.
- [ ] **Assign** — pick a driver's open assignment; surface the oversell warning if the backend flags it.
- [ ] **Delivery (driver view)** — see assigned/in-transit orders; mark dispatch, then deliver (capture per-line `emptyReceived` + optional payment) or fail (reason).
- [ ] **Order detail** — lines, status timeline, payments, outstanding, and "record payment" action; links to the customer.
- [ ] Routes guarded by role (operator/admin for entry+queue; delivery for the delivery view; developer bypass); nav entries added to the relevant roles in `AppLayout`'s `ROLE_NAV`.
- [ ] Manual smoke test of create → assign → dispatch → deliver (paid and on-credit) against the backend.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

> **Porting order:** backend track first (it owns the schema + the inventory
> prerequisites + the API the frontend reads); the frontend track ports right
> after.

## Out of Scope

- **Inventory reservations** — explicitly **not built** (decided 2026-06-08).
  Inventory moves only at delivery; oversell is an optional read-time check, not a
  reservation table. (Both this spec and the inventory spec had deferred the
  question here; the answer is "no reservation ledger.")
- **Invoices / `fulfilled` state / tax computation / daily cash reconciliation** —
  dropped from v2 (out of MVP scope; traceability, not accounting).
- **Prepayment before delivery** — payments are recorded at/after delivery for
  MVP; add a prepay path only if a real need appears.
- **Order amendment after `pending`** — lines are editable only while `pending`;
  changing a dispatched order = cancel + recreate. Revisit if operators need it.
- **Returns / refunds / RMA after delivery** — a delivered sale is corrected via
  the inventory ledger's `adjustment`/`return` paths, not an orders refund flow.
- **Push notifications to the driver on assignment** — dropped per
  [[../../../product/overview]]; reintroduce only with a concrete consumer (the
  ADR-012 event-mechanism carve-out).
- **Customer order analytics** (frequency, preferred method, lifetime value) —
  v1 had these; defer until a dashboard consumer exists.
- **Multi-delivery / split shipments per order** — v1's `order_deliveries`
  many-rows model; single delivery per order for MVP.

## Open Questions
_All resolved at `/focus` time, 2026-06-08:_

- **Overpayment on `POST /:id/payments`:** **REJECT (4xx).** An amount that would
  push `SUM(order_payments) > totalAmount` is refused — overpayment is treated as
  a data-entry error; change/tips aren't tracked. Invariant: paid ≤ total.
- **Oversell guard strictness:** **SOFT WARNING at assign, hard guarantee at
  deliver.** Assign returns a non-blocking shortfall warning (driver's current
  balance < order tank lines). The operator resolves it with a quick **"load more
  tanks"** action that calls the existing `inventory.recordLoad` (ADR-014) to top
  up the driver's open assignment, then proceeds. The real block is at deliver,
  where `recordSale` refuses to go negative (ADR-015). No reservation ledger.
- **`customer_debts.isPaid`/`paidAt`:** **KEEP as derived convenience flags**,
  flipped by the payment endpoint when an order's charge is fully covered. The
  netted balance view (`SUM(debts) − SUM(payments)`) remains the source of truth.
- **Closed-assignment guard at deliver:** **REJECT (409) with a re-assign
  message.** Deliver checks the bound `assignmentId` is still `open`; if it's
  `closed`/`carried`, it returns a clear "re-assign this order to the driver's
  current open assignment" error rather than letting `recordSale` throw its
  lower-level non-open error. (An assignment is a driver-day: `open` = live
  working day where sales are allowed; `closed` = end-of-day reconciled, no new
  sales; `carried` = rolled into the next day.)
