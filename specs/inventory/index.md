---
last-updated: 2026-06-27
---
# Inventory — lpg-store

---
project: lpg-store
domain: specs
category: inventory
last-updated: 2026-05-07
---

## Context Documents

Read these before working on inventory specs:

- [[../../eng/architecture]] — backend layout and conventions
- [[../../eng/legacy-bloat-analysis]] — why v1 grew so much; the ten drivers v2 must avoid
- [[../../eng/decisions]] — ADR-005 through ADR-010 are the inventory-shaping decisions
- [[../../eng/patterns/module-template]] — vertical module folder convention
- [[../../product/overview]] — daily workflow, tank exchange model, MVP scope
- `lpg-backend/legacy/docs/inventory/PRD - Inventory.md` — v1 PRD (reference for requirements; ignore the implementation status sections)
- `lpg-backend/legacy/src/db/schemas/inventory/` — v1 schema (reference for entity shapes; v2 storage model differs per ADR-007)
- `lpg-backend/legacy/src/services/inventory/` — v1 service code (reference; do not port the strategy pattern)

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[inventory-foundation/index\|inventory-foundation]] | done · backend ✓ frontend ✓ | Ledger-first inventory with three-state assignment workflow, six transaction kinds, customer empty-debt ledger, reconciliation/discrepancy reporting. The product's core. |
| [[day-handoff/index\|day-handoff]] | done · backend ✓ frontend ✓ | Two-signature day-end: driver physically counts → `close` (mandatory count, driver-only), operator reviews → `carry`/Consolidar (operator-only). Start-of-day carry-forward defaults + stale prior-day guard. Distributes effort off the lone operator without touching the ledger. |
| [[multi-type-fulfillment/index\|multi-type-fulfillment]] | done · backend ✓ frontend ✓ | Fulfil orders spanning a tank type the driver/store doesn't hold (e.g. 1×10kg + 1×45kg): smart **"Comprar y cargar"** assign-resolution (load if the store has it, else provider-purchase into the store then load) + clearer store stock-out 409 (names the tank type, guides to a Compra). Plus per-type **Movimientos** legibility (Tanque column; `load`→"Carga" so the refill row isn't blank). Display/flow only — no ledger change. |
| [[provider-purchase-cost/index\|provider-purchase-cost]] | done · backend ✓ frontend ✓ | Editable **cost-in** captured per provider purchase: optional per-line `unitCost` on the tank/item purchase request (default = catalog `purchase_price`, catalog not mutated), persisted on a new nullable `unit_cost` tx column; accounting **egress** values purchases at `COALESCE(unit_cost, holder snapshot) × qty` so the books reflect the cheaper provider price instead of the sell price. Fixes the over-stated egress in the accounting registry. |
| [[stale-day-banner/index\|stale-day-banner]] | done · frontend ✓ | Surface **unresolved prior days** (non-`carried` assignments dated before today — stuck `open` or counted-but-not-consolidated `closed`) in a **persistent banner above the tabs** on `/inventario`, so the office sees at a glance what must be cleared before today can start. Frontend-only: derived from the existing caller-scoped `GET /assignments?state=open\|closed`, filtered to `date < today` client-side — no backend change. |
| [[store-stock-first/index\|store-stock-first]] | done · backend ✓ frontend ✓ | Re-lead `/inventario` with the store's standing stock: store-stock becomes the **default first tab** (rename `Disponibilidad` → "Stock de tienda"), assignments/consolidation demoted. Plus a **never-blank zero-state** — list every active catalog tank type at 0/0 (merge `catalog.tankTypes` with availability) so a store with no `location` holders no longer shows a confusingly blank table (the observed "Stock en tienda empty" symptom). Now expanded (owner, 2026-06-27) to a **multi-location overview**: `/inventario` leads with a Resumen card per store the user oversees (from a new caller-scoped aggregate `GET /inventory/availability`); the old Disponibilidad becomes a per-store detail route `/inventario/tiendas/:id` where the never-blank catalog-merged table lives. **Backend track** (the aggregate endpoint) is implemented in lpg-backend; frontend track here. |
| [[store-stock-adjustments/index\|store-stock-adjustments]] | done · backend ✓ frontend ✓ | Set/correct a store's **standing full+empty stock directly**, in one step — no provider purchase, no open→adjust→close→carry dance. New `POST /inventory/stores/:storeId/adjustments` writes `adjustment`-kind rows on the store's `location` holder (signed deltas, required reason, actor+timestamp captured; **no** accounting egress); an **"Ajustar stock"** dialog (absolute or ±) on the store-stock surface. Reuses the existing holder + `adjustment` kind — no schema change. |
| [[store-history/index\|store-history]] | done · backend ✓ frontend ✓ | A **per-store history book**: read-only, newest-first stream of all `location`-holder tank+item movements (purchase/opening/load/sale/return/adjustment/carry) with **actor + timestamp + note**, composed from existing ledgers (no schema change). New `GET /inventory/stores/:storeId/history` + a **Movimientos / Historial** surface; **order events lazily fetched** on a button so the base request stays lean. |
| [[store-detail-products/index\|store-detail-products]] | done · backend ✓ frontend ✓ | Make `/inventario/tiendas/:id` **purchased-only + item-aware + tabbed**: tank stock becomes **holder-based** (purchased products incl. those at 0; never-purchased hidden), the page splits into **Balones · Artículos · Compras** tabs (no long scroll), and **Artículos** + **Compras** **lazy-load** on first open. **Backend track** = a new `GET /inventory/stores/:storeId/item-availability` (location item balances, holder-based) for the Artículos tab. |

## Edge cases the architecture must natively express

These were validated against v1 and the user's domain knowledge before drafting `inventory-foundation`. Any spec that touches inventory must remain compatible with all of them:

- **E1** Take 1 full, return 0 empty (debt accrues): inventory `{−1 full, 0 empty}`; customer `+1 empty owed (type X)`.
- **E2** Take 1 full, return 1 empty (normal swap): inventory `{−1 full, +1 empty}`; customer unchanged.
- **E3** Take 1 full, return 2 empties (settles prior debt): inventory `{−1 full, +2 empty}`; customer `−1 empty owed`.
- **E4** Walk-in cash sale, no customer record: same as E1 on inventory; nothing on the customer side.
- **E5** Customer brings empty back later, no purchase: inventory `{0 full, +1 empty}`; customer `−1 empty owed`.
- **E6** Driver receives day-opening assignment: paired `opening` transfer — `{−full}` on the store's `location` holder, `{+full}` on the driver's `assignment` holder (ADR-013).
- **E7** Failed delivery returns tanks to driver: reversal transaction(s) on the same assignment.
- **E8** Supplier delivers fulls in exchange for empties: `purchase` row on the store's `location` holder, not tied to any driver (ADR-013).
- **E9** Damaged/lost tank: `adjustment` ledger row with note.
- **E10** Day-end consolidation: `carry` hands the truck's leftovers back to the **location**; the operator then restocks from the provider and **opens the next day explicitly** with chosen quantities — so next-day opening need **not** equal today's closing (ADR-014). Empty-debt does **not** carry (lives on customers, not assignments).
- **E11** Reconciliation discrepancy at day-end: physical count vs ledger mismatch logged as `reconciliation` transaction.
