---
project: lpg-store
domain: specs
type: spec-track
spec: catalog-item-management
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-18
---

# Catalog Item Management — lpg-frontend-vue (Piloto) track

Shared spec: [[index]]

## Technical Notes

The catalog admin page already has Tipos de tanque / Artículos tabs with **create**
dialogs, and the Tiendas tab already does **edit + activate/deactivate + show-inactive**
(from `store-management`). This track brings the product tabs up to the same level by
reusing that exact pattern.

- Extend the create dialogs to double as **edit** dialogs (prefilled when given an
  existing row), mirroring `StoreFormDialog`'s create/edit dual mode.
- Add per-row **Editar** + an activate/deactivate control + a **show-inactive** toggle
  (`?all=1`) on both product tabs, mirroring the Tiendas tab.
- New catalog store/service methods: `PATCH /catalog/tank-types/:id` and
  `PATCH /catalog/items/:id` (mirror the existing store update method).
- Client-side validation mirrors the backend Zod (name length, price ≥0 ≤2dp, etc.),
  matching how the create dialogs already validate.
- Design-system tokens only, ≥44px touch targets, dialog emits `saved` so the list
  refreshes (per the established module conventions and `eng/design-system.md`).
- Gates: typecheck + build.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)
- `src/modules/catalog/` — the catalog module (tabs, dialogs, service, store, types)
- the existing tank-type / item create dialogs — extend to create+edit (mirror `StoreFormDialog`)
- the Tiendas tab + `StoreFormDialog` — the canonical edit/activate/show-inactive pattern to copy
- the catalog `service.ts` / `store.ts` — add `updateTankType` / `updateItem` calling the new `PATCH` endpoints

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-18] [lpg-frontend-vue] Implemented the frontend slice on the existing `catalog` module, mirroring the `store-management` Tiendas pattern.
- `types.ts` — `UpdateTankTypePayload` / `UpdateItemPayload` (all fields optional + `active`). `sizeLabel`/`description` are `string | null` (explicit null clears, matching backend `.nullable()`); item `unit` stays non-nullable (NOT NULL column — omitted when blank, never sent null).
- `service.ts` — `updateTankType(id)` → `PATCH /catalog/tank-types/:id`, `updateItem(id)` → `PATCH /catalog/items/:id` (mirror `updateStore`).
- `store.ts` — `updateTankType` / `updateItem` actions (loading + friendly error + refetch the respective list), exported.
- Converted the create-only dialogs to dual create/edit, mirroring `StoreFormDialog`: `TankTypeCreateDialog.vue` → `TankTypeFormDialog.vue` (prop `tankType?`), `ItemCreateDialog.vue` → `ItemFormDialog.vue` (prop `item?`). Form seeds from the row on open, `active` Switch shown only in edit, emits `saved`; edit sends the full set (size/description → null when blank), create unchanged. Validation mirrors backend (name length, prices ≥0).
- `CatalogView.vue` — `editingTankType`/`editingItem` refs (null = create); per-row `#actions` "Editar" button (Pencil) on both tables; "Nuevo" buttons reset to create mode. Show-inactive toggle + Estado badges already existed.
- Activate/deactivate is the in-dialog `active` Switch, exactly like the Tiendas tab (canonical pattern), not a separate per-row control.

[2026-06-18] [lpg-frontend-vue] All criteria for this repo met. Independent validation: all 6 shared acceptance criteria PASS, no functional bugs or design-system violations. Gates green (typecheck + build, PWA 73 entries / 772.25 KiB). Manual both-theme + phone-width smoke (edit a price, deactivate → drops from default list, show-inactive → reappears, restore) left to the operator.
