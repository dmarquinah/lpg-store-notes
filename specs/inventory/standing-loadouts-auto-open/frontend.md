---
project: lpg-store
domain: specs
type: spec-track
spec: standing-loadouts-auto-open
repo: lpg-frontend-vue
kind: frontend
track-status: not-started
last-updated: 2026-07-17
---

# Standing Loadouts → Auto-Open — lpg-frontend-vue track

Shared spec: [[index]]

> Ports after the backend track is `done`.

## Technical Notes

- **Loadout editor** per delivery driver (per-tank-type / item target quantities),
  reachable from the store detail or the driver's assignment row. Prefill from the
  saved template; clearable.
- **Store card "Abrir día"** — a single action opening all the store's
  not-yet-open drivers via `POST …/stores/:id/open-all`; success shows opened
  drivers + any **shortfall** advisories (loadout > available) without treating
  them as errors. A driver without a loadout still opens via the whole-pool sweep.
- **Per-store auto-open toggle** — opt into the scheduled morning open; clear copy
  that it applies loadouts automatically on the Lima business day.
- Single-driver store keeps its existing one-tap; no regression.
- Design-system: cards / `PageHeader` / `vue-sonner`; tokens + ≥44px targets;
  shortfall shown as a warning, not a blocking error.

## Related Files

<!-- Confirm exact paths within lpg-frontend-vue at /focus. -->

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/modules/inventory/views/InventoryView.vue` — store card "Abrir día" +
  results/shortfall toast.
- `src/modules/inventory/views/…StoreDetail…` — loadout editor entry + auto-open
  toggle.
- new loadout editor component + per-store auto-open control.
- `src/modules/inventory/api.*` / store — loadout CRUD + `open-all` payloads.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
