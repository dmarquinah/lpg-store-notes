---
last-updated: 2026-06-16
---
# Accounting — lpg-store

---
project: lpg-store
domain: specs
category: accounting
last-updated: 2026-06-15
---

## Context Documents

Read these before working on accounting specs:

- [[../../product/overview]] — the daily money flow (order entry → delivery → payment → debt; provider restock)
- [[../../eng/decisions]] — **ADR-012** (cross-module work = explicit service composition in one `db.transaction()`, no events/duplication), **ADR-007** (balance = `SUM` view), **ADR-009** (append-only ledger as audit trail), **ADR-013** (location-as-holder; provider `purchase` rows land on the `location` holder)
- [[../../eng/architecture]] — backend layout + module conventions
- [[../orders/orders-foundation/index]] — `order_payments` (the ingress ledger: `amount`, `method` ∈ cash/yape/plin/transfer, `occurred_at`, `recorded_by`), `orders.status='delivered'` = fulfilled, `orders.storeId`
- [[../inventory/inventory-foundation/index]] — provider purchases (the egress source): `tank_transactions`/`item_transactions` of `kind='purchase'`, valued at the holder's snapshot `purchase_price × qty`, plus the `purchase_surcharges` side table
- [[../stores/stores-and-catalog/index]] — `stores`, `store_assignments` (per-store scoping), `tank_types`/`inventory_items` prices
- [[../orders/orders-multi-location/index]] — `orders.storeId` + the `storeIdsForUser` / caller-scoping helpers every per-store read reuses

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[accounting-registry/index\|accounting-registry]] | done · backend ✓ frontend ✓ | Per-store **closing register** that groups a period's money events: auto-aggregates ingress (delivered-order payments by method) + egress (provider purchases + surcharges), allows free-form manual ingress/egress lines, and **closes** into a frozen snapshot. Foundation for the later weekly/monthly evaluation. |
| [[registry-source-drilldown/index\|registry-source-drilldown]] | done · backend ✓ frontend ✓ | Read-only **verification drill-down**: each registry block (ingress by method, egress provider) expands into its contributing source records grouped by **business day** (the manual "sum it day by day" check, in-place). New line-level read ports (composition, no duplicated queries); day subtotals reconcile to the block totals; **frozen into the snapshot** for closed registries (ADR-018 extended to detail). |
