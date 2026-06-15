---
project: lpg-store
domain: specs
type: spec-track
spec: orders-queue-history-search
repo: lpg-backend
kind: backend
track-status: not-started
last-updated: 2026-06-14
---

# Orders Queue / History / Search — lpg-backend track

Shared spec: [[index]]

## Technical Notes

Two additive changes; the queue's date/limit/offset filters already exist.

### Accent-insensitive customer search (the shared fix)

Today `customers/repository.ts` matches with
`or(ilike(customers.name, pattern), ilike(customers.phone, pattern))` — case- but
not accent-insensitive. Replace with `unaccent()` on both sides:

```ts
// customers/repository.ts (sketch)
const pat = `%${filter.search}%`;
conds.push(
  or(
    sql`unaccent(${customers.name}) ILIKE unaccent(${pat})`,
    sql`unaccent(${customers.phone}) ILIKE unaccent(${pat})`,
  ),
);
```

Migration `<n>_unaccent_search.sql`: `CREATE EXTENSION IF NOT EXISTS unaccent;`
(idempotent; lives in the `public` schema). This one path is shared by the
order-creation picker and the customers list. No functional index for now
(`unaccent` isn't `IMMUTABLE`; small dataset — note as a perf follow-up).

### `GET /orders` free-text `search`

Add `search` (min 2 chars) to `listOrdersQuerySchema` (mirror the customers
`searchSchema`: `z.string().trim().min(2).max(100).optional()`). The list query
already `leftJoin`s `customers` (for `resolvedCustomerName`/`Phone`), so the
filter can match both the registered customer and the walk-in snapshot columns:

```ts
// orders/repository.ts listOrders — when filter.search set
const pat = `%${filter.search}%`;
conds.push(
  or(
    sql`unaccent(${customers.name}) ILIKE unaccent(${pat})`,
    sql`unaccent(${customers.phone}) ILIKE unaccent(${pat})`,
    sql`unaccent(${orders.customerName}) ILIKE unaccent(${pat})`,
    sql`unaccent(${orders.customerPhone}) ILIKE unaccent(${pat})`,
  ),
);
```

Thread `search` through `service.listOrders` into `ListOrdersFilter`. It must
**compose** with the role/branch scope (`storeIds`/`assignmentIds`) and the
`status`/`dateFrom`/`dateTo` conditions already present — i.e. it's just another
`and(...)` condition, so scoping still bounds the result (no cross-branch leak).
Keep the single-query, no-N+1 shape.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)

To modify:

- `src/modules/customers/repository.ts` — `unaccent()` name/phone matching
- `src/modules/orders/types.ts` — `search` on `listOrdersQuerySchema`
- `src/modules/orders/repository.ts` — `search` in `ListOrdersFilter` + the joined/snapshot `unaccent` OR-match
- `src/modules/orders/service.ts` — thread `query.search` into the filter
- `src/db/migrations/<n>_unaccent_search.sql` — `CREATE EXTENSION IF NOT EXISTS unaccent`
- `src/modules/{customers,orders}/__tests__/*` — accent-insensitive match; orders `search` + scope/date composition

Context (read; do not needlessly modify):

- `src/modules/orders/repository.ts` `listOrders` — already `leftJoin`s `customers` + supports `storeIds`/`dateFrom`/`dateTo`
- `src/modules/customers/types.ts` — the `searchSchema` shape to mirror
- `src/lib/{errors,date}.ts`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
