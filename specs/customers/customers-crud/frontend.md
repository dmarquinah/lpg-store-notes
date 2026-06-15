---
project: lpg-store
domain: specs
type: spec-track
spec: customers-crud
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-08
---

# Customers CRUD — lpg-frontend-vue track

Shared spec: [[index]]

> **Depends on the [[backend]] track** — it consumes `GET/POST/PATCH
> /api/v1/customers`. Implement after the backend track ships (or against an
> agreed-stable contract).

## Technical Notes

New `customers` vertical module (`src/modules/customers/`), mirroring the
existing `users` and `catalog` modules per
[[../../../eng/patterns/frontend-module-template]]:

```
src/modules/customers/
  index.ts          # createCustomersModule({ apiClient, ... }) — wired in src/main.ts
  types.ts          # Customer, CustomerDebtSummary, create/update payloads (mirror backend Zod)
  service.ts        # GET/POST/PATCH /customers; unwrap the { customer | customers } envelope
  store.ts          # Pinia store: list + filters (search, showInactive), current customer
  routes.ts         # /clientes (list), /clientes/:id (detail/edit); meta.roles
  views/
    CustomersView.vue       # list + search + show-inactive toggle + debt badges
    CustomerDetailView.vue  # detail + edit form + debt summary
  components/
    CustomerFormDialog.vue  # create/edit form (client validation mirroring Zod)
```

- **List** — debounced search box (name/phone), "Mostrar inactivos" switch
  driving `?all=1`, per-row debt flag: a badge when `emptyTanksOwed > 0`
  ("debe envases") and/or `outstandingBalance > 0` ("debe S/ …"). Loading flag
  to avoid spinner flicker (as in `catalog`).
- **Form** — `name`, `phone`, `address`, `locationReference?`, `notes?`,
  `active`. Client-side validation mirroring the backend Zod (required name /
  phone / address; phone format). Surface the backend's duplicate-phone 409 as a
  field-level error on `phone`.
- **Detail** — debt summary panel: empty-tank debt rows by tank type (names
  resolved via the `catalog` module / the detail payload) + monetary outstanding
  balance. Read-only for now (settlement is out of scope).
- **Theme/UI** — reuse the shadcn-vue components already in `src/components/ui/`
  (table, input, switch, dialog, badge). No new design system work.

### Routing / nav

- Routes guarded `meta: { roles: ['operator', 'admin', 'developer'] }` (the
  single global guard in `src/router/index.ts` enforces it).
- Add a **"Clientes"** entry to the **operator** and **admin** nav in
  `AppLayout`'s `ROLE_NAV` (operators reach it via the drawer, as with
  "Inventario").

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-frontend-vue)

To be created:

- `src/modules/customers/{index,types,service,store,routes}.ts`
- `src/modules/customers/views/{CustomersView,CustomerDetailView}.vue`
- `src/modules/customers/components/CustomerFormDialog.vue`

To modify:

- `src/main.ts` — register `createCustomersModule(...)`
- `src/layouts/AppLayout.vue` — add "Clientes" to `ROLE_NAV` (operator + admin)

Pattern references (read, don't modify):

- `src/modules/users/` — closest analog (list + filters + guard-mirroring edit form)
- `src/modules/catalog/` — show-inactive toggle, `?all=1`, per-list loading flags, envelope unwrapping
- `src/modules/inventory/` — how the frontend renders server-sourced balances (debt summary analog)
- `src/lib/apiClient.ts`, `src/lib/{errors,types}.ts` — API wrapper + `ApiResponse<T>` envelope

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-08] [lpg-frontend-vue] Frontend track **done**. New `customers` vertical module (`src/modules/customers/{types,service,store,routes,index}.ts` + `views/{CustomersListView,CustomerDetailView}.vue` + `components/CustomerFormDialog.vue`) mirroring `users`/`catalog`: module-scoped `provideCustomersService`, store never touches the ApiClient. Built against the **already-shipped** backend contract (read from `lpg-backend/src/modules/customers/`), so envelope keys + shapes match exactly — list `{customers}` (CustomerListItem + `outstandingBalance`/`emptyTanksOwed`), detail/create/patch `{customer}` (CustomerDetail with `unpaidDebts` + `emptyDebtsByType`).
- **List** (`/clientes`): debounced (300ms) name/phone search that skips the API on a 1-char query (backend min-2), "Mostrar inactivos" switch → `?all=1`, per-row debt badges (`N envases` / `S/ x.xx` / "Al día"), Ver→detail + Editar.
- **Form dialog** (create + edit in one): client validation mirroring the backend Zod (name 1–100, phone 1–20, address ≥1); create omits empty optionals; edit sends only changed fields, clears emptied `locationReference`/`notes` to `null`, requires ≥1 change. Duplicate-phone **409** surfaced as a field-level error under the phone input (via a new `ConflictError` in `src/lib/errors.ts` + 409 mapping in `apiClient`; store exposes a `conflict` flag).
- **Detail** (`/clientes/:id`): contact card + debt summary — empty-tank debt by tank type (uses `tankTypeName` from the payload; no catalog dependency) with total, and monetary outstanding balance + unpaid-debt list.
- **Wiring**: `createCustomersModule` in `main.ts`, `customersRoutes` in the router (guarded `roles: [operator, admin, developer]`), "Clientes" added to the operator + admin (+ developer) `ROLE_NAV` in `AppLayout`.
- **Gates**: typecheck (vue-tsc) + build green. Independent validation agent confirmed all 5 frontend criteria + full contract match; it caught one latent bug — `isoToDisplay(createdAt)` rendered blank because the helper is date-only while `createdAt` is a full ISO datetime — fixed by passing the `slice(0,10)` date prefix. No test runner wired (per CLAUDE.md); manual smoke test against a running backend left to the operator.

[2026-06-08] [lpg-frontend-vue] All criteria for this repo met.
