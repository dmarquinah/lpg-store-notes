---
project: lpg-store
domain: specs
type: spec-track
spec: provider-purchase-cost
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-16
---

# Provider Purchase Cost — lpg-backend track

Shared spec: [[index]] · Egress consumer: [[../../accounting/accounting-registry/backend]]

## Technical Notes

A focused change to the **inventory** module's purchase write path plus its two
egress read ports. No change to the accounting module's formulas — it composes
`InventoryService.purchaseCostsForStorePeriod` /
`purchaseTotalsForStorePeriods`, and those now read the captured cost.

### Schema + migration

Add a **nullable** `unit_cost numeric(10,2)` to both `tank_transactions` and
`item_transactions` ([schema.ts](../../../../lpg-backend/src/modules/inventory/schema.ts)).
Nullable because the column is meaningful only for `kind='purchase'` rows — every
other kind (`sale`/`return`/`opening`/…) leaves it `NULL`. Generate the migration
with `npm run db:generate -- purchase_unit_cost` (named, per the project rule),
then apply with the migrator. No backfill: legacy purchases keep `NULL` and fall
back to the holder snapshot in valuation.

### Purchase request + recording

- `tankPurchaseSchema` / `itemPurchaseSchema`
  ([types.ts:79-111](../../../../lpg-backend/src/modules/inventory/types.ts)) —
  add an optional `unitCost` to each line item: `z.number().nonnegative()
  .finite()` with a ≤2-decimal refine (mirror the orders `round2` / accounting
  `positiveMoney` pattern; allow 0 but not negative).
- `recordTankPurchase` / `recordItemPurchase`
  ([service.ts:348-424](../../../../lpg-backend/src/modules/inventory/service.ts)) —
  when building each `insertTankTransaction` / `insertItemTransaction`, set
  `unitCost`. Resolution chain: `line.unitCost ?? lastPurchaseUnitCost(store,
  product) ?? catalog.purchase_price`. Store as a 2-decimal string
  (`toFixed(2)`), consistent with `purchase_surcharges`. **The catalog row is
  not mutated.**
- **Default = last purchase cost.** Add a repo read
  `lastPurchaseUnitCost(storeId, tankTypeId|itemId)` → the `unit_cost` of the
  most recent `kind='purchase'` tank/item transaction on that store's `location`
  holder (`ORDER BY occurred_at DESC LIMIT 1`), or `null` if none / still NULL.
  The catalog `purchase_price` (already loaded in `findOrCreateTankHolder` for
  the snapshot) is the final fallback on the first-ever purchase. This default
  is mostly a **frontend pre-fill** concern (the UI requires the field); the
  backend chain is the safety net for non-UI callers.
- `insertTankTransaction` / `insertItemTransaction` (repository) — extend their
  insert payload types to carry the optional `unitCost`.

### Correcting a purchase cost after the fact

Operators mis-enter under time pressure, so expose a correction path:

- **Route:** `PATCH /api/v1/inventory/purchases/:kind/:id/cost` (`kind` ∈
  `tank|item`) with body `{ unitCost }`, role-guarded identically to recording a
  purchase (operator own-store + admin/dev). A small Zod schema reuses the same
  `unitCost` validator.
- **Service** `updatePurchaseUnitCost(kind, txId, unitCost, caller)`: load the
  tx (now also returning `occurredAt`); reject if `kind !== 'purchase'`
  (`ValidationError`); resolve the holder → `storeId` + `storeIdsForUser` scope
  check (cross-store → 403); then the **accounted-period freeze** below; update
  `unit_cost`.
- **Frozen once accounted (user decision 2026-06-16).** If the purchase's
  business date (`toBusinessDate(tx.occurredAt)`) falls inside a **closed**
  registry for its store, the edit is rejected with `ConflictError` (409) for
  **every** role — an accounted period is final. Implemented as an **injected
  port** `AccountedPeriodGuard = (storeId, businessDate) => Promise<boolean>` on
  `InventoryService` (`setAccountedPeriodGuard`), satisfied by accounting's new
  `isStorePeriodAccounted` (→ repo `hasClosedRegistryCovering`: a `status=closed`
  registry whose `[periodStart, periodEnd]` covers the date). **Wired in
  `app.ts`** after accounting is built, so inventory never imports accounting
  (keeps the dependency one-directional; the late-bind breaks the would-be
  cycle). Unset guard (inventory standalone / tests) → no freeze. Reopening a
  closed period stays out of scope.
- **Append-only note:** editing `unit_cost` mutates a valuation *attribute*, not
  a ledger delta/quantity — inventory movement is untouched — so it doesn't
  violate ADR-009's append-only intent (it's metadata correction, like a note).

### Egress valuation (the two read ports)

Both currently multiply `holder.purchase_price × qty`. Change to
`COALESCE(tx.unit_cost, holder.purchase_price) × qty`:

- `purchaseCostsForStorePeriod`
  ([repository.ts:455-495](../../../../lpg-backend/src/modules/inventory/repository.ts)) —
  the two Drizzle `SUM(...)` expressions over `tankTransactions`/`itemTransactions`
  joined to their holders. Wrap the holder price in `COALESCE(tx.unit_cost, …)`.
- `purchaseTotalsForStorePeriods` (the batched/correlated-subquery variant used
  by the registry **list** totals) — apply the same `COALESCE` inside the
  `SUM(th.purchase_price * tt.full_delta)` / item subqueries. **This is raw SQL**
  the fakes can't cover — smoke-run it against the dev Postgres (as the original
  batched port was) to confirm shape and that `NULL` unit_cost rows still value
  via the holder price.

Surcharge sums are untouched.

### Snapshot interaction (accounting)

The accounting registry **freezes a snapshot on close** (ADR-018). Because the
egress port now returns cost-corrected figures, a registry **closed after** this
ships will freeze the corrected egress automatically — no accounting code change.
Registries **already closed** keep their old frozen totals (correct: closed
periods are final). Open registries recompute live and immediately reflect the
new valuation for purchases that carry a `unit_cost`.

### Reuse / don't reinvent

- Money: `numeric(10,2)` + `toFixed(2)`; round like the orders `round2` helper.
- Catalog price lookup: reuse the same `tank_types` / `inventory_items` select
  already done in `findOrCreateTankHolder` / `findOrCreateItemHolder` — don't add
  a second query path if the holder lookup can hand back the price.
- Errors: `src/lib/errors.ts` (`ValidationError` for a malformed `unitCost`).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/schema.ts` — add nullable `unit_cost` to
  `tank_transactions` + `item_transactions`.
- `src/modules/inventory/types.ts` — optional per-line `unitCost` on
  `tankPurchaseSchema` / `itemPurchaseSchema` ([types.ts:79-111](../../../../lpg-backend/src/modules/inventory/types.ts)).
- `src/modules/inventory/service.ts` — capture `unitCost` (entered or catalog
  default) in `recordTankPurchase` / `recordItemPurchase`
  ([service.ts:348-424](../../../../lpg-backend/src/modules/inventory/service.ts)).
- `src/modules/inventory/repository.ts` — extend `insertTankTransaction` /
  `insertItemTransaction` payloads; `COALESCE(tx.unit_cost, holder.purchase_price)`
  in `purchaseCostsForStorePeriod` + `purchaseTotalsForStorePeriods`
  ([repository.ts:455-545](../../../../lpg-backend/src/modules/inventory/repository.ts)).
- `src/modules/inventory/__tests__/*.ts` — cost-below-sell, omitted→catalog,
  legacy-NULL-fallback cases.
- `src/db/migrations/00NN_purchase_unit_cost.sql` — generated + applied.

Read-only context (no change needed):

- `src/modules/accounting/service.ts`/`repository.ts` — consumes the egress
  ports; valuation correction is transparent to it.
- `src/modules/catalog/schema.ts` — `tank_types`/`inventory_items.purchase_price`
  (the default source; not mutated).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-backend] Backend track **done**. Captured an editable provider cost-in per purchase and re-valued accounting egress against it. **Migration `0012_purchase_unit_cost`** (generated + applied): nullable `unit_cost numeric(10,2)` on **both** `tank_transactions` and `item_transactions`; existing rows stay `NULL`, no backfill. **Purchase requests** (`tankPurchaseSchema`/`itemPurchaseSchema`) gain an **optional** per-line `unitCost` (shared `unitCost` Zod: `≥0`, finite, ≤2dp). `recordTankPurchase`/`recordItemPurchase` persist it with the default chain **`entered ?? lastPurchase{Tank,Item}UnitCost(store, product) ?? holder.purchasePrice`** (the holder snapshot is the catalog price captured at creation — catalog **not** mutated). New repo reads `lastPurchase{Tank,Item}UnitCost` (most recent `kind='purchase'` on the location holder, `ORDER BY occurred_at DESC, id DESC`). **Egress valuation** now `COALESCE(tx.unit_cost, holder.purchase_price) × qty` in both `purchaseCostsForStorePeriod` (Drizzle) **and** the batched raw-SQL `purchaseTotalsForStorePeriods` (tank + item subqueries); `unit_cost IS NULL` rows still value at the holder snapshot (backward compatible). **Correction after the fact:** `PATCH /api/v1/inventory/purchases/:kind/:id/cost` (`canPurchase` = operator/admin, developer auto-passes) → `updatePurchaseUnitCost(kind, txId, unitCost, caller)`: 404 if missing, **`ValidationError`** if the row isn't `kind='purchase'`, **`ForbiddenError`** for a non-global caller whose `storeIdsForUser` excludes the tx's store. **No closed-period guard by design** — a closed registry reads its frozen `snapshot` (ADR-018), so a late correction only moves open registries' live egress; this also avoids an inventory→accounting dependency cycle. Surcharge valuation + all non-purchase kinds untouched (`unit_cost` stays `NULL`). New `provider-purchase-cost.test.ts` (6 tests at first, **8 after the follow-ups**: entered-below-sell, omitted→last-cost/catalog-on-first, legacy-NULL→snapshot, correction-moves-open-egress, reject non-purchase + cross-store, item entered cost); `FakeInventoryRepository` extended to mirror the real repo (COALESCE valuation, `lastPurchase*`, `getPurchaseTx`, `updatePurchaseTxCost`). The raw batched SQL was **smoke-run against dev Postgres** (store 1: tank 800 + surcharge 50 = 850, batched == summed, legacy NULL rows valued at snapshot). **Gates green:** typecheck + `biome check` (lint **and** format — the project's full gate, not just `biome lint`) + **93 tests** (was 85) + build; migration applied. Independent validation: **GREEN — all 7 backend criteria PASS**, fake faithful to real repo, no blocking issues. **Frontend track remains** (required cost field pre-filled from last purchase cost; edit-cost affordance in Movimientos).

[2026-06-16] [lpg-backend] Follow-up (user request) — **purchase-cost edits are now frozen once accounted**, reversing the earlier "no closed-period guard" call. The earlier design let the edit through and relied on the registry snapshot to shield closed books; the owner wants an accounted purchase to be *uneditable*, not silently divergent. Added accounting `isStorePeriodAccounted(storeId, businessDate)` → repo `hasClosedRegistryCovering` (a `status='closed'` registry whose `[periodStart, periodEnd]` covers the date). `InventoryService` gained an injected `AccountedPeriodGuard` (`setAccountedPeriodGuard`), consulted in `updatePurchaseUnitCost` after the scope check — if the purchase's `toBusinessDate(occurredAt)` is accounted → **`ConflictError` 409 for every role**; open/un-accounted periods still recompute live. `getPurchaseTx` now also returns `occurredAt`. **Wired in `app.ts`** (late-bind after accounting exists) so inventory never imports accounting — the reverse edge stays a one-line injected port, no dependency cycle. Both fakes extended (`FakeAccountingRepository.hasClosedRegistryCovering`, inventory `getPurchaseTx` occurredAt); +1 inventory test (closed → 409, then open → edit succeeds). Freeze query smoke-run on dev DB (open registry #1 → not accounted, correct). Gates green (typecheck + `biome check` + 94 tests + build).

[2026-06-16] [lpg-backend] Follow-up (user request) — added the **purchases LIST endpoint** the frontend edit affordance needs (no list existed; store purchases land on the location holder, invisible to the assignment-scoped `/transactions`). `GET /inventory/stores/:storeId/purchases?from=&to=` (`canPurchase`; operator own-store scoped, admin/dev global; bounds default to today's business date) → `service.listStorePurchases` over new repo read **`purchaseLinesForStorePeriod`** (tank + item `kind='purchase'` on the store's location holder, joined to `tank_types`/`inventory_items` for the name + left-joined `purchase_surcharges`, `COALESCE(unit_cost, holder.purchase_price)`, newest-first). Each `PurchaseLineView` carries `{kind, txId, productId, productName, qty, unitCost, lineCost, surcharge, occurredAt, businessDate, accounted}`; `accounted` reuses the injected `AccountedPeriodGuard` (memoized per business date) so the UI can show/freeze frozen rows without a failed request. **This is the same read the `registry-source-drilldown` spec planned** — it now exists, so that spec's egress drill-down can reuse it. +1 test (list: cost/lineCost/accounted + cross-store 403); **suite now 95 tests**; raw query smoke-run on dev DB (store 1: 20×Balón 10kg @ 40.00 + 50.00 surcharge). Gates green (typecheck + `biome check` + 95 tests + build).
