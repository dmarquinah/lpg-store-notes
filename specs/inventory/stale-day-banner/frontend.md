---
project: lpg-store
domain: specs
type: spec-track
spec: stale-day-banner
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: 2026-06-16
---

# Stale Day Banner — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Builds on: [[../day-handoff/frontend]] · Foundation: [[../inventory-foundation/frontend]]

## Technical Notes

Frontend-only. Extends the existing `inventory` module — **no new module, no new
endpoint**. Follow `eng/design-system.md` (Alert/Card tokens, status→badge
mapping, `Spinner`/`EmptyState`, ≥44px touch targets).

### Data: derive from the existing caller-scoped list

The page already fetches `state=closed` for the **Por consolidar** tab via
`fetchConsolidationList()` ([store.ts:257](../../../../../dev/personal/freelance/lpg-store/lpg-frontend-vue/src/modules/inventory/store.ts)).
Add a sibling that gathers the **stale** set:

- Fetch `listAssignments({ state: "open" })` **and** `listAssignments({ state:
  "closed" })` (both already caller-scoped server-side: operator → their
  branches, admin/dev → all). The `closed` call can reuse / share with
  `consolidationList` to avoid a duplicate request.
- Filter client-side to `a.date < todayISO()` (`@/lib/date`, the device-local
  ISO the rest of the page uses). Merge open + closed, sort by `date` ascending
  (oldest first — most overdue at the top), then `id`.
- Expose as a store getter/ref (e.g. `stalePriorDays`) + `loadingStale`. This is
  separate from the date/state-filtered `assignments` list so the banner is
  unaffected by the Asignaciones tab controls.

`InventoryAssignment` already carries `date` + `state` — no type change needed.

### Surface: persistent banner above the tabs

In [InventoryView.vue](../../../../../dev/personal/freelance/lpg-store/lpg-frontend-vue/src/modules/inventory/views/InventoryView.vue),
render the banner **between the `PageHeader` and the `<Tabs>`** (so it shows on
any tab), only when `stalePriorDays.length > 0`:

- A `warning`/`destructive`-toned `Alert` or `Card` headed e.g. **"Días
  anteriores sin resolver"** with a one-line explanation ("Resuélvelos antes de
  abrir el día de hoy").
- One row per stale day: `date` (via `isoToDisplay`), driver/store (the existing
  `assignmentLabel` map), a state `Badge` (`STATE_VARIANT`/`STATE_LABELS`
  already defined in the view), and a short action hint — `closed` → "Lista para
  consolidar", `open` → "Cerrar (forzar) y consolidar".
- Each row clicks through to the assignment detail (`goToAssignment(a.id)`),
  where the existing **Consolidar** / **Forzar cierre** actions live.
- After resolution the banner must refresh: re-run the stale fetch in
  `onMounted` and after the user returns / after a relevant write (mirror how
  `fetchConsolidationList` is already called on mount).

### Role gating

The Disponibilidad/assignment surfaces are already operator/admin/developer; the
banner inherits that scope. `delivery` doesn't reach this view. No extra gating
beyond what the route + backend caller-scoping already enforce.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To modify:

- `src/modules/inventory/store.ts` — add `stalePriorDays` + `loadingStale` and a
  fetch that merges past-dated `open` + `closed` (reuse the `closed` call backing
  `consolidationList`); filter `date < todayISO()`.
- `src/modules/inventory/views/InventoryView.vue` — render the persistent banner
  above the `<Tabs>`; reuse `assignmentLabel`, `STATE_LABELS`/`STATE_VARIANT`,
  `isoToDisplay`; call the stale fetch in `onMounted`.

Reference: [[../day-handoff/frontend|day-handoff frontend track]] for the
`consolidationList` pattern and the OpenDayDialog stale-day handling; the
existing `Alert` usage in `InventoryView.vue` for the error banner.

## Implementation Notes

<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-frontend-vue] Frontend track done. Persistent **"Días anteriores sin resolver"** banner added above the `/inventario` tabs (between the error Alert and `<Tabs>` in `InventoryView.vue`), rendering only when there's ≥1 unresolved prior day. **Data (no new endpoint):** `store.ts` gains `fetchStaleOpenDays()` → `listAssignments({state:'open'})` filtered to `date < todayISO()`, plus a computed `stalePriorDays` merging those past-dated open days with the past-dated slice of the already-fetched `consolidationList` (closed) — so the closed set costs no extra request — sorted oldest-first (date asc, then id). `service.ts` untouched. Each banner row shows `isoToDisplay(date)`, the catalog `assignmentLabel` (driver/store), a state `Badge` (reused `STATE_VARIANT`/`STATE_LABELS`), and an action hint (`closed` → "Lista para consolidar", `open` → "Cerrar (forzar) y consolidar"); clicking routes to the assignment detail (`goToAssignment`) where the existing Consolidar / Forzar cierre actions resolve it. Strict `< today` on both sources excludes today + future; `carried` can't appear (neither source carries it); a single-state assignment can't double-list. Refresh: `onMounted` calls `fetchStaleOpenDays()` + `fetchConsolidationList()`, and the view remounts on return (no keep-alive), so a resolved day drops off. **Reused** the design system: added a `warning` variant to the shared `Alert` (`border-warning/40 bg-warning/10 text-warning-text` — tokens only, mirroring `destructive`); rows are `min-h-11` (≥44px). Gates green: `npm run typecheck` + `npm run build` (PWA precache 68 entries / 769.69 KiB). Independent validation: **all 8 acceptance criteria PASS, no correctness bugs**. Manual smoke (leave a past `open` + a past `closed` day → both appear → resolve each → banner clears) left to the operator per the project's standing verification method. Single track → spec `done`.