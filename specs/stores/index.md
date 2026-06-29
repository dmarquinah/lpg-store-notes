---
project: lpg-store
domain: specs
category: stores
last-updated: 2026-06-18
---

# Stores & Catalog — lpg-store

## Context Documents

Read these before working on stores/catalog specs:

- [[../../eng/architecture]] — backend layout and conventions
- [[../../eng/legacy-bloat-analysis]] — why v1 grew; the ten drivers v2 must avoid
- [[../../eng/patterns/module-template]] — vertical module folder convention
- [[../../product/overview]] — store, roles, tank-exchange model, MVP scope
- `lpg-backend/legacy/src/db/schemas/locations/` — v1 stores + store_assignments (reference for shapes; drop `store_assignment_current_inventory`, the auto-routing anti-pattern)
- `lpg-backend/legacy/src/db/schemas/inventory/tank-type.ts`, `inventory-item.ts` — v1 catalog (reference for fields; trim `is_popular`/`scale`/`deleted_at` unless needed)

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[stores-and-catalog/index\|stores-and-catalog]] | done · backend ✓ frontend ✓ | Foundational reference data: `stores`, `store_assignments`, `tank_types`, `inventory_items` + single-store seed + read API (backend); tabbed admin catalog UI (frontend). |
| [[store-management/index\|store-management]] | done · backend ✓ frontend ✓ | Admin **write surface** for stores + store↔user assignments (create/edit store, assign/deactivate users). No schema change — endpoints + UI only. Unblocks multi-location testing (a 2nd store for transfer/switcher; operator assignments for branch scoping). |
| [[catalog-item-management/index\|catalog-item-management]] | done · backend ✓ frontend ✓ | Closes the one real parity gap from the 2026-06-18 v1 audit: the catalog is **create-only** for tank types & items (no `PATCH`). Adds admin **edit + deactivate/restore** for both (mirrors `PATCH /catalog/stores/:id`), so prices/names can be corrected and discontinued products retired without a DB edit. |
| [[real-catalog-seed/index\|real-catalog-seed]] | done · backend ✓ | Replaced the placeholder `db:seed` catalog with the **real catalog** from the operator's `inventario.xlsx`: **8 tank types** (4 `GLP` refills + 4 `Cilindro GLP` full cylinders, kept as separate independently-managed rows — owner rejected merging into a cylinder-price column, so **no schema change/migration**) + **6 inventory items** (Manguera `unit='m'`). Idempotent name-guarded, catalog-only (no opening stock), prod-disabled. |

## Notes

These are **reference/catalog** tables — low write volume, read by nearly every other module (inventory holders FK `stores`/`tank_types`/`inventory_items`; `inventory_assignments` FK `store_assignments`). Per [[../../eng/decisions]] ADR int-PK guidance, all four use integer (`serial`) PKs because they are referenced across tables. Canonical prices live here on the catalog; inventory holders snapshot the price in effect when a day opens (so daily sale-amount accountability uses that day's price).
