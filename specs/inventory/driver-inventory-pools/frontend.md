---
project: lpg-store
domain: specs
type: spec-track
spec: driver-inventory-pools
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: '"2026-07-18"'
---

# Per-Driver Inventory Pools — lpg-frontend-vue track

Shared spec: [[index]]

> Ports **after** the backend track is `done` (per the project porting order).
> The backend re-keys inventory to per-driver pools and exposes a per-driver
> breakdown + `storeAssignmentId` targeting; this track surfaces it.

## Technical Notes

The single-driver experience must be **pixel-identical** to today — all new UI is
gated on a store having **>1** delivery driver.

- **Resumen cards (`/inventario`, `InventoryView`).** LLENOS / VACÍOS and "En
  vehículos" show the **store total** (Σ driver pools) — unchanged for
  single-driver. For a **multi-driver** store, replace the single quick button
  with **one "Asignar stock del piso → {driver}" button per not-yet-opened
  driver** (identity = which button, a single click; see the mockup direction in
  the spec). A driver whose day is open renders an **Activo** chip instead of a
  button. A **"Repartir manualmente…"** link opens the existing general
  `OpenDayDialog` for the split-a-shared-pile case.
- **Quick-open call** passes the chosen driver's `storeAssignmentId`.
- **Store detail (`/inventario/tiendas/:id`).** Add a **per-driver breakdown** of
  the store's stock (each driver's pool), above/beside the existing
  catalog-merged totals table.
- **Purchase + adjust dialogs.** Add a **driver selector** shown **only** when the
  store has >1 delivery driver (single-driver auto-resolves server-side → no new
  control). The purchase auto-load toast/refresh already reflects the driver.
- **De-dupe messaging.** With per-driver assign live, the store-duplication
  workaround is no longer needed; no explicit migration UI here (existing
  duplicate stores keep working as separate stores).

Design-system: reuse `PageHeader` / card patterns / `vue-sonner` toast; tokens +
≥44px targets; the per-driver button list must stay legible on phones (stack, not
crowd).

## Related Files

<!-- Confirm exact paths from within lpg-frontend-vue at /focus; this repo's
     inventory module mirrors the backend module layout. -->

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/views/InventoryView.vue` — Resumen cards: per-driver
  assign buttons + Σ totals + Activo chips.
- `src/modules/inventory/views/…StoreDetail…` — per-driver stock breakdown.
- `src/modules/inventory/components/OpenDayDialog.*` — manual-split entry (kept).
- Purchase + adjust dialog components — conditional driver selector (>1 driver).
- `src/modules/inventory/api.*` / store — `quick-open` / purchase / adjust
  payloads gain `storeAssignmentId`; availability read consumes the per-driver
  breakdown.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
[2026-07-18] [lpg-frontend-vue] Frontend track implemented (surfaces the per-driver pool model; single-driver UX unchanged).

- **Plumbing** (`types.ts`, `service.ts`, `store.ts`): added optional `storeAssignmentId` to `TankPurchasePayload`/`ItemPurchasePayload` (via `ProviderPurchaseFields`) + `LocationAdjustmentPayload`. New `LocationPool` type + `PoolBreakdownResponse`. `service.quickOpen(storeId, storeAssignmentId?)` now posts `{ storeAssignmentId }` when given; new `service.getPoolBreakdown(storeId)` → `GET …/pool-breakdown`. Store: `quickOpen(storeId, storeAssignmentId?)`, `poolBreakdown` state + `fetchPoolBreakdown` (view) + non-mutating `getPoolBreakdown` (dialogs).
- **Resumen cards** (`InventoryView.vue`): store totals (Σ pools) unchanged. `deliveriesByStore` groups active delivery SAs per store. **Single-driver (count===1): markup unchanged** (one-tap sweep + Ver). **Multi-driver (≥2): one "Asignar todo el stock a {driver}" button per not-yet-opened driver** (`assignableDrivers` = drivers with no today-assignment; each posts that driver's `storeAssignmentId`, sweeping only their own pool — never "driver B gets 0"), plus a "Repartir manualmente" entry to the general `OpenDayDialog`; drivers with a day already show their state chip under "Hoy". **0-driver: Ver + Abrir día.** Quick-open spinner re-keyed from storeId → `quickOpeningSa` (storeAssignmentId) so one spinning button can't mislabel another.
- **OpenDayDialog** (correctness): the backend `openDay`/`load` source from the driver's OWN pool, but `getShopBalances` returns the store TOTAL. New `loadValidationBalances(saId)` validates the initial load against the **selected driver's own pool** (from `pool-breakdown`) on multi-driver stores; single-driver keeps `getShopBalances` (pool == total → unchanged). Hint/error wording flips to "pool del repartidor" only on multi. Stale-response guards on every await.
- **PurchaseDialog + StockAdjustmentDialog**: a driver `Select` shown **only when the store has >1 active delivery driver**; sends `storeAssignmentId` (single-driver omits → backend sole-resolves). Purchase auto-load notice/toast generalised from "sole driver" to the **attributed** driver (selected on multi). Adjust dialog rebuilds its "current" totals against the **selected driver's pool** (via `pool-breakdown`) so "set total"/negative-guard are pool-relative, not store-total-relative.
- **StoreDetailView**: new "Stock por repartidor" section on the Balones tab (each driver's pool + a read-only "Sin asignar (piso)" parking row), shown only when the store runs >1 pool; refreshed after purchase/adjust. Store total table unchanged.
- **Out of scope (per track):** no pool-transfer write UI (parking shown read-only; manual split = OpenDayDialog); duplicate-store merge deferred.
- **Known edge (low severity, out of the single-driver assumption):** a store with exactly one *active* delivery driver that ALSO holds parking / deactivated-driver pool stock validates the OpenDayDialog against the store total; the backend still rejects an over-load with a clear 409 (no corruption). Not reachable from the UI in normal single-driver operation (single-driver purchases/adjusts auto-resolve to the driver, never parking).
- **Validation:** independent agent audit — all 6 frontend acceptance criteria met, no functional bugs (spinner keying, stale-response races, selector leakage, `assignableDrivers` filter, payload attribution all correct).
- **Gates:** `npm run typecheck` ✓ · `npm run build` ✓ (InventoryView 25.5 kB, StoreDetailView 29.7 kB gzip'd within range).

[2026-07-18] [lpg-frontend-vue] Follow-on (user-directed, post-`done`): reworked the assignment UX around the per-driver model after operator feedback, and fixed the order-assign purchase attribution.

- **Resumen cards (`/inventario`)**: replaced the per-driver assign buttons with a SINGLE **"Abrir día"** that opens the whole crew via the new `quick-open-all` endpoint (each driver from their own pool; already-open drivers skipped, never fails). Label names the sole assignable driver or the count; hidden when none assignable. Toast summarizes opened/skipped. (The earlier per-driver-button design read as "assign the whole store to whoever I click first" — the single crew action removes that ambiguity.)
- **Store detail (`/inventario/tiendas/:id`)**: replaced the **Balones** tab with an **Asignaciones** tab (now the default) — per-driver cards (today's day-state chip or "Sin abrir hoy", each driver's own pool, a per-driver "Abrir día" / "Ver día" link), a read-only **"Sin asignar (piso)"** parking pool, plus the store totals + on-truck tables. Refreshes after quick-open / purchase / adjust.
- **Purchase + adjust dialogs**: the multi-driver driver selector is now **OPTIONAL** with a **"Sin asignar (parqueo)"** default (`PARKING` sentinel) — a real driver attributes to their pool, parking (or a single-driver store's auto-resolve) sends nothing. The adjust dialog rebuilds its "current" totals against the selected pool (driver or parking) via a `pools` ref + `effectiveCurrent()`.
- **AssignDialog "Abastecer y cargar" bug fix**: on multi-driver stores the shortfall purchase was landing on the **parking** pool while the follow-up load drew from the driver's (empty) pool → 409. Now it attributes the purchase to the **assigned** driver (auto-loads the bought fulls onto their open truck) and bases the have/load decision on that driver's own pool (`getPoolBreakdown`); unifies the former single/multi paths. Load math verified: exactly `shortfall` onto the truck, no double-load, for have {0, <shortfall, ≥shortfall}.
- Plumbing: `quickOpenAll` (service + store + `QuickOpenAllResult` type). Independent validation: no correctness bugs (per-pool sweep, skip/idempotency, load math, optional-parking attribution all sound). Gates: `npm run typecheck` ✓ · `npm run build` ✓.

[2026-07-18] [lpg-frontend-vue] Asignaciones-tab refinements (operator feedback):
- **Show where the stock actually is.** A driver with an OPEN/CLOSED day holds their stock on the **truck** (their tienda stock was swept at open), so the per-driver card now shows the **truck balances** (labelled "En su vehículo hoy") for a live day, and the tienda stock ("Inventario en tienda") otherwise — fixes the confusing "Sin stock en su inventario" on an active driver who clearly has stock on the assignment detail. New non-mutating `store.getAssignmentView(id)` + a `truckByDriver` map refreshed by a watcher on the crew / today's day-states.
- **Unassigned stock always visible.** The "Sin asignar (en tienda)" section renders even at zero (with an explanatory line) so it's discoverable.
- **Dropped the word "pool" from all UI copy** (confusing for Spanish speakers) → "inventario" / "sin asignar (en tienda)" in the purchase, adjust, open-day, and store-detail surfaces. typecheck + build green.

[2026-07-19] [lpg-frontend-vue] Two more Asignaciones-tab fixes (operator feedback):
- **"En vehículos" totals summed.** The backend `getLocationAvailability` returns one on-truck row PER driver's truck, so a multi-driver store showed the same tank type twice (and collided the `tankTypeId` row-key). Store detail now aggregates on-truck per tank type client-side (`onTruckTotals`) for the store-wide table; per-driver truck detail already lives in the driver cards. (Left the backend read per-truck by design — the aggregate is a display concern; revisit if another consumer needs it summed.)
- **Not-yet-opened driver reads "deactivated."** A driver with no day today now renders muted (dashed border + `bg-muted/20`, dimmed stock body, label "Inventario en tienda (sin abrir)"); a live/finished day shows at full strength. The "Abrir día" CTA stays crisp. typecheck + build green.

[2026-07-19] [lpg-frontend-vue] Follow-on: **"Agregar repartidor"** — one admin/dev action to add a driver to a store with a starting inventory, covering BOTH onboarding a new driver and consolidating a duplicate store (the deferred "merge duplicate stores" scope, driver+inventory slice).
- New **`AddDriverDialog`** on the store-detail **Asignaciones** tab, gated **admin/dev only** (`v-if="isAdmin"` on both the button and the dialog; backend enforces independently). Pick a delivery user not already active in this store; if they're active in **another** store → a **"Traer inventario de {tienda} y retirarlo de ahí"** toggle (carry + retire, default on); otherwise an optional **"Inventario inicial"** (tank + item rows, default empty → start with nothing).
- Orchestrates **create link → seed inventory → (carry) retire source**, each step guarded with a clear, **recoverable** message (link-only ⇒ "added empty, adjust in store"; seed-ok-retire-failed ⇒ "inventory copied, retire manually"). Submit is disabled until the driver/assignment lists load, so a pick before assignments resolve can't silently downgrade carry→manual.
- Plumbing: `seedDriver` (types + service + store); catalog `addStoreAssignment` returning the created link (the existing `createStoreAssignment` returns only a boolean).
- Independent validation caught a **cross-store double-count** (carry copied instead of moved) — fixed backend-side to a true move that zeroes the source. Gates: `npm run typecheck` ✓ · `npm run build` ✓.

[2026-07-20] [lpg-frontend-vue] Assignment detail (`/inventario/asignaciones/:id`) now shows the day's **"Conteo del día" (Inicio → Final)** on closed/consolidated days instead of the live truck balance — which reads 0 after consolidation (the carry hand-back zeroes the assignment). Reuses the existing day reconstruction (`GET /inventory/history?userId=&date=`): Inicio = `openingFull/Empty`, Final = `expectedFull/Empty` (everything except the close transfer = the counted end-of-day count, which survives consolidation). New `reconstructDay` plumbing (types `DayReconstruction`/`DayTypeBreakdown` + service + store `reconstruction`/`fetchReconstruction`); the view resolves the driver's userId from the catalog and guards the display on `reconstruction.assignmentId === viewed id`. Open days keep the live balance + actions unchanged; items are untouched (carry doesn't move them). typecheck + build green.

[2026-07-20] [lpg-frontend-vue] Refinement to the above: the "Conteo del día" now renders on **all** day states (not just closed/consolidated). Inicio always shows the opening (from the reconstruction); the second column is **"Actual"** while the day is **open** — reading the **live** truck balance so it stays reactive after each sale/load (no reconstruction refetch) — and **"Final"** once closed/consolidated (the reconstruction's `expected`). Reconstruction is fetched for all states now (for the Inicio column). Replaces the separate live "Tanques" table for open days. typecheck + build green.
