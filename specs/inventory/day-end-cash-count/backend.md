---
project: lpg-store
domain: specs
type: spec-track
spec: day-end-cash-count
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-10
---

# Day-End Cash Count — lpg-backend track

Shared spec: [[index]] · Foundation: [[../inventory-foundation/index]] · Day-end flow: [[../day-handoff/index]] · By-method aggregation reference: [[../../accounting/accounting-registry/index]]

## Technical Notes

The money and debt numbers already exist in the ledger — everything traces back to the assignment's **driver** (`storeAssignments.userId`) and **day** (`inventory_assignments.date`). The work is: a **new payment method (`fise`)**, a **new per-driver-day aggregation read** (existing aggregations are keyed on `storeId`), **new capture** of counted cash at close, **migrations** (enum + discrepancy table + discrepancy flag), and **surfacing** the recap on the operator's consolidation read. No change to how payments/debts are recorded.

### 0. New payment method: `vale FISE`

Add `'fise'` to `paymentMethodEnum` (`orders/schema.ts:33-38`) → drizzle-kit generates `ALTER TYPE order_payment_method ADD VALUE 'fise'`. Name the migration (`npm run db:generate -- add_fise_payment_method`). Postgres note: `ADD VALUE` can't be *used* in the same transaction it's added, but we only add it here (first use is later reads/writes), so the migrator's per-file transaction is fine. FISE is a partial LPG-subsidy tender, accepted anywhere `cash`/`yape`/`plin`/`transfer` are (delivery payment, `recordPayment`, the by-method aggregation) with no other behavior change. Add a Spanish label "Vale FISE" wherever methods are labelled.

### 1. Driver-day money/debt summary read (new)

New read (e.g. `GET /assignments/:id/day-summary`, or fold into the existing `GET /assignments/:id` detail). Returns for the one assignment:

- **Money by method (Option B — collected-today)** — from the assignment, derive `driverId = storeAssignments.userId` and the `date` window. Aggregate `order_payments` grouped by `method` where **`recordedBy = driverId` AND `occurredAt` in that business day** (mirror the SQL shape of `paymentsByMethodForStorePeriod` `orders/repository.ts:329-354`, but key on `recordedBy` + day, **not** `storeId`/period). This deliberately includes payments **servicing a prior debt** (`order_payments.customerId` set) — that cash is physically in the driver's hands. It is the **expected** figure the driver's count reconciles against. Include the new `fise` method.
- **Tanks handed out** — per tank type, from the assignment's `sale`-kind tank transactions (already reconstructed for balances in `toAssignmentView`).
- **Money debt accrued that day** — `customer_debts` rows whose `orderId` → an order on this assignment, `createdAt` within the business day (`customers/schema.ts:37-56`; written by `createCharge` `orders/service.ts:403-413`). Return the **delta accrued** (not the cumulative `customer_debt_balance` view), as **both** a total **and** an itemized list (per customer/order) so the driver can verify the figure.
- **Empty-tank debt accrued that day** — `customer_empty_debts` rows via `refTankTransactionId` → `tank_transactions` (holder/`refOrderId`) → this assignment, `occurredAt` within the day (`inventory/schema.ts:195-210`; written in `recordSale` `inventory/service.ts:393-412`). Again a **delta** (not the `customer_empty_debt_balance` view), returned as a **total plus itemized** per customer/type.

Use `src/lib/date.ts` (`toBusinessDate`/`businessToday`, `America/Lima`) or the fixed `-05:00` SQL window already used by the accounting queries — both valid (Lima has no DST). Anchor the "day" to `inventory_assignments.date`.

### 2. Capture counted cash at close (new + migration)

Extend the close path (`closeAssignment` `inventory/service.ts:964-1050`, `closeSchema` `inventory/types.ts:268-274`) to accept, alongside `finalCounts`:

- `cashCounts: [{ method, counted }]` — the driver's counted money per method.
- `cashNote?: string` — the discrepancy explanation.

At close, compute **expected per method** from part 1 and persist a snapshot to a **new table** (migration) — proposed `assignment_cash_counts(assignment_id, method, expected numeric(10,2), counted numeric(10,2), note, created_at)` — the money analogue of the `reconciliation` tank rows. Validation: if `counted ≠ expected` for any method and `cashNote` is empty → **400** (mirrors the mandatory-tank-count rule). A clean close needs no note. Admin/developer override may close without the note, consistent with the existing tank-close override (`caller: {id, role}` already threaded through `closeAssignment` per ADR-017).

### 3. Discrepancy discoverability (new)

A closed-with-discrepancy day must be **findable by admin/operator** without scanning. Options (pick at `/focus`): a boolean `has_cash_discrepancy` on the assignment set at close, plus a `GET /assignments?hasCashDiscrepancy=true` filter; or derive from the presence of a non-empty `assignment_cash_counts.note`. Mirror the intent of the `cierre tardío` zero-delta annotation (ADR-017) — an attributable, queryable marker seeding the admin "who isn't following the flow" signal. Keep the note attributable (the closing driver is the assignment owner; capture `created_at`).

### 4. Surface the recap on consolidation/verify (extend)

The operator's `closed`-awaiting-carry review (the `carry` path `inventory/service.ts:1056-1099` and the assignment detail / discrepancies reads `getAssignment` `:1102-1105`, `getDiscrepancies` `:1166-1170`) must expose the **same money/debt recap** + the persisted counted-vs-expected + `cashNote`, so the second signature covers money. Likely: include the summary (part 1) and the stored `assignment_cash_counts` in the detail/discrepancies response the operator already fetches.

### Notes

- No change to `order_payments`, payment methods, partial-payment logic, or the debt ledgers — this reads and attests over them.
- Keep `carry` idempotent and the tank `reconstructDay`/hand-back behaviour exactly as in the foundation.
- The `open|closed|carried` enum and the `day-handoff` role split (driver closes, operator carries) are unchanged.
- Add an index only if a lookup proves hot (e.g. `assignment_cash_counts(assignment_id)` comes with the table; a `has_cash_discrepancy` filter may want an index — low volume, likely skip).
- Record an ADR if the discrepancy-record shape / discoverability marker is a decision worth freezing (extends the ADR-017 audit-trail line).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — new day-summary route; extend close payload; optional `?hasCashDiscrepancy` filter on the assignments list.
- `src/modules/inventory/service.ts` — `closeAssignment` captures/validates `cashCounts` + `cashNote` and computes expected; new `daySummary(assignmentId)`; surface recap in the consolidation/detail read.
- `src/modules/inventory/types.ts` — extend `closeSchema` (`cashCounts`, `cashNote`); day-summary response type.
- `src/modules/inventory/repository.ts` — assignment-scoped money-by-method + debt-accrued queries; persist/read `assignment_cash_counts`; discrepancy flag/filter.
- `src/modules/orders/schema.ts` — add `'fise'` to `paymentMethodEnum` (`:33-38`).
- `src/modules/orders/repository.ts` — reference/reuse `paymentsByMethodForStorePeriod` (`:329-354`); add a **`recordedBy`+day**-keyed by-method variant (Option B) here or compose in inventory.
- `src/modules/orders/types.ts` / validation — ensure the payment-method schema accepts `fise`.
- `src/db/migrations/<n>_*.sql` — **two named migrations:** `add_fise_payment_method` (enum `ADD VALUE`) and `day_end_cash_count` (new `assignment_cash_counts` table + `has_cash_discrepancy` flag on assignments).
- `src/lib/date.ts` — reuse for business-day boundaries (no change expected).
- `src/modules/inventory/__tests__/` — day-summary read (money by method + debt delta), counted-vs-expected capture, discrepancy-note-required (400), admin override, discoverability query.

Reference: [[backend|day-handoff backend track]] for the close/carry/override plumbing (`caller: {id, role}`, ADR-017) and [[../../accounting/accounting-registry/index]] for the by-method aggregation shape being re-scoped to the assignment.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-07-10] [lpg-backend] Backend track **done**. Recorded **ADR-021**. All backend acceptance criteria met; gates green (typecheck + biome `check` + **174 tests** (was 169) + build); migration **0017** applied to the dev DB.

- **`fise` payment tender.** Added `'fise'` to `order_payment_method` (`orders/schema.ts`). Migration `0017_day_end_cash_count` emits `ALTER TYPE … ADD VALUE 'fise'` — **applied cleanly on this Postgres, no transaction-block error** (the value is added, not used, in the same migration). Order flows accept it automatically (same enum); the inventory cash-count schema validates against a local `PAYMENT_METHODS` list incl. `fise`.
- **`assignment_cash_counts` table + `has_cash_discrepancy` flag** (both in migration 0017). The cash-count `method` is a **plain `text`** column (NOT the orders enum) to avoid an inventory↔orders schema import cycle — validated at the service/zod layer instead. Table has a `uq_assignment_cash_count (assignment_id, method)` unique index.
- **Day-summary read** — `GET /assignments/:id/day-summary` → `InventoryService.getDaySummary`. Composes: **tanks handed out** (`tanksHandedOutByAssignment`, `-Σ full_delta` of `sale` rows) + **empty-debt accrued** delta (`emptyDebtAccruedByAssignment`, `customer_empty_debts → tank_transactions → tank_holders.assignment_id`, itemized per customer/type) — both inventory-local — plus the **money half** (cash-by-method + money-debt) from the injected resolver. Returns totals **and** itemized lists.
- **Money half = injected orders read port (ADR-012).** New `DriverDayMoneyResolver` port on inventory (`setDriverDayMoneyResolver`, wired in `app.ts` to `OrdersService.driverDayMoney`). Orders adds `paymentsByMethodForDriverDay(driverId, from, to)` (**Option B** — `recorded_by = driver` + `occurred_at` in the Lima `-05:00` day, group by method; includes prior-debt collection) and `debtChargesForAssignment(assignmentId)` (`customer_debts ⋈ orders` by `assignment_id`, resolved to customer names). Unset resolver → zero money (standalone/unit-test safe).
- **Cash attestation at close.** `closeSchema` gained optional `cashCounts` + `cashNote`. `closeAssignment` computes expected per method (resolver) **before the transaction**, so a missing-note rejection aborts before any write; a counted≠expected gap without `cashNote` → **400**; a clean count needs no note; admin override that omits `cashCounts` bypasses (consistent with ADR-017). Persists per-method `expected/counted/note` to `assignment_cash_counts` and sets `has_cash_discrepancy` inside the close transaction.
- **Discoverability.** `has_cash_discrepancy` flag + `GET /assignments?hasCashDiscrepancy=true` filter (threaded through `ListAssignmentsQuery` → `AssignmentFilter` → `listAssignments`). The operator's verify surface reads the same `day-summary` recap (counted-vs-expected + notes), so the second signature covers money.
- **Unchanged:** tank `finalCounts`/`carry`/discrepancy reads and all day-handoff guards untouched; no change to how payments/debts are recorded (only the `fise` value added).
- **Decisions vs. Technical Notes:** single combined migration `0017` (not two) since drizzle-kit emits one file per generate — enum + table + column together, safe because `fise` isn't used in-migration. Empty-debt itemization returns customer **IDs** (frontend resolves names it already holds); money-debt itemization carries resolved **names** (orders `customers` join). The `day-summary` endpoint (not the `carry`/detail response) is the single recap surface for both signatures.
- **Real-DB note:** the four new reads are exercised by the in-memory fakes with a stubbed money resolver; the live SQL mirrors the battle-tested `paymentsByMethodForStorePeriod` and is left to the owner's standing manual smoke (per project convention for `.existing()`-style reads). Frontend track carries the driver recap + cash-count form + button gating + operator verify UI.

[2026-07-10] [lpg-backend] Bug fix (post-completion, owner smoke): **`moneyDebt` in the day-summary double-counted partially-paid orders.** `createCharge` writes the **full order total** to `customer_debts` (the at-delivery payment nets against it only in the customer *balance* view), so `debtChargesForAssignment` was reporting the gross charge as "credit accrued today" — e.g. an order of 60 paid 50 showed **60** instead of the true new credit **10** (double-counting the 50 already under cash). Fixed by **netting each charge by same-day payments against that order**: `debtChargesForAssignment(assignmentId, from, to)` now subtracts `SUM(order_payments.amount WHERE order_id = charge.order_id AND occurred_at in the business day)`; `driverDayMoney` drops charges that net to ≤ 0. This makes the read actually match the AC ("the delta, not the cumulative balance"). Real-DB SQL (the orders fake returns `[]`; day-summary unit tests inject a money resolver), covered by owner smoke. `npm run typecheck` + biome + **174 tests** + build green.

[2026-07-10] [lpg-backend] Fixed a runtime crash in the above netting change: the first cut passed raw `Date` objects into a `sql` correlated subquery, which postgres-js can't serialize (`day-summary` 500'd with "string argument … Received an instance of Date"). Reworked to a **grouped `paid_in_day` subquery** whose window uses drizzle's `gte`/`lte` operators (correct Date→timestamp mapping for the `withTimezone` column), LEFT-JOINed to the charges — same `credit = total − paid-today` result, no raw Date params. typecheck + biome + 174 tests green; `day-summary` verified live.

[2026-07-10] [lpg-backend] Added **`storeId` + `storeName`** to `DaySummary` so the driver's `/mi-dia` names its location from its own caller-scoped read (no catalog dependency). `getDaySummary` already loaded the `StoreAssignmentRef` (has `storeId`); resolved the name via the existing `storeNamesByIds` (folded into the summary's `Promise.all`). typecheck + biome + 174 tests green.

[2026-07-10] [lpg-backend] Added a **per-order breakdown** to the day-summary (owner request — one unified money+tanks view instead of cash-by-method + tanks-by-type separately). `DaySummary` gained `orders: DayOrder[]` — each delivered order on the assignment with `{ total, paid, credit, tanks:[{tankTypeId,qty}] }`. Composed in the injected `driverDayMoney` resolver from two new orders-repo reads: `deliveredOrdersForAssignment` (orders + `paidAggSubquery`, status=delivered) and `tankLinesForAssignment` (`order_items` lineType=tank, grouped qty per type); `credit = max(0, total − paid)`. `DriverDayMoney.orders` is optional (unset resolver → `[]`). No schema change. typecheck + biome + 174 tests green.

[2026-07-10] [lpg-backend] Added **per-order payment methods** to the day-summary orders: `DayOrder.payments: { method, amount }[]` (money collected on each delivered order, grouped by method — partials may split it). New repo read `paymentLinesForAssignment` (`order_payments ⋈ orders`, status=delivered, grouped by order+method); composed into `driverDayMoney`'s per-order list. (`OrderPaymentLineRow` — distinct from the pre-existing accounting `PaymentLineRow`.) typecheck + biome + 174 tests green.

[2026-07-10] [lpg-backend] Added **per-line cost** to the day-summary order tanks: `DayOrder.tanks[].lineTotal` (Σ `order_items.line_total` per type, from `tankLinesForAssignment`) so the recap can itemize each order line's money. typecheck + biome + 174 tests green.

[2026-07-10] [lpg-backend] Reviewed the day-summary composition for scalability (owner concern re: "multiple loops"). Confirmed it's already **O(n) linear** — no N+1, no nested loops, no filter-in-map; each of the 5 reads is a single indexed query (`assignment_id` / `recorded_by`+`occurred_at`) run in parallel, and the data is bounded per driver-day. DRYed the two order-grouping loops into a small `groupByOrder<T,V>` helper (still one O(n) pass each, drops a redundant `Map.set` per row). No behavior change; typecheck + biome + 174 tests green.
