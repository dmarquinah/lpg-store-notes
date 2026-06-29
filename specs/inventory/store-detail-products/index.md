---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../store-stock-first/index]]"
  - "[[../inventory-foundation/index]]"
last-updated: 2026-06-28
---

# Spec: Store Detail Products (purchased-only stock, items, tabbed)

## Problem Statement

The per-store detail view `/inventario/tiendas/:id` (from
[[../store-stock-first/index|store-stock-first]]) has three gaps the owner hit in
testing:

1. **Shows products that were never purchased.** The Stock en tienda table merges
   in **every active catalog tank type** at 0/0. The owner only wants the products
   the store **actually deals in** — i.e. anything it has **already purchased**
   (has a `location` holder), *including* ones depleted back to 0. A type the
   store has **never purchased** should **not** appear at all. (The
   `tank_balance` view already returns a holder row at 0 for a purchased-then-
   drained product, and nothing for a never-purchased one — so the fix is to stop
   the catalog merge and read holder-based balances directly.)
2. **No item (accessory) stock.** A store stocks **items** too (e.g. Manguera),
   but the detail page shows only tanks. There is currently **no API** for a
   store's location **item** balances (`item_balance` is only queried per
   assignment) — so this needs a small backend read.
3. **One long scroll.** Stock en tienda + En vehículos + Compras al proveedor are
   stacked; as the catalog grows this becomes a lot of scrolling.

## Proposed Solution

Make the detail page **purchased-only**, **tabbed**, and **item-aware**, loading
the heavier tabs lazily.

- **Holder-based tank stock (frontend).** Drop the catalog merge; bind the Stock
  en tienda table to the store's **holder balances** (`availability.shop`). A
  purchased-then-0 product shows at 0; a never-purchased one is absent. The
  empty-state reads as "no purchases yet", not "blank".
- **Tabbed layout (frontend):** **Balones · Artículos · Compras**. Each tab is
  short, so the page never becomes one long scroll regardless of catalog size.
- **Lazy loading (frontend):** the **Artículos** and **Compras** tabs fetch their
  data only **on first open** (not on page mount) — the default Balones tab loads
  the tank availability the page already had.
- **Per-store item availability (backend, track [[backend]]):** a new
  `GET /inventory/stores/:storeId/item-availability` returning the store's
  **location item balances** from the `item_balance` view — holder-based, same
  rule as tanks (purchased-incl-0 shown, never-purchased absent). The Artículos
  tab consumes it.

Backend contract in [[backend]]; the tabs + holder-based stock + lazy loads in
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Backend (lpg-backend) — implemented in its own repo:**

- [ ] New `GET /api/v1/inventory/stores/:storeId/item-availability` returns
      `{ items: [{ inventoryItemId, qty }] }` from the **`item_balance` view**
      filtered to **`kind='location'`** for that store — **holder-based**: a
      never-purchased item is **absent**; a purchased-then-0 item appears at
      `qty: 0`. Auth mirrors the existing tank availability route
      (`/stores/:storeId/availability`).
- [ ] Repo read `itemBalancesByLocation(storeId)` over `item_balance`; service
      maps rows to `{ inventoryItemId, qty }` (mirrors `toTankRows`). Tests cover
      never-purchased-absent + purchased-then-0-present; existing tests stay green;
      typecheck / biome / build green.

**Frontend (lpg-frontend-vue):**

- [ ] The Stock en tienda (tanks) table on `/inventario/tiendas/:id` is
      **holder-based** (`availability.shop` directly, **no catalog merge**):
      products the store has purchased show (incl. those at 0); never-purchased
      tank types do **not** appear. Empty-state copy reflects "no purchases yet"
      (not a generic blank).
- [ ] The detail page is restructured into **Balones · Artículos · Compras** tabs;
      no single long scroll regardless of how many products exist.
- [ ] **Artículos** and **Compras** tabs **lazy-load** on first open (no fetch
      until the tab is activated); the default Balones tab keeps loading tank
      availability + En vehículos on mount.
- [ ] The **Artículos** tab lists the store's **item stock** (item name + qty)
      from the new endpoint — holder-based (purchased-incl-0 shown,
      never-purchased absent); item names resolve via the catalog.
- [ ] "Registrar compra" + cost-correction still work (purchases now live in the
      Compras tab); navigating between stores resets to the Balones tab + the
      correct store's data.
- [ ] `npm run typecheck` + `npm run build` green. Manual smoke (purchased-only
      tanks, item stock under Artículos, lazy tabs) left to the operator once the
      backend endpoint is live.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Order-create step 2 product picker.** The same "don't make me scroll the whole
  catalog" theme is being addressed there too (category tabs + compact rows) as a
  separate frontend follow-on in the `orders` module — not part of this spec.
- **On-truck item balances.** Only location item stock is exposed here; items on a
  driver's truck are out of scope.
- **Setting a store's standing stock** — that's
  [[../store-stock-adjustments/index|store-stock-adjustments]].
- **Aggregating items into the overview cards.** The Resumen cards stay tank-only;
  item stock lives on the detail page.

## Open Questions

- **Auth on the item-availability endpoint.** Proposed: mirror the existing tank
  `/stores/:storeId/availability` (authenticated, unscoped) for consistency.
  Tighten to operator-own-store/admin later if store reads get scoped across the
  board.
- **Lazy re-fetch policy.** Proposed: fetch on each tab open (cheap GET, always
  fresh); switch to a per-store cache only if it proves chatty.
