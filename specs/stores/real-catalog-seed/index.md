---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../stores-and-catalog/index]]"
  - "[[../catalog-item-management/index]]"
last-updated: 2026-06-18
---

# Spec: Real Catalog Seed (from `inventario.xlsx`)

## Problem Statement

v2's `db:seed` (`src/db/seed.ts`) seeded a **fictional placeholder catalog** — two
tank types (`Balón 10kg`, `Balón 45kg`) with made-up prices (40/60, 160/220) and a
single `Válvula` item. The business has a **real, curated product catalog** (the
operator's `inventario.xlsx` export) that should be the canonical seed so dev/demo —
and a fresh production bootstrap — reflect the real product line and current pricing,
not placeholders.

The export holds **14 rows**: 4 gas cylinder sizes/grades sold as a **GLP refill**
(customer brings an empty) **and** as a **Cilindro** (full new cylinder, no empty
exchanged), plus 6 accessories. Source of truth (parsed from the sheet, columns
*Producto · Medida · Cantidad · P. Compra · P. Venta*):

**Gas products (→ `tank_types`, 8 rows)** — refill + full-cylinder are **separate
catalog entries**, each managed independently with its own price:

| Tank type | sizeLabel | compra / venta |
|-----------|-----------|-----------------|
| GLP 10kg Premium | 10kg | 42 / 55 |
| GLP 10kg Normal  | 10kg | 41 / 54 |
| GLP 5kg Normal   | 5kg  | 22 / 31 |
| GLP 45kg         | 45kg | 180 / 250 |
| Cilindro GLP 10kg Premium | 10kg | 65 / 80 |
| Cilindro GLP 10kg Normal  | 10kg | 65 / 80 |
| Cilindro GLP 5kg Normal   | 5kg  | 60 / 70 |
| Cilindro GLP 45kg         | 45kg | 210 / 260 |

**Accessories (→ `inventory_items`, 6 rows)**:

| Item | unit | compra / venta |
|------|------|-----------------|
| Válvula Normal | unidad | 20 / 35 |
| Adaptador Presión alta | unidad | 10 / 20 |
| Válvula Premium PB | unidad | 30 / 40 |
| Válvula Premium PA | unidad | 20 / 40 |
| Manguera alta | m | 4 / 8 |
| Manguera baja | m | 3 / 6 |

The sheet's `Cantidad` column (current stock) is **not** seeded — owner decision:
catalog only; opening stock is entered through the normal inventory flow.

## Proposed Solution

**Pure seed-data change — no schema change, no migration.** Owner decision: keep the
GLP refill and the Cilindro full-cylinder as **separate, independently-managed catalog
entries** (rejected a merged tank type with a second cylinder-price pair — see below).
Both are cylinders, so all 8 gas products live in `tank_types` (sharing the table/UI
but each its own row + price); the 6 accessories go in `inventory_items` (Manguera by
the metre, `unit='m'`).

Rewrite the catalog block of `src/db/seed.ts` to be data-driven from the 8 tank types
+ 6 items above, keeping the existing **idempotent, guarded-by-name** insert pattern
(re-running never duplicates). The demo store / users / customers seed stays as-is, and
the seed remains **disabled in production**.

> **Rejected alternative (a mid-implementation correction):** adding nullable
> `cylinder_purchase_price` / `cylinder_sell_price` columns to `tank_types` to hold both
> prices on one row. The owner wants the refill and the full cylinder managed
> *separately*, so they are distinct rows — no schema change at all.

Backend detail in [[backend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across all tracks. -->

- [x] `db:seed` seeds the **real catalog** from `inventario.xlsx`: **8 tank types**
      (4 `GLP …` refills + 4 `Cilindro GLP …` full cylinders, each with its own
      compra/venta from the tables above) + **6 inventory items** (Válvula ×3 +
      Adaptador `unit='unidad'`; Manguera ×2 `unit='m'`).
- [x] GLP refill and Cilindro are **separate `tank_types` rows** — no schema change, no
      merged price field; both managed independently.
- [x] The seed is **idempotent** — guarded by name, re-running inserts no duplicates
      (verified: 2nd run seeds 0 rows) — and remains **disabled in production**
      (`NODE_ENV==='production'` guard intact).
- [x] **Catalog only:** no opening stock / inventory holders seeded from `Cantidad`; the
      demo store / users / customers seed is unchanged.
- [x] Gates green: `npm run typecheck`, `npm run check` (Biome), `npm test` (102 pass,
      unchanged), `npm run build`. Seed verified against the dev DB (all 14 rows present,
      correct prices/units).

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |

## Out of Scope

- **Schema change / cylinder-price column** on `tank_types` — explicitly rejected; the
  refill and full cylinder are separate rows.
- **Opening stock** from the `Cantidad` column — owner chose catalog-only; stock is
  entered via the normal day-open / purchase flow.
- **Frontend** — no UI work; the existing catalog admin tabs render the new rows as-is.
- **Per-customer pricing** (legacy `cliente.precio10k/precio45k`) — dropped in v2 in
  favor of catalog price + per-order override.
- **The `28042022-v2.sql` legacy dump** — confirmed 2022 dev/test data (Cliente Test,
  Local Test, Diego Admin/Supervisor/Usuario, test sales); not migrated.

## Open Questions (resolved)

- **Cilindro home** → all 8 gas products as `tank_types` (owner decision); the 4
  Cilindro rows are *not* inventory_items.
- **Display names** → kept the sheet's `GLP …` / `Cilindro GLP …` naming (typo
  `Premiun→Premium` fixed, `5kg` normalized).
- **Re-seed of an already-seeded dev DB** → the name-guarded insert leaves the 3 old
  placeholder rows (`Balón 10kg`, `Balón 45kg`, `Válvula`) coexisting with the new
  curated ones; `db:reset && db:seed` is the clean path. A fresh DB seeds only the 14
  real rows. (Cleaning the existing dev DB's placeholders is an operator choice, not
  baked into the seed.)
