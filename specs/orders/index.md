---
last-updated: 2026-06-14
---
# Orders — lpg-store

---
project: lpg-store
domain: specs
category: orders
last-updated: 2026-06-08
---

## Context Documents

Read these before working on orders specs:

- [[../../product/overview]] — daily workflow (phone-driven order entry → delivery → debt), tank-exchange model, MVP scope
- [[../../eng/decisions]] — **ADR-012** (cross-module work = explicit service composition in one `db.transaction()`), **ADR-010** (empty-debt as a signed-delta customer ledger), **ADR-007** (balance = SUM view), **ADR-013/014/015** (holder model; inventory never negative; today-only open)
- [[../../eng/architecture]] — backend layout, module conventions; "What v2 deliberately drops from v1" (the 998-LOC order workflow repo)
- [[../../eng/legacy-bloat-analysis]] — why v1 grew; patterns orders must avoid
- [[../inventory/inventory-foundation/index]] — the ledger orders writes through (`recordSale`/`recordReturn`, holders, E1–E11)
- [[../inventory/index]] — the **E1–E11 edge-case catalogue** every inventory-touching spec must remain compatible with
- [[../customers/customers-crud/index]] — the registry orders attaches to + the `customer_debts` table orders is the writer of
- [[../stores/stores-and-catalog/index]] — `tank_types`/`inventory_items` (prices), `store_assignments`
- `lpg-backend/legacy/docs/orders/PRD - Orders.md` — v1 PRD (reference for requirements; ignore implementation-status sections)
- `lpg-backend/legacy/src/{db/schemas,services,routes}/orders/` — v1 order code (reference for shapes/rules; **do not** port reservations, the 998-LOC workflow repo, invoices, or the strategy pattern)

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[orders-foundation/index\|orders-foundation]] | done · backend ✓ frontend ✓ | Order lifecycle `pending → assigned → in_transit → delivered` (+cancelled/failed). No reservations — inventory moves at delivery via `inventory.recordSale`/`recordItemSale` in one `db.transaction()`. Catalog-default pricing (override-able), partial payments (new `order_payments` ledger), monetary debt written to `customer_debts`. The business's money-recording workflow. |
| [[orders-queue-history-search/index\|orders-queue-history-search]] | approved · backend ◻ frontend ◻ | Make the growing queue manageable: **Activos/Historial** split (active queue by default; finished orders by **date range**), free-text **customer search** on `GET /orders`, and an **accent-insensitive** customer search fix (`unaccent`) that also repairs the order-creation picker ("Maria" ⇄ "María"). |
| [[orders-multi-location/index\|orders-multi-location]] | done · backend ✓ frontend ✓ | Adds a location + ownership dimension to orders: `orders.storeId` (owning branch, set at create) + `ownerId` (= creator; pre-delivery management owner-gated); store-scoped operators via `store_assignments` (admin = global); `POST /:id/transfer` handoff to another branch; **atomic conditional state transitions** (409 on lost-update). Fixes multi-operator coordination + the true assign race. |
