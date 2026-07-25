---
project: lpg-store
domain: specs
type: spec-track
spec: provider-management
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-09
---

# Provider Management — lpg-backend track

Shared spec: [[index]] · Debt pattern mirrored from [[../../customers/customers-crud/index]] · Cost default chain from [[../../inventory/provider-purchase-cost/backend]]

## Technical Notes

**Module layout.** New vertical module `src/modules/providers/`
(`routes/service/repository/schema/types/index.ts`) with a
`createProvidersModule({ db, deps })` factory composed in `src/app.ts`, per the
module template. `providers/schema.ts` re-exported through `src/db/schema.ts`
(the drizzle aggregator) so migrations pick it up. Follow the `customers` module
as the closest structural template.

**Data model (new tables, migration continues from 0014):**

- `providers` — `id serial PK`, `name text` (unique among active), `phone text?`,
  `notes text?`, `active boolean default true`, `createdAt`/`updatedAt`. Int PK:
  it is referenced by purchases and every debt table (project convention —
  int PKs for shared/referenced tables, UUID only for leaf tables).
- `provider_prices` — `id`, `providerId → providers.id`, `productKind` enum
  (`'tank' | 'item'`), `productId int` (tankTypeId or inventoryItemId — soft ref,
  mirrors how the ledger references products), `unitPrice numeric(10,2)` (≥ 0,
  ≤ 2 dp), `active`, timestamps. Partial-unique on `(providerId, productKind,
  productId) WHERE active`. **Seeded on provider creation**: create a row per
  active catalog tank type + inventory item, `unitPrice = catalog.purchase_price`
  (read from `tankTypes`/`inventoryItems`); admin edits afterward. New catalog
  products added later have no row until priced — the purchase default falls back
  to last-cost/catalog for those.
- `provider_empty_debts` — direct mirror of `customer_empty_debts`
  (inventory/schema.ts:173-188): `id`, `providerId`, `tankTypeId`, `delta int`
  (signed), `refTankTransactionId?`, `createdAt`. Append-only (ADR-009).
  Balance view `provider_empty_debt_balance` → `empties_owed = SUM(delta)`
  grouped by `(providerId, tankTypeId)` (mirror the SQL in
  migration 0002:109 / schema.ts:224-228). Sign convention: **positive = we owe
  the provider empties**.
- `provider_debts` — money charges, append-only: `id`, `providerId`,
  `amount numeric(10,2)`, `description text`, `refKind text?`
  (`'tank_purchase'|'item_purchase'|'manual'`), `refId int?`, `createdAt`.
  (Charges accrue; they are not flipped to paid — payments net against them,
  matching the partial-payment requirement, unlike `customer_debts.isPaid`.)
- `provider_payments` — mirror of `order_payments`: `id`, `providerId`,
  `amount numeric(10,2)` (> 0), `method` (reuse the existing payment-method
  enum from orders if shareable, else a providers-local enum), `note text?`,
  `recordedBy → users.id`, `occurredAt timestamptz`. Balance:
  `outstanding = SUM(provider_debts.amount) − SUM(provider_payments.amount)`
  per provider (a `provider_debt_balance` view or a repo aggregate).

**Purchase changes (inventory module — the cross-module edit):**

- Add nullable `provider_id → providers.id` to `tank_transactions` and
  `item_transactions` (same migration or a paired one; set only for
  `kind='purchase'`). Existing rows NULL — no backfill.
- `types.ts`: add `providerId` (optional on the wire) to `tankPurchaseSchema`
  (types.ts:119-142) and `itemPurchaseSchema` (types.ts:144-160); add an
  optional `amountPaid` (numeric, ≥ 0) at the request level (or per the finalized
  Open Question). `unitCost` primitive stays as-is (types.ts:20-26).
- `service.ts` `recordTankPurchase` (service.ts:634-696): persist `providerId`
  on the inserted tx; extend the `unitCost` default (service.ts:667-671) to try
  `provider_prices` for `providerId + tankTypeId` **before**
  `lastPurchaseTankUnitCost` → catalog. After the existing empty guards
  (service.ts:651-660), when `providerId` present, write a
  `provider_empty_debts` delta `= qty − emptyReturned` (positive shortfall) or
  the capped settle on over-return, referencing the new tx id. The **empty shortfall defaults to a tank debt (no monetary surcharge)**; the existing `purchase_surcharge` path stays available when the operator bills the shortfall in cash instead. When `providerId` present, write a `provider_debts` charge `= lineCost + surcharge`; if the optional `amountPaid` is given, write a `provider_payments` row (the standalone `POST /providers/:id/payments` covers deferred / partial payments). All inside the existing `db.transaction()`.
- `recordItemPurchase` (service.ts:698-734): same provider attribution + price
  default + money charge (no empty dimension for items).
- **Composition, not import** (ADR-012): inventory must not import the providers
  module. Inject the provider-write ports (accrue empty debt, accrue money
  charge, resolve provider price) into `InventoryService` via a setter wired in
  `app.ts` — the same late-bind injected-port pattern used for the accounting
  `accountedGuard` (see `provider-purchase-cost` backend track,
  `updatePurchaseUnitCost` service.ts:743-771) and the notifications trigger.

**Endpoints (providers module):**
- `GET /providers` (list + `?search=` name/phone, active filter), `GET /providers/:id`
  (detail: money outstanding + empties owed per type + price list), `POST /providers`,
  `PATCH /providers/:id` (partial + soft `active`).
- `GET /providers/:id/prices`, `POST`/`PATCH` price rows.
- `POST /providers/:id/payments` (record payment, partial allowed; role-guarded).
- Empties-owed read on the detail (or `GET /providers/:id/empty-debts`, mirroring
  the customer empty-debt route at routes.ts:337-345).

**Accounting (deliberate change to the done `accounting-registry`):** egress
becomes **cash paid** — sum `provider_payments` by payment business date for the
store + period, replacing the purchase-value egress
(`purchaseCostsForStorePeriod` / `purchaseTotalsForStorePeriods`,
service.ts:1126-1152; consumed at `accounting/service.ts:348-372` via the source
ports at `accounting/service.ts:71-82`). Keep purchase value available as
**Compras recibidas** and expose **Deuda a proveedores** (running balance /
period delta) so the registry shows `pagos + Δdeuda = compras`. Legacy purchases
(no provider / payment) keep valuing at purchase value via a NULL-provider
branch. Freeze all three figures in the closed-registry snapshot exactly like
today's egress; keep `registry-source-drilldown` (`purchaseLinesForStorePeriod`)
consistent. Read provider-payment totals via an **injected accounting→providers
port** (ADR-012), not a direct import — same late-bind pattern as the accounting
`accountedGuard`. Existing accounting tests are **updated** (not merely kept
green) to assert the cash-basis egress + the reconciliation identity.

**Testing:** lifecycle integration tests (`node --test`, no jest), 2–5 per new
surface, each walking a full happy path (project convention). Extend the
inventory fakes faithfully as `provider-purchase-cost` did. Cover: credit
purchase → empty shortfall debt → partial money payment → over-return settles
tank debt → payment clears money balance. Keep existing inventory + accounting
tests green.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `src/modules/customers/{schema,service,repository,routes,types,index}.ts` — the registry + monetary-debt pattern to mirror for the new `providers` module
- `src/modules/inventory/service.ts` — `recordTankPurchase` (634-696), `recordItemPurchase` (698-734), `unitCost` default (667-671), empty guards (651-660), customer empty-debt settle logic in `recordSale`/`recordReturn` (343-475) to mirror; `updatePurchaseUnitCost` injected-guard pattern (743-771)
- `src/modules/inventory/schema.ts` — `tank_transactions`/`item_transactions` (113-145), `customer_empty_debts` (173-188) + balance view (224-228) to mirror as `provider_empty_debts`; add `provider_id`
- `src/modules/inventory/types.ts` — `tankPurchaseSchema` (119-142), `itemPurchaseSchema` (144-160), `unitCost` primitive (20-26)
- `src/modules/inventory/{routes,repository,kindToDelta}.ts` — purchase routes (250-296), balance reads, `kindToTankDelta`
- `src/modules/accounting/{service,repository,schema}.ts` — egress breakdown (service.ts:348-372) + source ports (71-82): switch egress to cash-paid, add **Compras recibidas** / **Deuda a proveedores** reconciling lines, freeze in snapshot, keep `registry-source-drilldown` consistent; wire the accounting→providers payment-totals port in `app.ts`
- `src/db/schema.ts` (drizzle aggregator), `src/db/migrations/` (0002 balance-view SQL, 0012 unit_cost as precedent) — new migration from 0014
- `src/app.ts` — module composition + injected-port wiring (mirror accounting `accountedGuard` late-bind)

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [lpg-backend] description of what was done -->

[2026-07-09] [lpg-backend] Backend track **done**. Built the `providers` module and reworked the purchase + accounting egress paths end-to-end.

**New `providers` module** (`src/modules/providers/{schema,types,repository,service,routes,index}.ts`, mirrors `customers`): CRUD + name/phone accent-insensitive search + soft delete (admin-gated writes, operator+admin reads). Create **seeds a `provider_prices` row per active catalog tank type + item at the catalog `purchase_price`** (in one tx); admin edits via `PUT /providers/:id/prices`. Detail composes money-outstanding + empties-owed + price list. `POST /providers/:id/payments` records a (partial) payment, store-scoped (operator own-store via `store_assignments`, admin any).

**Data model** (all provider tables in `providers/schema.ts` to stay cycle-free — `orders/schema` already imports `inventory/schema`, so `inventory/schema` can't import `orders`; the provider tables keep a local `provider_payment_method` enum): `providers` (+ `uq_provider_active_name` partial-unique, migration **0015**), `provider_prices` (partial-unique active per provider+product), `provider_empty_debts` (+ `provider_empty_debt_balance` view), `provider_payments` (+ `provider_money_balance` view = Σ purchase value on `*_transactions.provider_id` − Σ payments). Migration **0014_providers** adds these + a nullable `provider_id` FK on `tank_transactions`/`item_transactions`. `refTankTransactionId` is a soft ref (no FK) so `providers/schema` imports only `catalog`+`auth` (one-way `inventory/schema → providers/schema`).

**Money debt is DERIVED, not a charge table**: the purchase txs already carry `provider_id` + `unit_cost`, so `provider_money_balance` computes charges directly (no `provider_debts` table needed).

**Purchase rework** (`inventory/service.ts`): `providerId` (optional on wire) persisted on the purchase tx; `unitCost` default chain gains a top tier — **provider price (injected `ProviderPriceResolver`, wired in `app.ts` to `providers.service.unitPriceFor`, no inventory→providers import)** → last cost → catalog. Empty shortfall `qty − emptyReturned` accrues a **`provider_empty_debts`** positive delta; an over-return settles prior debt (negative, capped at owed — mirrors `recordSale`). Never blocks; the `emptyReturned ≤ qty` guard is relaxed **only when a provider is present** (a provider-less purchase keeps the legacy 409). Optional inline `amountPaid`+`paymentMethod` writes a `provider_payments` row in the same tx.

**Accounting egress → cash basis** (`accounting/service.ts` + `types.ts`; deliberately modifies the done `accounting-registry`): egress that nets is now **cash paid to providers** (`provider_payments` by payment date via new `AccountingEgressSource` payment ports), replacing purchase-value egress. The breakdown adds **`comprasRecibidas`** (goods received) + **`deudaDelta`** so `providerPayments + deudaDelta === comprasRecibidas`; frozen in the close snapshot. Legacy (provider-less) purchases → 0 cash egress + goods-received memo. The drill-down still decomposes purchases and now reconciles to `comprasRecibidas`.

**Tests**: `providers.test.ts` (2 — registry+pricing seed/edit/search/gating; payment scoping+netting) + `inventory/provider-purchase.test.ts` (3 — empty-debt accrue+settle; provider-price default + inline payment; provider-less no-debt + guard). Existing accounting tests **updated to cash-basis** (a/d/e) + new case (f) asserting the reconciliation identity + partial-payment egress. **168 tests pass**; `typecheck` + `biome check` + `build` green; migrations 0014 + 0015 applied to the dev DB (views validated live). All acceptance criteria for this repo met.

Remaining: the **frontend track** (lpg-frontend-vue) — providers module UI (list with debt badges, form, detail with price list + record-payment), provider selector + empty-debt note on the purchase dialog, and the accounting registry detail's cash-paid egress + Compras/Deuda reconciling lines.

[2026-07-09] [lpg-backend] Renamed the accounting egress breakdown keys for cross-repo naming consistency (they were the only Spanish-named keys among otherwise-English egress fields): `comprasRecibidas → goodsReceived`, `deudaDelta → debtDelta`, in `accounting/{types.ts,service.ts}` + the accounting tests. Spanish is retained only in the UI labels ("Compras recibidas" / "Deuda a proveedores"). The reconciliation identity is now `providerPayments + debtDelta = goodsReceived`. Quality gate re-run green: `typecheck` + `biome check` + **168 tests pass**. Done alongside the frontend track so both repos stay in sync on the wire contract.

[2026-07-09] [lpg-backend] **Store purchase-detail enrichment** (post-`done` follow-up, agreed in-session — recorded here so the backend track reflects it). Added an explicit **`purchase_id` (uuid)** to `tank_transactions`, `item_transactions`, and `provider_payments` (migration **0016_purchase_id_grouping**, applied to dev DB; + a btree index on each). `recordTankPurchase`/`recordItemPurchase` generate one `purchaseId` per call (`node:crypto` `randomUUID`) and stamp it on every line **and** the inline `amountPaid` payment, so the store purchase list can group a multi-product purchase and show per-purchase paid/owed. Standalone provider-level payments keep `purchase_id = NULL`.

`GET /inventory/stores/:storeId/purchases` now returns **`{ purchases, purchasePayments }`**: each `PurchaseLineView` gained `purchaseId`, `providerId`, `providerName` (left-joined), and `emptyDebt` (the `provider_empty_debts.delta` for that tank tx — signed; 0 for items/no-provider); `purchasePayments` maps `purchaseId → Σ inline payment` (new repo `purchasePaidTotalsForStorePeriod`, grouping `provider_payments` by `purchase_id`). The shared `purchaseLinesForStorePeriod` (also used by the accounting drill-down) only gained columns — the accounting consumer is unaffected. New lifecycle test asserts grouping + provider + per-line empty debt + per-purchase paid. Gate green: typecheck + biome + **169 tests** + build.

Note: reassigning a provider to an already-created provider-less purchase is still unsupported (would need a dedicated endpoint) — out of scope here.
