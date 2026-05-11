---
project: lpg-store
domain: eng
last-updated: 2026-05-08
---

# Why the v1 Frontend Grew to ~32.7k LOC for ~25 Screens

A diagnosis of the architectural patterns that inflated the v1 Vue dashboard, written so the v2 frontend specs can cite specific drivers when justifying design choices. This is the frontend counterpart to [[legacy-bloat-analysis]].

## Headline numbers

| Area | LOC | Note |
|------|-----|------|
| Total v1 frontend | **32,718** | non-test `*.ts` + `*.vue` under `src/` (212 files) |
| `components/` | **15,776** | 112 files, role-split: `admin/`, `delivery/`, `operator/`, `common/` |
| `stores/` (Pinia) | **4,895** | 17 files, role-split + 3 root stores |
| `services/` | **3,432** | 18 files; mixed role-split + global services |
| `views/` | **2,727** | 12 files, one per route × role |
| `i18n/` | **2,567** | 20 files; 9 modules × 2 locales + loader |
| `types/` | **1,365** | 8 files; flat tree, several dual-API duplicates |
| `notificationService.ts` + `notificationStore.ts` | **938** | Firebase FCM stack (slated for removal in v2) |

The product has roughly **25 screens** (login + register + 4 admin views with subroutes + 5 delivery views + 4 operator views + shared profile/settings). ~33k LOC for ~25 screens (~1.3k LOC per screen) is not a feature problem — it is an architecture problem, mirroring the backend's diagnosis.

## Ten drivers of bloat

Each driver lists the v1 files (still under `src/` today, since the frontend reset has not happened yet) that exemplify it, so the link survives even if a path moves into `legacy/`.

### 1. Role-as-folder duplication

`components/`, `views/`, `stores/`, `services/`, and `layouts/` are each split into `admin/`, `operator/`, `delivery/` (and an empty `superadmin/`). The same business feature shows up in 2–3 role folders with parallel components and stores:

- `components/admin/inventory/` + `components/delivery/inventory/`
- `components/operator/orders/` + `components/delivery/orders/`
- `stores/admin/adminInventoryStore.ts` + `stores/admin/adminInventoryTransactionStore.ts` + `stores/delivery/deliveryInventoryStore.ts` (3 stores for *one* domain)
- `services/delivery/orders.ts` + `services/operator/orderService.ts`

A v2 module aligned by domain (`modules/inventory/`, `modules/orders/`) — with role-aware UI variants where they actually differ — collapses these triples into one source of truth. Most divergence between role variants is presentational (column choices, button gates), not data-shape.

**Cost**: every cross-cutting change ("the order status enum changed") fans out into 2–3 places that drift independently.

### 2. i18n stack for one deployment language

`src/i18n/locales/{en,es}/*` is **2,409 LOC** of translation strings across 9 modules × 2 locales. `i18n/index.ts` (124) and `i18n/router.ts` (34) plus `meta.i18nModules` on every route definition wire up lazy-loaded modules that load the right bundle on navigation.

The product is sold to a single Spanish-speaking Peruvian operator. There is no language switcher in production and no path to multilingual users in the MVP. The English locale exists as scaffolding for a future that never materialized.

**Cost**: every UI string carries a translation key and a parallel `en.ts` entry; every new screen requires a new `i18nModules` declaration; `vue-i18n` adds runtime weight. **Verdict**: drop entirely in v2. Spanish becomes plain Vue template text. Reintroduce only when a real second-language user appears.

### 3. Dual transaction API surface

`src/services/inventoryService.ts` (189 LOC) carries two parallel APIs:

- the **complex** transaction API (separate full/empty deltas, manual sign math, used by `TransactionForm.vue`)
- the **simplified** transaction API (single quantity field, business logic on the backend, validation endpoint, type metadata endpoint)

`src/types/inventoryTypes.ts` carries both request/response shapes (`CreateTankTransactionRequest` *and* `EnhancedTankTransactionRequest`, `BatchTankTransactionsRequest` *and* `EnhancedBatchTankTransactionsRequest`). `adminInventoryTransactionStore.ts` (309 LOC) exposes both surfaces.

Backend v2 collapses to a single signed-delta API per ADR-005. Frontend v2 follows: one shape, one store method, one form. Drop the deprecated paths during the inventory module port.

### 4. Five-state inventory workflow types

`src/types/inventoryTypes.ts` defines `InventoryStatusType` with `CREATED / ASSIGNED / CONSOLIDATED / VALIDATED / OBSERVED`, mirroring the v1 backend. Status-specific UI helpers (`useInventoryStatusBadge.ts`, `StatusUpdateForm.vue`, sort priority, transition matrix) compound around the five states.

Backend v2 collapses to **three states** (open / closed / carried) per ADR-008. Frontend types must follow when porting, dropping `VALIDATED` and `OBSERVED` from the enum and rewriting the badge/transition logic to fit.

### 5. Firebase FCM push notifications (~940 LOC)

`src/services/notificationService.ts` (460 LOC) + `src/stores/notificationStore.ts` (478 LOC) + `src/services/firebase/config.ts` + the SDK in `dependencies` add up to a full push pipeline: token registration, foreground handlers, permission UX, manual activation gating.

[[../product/overview]] explicitly drops FCM from v2 MVP scope ("was built in v1, dropped from v2 unless a clear delivery-confirmation use case re-emerges"). Carrying the v1 stack into the v2 frontend would re-introduce ~1k LOC, the Firebase dependency, an FCM project, and the service-worker integration — for a feature the product does not need.

**Verdict**: drop entirely. PWA installability stays (it is independent of FCM); push notifications wait for a real use case.

### 6. Per-role-per-feature Pinia stores

Inventory is the worst offender:

- `stores/admin/adminInventoryStore.ts` (318)
- `stores/admin/adminInventoryTransactionStore.ts` (309)
- `stores/delivery/deliveryInventoryStore.ts` (497)

That is **1,124 LOC** across three stores for one domain. Each store fetches the same backend resource through a different role lens, with overlapping state shapes and mostly-parallel actions. The split is done because the role-folder layout *forces* it — there is no shared module to put a single store in.

Same pattern in orders: `stores/delivery/deliveryOrdersStore.ts` (587) + `stores/operator/operatorOrderStore.ts` (543) + `stores/operator/operatorOrderAssignmentStore.ts` (279) = **1,409 LOC**.

A v2 module pattern places one store per domain (e.g. `modules/inventory/store.ts`) with role-aware getters/actions where roles legitimately diverge.

### 7. Role-scoped layouts × role-scoped sidebars

`layouts/admin/BaseLayout.vue`, `layouts/delivery/BaseLayout.vue`, `layouts/operator/BaseLayout.vue`, plus `components/admin/Sidebar.vue` and `components/operator/Sidebar.vue` (and a root `Sidebar.vue`) — three near-identical shells differing mostly in nav items and theme accents.

Layout total is small (163 LOC), but the *pattern* of "make a parallel folder per role" is what compounds elsewhere. v2 should have one layout component that takes nav config as a prop.

### 8. Service-worker generation + PWA manifest hard-coding

`scripts/generate-service-worker.js` runs before every `dev` and `build`. `vite-plugin-pwa` is also configured (via `VitePWA({})` in `vite.config.ts`), creating a second SW pipeline. The manifest in `vite.config.ts` hard-codes `start_url: "/delivery/dashboard"` and shortcut URLs to delivery-only routes.

Two SW pipelines is one too many. The PWA manifest should also account for the operator and admin landing pages, not just delivery. **Verdict in v2**: keep `vite-plugin-pwa`, drop the custom `generate-service-worker.js` script, parameterize the manifest start_url by deployment.

### 9. `services/api/apiService.ts` reads localStorage directly

`apiService.ts` does `const token = localStorage.getItem("token")` on every request. This couples the fetch wrapper to the auth store's persistence choice and makes the wrapper untestable without stubbing `localStorage`. Routes also redirect on auth state via direct `authStore` access in the guard chain — the dependency graph from "I need to call the API" to "auth knows where the token lives" is implicit.

v2 inverts this: the API client takes a `getToken()` callback (or a `tokenProvider` instance) injected by the auth module's composition. localStorage stays inside `modules/auth/` only.

### 10. Empty `superadmin/` folder + speculative role splitting

`src/stores/superadmin/` exists and is empty. `UserRoles.SUPERADMIN` is defined and routed (`/superadmin` redirects to `admin-users`), but no UI is unique to it. Same pattern with the `roleDefaultRoutes` table including SUPERADMIN as a synonym of ADMIN.

Speculative roles + speculative folders compound driver 1: every new feature has to decide "do I add a `superadmin/` variant?" when no use case exists. v2 collapses SUPERADMIN to a permission flag on the ADMIN role, or removes it until a real divergence emerges.

## Diagnosis in one sentence

The screen count was small. The bloat was **organizing by user role at every layer** (folders, stores, services, layouts, route trees), plus **scaffolding for futures that never arrived** (English locale, two transaction APIs in parallel, push notifications, superadmin role) — both compounding into 2–3× duplication for every feature.

## What v2 takes from this

The first frontend spec to apply these is `[[../specs/frontend-bootstrap/v2-skeleton]]` (status: draft). It bakes the lessons into rules:

- **D1, D6, D7** are killed by **module-by-domain layout** (`src/modules/<feature>/` with role-aware variants inside) — analogous to backend [[patterns/module-template]] but adapted for Vue/Pinia.
- **D2** is killed by **dropping i18n** entirely until a real second-language user appears.
- **D3, D4** ride along with the inventory module port (waits on backend ADR-005 + ADR-008 to land).
- **D5** is killed by **dropping Firebase + notification stack** from the v2 dependency tree.
- **D8** is killed by **using `vite-plugin-pwa` only** and parameterizing the manifest.
- **D9** is killed by **injecting a token provider** into the API client at module-composition time.
- **D10** is killed by **collapsing SUPERADMIN** until divergence is real.

The order in which v2 modules port should match the backend's spec order so frontend lands on top of working backend endpoints — see `[[../specs/index]]` for the porting roadmap.
