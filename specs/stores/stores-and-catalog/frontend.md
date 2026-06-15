---
project: lpg-store
domain: specs
type: spec-track
spec: stores-and-catalog
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-05
---

# Stores & Catalog — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

Deferred. The catalog/stores admin UI is **not required** to unblock the inventory backend. The inventory frontend track only needs to *read* tank types/items when recording transactions, which it can do directly against the catalog read endpoints once the backend track ships.

When this track is picked up: an admin `catalog` module (list + create tank types / items, list stores/assignments) mirroring the `users` module pattern.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To be created (later): `src/modules/catalog/` vertical module.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
[2026-06-05] [lpg-frontend-vue] Built the `catalog` admin UI (`src/modules/catalog/`), mirroring the `users` module: `types`/`service`/`store`/`routes`/`index` + `views/CatalogView.vue` + `components/{TankTypeCreateDialog,ItemCreateDialog}.vue`. Tabbed page (new hand-authored `components/ui/tabs/` reka-ui component) with three tabs — Tipos de tanque, Artículos (list + admin create, client-side validation mirroring the backend Zod), and read-only Tiendas. Single "Mostrar inactivos" switch drives `?all=1` across all lists; per-list loading flags avoid spinner flicker. Service calls `/catalog/{stores,tank-types,items}` and unwraps the `{stores|tankTypes|items|tankType|item}` envelopes; prices sent as numbers. Wired in `main.ts` + router; `/catalog` guarded `roles: [admin, developer]`; "Catálogo" added to all three admin nav arrays. Also retheme: burnt-orange flame primary (`18 85% 40%`, white foreground for WCAG-AA button text) + ring in `style.css` (:root + .dark), replacing the stock blue that clashed with the dark-blue background — on-brand for *la llama piloto*. typecheck + build green; independent validation: all frontend criteria met.
