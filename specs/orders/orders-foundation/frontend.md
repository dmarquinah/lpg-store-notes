---
project: lpg-store
domain: specs
type: spec-track
spec: orders-foundation
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-10
---

# Orders Foundation — lpg-frontend-vue track

Shared spec: [[index]]

> Ports **after** the backend track is `done` (it owns the API this reads).

## Technical Notes

### Module pattern

New `src/modules/orders/` vertical slice mirroring the established
`customers`/`inventory`/`users` modules
([[../../../eng/patterns/frontend-module-template]]): `types`/`service`/`store`/
`routes`/`index` + `views/` + `components/`. Wire in `src/main.ts` and the router;
role variants live **inside** the module (sibling components), never as parallel
role folders (the v1 anti-pattern — [[../../../eng/frontend-bloat-analysis]]).
Service hits `/api/v1/orders` through the shared `ApiClient`. No i18n
(hardcoded Spanish), no new global stores beyond the module store.

### Screens / role surfaces

- **Order entry (operator)** — the phone-driven create form, the product's core
  operator interaction ([[../../../product/overview]]):
  - Customer step: search/select by **name or phone** (reuse the `customers`
    module's service/search), **or** a quick walk-in (name + phone, no record).
    Include the **inline "new customer on the call"** create (the `customers`
    spec deferred this here) — a compact dialog reusing the customers create form,
    surfacing the duplicate-phone 409 as a field error.
  - Lines: add **tank** / **item** lines from the `catalog`; `unitPrice`
    pre-filled from catalog `sellPrice`, **editable** per line; per-tank
    `emptyReceived` defaulting to qty (the exchange).
  - Live `totalAmount`. Client validation mirrors the backend Zod schema.
- **Order queue / list** — filter by `status`; show `paymentStatus` + outstanding
  badges; per-row role-aware actions (assign / dispatch / cancel).
- **Assign** — pick a driver's **open** assignment (reuse inventory's assignment
  picker / `store-assignments`); show the backend oversell warning if returned.
- **Delivery view (driver)** — list orders assigned/in-transit to the driver;
  **dispatch**, then **deliver** (capture per-line `emptyReceived` + an optional
  payment with method) or **fail** (reason).
- **Order detail** — lines, **status timeline** (from `order_status_history`),
  payments list, outstanding, and a **record-payment** action (usable after
  delivery to pay down debt); link through to the customer.

### Routing & nav

Routes guarded by role: operator + admin for entry + queue + detail + record-
payment; delivery for the delivery view + dispatch/deliver/fail; developer bypass.
Add nav entries in `AppLayout`'s `ROLE_NAV` — "Pedidos" for operator/admin, a
delivery entry (e.g. "Mis entregas") for the driver drawer.

### Reuse, don't duplicate

- Customer lookup/create → the `customers` module (service + create form).
- Tank-type / item names + prices, assignment picker → the `catalog` /
  `inventory` modules already shipped.
- Money/empty-debt display conventions → match the `customers` detail debt
  summary already built.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To create:

- `src/modules/orders/{types,service,store,routes,index}.ts`
- `src/modules/orders/views/` — order list/queue, order entry, order detail, delivery view
- `src/modules/orders/components/` — line editor, customer-select/quick-create, payment dialog, status timeline

To modify:

- `src/main.ts` — register the orders module
- `src/router/index.ts` — mount orders routes (role guards)
- `src/layouts/AppLayout.vue` — add "Pedidos" / delivery nav entries to `ROLE_NAV`

Context (read / reuse):

- `src/modules/customers/` — customer search/select + create form (inline quick-create), debt-display conventions
- `src/modules/catalog/` — tank-type/item lists + prices
- `src/modules/inventory/` — assignment picker / store-assignments, balance displays
- `src/lib/{apiClient,errors}.ts` — `ApiClient`, `ConflictError` (409 → field error) and friends
- `src/modules/auth/` — role guards / current user

v1 reference (UX flows only; ignore the role-as-folder structure and i18n):

- `lpg-frontend-vue/legacy/src/components/{operator,delivery}/orders/`
- `lpg-frontend-vue/legacy/docs/orders/`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->


[2026-06-10] [lpg-frontend-vue] **Frontend track complete.** New `orders` vertical
module (`types`/`service`/`store`/`routes`/`index` + `views/` + `components/`)
mirroring customers/inventory; wired in `main.ts`, `router/index.ts`, and
`AppLayout` ROLE_NAV ("Pedidos" for operator/admin/developer, "Mis entregas" for
delivery). Two role surfaces in ONE module via role-aware routes: office queue +
multi-step entry under `/pedidos`; driver list under `/mis-entregas`
(roles delivery+developer).

- **Entry wizard** (operator-speed, modeled on v1's `Stepper`/`QuickOrderForm`,
  minus v1's payment step — v2 takes payment at delivery): Cliente → Productos →
  Resumen, driven by `OrderStepper`. `CustomerSelect` reuses `useCustomersStore`
  search + walk-in + inline create (reuses `CustomerFormDialog`, which now emits
  `created`; the customers store `createCustomer` now returns/sets the created
  detail). `OrderLinesEditor` adds tank/item lines from the catalog with editable
  prefilled prices + live total (`OrderCartSummary`). Resumen shows a prominent
  "Total a pagar" + a **required Método de pago** select.
- **Queue** (`OrdersListView`): status filter, payment/outstanding badges, per-row
  assign/dispatch/cancel by status. **Assign** (`AssignDialog`) picks a driver's
  open inventory assignment (reuses inventory/catalog stores), surfaces the
  backend oversell warnings, and offers "cargar balones faltantes" via
  `inventory.recordLoad`.
- **Detail** (`OrderDetailView`, shared by both surfaces): lines, money summary,
  `StatusTimeline`, payments, linked inventory tx, customer link, and a
  role+status-aware action bar (assign/dispatch/deliver/fail/cancel/record-pago).
- **Driver view** (`DeliveryListView`): the driver's assigned/in-transit orders
  (load-on-mount + manual "Actualizar" — no polling, per decision); dispatch,
  deliver (`DeliverDialog`: per-line emptyReceived + optional payment, method
  prefilled from the order), fail. Labels/variants centralized in `orderLabels.ts`.

**Cross-repo [lpg-backend] changes (user-approved during this track):**
1. **Opened `GET /orders` to the delivery role**, scoped to the driver's own
   assignments: route guard `office`→`anyStaff`; `service.listOrders(query,
   caller)` derives the assignment set via `inventory.listAssignments({userId})`;
   repo gained `assignmentIds` (inArray; empty → []). Also consolidated the
   duplicate `field`/`anyStaff` role sets into one `anyStaff`.
2. **Added nullable `orders.payment_method`** (the method agreed at registry) —
   migration `0007_orders_payment_method.sql`; `createOrderSchema.paymentMethod`
   optional (UI requires it); persisted + returned on summary/detail; prefilled
   into the delivery payment.

**Gates:** frontend typecheck + build green. Backend typecheck + lint + **54
tests** + build green; migration `0007` applied to the dev DB. Independent
validation agent confirmed all 8 frontend criteria met, no high/medium contract
or logic bugs.


[2026-06-10] [lpg-frontend-vue] Follow-up: the driver `DeliveryListView` now shows
the driver's **full** order list (the backend already returns all the driver's
orders) with a client-side bucket filter — **Por entregar** (assigned/in_transit,
default) · **Con saldo pendiente** (delivered & not fully paid) · **Entregadas** ·
**Todas** — plus a Pago column with the outstanding balance, so drivers can reach
past and still-owing orders (previously the list hard-filtered to active only).
Recording the remaining payment stays **office-only** (operator/admin) by
decision; drivers view the balance and order detail but don't record payments.