---
project: lpg-store
domain: specs
type: spec-track
spec: accounting-registry
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-15
---

# Accounting Registry — lpg-backend track

Shared spec: [[index]] · Ingress source: [[../../orders/orders-foundation/index]] · Egress source: [[../../inventory/inventory-foundation/index]]

## Technical Notes

A new `accounting` vertical module that **reads** the existing money ledgers and **owns** only the register + manual-entry tables. No changes to `orders` or `inventory` write paths.

### New module: `src/modules/accounting/`

Standard vertical layout (`{routes,service,repository,schema,types,index}.ts`); `index.ts` exposes `createAccountingModule({ db, deps })` and is composed in `src/app.ts` **after** orders + inventory (it depends on their services for reads). Follow [[../../../eng/patterns/module-template]] and the `orders` module's factory (`createOrdersModule`) as the closest precedent — orders already composes `inventory` + `customers` repos inside one `withTransaction` (ADR-012); accounting does the read-side equivalent.

### Tables (migration `0011_accounting.sql`)

- **`accounting_registries`** — serial PK; `store_id` int NOT NULL (FK → `stores`); `period_start` / `period_end` (store as `date`, Lima business dates from `src/lib/date.ts` — never raw UTC); `status` enum `'open' | 'closed'` default `'open'`; `opened_by` int NOT NULL (FK → `users`); `closed_by` int? / `closed_at` timestamptz? (set on close); `notes` text?; snapshot columns (see Close). Integer PK — referenced by entries (per the project's int-PK-for-shared-tables rule).
  - **No overlap per store**: opening a register whose `[period_start, period_end]` intersects an existing register for the same `store_id` → `ConflictError` (409). Enforce in the service with a range check; optionally back with an exclusion constraint if convenient.
- **`accounting_entries`** — serial PK; `registry_id` int NOT NULL (FK → `accounting_registries`, the int PK); `direction` enum `'ingress' | 'egress'`; `amount` `numeric(10,2)` NOT NULL CHECK > 0; `category` varchar; `method` `order_payment_method`? (reuse the orders enum for ingress lines paid by a channel; nullable for egress/expenses); `note` text?; `occurred_at` timestamptz default now(); `recorded_by` int NOT NULL (FK → `users`). Insert/delete **only while the parent registry is `open`** (service guard → 4xx on a `closed` parent).

### Aggregation (the registry detail)

For an `open` register, compute live; for a `closed` one, serve the frozen snapshot. Live computation, scoped to `store_id` + `[period_start, period_end]` (business-date compare):

- **Ingress** — `SUM(order_payments.amount)` grouped by `order_payments.method`, restricted to payments whose order is `delivered` **and** whose owning `orders.store_id` = the register's store, and whose `occurred_at` business date ∈ the window. (Membership keyed by the *payment's* business date — see [[index]] Open Questions.) Reuse the orders module via composition rather than re-querying `order_payments` from accounting if a thin orders read method (e.g. `paymentsForStorePeriod(storeId, from, to)`) keeps the join logic in orders.
- **Egress** — provider purchases on the store's `location` holder: `tank_transactions`/`item_transactions` with `kind='purchase'`, valued `holder.purchase_price × qty` (`qty` = `full_delta` for tanks, `delta` for items), **plus** `purchase_surcharges.amount` for those tank transactions. Expose via an inventory read method (e.g. `purchasesForStorePeriod(storeId, from, to)`) so the valuation/join stays in the inventory module.
- **Manual** — the registry's `accounting_entries`, summed by direction.
- **Net** = total ingress − total egress (operational + manual).

### Close = freeze

`POST /accounting/registries/:id/close` (admin/developer): set `status='closed'`, `closed_by`, `closed_at`, and **persist the aggregated totals** onto the register (denormalized columns or a JSON snapshot column — decide at build; see Open Questions). After close: detail reads the snapshot; entry insert/delete → 4xx; re-close → 409/no-op.

### Scoping & permissions

Mirror the orders/inventory caller-scoping: admin/developer global; scoped users via `storeIdsForUser` (already used by `listAssignmentsForCaller` / orders queue). A `storeId` filter is **intersected** with the caller's scope (tampered id → 0 rows, never widening). Cross-store register/entry access → 404/403. Reuse `createRequireRole` from auth middleware (developer auto-passes). **Role split (resolved):** admin/dev manage every store; **operators** may open a register + add/remove **manual entries** + view detail for their own `storeIdsForUser` store(s); **`close` is admin/dev only** (route-level `requireRole('admin')` + the operator open/entry paths add a service-level own-store check). Membership/aggregation windows compare on the *payment's* business date for ingress.

### Reuse / don't reinvent

- Lima business dates: **`src/lib/date.ts`** (`businessToday`, `toBusinessDate`) — every window boundary and membership compare. Never raw UTC.
- Errors: `src/lib/errors.ts` (`ConflictError` 409 for overlap, `ForbiddenError`/`NotFoundError` for scope, `ValidationError` 400).
- Money: `numeric(10,2)` everywhere; round consistently with the orders `round2` helper pattern.
- Payment-method enum: reuse the orders `order_payment_method` (cash/yape/plin/transfer) — do not define a second.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To create:

- `src/modules/accounting/schema.ts` — `accounting_registries` + `accounting_entries` tables, status/direction enums.
- `src/modules/accounting/types.ts` — Zod schemas (open register, add entry, close) + response types (registry summary, detail with ingress-by-method/egress/manual/net breakdown).
- `src/modules/accounting/repository.ts` — register + entry CRUD; overlap check; snapshot persistence; the only file importing `schema.ts`.
- `src/modules/accounting/service.ts` — lifecycle (open/close, overlap 409, scope), aggregation composing orders + inventory reads, entry guards (open-only).
- `src/modules/accounting/routes.ts` — role-guarded REST (`/api/v1/accounting/registries…`).
- `src/modules/accounting/index.ts` — `createAccountingModule({ db, deps })` factory.
- `src/modules/accounting/__tests__/accounting.test.ts` — lifecycle + overlap + scoping + freeze tests.
- `src/db/migrations/0011_accounting.sql` — generated via `npm run db:generate -- accounting`, then applied.

To modify / read from (composition, not duplication):

- `src/app.ts` — compose `createAccountingModule` after orders + inventory; mount `/api/v1/accounting`.
- `src/modules/orders/{service,repository}.ts` — add a read for `order_payments` of delivered orders by store + period (grouped by method). Reference: `paymentsForOrder`/`sumPaymentsForOrder` ([repository.ts:285-300]), `storeIdsForUser` ([repository.ts:250-256]), `order_payments` schema ([schema.ts:120-144]), status enum ([schema.ts:22-29]).
- `src/modules/inventory/{service,repository}.ts` — add a read for provider purchases (tank+item `kind='purchase'`) valued at `purchase_price × qty` + surcharges, by store + period. Reference: `recordTankPurchase`/`recordItemPurchase` ([service.ts:345-421]), `tank_holders.purchase_price` ([schema.ts:54-82]), `purchase_surcharges` ([schema.ts:185-190]), tx-kind enum ([schema.ts:23-32]).
- `src/lib/date.ts`, `src/lib/errors.ts`, `src/modules/auth/middleware.ts` (`createRequireRole`) — reused, unchanged.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-15] [lpg-backend] Backend track **done**. New `accounting` vertical module (`src/modules/accounting/{schema,types,repository,service,routes,index}.ts`) composed in `app.ts` **after** orders + inventory. **Tables** (migration `0011_accounting.sql`, applied): `accounting_registries` (storeId, periodStart/periodEnd `date`, status `open|closed`, openedBy/closedBy/closedAt, notes, `snapshot` jsonb, period-order CHECK) + `accounting_entries` (registryId, direction `ingress|egress`, amount numeric(10,2) CHECK >0, category, `method` reusing the **existing** `order_payment_method` enum, note, occurredAt, recordedBy). Two new enums (`accounting_registry_status`, `accounting_direction`); the payment-method enum was **reused, not recreated**. **Composition (ADR-012, no duplication):** the breakdown reads via two structural ports — `AccountingIngressSource` / `AccountingEgressSource` — satisfied by `OrdersService.paymentsByMethodForStorePeriod` (delivered-order `order_payments` grouped by method, **keyed by the payment's** `occurredAt` business date) and `InventoryService.purchaseCostsForStorePeriod` (location-holder `kind='purchase'` valued `purchase_price × qty` for tanks+items + `purchase_surcharges`). Both added as repo methods (+ fakes) + 1-line service delegates. **Also wired item-holder price snapshot** in `inventory/repository.findOrCreateItemHolder` (was hardcoded `'0.00'` "until the reports spec" — this is that spec) so item egress isn't silently zero; mirrors the tank-holder lookup, lenient `0.00` fallback. No existing test depended on the old behaviour. **Lifecycle:** open (overlap per store → 409; operator must own the store, admin/dev any) → live breakdown (open) / **frozen `snapshot`** (closed) → add/remove manual entries (open-only → 409 on closed) → close (**admin/dev only**) freezes the snapshot. **Scoping:** admin/dev global; operators limited to `storeIdsForUser` for open/manage/view; `?storeId` intersected with scope (tampered id → no rows, never widens); cross-store → 404/403. **Tests:** new `accounting.test.ts` (3 lifecycle/overlap/scoping tests; fakes-based incl. a `FakeAccountingRepository` + `seedDeliveredPayment` + inventory price-seeding) → **84 total** (was 81). Independent validation: **all 7 backend criteria MET, no gaps** (ingress payment-date-keyed, overlap = canonical range-intersect, scoping un-widenable confirmed). Gates green: typecheck + lint + 84 tests + build; migration applied to dev DB. **Frontend track remains.**

**Deferred / noted:** manual per-order exclusion from a registry (kept out by the non-overlapping-window decision); reopen of a closed registry; cross-period weekly/monthly reports (the spec's stated posterior work, out of scope).

Recorded **ADR-018** — closing a registry freezes a `snapshot` (deliberately overrides ADR-007's derive-don't-store rule, scoped to closed financial periods) so a back-dated payment can't retroactively move a closed period's books.

[2026-06-15] [lpg-backend] Follow-up (post-track): **`GET /registries` list now carries headline totals** (`net`, `ingressTotal`, `egressTotal`) per row, **without N+1**. **Closed** registries read them from the frozen `snapshot` (zero extra queries; respects ADR-018). **Open** registries are computed in **one batched pass**: new batch read ports `OrdersService.paymentTotalsForStorePeriods` + `InventoryService.purchaseTotalsForStorePeriods` (each a single SQL pass over a `VALUES` list of the open registries' windows, keyed back by registry id — ingress via JOIN+GROUP BY, egress via correlated subqueries summing tank+item+surcharge), plus `AccountingRepository.entryTotalsByRegistry` (one grouped query for manual lines). So the whole page is **O(1) queries regardless of row count**, never per-row. New `RegistrySummaryView` (extends `RegistryView` with the three totals; `null` only in the defensive closed-without-snapshot case). The raw batched SQL (VALUES/CTE + param casts + correlated subqueries) was **smoke-run against the dev Postgres** (the fakes can't cover raw SQL) — confirmed `{key:number, total:string}` shape, empty-periods guard, and real store-1 data. +1 test (list totals: open batched + closed snapshot) → **85 total**. Gates green (typecheck + lint + 85 tests + build). Frontend can render net per row straight off the list; track stays `done`.