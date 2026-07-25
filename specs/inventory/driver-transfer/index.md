---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on:
  - "[[../driver-inventory-pools/index]]"
last-updated: 2026-07-20
---

# Spec: Driver Transfer — move a driver between stores with their inventory, active day, and orders (duplicate-store consolidation)

## Problem Statement

Admins created **virtual duplicate stores** to get the old single-driver
quick-assign. With per-driver pools ([[../driver-inventory-pools/index]]) that
hack is no longer needed, and the shipped **"Agregar repartidor"** flow
(`seed-driver` + link create/retire, delivered as a follow-on to
`driver-inventory-pools`) lets an admin move a driver + their standing inventory
into the real store. But testing surfaced three defects in that move:

1. **Orphaned open days.** `seed-driver` empties the driver's old day's truck but
   leaves the **day `open`**, still pointing at the now-retired link. The
   `/inventario` "Hoy" section lists today's inventory days per store regardless
   of link-active, so the retired-link day keeps showing the driver's name in the
   **old** store — the driver appears **twice** (old + new) after a transfer, and
   again per hop on a round-trip.
2. **Duplicate links.** `createStoreAssignment` only blocks a duplicate **active**
   link, so re-adding a driver to a store they previously worked in inserts a
   **second** `store_assignments` row instead of reactivating the inactive one.
3. **Orders left behind.** The driver's not-yet-delivered **orders** stay in the
   old store (their `store_id` unchanged) and were bound to the day whose truck we
   zeroed — stranded and showing the wrong location.

## Proposed Solution

Make the carry a **true move of the driver's whole active context**, not a
copy-stock-and-orphan-the-day:

- **Move the active day, don't zero-and-orphan it.** Re-point each of the
  driver's **unconsolidated** days (`inventory_assignments.store_assignment_id`)
  to the **new** link. The truck stock and the day's orders ride along untouched.
  This removes the orphan from the old store's "Hoy", preserves order statuses
  (they stay bound to the same day, which is now in the new store and still holds
  the stock to deliver), and moves the orders to the new store.
- **Move the day's orders' `store_id`** to the target store — for **all** orders
  on the moved days, delivered included. (Owner confirmed: moving already-delivered
  orders/sales to the target is desired — the duplicate source store will be
  deleted, so full consolidation is correct.) Cross-module via an injected orders
  port (ADR-012 — inventory never imports the orders module).
- **Reactivate, don't duplicate.** Adding a driver to a store reactivates an
  existing **inactive** link for `(store, user)` instead of inserting a new row.
- **Standing stock (pool)** moves as today (`seed-driver` copy+zero, pool only) —
  the day's truck is no longer folded into the pool (the day itself moves).
- **Retire the source link** after the move (as today).

Manual onboarding (a brand-new driver, no source store) is unchanged.

Backend contract + orders port in [[backend]]; the "Agregar repartidor" carry
orchestration + view refresh in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

- [ ] After transferring a driver from store S → T, **no** `open`/non-consolidated
      day for that driver remains under **S** (no orphaned day, no ghost driver in
      S's "Hoy"). The driver appears exactly once, under **T**.
- [ ] The driver's unconsolidated day(s) are **re-pointed** to the target link —
      truck stock preserved, deliverable from the day after the move.
- [ ] **Every order** on the moved days has its `store_id` set to the target
      store; **assigned / in-transit** orders keep their status and `assignment_id`
      (still deliverable); delivered/failed/cancelled move too (their `store_id`
      follows the day) but are otherwise untouched. A status-history line records
      the move for non-terminal orders.
- [ ] Adding a driver to a store **reactivates** an existing inactive
      `(store, user)` link instead of creating a duplicate row; no store
      accumulates `inactive + active` link pairs from repeated transfers.
- [ ] Standing stock (pool) is moved to the target; the source driver's pool +
      re-pointed days leave the source holding nothing for that driver.
- [ ] **Admin / developer only** at every layer (route `requireRole('admin')` +
      service `isGlobal` guard + UI gate); operator/delivery cannot reach it.
- [ ] `/inventario` Resumen + store-detail Asignaciones refresh after a transfer
      (no stale duplicate display); a round-trip (S→T→S) leaves a clean single
      driver + single active link + single day.
- [ ] **Gates:** `npm run typecheck` + `npm run check` + `npm test` + `npm run
      build` green in lpg-backend; `typecheck` + `build` green in
      lpg-frontend-vue. New tests: day re-point, order `store_id` move (+ status
      preserved for non-terminal), link reactivation, admin-only.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Full multi-store merge** beyond a single driver — customers are global
  (unaffected); accounting registries are not merged (though a moved day's
  delivered-order sales follow it to the target, which is the desired
  consolidation for a disposable duplicate store).
- **Blocking / special-casing in-transit** — the move carries in-transit orders
  with status preserved (re-pointed day), so no guard is needed.
- **Deleting the duplicate store** itself — the admin does that manually once its
  drivers/inventory/active orders have been consolidated out.

## Notes

- Supersedes the orphan-leaving behavior of the shipped `seed-driver` carry
  (documented in [[../driver-inventory-pools/frontend]] /
  [[../driver-inventory-pools/backend]]); `seed-driver` stays for **manual**
  onboarding.
- A one-off test-DB cleanup (deleting two orphaned open days + confirming no
  duplicate active links) was performed 2026-07-20 while diagnosing defect #1.
