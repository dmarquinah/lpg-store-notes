---
project: lpg-store
domain: specs
type: spec-track
spec: registry-source-drilldown
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-16
---

# Registry Source Drill-down — lpg-backend track

Shared spec: [[index]] · Aggregate ports it decomposes: [[../accounting-registry/backend]]

## Technical Notes

Read-only enrichment. The accounting module already composes two aggregate read
ports; this adds their **line-level siblings** and threads the detail through
the existing open-vs-snapshot branch.

### New read ports (composition, ADR-012)

Mirror the existing aggregate ports, but return rows instead of sums. Keep the
joins in the owning module:

- **Orders** — alongside `paymentsByMethodForStorePeriod`
  ([service.ts:523](../../../../lpg-backend/src/modules/orders/service.ts),
  [repository.ts:315](../../../../lpg-backend/src/modules/orders/repository.ts)),
  add `paymentLinesForStorePeriod(storeId, from, to)` → one row per
  `order_payments` of a `delivered` order in the store + window:
  `{orderId, customerName, amount, method, occurredAt}`. Same Lima `-05:00`
  business-date window and the same payment-date membership keying. Join
  `orders` → `customers` for the name (orders already joins users/customers
  elsewhere; reuse that pattern).
- **Inventory** — `purchaseLinesForStorePeriod(storeId, from, to)` **already
  exists** (built by [[../../inventory/provider-purchase-cost/backend]],
  2026-06-16): one row per `kind='purchase'` tank/item tx on the store's
  `location` holder — `{kind, txId, productId, productName, qty, unitCost,
  surcharge, occurredAt}`, `unitCost = COALESCE(tx.unit_cost,
  holder.purchase_price)`, names joined from `tank_types`/`inventory_items`,
  surcharge left-joined. **Reuse it** for the egress drill-down (group by
  business day, add `lineCost`); no new inventory port needed. The service-level
  `InventoryService.listStorePurchases` adds the `accounted` flag + scoping if a
  caller-facing variant is wanted.

Expose both via the structural read-port interfaces accounting already uses
(`AccountingIngressSource` / `AccountingEgressSource`) — extend those ports +
their fakes rather than introducing a new composition seam.

### Detail assembly (open) — day grouping

In the accounting service, after fetching the line ports for an **open**
registry, group both sides by **Lima business date** (`src/lib/date.ts` —
`toBusinessDate`, never raw UTC) into per-day buckets with a subtotal and the
day's records. Shape (illustrative):

```
drilldown: {
  ingress: [{ date, subtotal, byMethod, lines: [{orderId, customerName, amount, method}] }],
  egress:  [{ date, subtotal, lines: [{kind, productName, qty, unitCost, lineCost, surcharge}] }],
}
```

The drill-down decomposes the **same** numbers as `breakdown` — assert in a test
that `sum(ingress day subtotals)` equals `breakdown.ingress.operationalTotal`
and likewise for egress, so the two can never silently disagree.

### Detail assembly (closed) — freeze

The registry already freezes a `snapshot` jsonb on close (ADR-018). **Extend the
snapshot** to also persist the day-grouped drill-down (or enough to rebuild it).
On close, capture the line ports' output into the snapshot next to the totals;
the closed-registry detail reads the drill-down from the snapshot, never live.
This keeps a closed period's line detail consistent with its frozen totals even
if a back-dated payment/purchase lands afterward. Migration: only if the
snapshot column needs a shape note — the column is `jsonb`, so likely **no
migration**, just a richer payload (confirm the existing `snapshot` typing in
`schema.ts`/`types.ts`).

### Surface

Decide between (a) extending `RegistryDetailView` with a `drilldown` field on the
existing `GET /registries/:id`, or (b) a `GET /registries/:id/lines`
sub-resource (lighter default detail; lazy drill-down). Either way the route is
role-guarded identically to the detail (admin/dev global; operator own store;
`?storeId` intersected, cross-store → 404/403) and the closed case reads the
snapshot. See [[index]] Open Questions.

### Reuse / don't reinvent

- Business dates: `src/lib/date.ts` for every window boundary and day bucket.
- Money rounding: the orders `round2` pattern; `numeric(10,2)` strings → numbers
  at the view boundary (as `toRegistryView`/`toEntryView` already do).
- Scoping: the existing `storeIdsForUser` intersect used by the registry detail —
  do **not** re-implement scope here.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/orders/repository.ts` + `service.ts` — add
  `paymentLinesForStorePeriod` ([repository.ts:315](../../../../lpg-backend/src/modules/orders/repository.ts),
  [service.ts:523](../../../../lpg-backend/src/modules/orders/service.ts)).
- `src/modules/inventory/repository.ts` + `service.ts` — add
  `purchaseLinesForStorePeriod` ([repository.ts:455](../../../../lpg-backend/src/modules/inventory/repository.ts)).
- `src/modules/accounting/types.ts` — drill-down view types on
  `RegistryDetailView`; extend the `snapshot` payload type.
- `src/modules/accounting/service.ts` — assemble day-grouped drill-down (open) /
  read it from `snapshot` (closed); extend the close path to freeze it; extend
  the `AccountingIngressSource`/`AccountingEgressSource` ports + fakes.
- `src/modules/accounting/repository.ts` — persist the richer snapshot.
- `src/modules/accounting/routes.ts` — the detail (or new `/lines` sub-resource)
  route, role-guarded as today.
- `src/modules/accounting/__tests__/accounting.test.ts` — drill-down reconciles
  to totals; frozen for closed.

Read-only context:

- `src/lib/date.ts` — Lima business-date helpers for day bucketing.
- `src/modules/accounting/schema.ts` — existing `snapshot` jsonb column.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-backend] Backend track **done**. Read-only per-block, day-grouped drill-down over the existing registry, composed (ADR-012) — no duplicated queries, no migration.

**Open questions resolved at build start:** snapshot freezes the **full** line detail (~30–50 KB/fortnight, jsonb-fine); ingress = **per-day subtotal + lines** with method as a per-line column (no redundant per-day byMethod map); surface = a new **`GET /registries/:id/lines`** sub-resource (lazy, keeps the default detail light). Recorded in [[index]] Open Questions.

**New line-level read ports (composition):**
- **Orders** — `paymentLinesForStorePeriod(storeId, from, to)` (repository.ts + service delegate): one row per `order_payments` of a **delivered** order in the store + Lima `-05:00` window, **same predicate-for-predicate** as the aggregate `paymentsByMethodForStorePeriod` (membership keyed on the **payment's** date). Returns `{orderId, customerName, amount, method, occurredAt}`; `customerName = COALESCE(customers.name, orders.customer_name_snapshot, 'Cliente')` via the existing customers leftJoin. New `PaymentLineRow` type.
- **Inventory** — reused the **pre-existing** `purchaseLinesForStorePeriod` (from `provider-purchase-cost`); only added a thin **`InventoryService` delegate** so the service (used in `app.ts`) also satisfies the extended egress port (the fake repo already had it).
- Both threaded through the structural ports accounting already uses — `AccountingIngressSource` / `AccountingEgressSource` gained a `*LinesForStorePeriod` method (return shapes declared **inline** in `service.ts` — `IngressLineRow`/`EgressLineRow` — so accounting never imports orders/inventory row types).

**Drill-down assembly** (`accounting/service.ts`): pure module-level `groupIngress`/`groupEgress` bucket rows by **Lima business date** (`toBusinessDate`, never raw UTC), ascending days, per-day subtotal. Ingress day subtotal = Σ amount; egress day subtotal = Σ (`lineCost` + `surcharge`), `lineCost = unitCost × qty`. So Σ ingress subtotals == `breakdown.ingress.operationalTotal` and Σ egress == `breakdown.egress.providerTotal` (manual entries are **excluded** by design — they're the detail's `entries`). Rounding can't diverge: every money column is `numeric(scale:2)` and qty is integer, so per-line `round2` is a no-op and the per-line sum equals the SQL aggregate to the cent (asserted in the test).

**Freeze (ADR-018 extended to detail):** `RegistrySnapshot extends RegistryBreakdown { drilldown? }` — the drill-down rides on the **same `snapshot` jsonb column** as an extra field, so existing `as RegistryBreakdown` casts + the list-totals path keep working and **no migration / backfill** is needed. `closeRegistry` now freezes `{ ...breakdown, drilldown }`. `getRegistryLines` serves the frozen `snapshot.drilldown` for closed registries (**never recomputed** → can't drift; legacy pre-spec snapshots with no drilldown degrade to empty arrays) and computes live only while open.

**Surface & scoping:** new `GET /registries/:id/lines` (routes.ts) guarded by the **same `office` role** as the detail and scoped via the existing `requireVisibleRegistry` (cross-store → 404). No write routes; aggregate totals/net untouched. `AccountingRepository.closeRegistry(snapshot: unknown)` already stored any object → **repository unchanged**.

**Tests:** new test **(e)** in `accounting.test.ts` — open registry: a delivered payment + a provider purchase appear as drill-down lines under the right Lima day, excluded records (other store / out-of-window / non-delivered) absent, and Σ subtotals reconcile to `operationalTotal`/`providerTotal`; after close, a new in-window payment does **not** alter the frozen lines (still 2 days, sum 150). Added `paymentLinesForStorePeriod` to the orders fake. **96 tests total** (was 95). Independent validation: **all 5 backend criteria MET, no correctness bugs** (surcharge PK rules out fan-out; rounding exact under the 2-dp schema). Gates green: typecheck + `biome check` (88 files) + 96 tests + build.

**Frontend track remains** (expandable per-block UI consuming `GET /registries/:id/lines`).