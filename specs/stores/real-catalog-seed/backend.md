---
project: lpg-store
domain: specs
type: spec-track
spec: real-catalog-seed
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-18
---

# Real Catalog Seed — lpg-backend track

Shared spec: [[index]]

## Technical Notes

**Pure seed-data change — no migration, no schema/types/service/repository edits.**
The `tank_types` / `inventory_items` tables already hold everything needed (name,
sizeLabel/unit, purchase/sell price). The 8 gas products (4 GLP refills + 4 Cilindro
full cylinders) are distinct `tank_types` rows; the 6 accessories are `inventory_items`
(Manguera `unit='m'`).

- `src/db/seed.ts` → `seedCatalog`: replaced the inline `TANK_TYPES` (2) + `ITEMS` (1)
  arrays with the curated 8 + 6 from [[index]]. Prices stored as 2-dp strings.
- Kept the existing **idempotent guard** — each row checked by `name` before insert,
  so re-runs never duplicate. Store / users / customers seed blocks untouched. The
  `NODE_ENV === 'production'` guard in `run()` stays.
- No new automated test: the change is dev-only seed data (seed scripts aren't
  unit-tested in this project), the catalog create/list/PATCH lifecycle is already
  covered by `catalog.test.ts`, and the seed was verified directly against the dev DB.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `src/db/seed.ts` — `seedCatalog`: curated 8 tank types + 6 inventory items, idempotent-by-name
- `inventario.xlsx` (repo root) — source of truth for names/units/prices

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-18] [lpg-backend] **Scope corrected mid-implementation.** Initial plan added
nullable `cylinder_purchase_price`/`cylinder_sell_price` columns to `tank_types` to keep
both prices on one row; the owner rejected merging — refill and full cylinder must be
managed **separately**. Reverted the `schema.ts` edit (no migration generated) and
seeded all 8 gas products as **distinct `tank_types` rows** instead.

Final change is `src/db/seed.ts` only: replaced the 2 placeholder tank types + 1 item
with the real catalog from `inventario.xlsx` — 8 tank types (`GLP 10kg Premium` 42/55,
`GLP 10kg Normal` 41/54, `GLP 5kg Normal` 22/31, `GLP 45kg` 180/250, and the four
`Cilindro GLP …` full cylinders 65/80, 65/80, 60/70, 210/260) + 6 inventory items
(`Válvula Normal` 20/35, `Adaptador Presión alta` 10/20, `Válvula Premium PB` 30/40,
`Válvula Premium PA` 20/40, `Manguera alta`/`baja` `unit='m'` 4/8 and 3/6). Idempotent
name-guard pattern preserved.

Verified: `npm run typecheck`, `npm run check` (biome, 95 files clean), `npm test`
(**102 pass**, unchanged — no logic touched), `npm run build` all green. Ran
`npm run db:seed` against the dev DB → all 14 rows inserted with correct prices/units; a
2nd run seeded **0** rows (idempotent). Note: on this already-seeded dev DB the 3 old
placeholder rows (`Balón 10kg`, `Balón 45kg`, `Válvula`) coexist with the new curated
ones — `db:reset && db:seed` gives the clean 14-row state; a fresh/production bootstrap
seeds only the real catalog.

[2026-06-18] [lpg-backend] All criteria for this repo met → track done; single-track
spec → overall `done`.
