---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../auth/auth-foundation/index]]"
last-updated: 2026-06-05
---

# Spec: Stores & Catalog (foundational reference data)

## Problem Statement

`inventory-foundation` (the product's core) FKs into four tables that do not exist yet in v2 — only the auth tables (`users`, `delivery_profiles`, `user_invitations`) are built. Before any inventory work can lay down its holder/ledger tables, the v2 backend needs:

- **`stores`** — physical locations that hold standing inventory (one store for the MVP, seeded; multi-store non-breaking later).
- **`store_assignments`** — which user (driver/operator) belongs to which store. `inventory_assignments` keys on `(storeAssignmentId, date)`.
- **`tank_types`** — the cylinder catalog (e.g. 10kg, 45kg) with canonical purchase/sell prices.
- **`inventory_items`** — non-tank catalog (valves, hoses, accessories) with prices.

This spec delivers those tables, a single-store dev seed (store + assignments for the seeded users + a few tank types/items), and a thin read API so the frontend and inventory module can list them. It deliberately stays small — it is plumbing that unblocks inventory, not a feature in itself.

## Proposed Solution

- **Backend:** one vertical module (`src/modules/catalog/`) owning the four reference tables, Drizzle migrations, an extended `db:seed` (one store, store_assignments for the seeded operator+delivery users, seed tank types + items), and minimal read endpoints (`GET /stores`, `GET /tank-types`, `GET /items`) plus admin create for catalog rows. Int (`serial`) PKs throughout (referenced across tables). Canonical prices live on `tank_types`/`inventory_items`; inventory holders snapshot them at day-open.
- **Frontend:** (deferred / thin) an admin catalog view can come later; the inventory frontend track only needs to *read* tank types when recording transactions. No frontend work is required to unblock the inventory backend.

Detailed schema + routes live in [[backend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Backend (lpg-backend):**

- [ ] `stores`, `store_assignments`, `tank_types`, `inventory_items` tables exist with integer (`serial`) PKs; Drizzle migration in `src/db/migrations/` (no `db:push`); re-exported from `src/db/schema.ts`.
- [ ] `store_assignments` links `userId → users.id` and `storeId → stores.id`, with an index on `(storeId)` and `(userId)` and a uniqueness guard so a user isn't double-assigned to the same store while active.
- [ ] `tank_types` and `inventory_items` carry canonical `purchasePrice`/`sellPrice` (`numeric(10,2)`) and an `active` flag.
- [ ] `src/modules/catalog/` follows [[../../../eng/patterns/module-template]]; only `repository.ts` imports `schema.ts`; types are Zod + `z.infer<>` (no `dtos/`, no `I*` interfaces, no DI container).
- [ ] `db:seed` is extended idempotently: one store, `store_assignments` binding the seeded operator + delivery users to it, ≥2 tank types and ≥1 inventory item. Re-running the seed does not duplicate rows.
- [ ] Read endpoints return Zod-validated shapes: `GET /api/v1/stores`, `GET /api/v1/catalog/tank-types`, `GET /api/v1/catalog/items` (active-only by default, `?all=1` to include inactive). Admin-only create for tank types / items. Errors flow through `src/lib/errors.ts`.
- [ ] At least 2 lifecycle tests (per [[../../../eng/decisions]] test guidance): seed→list a store+assignment; create→list a tank type.

**Frontend (lpg-frontend-vue):**

- [x] Admin catalog management UI — `src/modules/catalog/` (tabbed page): list + admin create for tank types & items, read-only stores list, show-inactive toggle. Built mirroring the `users` module.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Multi-store management UX** — single store seeded; multi-store is structurally supported (FKs) but no management screens.
- **Store-level standing stock ledger** — that's `inventory-foundation`'s `location` holder, not this spec. This spec only provides the `stores` row the holder points at.
- **Catalog admin UI** — deferred to a later frontend spec.
- **Soft-delete / price history** — `active` flag only; no `deleted_at`, no price-change audit (add when a consumer needs it).

## Open Questions

- Single `catalog` module vs. splitting `stores` (stores + assignments) from `catalog` (tank_types + items)? Default: one module for MVP; revisit if it grows. (Resolve in the backend track's plan.)
