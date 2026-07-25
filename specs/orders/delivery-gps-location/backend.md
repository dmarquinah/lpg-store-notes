---
project: lpg-store
domain: specs
type: spec-track
spec: delivery-gps-location
repo: lpg-backend
kind: backend
track-status: done
last-updated: 2026-07-15
---

# GPS delivery location (pin per customer) — lpg-backend track

Shared spec: [[index]]

## Technical Notes
**Final model: customers-only sticky pin.** No `orders` columns — the order read
resolves the customer's pin through the existing `customers` join.

- **Schema.** Nullable `latitude`/`longitude` (`numeric(9,6)`) on `customers`
  (`src/modules/customers/schema.ts`) with three DB CHECKs: both-or-neither
  (`(latitude IS NULL) = (longitude IS NULL)`), lat ∈ [-90,90], lng ∈ [-180,180].
  Migration `0018_customer_gps_location`.
- **Driver capture at delivery.** `deliverOrderSchema` gains optional
  `location: { latitude, longitude }` (WGS84-bounded). In `deliverOrder`'s tx
  (ADR-012), when `location` is present and the order has a `customerId`, call
  `customers.setLocation(customerId, lat, lng, { overwrite: false }, h.customers)`.
  **Fill-when-empty only** — delivery never overwrites. Walk-ins → skipped; never blocks.
- **Sticky write rule.** `CustomersRepository.setLocation` runs a conditional
  `UPDATE ... WHERE id = $id AND ($overwrite OR latitude IS NULL)` — delivery passes
  `overwrite: false` (no-op once a pin exists); the office PUT passes `true`.
- **Pin update endpoint.** Dedicated `PUT /customers/:id/location`
  (`setCustomerLocationSchema`, both coords required + range-validated) →
  `CustomersService.updateLocation(id, lat, lng, caller)` →
  `setLocation(..., { overwrite: true })`. The general `PATCH /customers/:id` carries
  no coordinates. Route allows `operator`/`admin`/`delivery` (registered before the
  blanket operator/admin guard).
- **Delivery-role guard (ADR-012).** For a `delivery` caller, `updateLocation`
  consults an injected `DeliveryLocationAuthorizer` (late-bound in `app.ts`): allowed
  only when the driver has a live order (`hasOrderForCustomerOnAssignments`, non-
  terminal) for the customer on one of **today's** assignments
  (`inventory.listAssignments({ userId, date: businessToday() })`). Unset authorizer
  or no match → `ForbiddenError`. Operators/admins skip the check.
- **Reads.** `PublicCustomer` exposes `latitude`/`longitude`; the orders repo's
  customer join selects `resolvedCustomerLatitude`/`Longitude` and `OrderSummary`
  exposes `customerLatitude`/`customerLongitude` for the Maps link.
- **Validation.** Coordinate ranges enforced in Zod (matching the DB CHECKs) on
  both the deliver `location` object and the customer update; a lone lat/lng is a
  400 before any write.

## Related Files

### lpg-backend (/home/diegomh/dev/personal/freelance/lpg-store/lpg-backend)
- `src/modules/customers/schema.ts` — `latitude`/`longitude` columns + CHECKs.
- `src/db/migrations/0018_customer_gps_location.sql` — the migration.
- `src/modules/customers/types.ts` — `setCustomerLocationSchema`; `PublicCustomer` + `toPublicCustomer`; `latitudeSchema`/`longitudeSchema`.
- `src/modules/customers/routes.ts` — `PUT /:id/location`.
- `src/modules/customers/repository.ts` — `SetLocationOptions`, `ICustomersRepository.setLocation` + impl.
- `src/modules/customers/service.ts` — `setLocation` seam (orders tx); `updateLocation` (PUT) + `DeliveryLocationAuthorizer` port/setter.
- `src/app.ts` — late-binds the authorizer (inventory today-assignments + orders `hasOrderForCustomerOnAssignments`).
- `src/modules/orders/service.ts` — `deliverOrder` capture; `toSummary` coords; `hasOrderForCustomerOnAssignments`.
- `src/modules/orders/types.ts` — `deliverOrderSchema` `location`; `OrderSummary` coords.
- `src/modules/orders/service.ts` — `deliverOrder` in-tx capture; `toSummary` coords.
- `src/modules/orders/repository.ts` — `OrderRow` + both join selects + `toOrderRow` resolved coords.
- `src/modules/customers/__tests__/helpers.ts` + `customers.test.ts` — fake `setLocation`/`seedCustomer` coords; office set/clear/half-pin test.
- `src/modules/orders/__tests__/helpers.ts` + `orders.test.ts` — fake resolved coords; deliver-capture lifecycle tests.

## Implementation Notes
<!-- [YYYY-MM-DD] [lpg-backend] description -->
- [2026-07-15] [lpg-backend] Implemented the customers-only sticky-pin model. Added `customers.latitude/longitude` (`numeric(9,6)`) + CHECKs, migration `0018_customer_gps_location`. Coords exposed on `PublicCustomer` and `OrderSummary` (`customerLatitude`/`customerLongitude`) via the existing join.
- [2026-07-15] [lpg-backend] Per owner feedback, split the write paths: **delivery capture fills an empty pin only** (dropped the `updateLocation` overwrite flag), and **correcting an existing pin is a dedicated `PUT /customers/:id/location`** (`updateLocation` service method, overwrite:true) instead of folding coords into the general `PATCH`.
- [2026-07-15] [lpg-backend] Opened `PUT /customers/:id/location` to the **delivery** role under a guard: a driver may correct a pin only for a customer they have a live order for on **today's** assignment. Implemented as a late-bound `DeliveryLocationAuthorizer` (ADR-012) composed in `app.ts` from `inventory.listAssignments({date: today})` + `orders.hasOrderForCustomerOnAssignments`; unset/no-match → 403. Route registered before the blanket operator/admin guard. Full gate green (typecheck, biome, 184 tests, build). All backend acceptance criteria met — track done; frontend track remains.
