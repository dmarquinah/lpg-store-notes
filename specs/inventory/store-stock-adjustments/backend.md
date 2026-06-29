---
project: lpg-store
domain: specs
type: spec-track
spec: store-stock-adjustments
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-28
---

# Store Stock Adjustments — lpg-backend track

Shared spec: [[index]] · History consumer: [[../store-history/backend]]

## Technical Notes

A small additive change to the **inventory** module: a location-scoped sibling of
the existing assignment-scoped adjustment path. The `adjustment` transaction kind
and the `location` holder already exist (`inventory-foundation`) — this just
exposes a write path to them for a store rather than a driver-day.

### Request + route

- **Route:** `POST /api/v1/inventory/stores/:storeId/adjustments`, role-guarded
  identically to recording a purchase (`canPurchase`: operator own-store +
  admin/dev; developer auto-passes). Reuse the `storeIdsForUser` scope check
  already used by `recordTankPurchase` (cross-store → `ForbiddenError` 403).
- **Body (Zod):** `{ notes: string (non-empty), lines: Array<{ tankTypeId: int>0,
  fullDelta: int, emptyDelta: int }> }` with `≥1` line and each line having at
  least one non-zero delta. Mirror the integer/finite validation style of the
  existing purchase schemas in `types.ts`.

### Service — `recordLocationAdjustment`

- One `repo.transaction(...)`. For each line: `findOrCreateTankHolder({ kind:
  'location', storeId }, tankTypeId)`, read the holder's current balance
  (`tankBalancesByLocation(storeId)`), and **reject if `current + delta < 0`** for
  full or empty (`ConflictError` 409 — message names the tank type, like the
  `loadFromLocation` stock-out error). Then `insertTankTransaction({ holderId,
  fullDelta, emptyDelta, kind: 'adjustment', userId: caller.id, notes })`.
  `unit_cost` stays **NULL** (not a purchase). No `refTransactionId` pairing — a
  location adjustment is standalone (unlike the paired opening/carry transfers).
- The whole request is atomic: any line failing the negative-balance check rolls
  the transaction back, nothing is written.
- **Actor + timestamp** come for free from the existing `insertTankTransaction`
  (`user_id`, `occurred_at default now()`), which is what
  [[../store-history/index|store-history]] reads back.

### Why not the assignment-adjustment path or a purchase

- The existing `POST /assignments/:id/adjustments` writes on the **assignment**
  holder — wrong target (it would require inventing a driver-day). This new
  endpoint writes on the **location** holder directly.
- A **purchase** carries supplier semantics (full-for-empty swap default,
  `unit_cost`, accounting egress via `purchaseCostsForStorePeriod`). An adjustment
  must **not** hit egress — keep `kind='adjustment'` so the registry (which sums
  only `kind='purchase'`) ignores it. Add a test asserting an adjustment leaves
  egress unchanged.

### Reuse / don't reinvent

- `findOrCreateTankHolder`, `tankBalancesByLocation`, `insertTankTransaction`,
  `kindToTankDelta` — all already in the service/repository.
- Errors: `src/lib/errors.ts` (`ValidationError`, `ConflictError`,
  `ForbiddenError`).
- Scope helper: the same `storeIdsForUser` / caller-scope used by the purchase
  routes.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/inventory/routes.ts` — add `POST /stores/:storeId/adjustments`
  (guard like the tank-purchase route).
- `src/modules/inventory/types.ts` — Zod schema for the location-adjustment body
  (`notes` + `lines[]` with signed integer deltas).
- `src/modules/inventory/service.ts` — `recordLocationAdjustment(storeId, input,
  caller)`: scope check, per-line find-or-create holder + negative-balance guard +
  `insertTankTransaction(kind='adjustment')`, all in one transaction.
- `src/modules/inventory/__tests__/*.ts` — seed-from-zero, multi-type atomic,
  negative-balance reject (nothing written), cross-store 403, adjustment-does-not-
  hit-egress.

Read-only context (no change):

- `src/modules/inventory/schema.ts` — `tank_holders` (`kind='location'`),
  `tank_transactions` (`adjustment` kind, `user_id`, `occurred_at`).
- `src/modules/inventory/repository.ts` — `findOrCreateTankHolder`,
  `tankBalancesByLocation`, `insertTankTransaction`.
- `src/modules/accounting/` — egress sums only `kind='purchase'` (confirms an
  adjustment is invisible to the registry; no change here).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-28] [lpg-backend] Backend track DONE — all 7 backend criteria met; independent validation GREEN; gates green (typecheck + `biome check` + **134** tests + build).

- **Route:** `POST /api/v1/inventory/stores/:storeId/adjustments` (routes.ts), guarded by `canPurchase` (`requireRole('operator','admin')`; **developer auto-passes** the role guard). Returns `201 { transactions }`.
- **Schema** (`locationAdjustmentSchema`, types.ts): `{ notes: requiredNote(non-empty, ≤500), lines: [{ tankTypeId, fullDelta, emptyDelta }] }` — `≥1` line, each line `≥1` non-zero delta (zero-effect **rejected**, 400), and **no duplicate `tankTypeId`** (mirrors the purchase-schema dedup so the frontend's absolute "set to N" maps to exactly one delta).
- **Service** (`recordLocationAdjustment`, service.ts): own-store scope check via `storeIdsForUser` (operator) / `isGlobal` (admin/dev) → cross-store `ForbiddenError` **403**. Inside one `repo.transaction`: a **validate-all-before-write** pass (catalog-type existence via `tankTypeLabel` → **400** `ValidationError` on unknown; negative-result guard → **409** `ConflictError` naming the type) precedes any insert, so a single bad line leaves **nothing written even on the no-rollback in-memory fake** (real Postgres also rolls back — belt + suspenders). Then per line `findOrCreateTankHolder({kind:'location', storeId})` + `insertTankTransaction({ kind:'adjustment', userId, notes })` — **no `unitCost`** (NULL), so the accounting registry (sums only `kind='purchase'`) never sees it.
- **No schema/migration change** — reuses the existing `adjustment` kind + `location` holder.
- **Decisions** (Open Questions): reuse `adjustment` kind; note required free-text (presets are the frontend's concern); zero-effect + duplicate-type lines rejected; tank type validated by **existence only** (NOT restricted to `active`, so a correction/merma on a deactivated-but-physically-present type stays possible — owner-confirmable). Validation errors are **400** (this codebase's `ValidationError` → 400, not the 422 the spec text guessed).
- **Tests** (store-stock-adjustments.test.ts, 15): seed-from-zero multi-type, ± correction (append-only, two rows), negative-balance reject (nothing written), unknown-type validate-all (nothing written), cross-store 403, own-store operator ok, no-egress (purchase totals stay 0); HTTP 201/403/409/400×3/delivery-403/401.
- **Note for the consumer spec:** `recordTankPurchase` does **not** actually enforce own-store scoping (the spec text assumed it did) — this endpoint follows the real pattern from `listStorePurchases`/`updatePurchaseUnitCost` and adds the explicit 403. Worth back-filling the same check on the purchase routes later.
