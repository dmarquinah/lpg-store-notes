---
project: lpg-store
domain: specs
type: spec-track
spec: catalog-item-management
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-18
---

# Catalog Item Management — lpg-backend track

Shared spec: [[index]]

## Technical Notes

Pure additive write surface on the existing `catalog` module — copy the
already-shipped `PATCH /catalog/stores/:id` pattern for tank types and items.

- The router (`src/modules/catalog/routes.ts`) already has `POST /tank-types`,
  `POST /items`, and `PATCH /stores/:id` + `PATCH /store-assignments/:id`
  (admin-gated via `requireRole('admin')`, parsing with the module's `parse()`
  helper). Add `PATCH /tank-types/:id` and `PATCH /items/:id` in the same style.
- Add `updateTankTypeSchema` / `updateItemSchema` in `types.ts` (partial of the
  create schemas + an optional `active` boolean), mirroring `updateStoreSchema`.
- Add `updateTankType` / `updateItem` service methods mirroring `updateStore` /
  `setStoreAssignmentActive`; the repository update follows the existing pattern.
- **Check the schema first:** `src/modules/catalog/schema.ts` — do `tank_types` /
  `inventory_items` already have an `active` (or soft-delete) flag and do the list
  queries already honor `?all=1` for them? `stores` does; confirm the product
  tables do too. If the `active` column is missing, add a tiny additive migration
  (named explicitly, per `feedback_named_migrations`).
- **Price-edit safety:** inventory holders snapshot the catalog price at day-open,
  so a price PATCH only affects future opens — confirm nothing reads the live
  catalog price for a closed/consolidated day. Don't mutate historical rows.
- Tests: extend the catalog/stores test file (or add `catalog-item-management.test.ts`)
  with a lifecycle — create → edit a field → deactivate (gone from default list,
  present with `?all=1`) → restore; assert admin-gating. Keep it to the 2–5
  happy-path lifecycle tests the project favors (`feedback_lifecycle_tests_over_unit_enum`).
- Gates: `npm run typecheck`, `npm run check`, `npm test`, `npm run build`.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `src/modules/catalog/routes.ts` — add `PATCH /tank-types/:id` + `PATCH /items/:id` (mirror `PATCH /stores/:id`)
- `src/modules/catalog/types.ts` — add `updateTankTypeSchema` / `updateItemSchema` (mirror `updateStoreSchema`)
- `src/modules/catalog/service.ts` — add `updateTankType` / `updateItem` (mirror `updateStore`)
- `src/modules/catalog/repository.ts` — update queries (mirror the store update)
- `src/modules/catalog/schema.ts` — verify `active`/soft flag on `tank_types` + `inventory_items`; migration only if absent
- `src/modules/catalog/*.test.ts` — lifecycle test for edit + deactivate/restore
- `legacy/src/routes/productRoutes.ts` — v1 reference for the fields/behavior (don't port the search/restore-as-separate-endpoint shape)

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-18] [lpg-backend] Implemented the full backend slice. Verified open questions first: both `tank_types` and `inventory_items` already carry `active` (schema.ts) and their list queries honor `?all=1` — **no migration needed**. No field is unsafe to PATCH: inventory references catalog by id and snapshots prices at day-open, and `sizeLabel`/`unit` are display-only (grep confirmed no inventory logic reads them). Also checked seeding: legacy had a seed but it was demo/sample data (fictional product names), not a real curated catalog; v2's seed is equivalent dev-only fixtures (disabled in production) — production starts empty by design, admins populate via UI. Nothing to migrate.

Changes (pure additive on the existing `catalog` module, mirroring `PATCH /catalog/stores/:id`):
- `types.ts` — `updateTankTypeSchema` / `updateItemSchema`: partial of the create shapes (reusing `nameSchema`/`priceSchema`) + optional `active`, `.refine` requiring ≥1 field. `sizeLabel`/`description` are `.nullable().optional()` (explicit null clears); `unit` stays non-nullable (column is NOT NULL).
- `repository.ts` — `TankTypePatch`/`ItemPatch` + `updateTankType`/`updateItem` (pure `UPDATE … SET …, updatedAt RETURNING`, returns row or null). No uniqueness/conflict logic (catalog tables have no unique-name constraint, unlike stores).
- `service.ts` — `updateTankType`/`updateItem`: build patch from defined keys, `.toFixed(2)` prices, 404 if missing. No conflict check.
- `routes.ts` — `PATCH /tank-types/:id` + `PATCH /items/:id`, `requireRole('admin')`.
- Tests — fake repo gained `seedItem`/`updateTankType`/`updateItem`; 2 lifecycle tests (tank-type edit+price+deactivate/?all=1/restore/404; item edit+null-clear+deactivate/restore + non-admin 403 + empty-body 400 + invalid-value 400).

Gates all green: `npm run typecheck`, `npm run check` (biome), `npm test` (102 pass), `npm run build`. Validation agent confirmed all 5 backend acceptance criteria met.

[2026-06-18] [lpg-backend] All criteria for this repo met.
