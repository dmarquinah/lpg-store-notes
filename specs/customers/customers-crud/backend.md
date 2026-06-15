---
project: lpg-store
domain: specs
type: spec-track
spec: customers-crud
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-06-08
---

# Customers CRUD — lpg-backend track

Shared spec: [[index]]

## Technical Notes

### Data model (v2 conventions: `serial` int PK, snake_case columns, `numeric` prices, `timestamptz`)

- **`customers`**
  - `id` serial PK
  - `name` varchar(100) notNull
  - `phone` varchar(20) notNull, **unique** (the lookup key; one customer per number)
  - `address` text notNull (delivery address)
  - `locationReference` text? (free-text delivery hint, e.g. "portón rojo junto a la panadería")
  - `notes` text? (operator notes)
  - `active` bool default true (soft delete — match `catalog` convention; no `deleted_at`)
  - `createdAt`/`updatedAt` timestamptz
  - Index on `phone` (covered by the unique constraint); consider a lower(name) index only if search proves slow (single small-city base — likely unnecessary).

- **`customer_debts`** (NEW — monetary debt; written later by `orders`)
  - `id` serial PK
  - `customerId` int FK → `customers.id` notNull
  - `orderId` int? — **soft reference** (no FK; `orders` table doesn't exist yet). Mirror the pattern inventory used for `customerId`. Harden to a real FK when orders lands.
  - `amount` numeric(10,2) notNull, check `amount > 0`
  - `description` text?
  - `isPaid` bool default false
  - `paidAt` timestamptz?
  - `createdAt`/`updatedAt`
  - Index `(customerId, isPaid)` for the balance aggregation.

- **`customer_debt_balance` view** — `customerId`, `outstandingBalance` = SUM(amount) WHERE NOT isPaid, grouped by customer. Same shape/pattern as the inventory balance views. Zero/absent for customers with no unpaid debt.

### FK hardening (the soft reference inventory left behind)

`inventory`'s migration created `customer_empty_debts.customer_id` as a plain
integer (see `src/modules/inventory/schema.ts` — "soft reference until the
customers module lands"). This track's migration adds:

```
ALTER TABLE customer_empty_debts
  ADD CONSTRAINT customer_empty_debts_customer_id_customers_id_fk
  FOREIGN KEY (customer_id) REFERENCES customers(id);
```

Keep the existing `idx_customer_empty_debts_cust_type`. The empty-tank debt
**balance view** (`customer_empty_debt_balance`, ADR-010, already created by
inventory) is read as-is — no changes; this module just queries it per customer
and groups by `tank_type_id` (join `tank_types` for the label).

### Module layout

`src/modules/customers/{index,routes,service,repository,schema,types}.ts` per
[[../../../eng/patterns/module-template]]. `index.ts` exposes
`createCustomersModule({ db })`; mounted in `src/app.ts`. Only `repository.ts`
imports `schema.ts`; request/response shapes are Zod + `z.infer<>`.

Per ADR-009, the customers **service** is the place other modules (orders) will
inject as a `dep` to validate a customer exists before referencing it. Keep a
small `customerExists(id)` / `getById` surface in mind even though no consumer
calls it yet this spec.

### Routes (thin: validate → service → respond; errors via `src/lib/errors.ts`)

- `GET /api/v1/customers` — list. Query: `search?` (name OR phone substring, min 2 chars → else 400), `all?` (`1` includes inactive; default active-only), `limit?`/`offset?`. Each row includes `outstandingBalance` (monetary) and `emptyTanksOwed` (total empties owed) so the UI can show a debt flag without N+1 calls.
- `GET /api/v1/customers/:id` — detail. Includes the unpaid `customer_debts` rows + `outstandingBalance`, and empty-tank debt grouped by tank type `[{ tankTypeId, tankTypeName, owed }]`. 404 if not found.
- `POST /api/v1/customers` — body `{ name, phone, address, locationReference?, notes? }`. Duplicate phone → 409 `ConflictError`.
- `PATCH /api/v1/customers/:id` — partial `{ name?, phone?, address?, locationReference?, notes?, active? }`. Phone collision → 409. `active:false` = soft delete, `active:true` = restore.

Role guard: **operator + admin** (developer bypass) on all routes, via
`src/modules/auth/middleware.ts`.

### Seed (optional, idempotent)

Optionally extend `src/db/seed.ts` with 1–2 sample customers (guarded on phone
existence so re-seeds don't duplicate) to make the frontend list non-empty in
dev. Not required for acceptance.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To be created:

- `src/modules/customers/{index,routes,service,repository,schema,types}.ts`
- `src/modules/customers/__tests__/customers.test.ts`
- `src/db/migrations/<n>_customers.sql` (`npm run db:generate -- customers`) — `customers` + `customer_debts` tables, `customer_debt_balance` view, and the `customer_empty_debts` FK constraint

To modify:

- `src/db/schema.ts` — re-export `../modules/customers/schema`
- `src/app.ts` — mount `createCustomersModule({ db })`
- `src/db/seed.ts` — (optional) seed sample customers
- `src/modules/inventory/schema.ts` — drop the "soft reference" comment / add the FK on `customerEmptyDebts.customerId` referencing `customers.id` (so the Drizzle model matches the migration)

Context (do not modify — read for the debt model):

- `src/modules/inventory/schema.ts` — `customer_empty_debts` table + soft `customerId`
- `src/modules/inventory/` — `customer_empty_debt_balance` view (ADR-010)
- `src/modules/catalog/schema.ts` — `tank_types` (for empty-debt-by-type labels)

v1 reference (shapes only; **do not** port the dropped fields):
`legacy/src/db/schemas/customers/{customers,customer-debts}.ts`,
`legacy/src/services/customerService.ts`,
`legacy/docs/customers/PRD - Customers.md`.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-08] [lpg-backend] Backend track **done**. New `customers` vertical module (`src/modules/customers/{schema,types,repository,service,routes,index}.ts`) mirroring `catalog`/`users`:
- **`customers`** table (serial PK, `name` varchar(100), `phone` varchar(20) **unique**, `address` text notNull, `locationReference?`/`notes?`, `active` default true, timestamps) and **`customer_debts`** (FK→`customers.id`, soft `orderId`, `amount` numeric(10,2) + `> 0` check, `isPaid`/`paidAt`, index `(customerId,isPaid)`). `customer_debt_balance` view (`pgView(...).existing()`, hand-written `CREATE VIEW` in the migration like the inventory balance views) = `SUM(amount) WHERE NOT is_paid` per customer.
- **FK hardening**: migration `0005_customers.sql` adds `customer_empty_debts_customer_id_customers_id_fk`; `idx_customer_empty_debts_cust_type` retained. Drizzle model updated in `inventory/schema.ts` (`.references(() => customers.id)`), so model matches DB.
- **Read API** (`/api/v1/customers`, guarded operator+admin, developer bypass): `GET /` list with `?search=` (name OR phone `ilike`, min-2-else-400), `?all=1`, `?limit`/`?offset`; each row carries `outstandingBalance` (monetary, via the view) + `emptyTanksOwed` (summed `customer_empty_debt_balance`) in one query — no N+1. `GET /:id` detail = unpaid `customer_debts` rows + monetary balance + empty-tank debt grouped by tank type (joined `tank_types` for labels), 404 unknown. `POST /` (409 dup phone). `PATCH /:id` partial incl. `active` soft-delete, 409 on phone collision.
- Wired in `app.ts`; `customers/schema` re-exported from `db/schema.ts`; `db/seed.ts` seeds 2 sample customers (phone-guarded). 4 lifecycle tests added.
- **Gates**: typecheck + lint + build green; 48 tests pass (4 new). Migration applied to the dev DB and **smoke-verified**: `customer_debt_balance` view live (empty → reads 0), empty-debt FK present, seed customers inserted. Independent validation agent confirmed all 11 backend criteria met, no gaps.
- Note for `orders`: per ADR-009/ADR-012 the customers **service** is the injection point to validate a customer exists; `findById` is the seam (no consumer yet). `orders` is the future writer of `customer_debts` (balance reads 0 until then).