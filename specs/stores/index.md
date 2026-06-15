---
project: lpg-store
domain: specs
category: stores
last-updated: 2026-06-05
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
| [[store-management/index\|store-management]] | approved · backend ◻ frontend ◻ | Admin **write surface** for stores + store↔user assignments (create/edit store, assign/deactivate users). No schema change — endpoints + UI only. Unblocks multi-location testing (a 2nd store for transfer/switcher; operator assignments for branch scoping). |

## Notes

These are **reference/catalog** tables — low write volume, read by nearly every other module (inventory holders FK `stores`/`tank_types`/`inventory_items`; `inventory_assignments` FK `store_assignments`). Per [[../../eng/decisions]] ADR int-PK guidance, all four use integer (`serial`) PKs because they are referenced across tables. Canonical prices live here on the catalog; inventory holders snapshot the price in effect when a day opens (so daily sale-amount accountability uses that day's price).
