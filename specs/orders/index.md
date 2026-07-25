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
| [[orders-queue-history-search/index\|orders-queue-history-search]] | done · backend ✓ frontend ✓ | Make the growing queue manageable: **Activos/Historial** split (active queue by default; finished orders by **date range**), free-text **customer search** on `GET /orders`, and an **accent-insensitive** customer search fix (`unaccent`) that also repairs the order-creation picker ("Maria" ⇄ "María"). |
| [[orders-multi-location/index\|orders-multi-location]] | done · backend ✓ frontend ✓ | Adds a location + ownership dimension to orders: `orders.storeId` (owning branch, set at create) + `ownerId` (= creator; pre-delivery management owner-gated); store-scoped operators via `store_assignments` (admin = global); `POST /:id/transfer` handoff to another branch; **atomic conditional state transitions** (409 on lost-update). Fixes multi-operator coordination + the true assign race. |
| [[order-event-timeline/index\|order-event-timeline]] | done · backend ✓ frontend ✓ | Unify the order detail's separate **Historial** + **Pagos** cards into one full-width **horizontal timeline** merging status changes and payments by datetime, each node naming the actor(s). Backend enriches the detail with actor **names** (role-safe; `GET /users` is admin-only) + a new nullable `order_status_history.target_user_id` populated on assign so the **Asignado** node shows operator → driver. |
| [[single-delivery-auto-assign/index\|single-delivery-auto-assign]] | draft · backend ◻ frontend ◻ | **Backlog / deferred.** When an order is created for a store with **exactly one** active delivery, **auto-assign** that driver via the existing assign transition (same "Asignado" history node); 0/>1 drivers → unchanged. The "maybe expand" of [[../inventory/quick-assignment/index\|quick-assignment]] — split off so the inventory work ships focused. |
| [[copy-order-share/index\|copy-order-share]] | done · frontend ✓ | **Frontend-only.** A **"Copiar datos"** button on the order detail (and optionally the create-order success step) that formats the full order into a phone-legible Spanish text block and copies it to the clipboard, so the admin can relay orders to still-learning workers over WhatsApp. No backend change — all data is already on the detail payload. |
| [[delivery-record-payment/index\|delivery-record-payment]] | done · backend ✓ frontend ✓ | Let the **delivery** role record a payment/collection (cobro) against an already-delivered order's pending debt — like operators/admins but **only for orders on their own assignments**. Widens the `POST /:id/payments` guard + reuses the existing `assertVisible` ownership boundary; no new endpoint, no schema change. Frontend exposes the existing cobro affordance to drivers. |
| [[deliver-rehome-open-day/index\|deliver-rehome-open-day]] | draft · backend ◻ | **Auto-rehome a stale-day order to today's open inventory on deliver.** When an order's bound driver-day was closed/consolidated before it was delivered, `deliverOrder` auto-rebinds it to the **same driver+store's** open day today (matched by `storeAssignmentId`) and records the sale there — instead of the dead-end "reasigne el pedido" error. No open day today → clear 409. Atomic rebind via the delivered transition patch; timeline reason notes the rehome. Backend-only, no migration. |
| [[delivery-gps-location/index\|delivery-gps-location]] | done · backend ✓ frontend ✓ | Driver **pins the exact GPS point** at delivery so future orders to the same customer can navigate via **Google Maps**. Final model: a **single sticky pin on the customer only** — nullable `lat/lng` on `customers` (migration `0018`, no `orders` columns); optional capture at `deliver` saves it **fill-when-empty / explicit-overwrite** (never drifts per delivery); office set/correct via `PATCH /customers/:id`; order read exposes the resolved pin for the Maps link. Multiple-locations-per-customer deferred. |
