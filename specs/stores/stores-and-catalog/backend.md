---
project: lpg-store
domain: specs
type: spec-track
spec: stores-and-catalog
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-05
---

# Stores & Catalog — lpg-backend track

Shared spec: [[index]]

## Technical Notes

### Data model (v2 conventions: `serial` int PK, snake_case columns, `numeric` prices, `timestamptz`)

- **`stores`** — `id` serial PK, `name` varchar(120) notNull, `address` varchar(255)?, `phone` varchar(32)?, `active` bool default true, `createdAt`/`updatedAt` timestamptz.
- **`store_assignments`** — `id` serial PK, `storeId` int FK → `stores.id` notNull, `userId` int FK → `users.id` notNull, `active` bool default true, `createdAt`/`updatedAt`. Indexes `(storeId)`, `(userId)`; partial unique `(storeId, userId) WHERE active` so a user isn't double-bound to one store.
- **`tank_types`** — `id` serial PK, `name` varchar(60) notNull, `sizeLabel` varchar(20)? (e.g. "10kg"), `purchasePrice` numeric(10,2) notNull, `sellPrice` numeric(10,2) notNull, `active` bool default true, timestamps.
- **`inventory_items`** — `id` serial PK, `name` varchar(100) notNull, `description` text?, `unit` varchar(20) default 'unidad', `purchasePrice` numeric(10,2) notNull, `sellPrice` numeric(10,2) notNull, `active` bool default true, timestamps.

Drop from the v1 shapes: `store_assignment_current_inventory` (auto-routing anti-pattern), `is_popular`, `scale`, `deleted_at` (use `active`).

Prices are **canonical here**; `inventory-foundation`'s holders snapshot `purchasePrice`/`sellPrice` at day-open so each day's sale-amount accountability uses that day's price.

### Module layout

`src/modules/catalog/{index,routes,service,repository,schema,types}.ts` per [[../../../eng/patterns/module-template]]. `index.ts` exposes `createCatalogModule({ db })`; mounted in `src/app.ts`. (Open question in [[index]]: one `catalog` module vs. splitting `stores`; decide in the plan.)

### Routes

- `GET /api/v1/stores` — list stores (active-only; `?all=1`).
- `GET /api/v1/catalog/tank-types` — list tank types (active-only; `?all=1`).
- `GET /api/v1/catalog/items` — list inventory items (active-only; `?all=1`).
- `POST /api/v1/catalog/tank-types`, `POST /api/v1/catalog/items` — admin-only create.

Thin routes: validate (Zod) → service → respond. Errors via `src/lib/errors.ts`. Role guard reuses `src/modules/auth/middleware.ts`.

### Seed

Extend `src/db/seed.ts` idempotently: insert one `stores` row; `store_assignments` binding the seeded `operator@lpg.local` + `delivery@lpg.local` users to it; ≥2 `tank_types` (e.g. 10kg, 45kg) and ≥1 `inventory_items`. Guard each insert on existence (re-runnable).

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To be created:

- `src/modules/catalog/{index,routes,service,repository,schema,types}.ts`
- `src/modules/catalog/__tests__/catalog.test.ts`
- `src/db/migrations/<n>_stores_and_catalog.sql` (drizzle-kit generate)

To modify:

- `src/db/schema.ts` — re-export `../modules/catalog/schema`
- `src/db/seed.ts` — seed store + assignments + catalog
- `src/app.ts` — mount `createCatalogModule({ db })`

v1 reference: `legacy/src/db/schemas/locations/{stores,store-assignments}.ts`, `legacy/src/db/schemas/inventory/{tank-type,inventory-item}.ts` (shapes only; do **not** port `store_assignment_current_inventory`).

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
[2026-06-05] [lpg-backend] Implemented the `catalog` module (`src/modules/catalog/`): `stores`, `store_assignments`, `tank_types`, `inventory_items` (serial PKs, `numeric(10,2)` prices, partial-unique active store-assignment guard `uq_store_assignments_active`); migration `0001_stores_and_catalog.sql`; read endpoints `GET /api/v1/catalog/{stores,tank-types,items}` (active-only, `?all=1`) + admin-only `POST /tank-types|/items`; idempotent `db:seed` extension (1 store, operator+delivery assignments, 2 tank types, 1 item). Resolved the open question: a **single** `catalog` module (not split). Also made the `db:generate` npm script require an explicit migration name (`npm run db:generate -- <name>`). 3 lifecycle tests; typecheck/lint/test/build green; migrate+seed verified against the dev DB (re-seed idempotent). Independent validation: all backend acceptance criteria met.

[2026-06-06] [lpg-backend] Added `GET /api/v1/catalog/store-assignments` (filters `?storeId=`, `?userId=`, `?all=1`) — returns enriched driver↔store mappings `{ id, active, store:{id,name}, user:{id,name,role} }` by joining `store_assignments ⨝ stores ⨝ users`. Needed by the inventory **open-day** form to offer a real driver picker instead of a raw `storeAssignmentId`. Repurposed the previously-dead `listStoreAssignments(storeId)` repo method into `listStoreAssignmentDetails(filter)`. +1 test (37 total); join verified against the dev seed (operator + delivery bound to Tienda Principal).