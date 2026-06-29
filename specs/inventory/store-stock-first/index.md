---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../inventory-foundation/index]]"
  - "[[../stale-day-banner/index]]"
  - "[[../day-handoff/index]]"
last-updated: 2026-06-27
---

# Spec: Store Stock First (multi-location inventory overview)

## Problem Statement

On `/inventario` the operator/admin's **primary** concern — *"what does each of my
stores hold right now, and what's happening there today?"* — is the hardest thing
to reach. The page leads with the **Asignaciones** tab (a daily driver-assignment
log, a *complementary* concern), and a store's actual stock lives under a third
tab labelled **"Disponibilidad"** — a word that doesn't say "this is your store's
inventory", behind a **single-store picker** that forces the user to inspect one
location at a time. An owner/admin who oversees several branches has **no
at-a-glance board** of all their locations; an operator's own store's state is two
clicks deep behind an unhelpful label.

Two concrete failures observed:

1. **No cross-location view.** Availability is per-store (`GET
   /inventory/stores/:id/availability`), surfaced through a picker. To compare
   branches the user re-picks, re-reads, repeats. The mental model is
   *all-my-locations first*; the UI is *one-store-if-you-find-the-tab*.
2. **A confusingly blank table.** The **"Stock en tienda"** table renders only
   tank types that already have a `location` holder for the store (the backend
   `tank_balance` view `LEFT JOIN`s *existing* holders). A store never credited
   any standing stock shows a **completely empty table** — indistinguishable, to
   the user, from a bug. (This is the "Stock en tienda was empty" symptom from
   testing. It is **not** data loss: draining a store's tanks into a delivery
   leaves the holder row at **0/0**; an empty table means *no holder exists yet*.
   The missing "give a store an opening balance" capability is
   [[../store-stock-adjustments/index|store-stock-adjustments]].)

The office needs the **daily state of every location it's responsible for** to be
the first thing on the page, and a store with no stock yet must read as **0**, not
a blank.

## Proposed Solution

Re-lead `/inventario` with a **"Resumen" overview of every location the user is
associated with** (operator → their stores; admin/developer → all), backed by a
new aggregate endpoint, and move the single-store deep view to a dedicated detail
route. (Owner decisions 2026-06-27.)

- **New backend aggregate endpoint** (track [[backend]], implemented in
  `lpg-backend`): `GET /inventory/availability` returns, in **one caller-scoped
  request**, each store's `{ storeId, storeName, shop[], onTruck[] }`. This
  replaces N per-store calls and is the data behind the overview. A store with no
  holders still appears (empty arrays) so the UI renders it at 0.
- **Resumen overview (default tab).** A responsive grid of **location cards**, one
  per store: headline **Llenos / Vacíos** totals + a per-type breakdown of what
  the store holds, an **"En vehículos"** total, **today's** assignment(s) (driver
  + day-state badge, or "Sin asignación · Abrir día"), and quick actions (**Ver ·
  Registrar compra · Abrir día**). Never-blank: a store with no stock reads 0.
- **Tabs reordered + renamed:** `[ Resumen | Asignaciones | Por consolidar ]`. The
  "Disponibilidad" tab is **retired** — its content becomes the per-store detail
  route. The stale-day banner stays above the tabs.
- **Dedicated per-store detail route** `/inventario/tiendas/:id`: the deep
  single-store view (never-blank per-type stock table merging the catalog tank
  types, On-vehículos, Compras al proveedor + cost-correction, "Registrar
  compra"). A card's **Ver** navigates here.
- **Preserve** operator store-scoping, the stale-day banner
  ([[../stale-day-banner/index|stale-day-banner]] not regressed), purchases +
  cost-correction, and the design system.

Backend contract in [[backend]] (built separately); the overview + detail route in
[[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

**Backend (lpg-backend) — implemented in its own repo:**

- [ ] New `GET /api/v1/inventory/availability` returns one entry per the caller's
      scoped stores: `{ storeId, storeName, shop: {tankTypeId, fullTanks,
      emptyTanks}[], onTruck: {…}[] }`. Caller-scoped: admin/developer → all
      active stores; operator → only `storeIdsForUser`. Role-guarded to
      operator/admin (developer alias).
- [ ] Optional `?storeId=` narrows the result, **intersected** with the caller's
      scope (an id outside scope yields no entry — never a widening grant).
- [ ] A scoped store with **no `location` holders still appears** (empty `shop` /
      `onTruck` arrays) so the overview can render it at 0 — the aggregate never
      silently drops a store.
- [ ] Implemented with **grouped queries** across the caller's stores (one shop
      read + one on-truck read), not an N-loop; on-truck mirrors the existing
      open-assignment logic.
- [ ] Tests cover scoping (operator → own only; admin → all; `?storeId=`
      intersect), the response shape, and the empty-store-included case; existing
      tests stay green; typecheck / biome / build green.

**Frontend (lpg-frontend-vue):**

- [ ] `/inventario` leads with a **Resumen** tab (default) rendering **one
      location card per store the user is associated with** (operator → their
      stores; admin/developer → all), from the new aggregate endpoint. Each card
      shows: headline **Llenos / Vacíos** totals, a **per-type list** of the types
      the store holds, an **"En vehículos"** total, **today's** assignment(s)
      (driver + state badge) or a **"Sin asignación · Abrir día"** affordance, and
      actions **Ver · Registrar compra · Abrir día**.
- [ ] **Never-blank:** a store with no stock renders as **0 / 0** (the card is
      present, its per-type area reads "sin stock aún") — never an absent/blank
      card.
- [ ] Tabs are reordered/renamed to **[ Resumen · Asignaciones · Por consolidar ]**;
      the **"Disponibilidad"** tab is retired. The **stale-day banner** still
      renders above the tabs ([[../stale-day-banner/index|stale-day-banner]] not
      regressed); Asignaciones + Por consolidar keep their content + the
      consolidation badge count.
- [ ] A new per-store detail route **`/inventario/tiendas/:id`** shows that store's
      deep view: a **never-blank** per-type stock table (every **active catalog
      tank type**, 0/0 if untouched, by merging `catalog.tankTypes`), On-vehículos,
      **Compras al proveedor** + cost-correction, and **"Registrar compra"**. A
      card's **Ver** navigates here; the route is operator/admin/developer-gated
      like the page.
- [ ] Today's assignment status on the cards is **decoupled** from the Asignaciones
      tab's date/state filter (a dedicated today fetch), so changing that filter
      does not alter the overview.
- [ ] Operator store-scoping, role nav, and the route's `meta.roles` are unchanged;
      design-system compliant (ResponsiveTable, tokens, ≥44px — `eng/design-system.md`);
      `npm run typecheck` + `npm run build` green. Manual smoke (cards per store,
      0-state, drill-in) once the backend endpoint is live.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Giving a store an opening/standing balance** — that's
  [[../store-stock-adjustments/index|store-stock-adjustments]]. This spec presents
  stock (including a true 0 state) and adds the aggregate; it doesn't add a way to
  *set* stock.
- **The per-store history / Movimientos tab** — that's
  [[../store-history/index|store-history]]; it will hang off the per-store detail
  route this spec creates.
- **Cross-store roll-ups / analytics** (totals across all branches, charts). The
  overview is per-location cards; an aggregate dashboard is a later idea.
- **Changing how balances are computed.** The aggregate composes the existing
  `tank_balance` view; no view/migration change.

## Open Questions

- **`includeTrucks` on the aggregate.** Proposed: the aggregate **always** returns
  `onTruck` (the cards show "En vehículos"); no param needed. Revisit if the
  on-truck read proves costly at scale.
- **Card per-type density.** Proposed: list only the types the store actually holds
  (non-zero) on the card, with the full every-active-type table reserved for the
  detail route. Confirm during build if owners want all types on the card too.
