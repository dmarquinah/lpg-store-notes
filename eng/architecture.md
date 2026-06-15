---
last-updated: '"2026-05-08"'
---
# Architecture — lpg-store

---
project: lpg-store
domain: eng
last-updated: 2026-05-07
---

## Backend (lpg-backend, v2)

### Stack

- Node.js 22, TypeScript
- Express 5
- Drizzle ORM + Postgres (postgres-js)
- Zod for validation
- Pino for logs
- Redis (ioredis) for cache
- Node's built-in test runner (`node --test`), no jest
- Docker (multi-stage, distroless final at ~175MB), GHCR + GitHub Actions, VPS as host

### Layout

```
src/
  app.ts                # express factory: middleware + module mounts
  server.ts             # http server boot
  config/env.ts         # zod-validated env
  db/                   # drizzle client, migrations, schema re-exports
  middleware/           # error handler, request logger, 404
  lib/                  # logger, errors, cache
  modules/              # vertical feature modules (populated as features port)
```

Each feature is a self-contained vertical module — see [[patterns/module-template]] for the convention.

### Key conventions

- **Composition root** is `src/app.ts`; no DI container. Each module exposes `createXxxModule({ db, deps })` which returns a `{ router, ... }` object that `app.ts` mounts.
- **Repositories** are the only files that import schema. Services accept repository instances; routes accept services. Keeps types narrow and enables easy substitution for tests.
- **Migrations are real**: `drizzle-kit generate` produces SQL files in `src/db/migrations/`, which run via `drizzle-orm/postgres-js/migrator`. No `db:push` after the first deploy.
- **No path aliases** (`@/...`). `tsc` doesn't rewrite them at compile time and the runtime resolver doesn't know about them, so the distroless image was failing to require modules. Relative imports only.
- **Errors**: `src/lib/errors.ts` (`AppError` and subclasses); the central `errorHandler` middleware turns them into JSON responses. Zod errors get the same treatment.

### What v2 deliberately drops from v1

Lessons from v1 (now under `legacy/src/`) baked into the v2 reset:

- Custom DI container with 7 module files → composition in `app.ts`
- 5-strategy Transaction Strategy Pattern (~1k LOC, transfer was incomplete) → revisit per-feature when porting; the pattern earns its keep only with real polymorphism
- 998-LOC `PgOrderWorkflowRepository` mixing CRUD + analytics + workflow + querying → split into focused services per port
- 8-permutation `increment*/decrement*/byInventoryId/byAssignmentId` repository methods → collapse to signed-delta methods when porting
- Generic `audit_logs` table → drop; rely on order_status_history + tank/item_transactions as the trail
- Vehicles + finance + notifications schemas → drop entirely (out of MVP scope)
- Fly.io + Supabase deploy + multi-env config → single Docker stack on VPS

### Related Files

Backend (current v2):

- `lpg-backend/src/app.ts`
- `lpg-backend/src/server.ts`
- `lpg-backend/src/config/env.ts`
- `lpg-backend/src/db/{client,schema,migrate}.ts`
- `lpg-backend/src/middleware/errorHandler.ts`
- `lpg-backend/src/lib/{logger,errors,cache}.ts`
- `lpg-backend/Dockerfile`
- `lpg-backend/docker-compose.yml`
- `lpg-backend/.github/workflows/main.yml`

V1 archive (reference only):

- `lpg-backend/legacy/src/` — full v1 source
- `lpg-backend/legacy/docs/` — v1 PRDs (Orders, Inventory, Customers, Products)
- `lpg-backend/legacy/CLAUDE.local.md` — original architecture notes

## Frontend (lpg-frontend-vue)
Operator dashboard + delivery PWA, served as a single Vue 3 SPA. **Currently still v1**: ~32.7k LOC across 212 files under `src/`, organized by user role. v2 has not started — the repo will follow the same playbook as the backend (legacy archive + clean reset + module ports), but on a different cadence so frontend modules can land on backend modules as they ship.

### Stack (v1, current)

- Vue 3 (Composition API) + TypeScript, Vite 5
- Pinia for state, Vue Router 4 for routing, Vue i18n 9 for translations
- Tailwind CSS + Headless UI + Heroicons
- `@vuepic/vue-datepicker`, Chart.js + vue-chartjs, Leaflet (delivery map)
- Firebase (FCM push notifications)
- `vite-plugin-pwa` + Workbox (installable PWA, NetworkFirst on `/v1/*`)

### Layout (v1, current)

```
src/
  main.ts                 # createApp + pinia + router + i18n + tooltip plugin
  App.vue
  router/                 # routes.ts, index.ts (guards), translations.ts
  layouts/                # role-scoped: admin/, delivery/, operator/ + AuthLayout.vue
  views/                  # role-scoped: admin/, delivery/, operator/, auth/
  components/
    common/               # Toast, DatePicker, Stepper
    admin/                # inventory/, stores/, users/ (+ Sidebar.vue)
    delivery/             # dashboard/, inventory/, orders/
    operator/             # dashboard/, orders/ (+ Sidebar.vue)
    Header.vue, Sidebar.vue, LoadingSpinner.vue, ReloadButton.vue
  stores/                 # role-scoped: admin/, delivery/, operator/, superadmin/ (empty)
                          # + authStore, configStore, notificationStore at root
  services/
    api/                  # apiService.ts, apiUtils.ts, apiErrors.ts, types.ts
    delivery/             # dashboard.ts, inventory.ts, orders.ts
    operator/             # orderService.ts
    firebase/              # FCM config
    authService, customerService, inventoryService, productService,
    storeService, userService, notificationService,
    pwaInstallHandler, serviceWorkerHandler
  i18n/                   # index.ts, router.ts, locales/{en,es}/{auth,core,dashboard,
                          # inventory,orders,profile,settings,stores,users}.ts
  types/                  # auth, customerTypes, inventoryTypes, orderTypes,
                          # productTypes, storeTypes, userTypes, dashboard
  composables/, utils/, directives/, plugins/
```

### Key v1 conventions (for reference; v2 will not inherit all)

- **Role-as-folder**: `components/`, `views/`, `stores/`, `services/`, `layouts/` each split into `admin/ | operator/ | delivery/`. The same feature (e.g. orders) appears in 2–3 role folders with parallel components and stores.
- **Role-based routing** in `src/router/index.ts`: route meta carries `requiresAuth`, `roles[]`, `i18nModules[]`. A guard chain (`handleUnauthenticatedAccess` → `handleAuthPagesAccess` → `handleMissingUserRole` → `handleRoleBasedAccess`) enforces redirects + lazy user fetch.
- **Lazy-loaded translations**: each route declares `meta.i18nModules`, and `setupI18nRouter` loads only those locale modules on navigation.
- **Auth via localStorage token**: `apiService.ts` reads `localStorage.getItem("token")` and injects `Authorization: Bearer …` on every request. Base URL = `VITE_API_URL` (default `http://localhost:5000`) + `/v1`.
- **API response shape**: `{ data, pagination?, error? }`. `apiRequestUnwrapped` calls `unwrap()` to throw on `error` or return `data`.
- **PWA**: `scripts/generate-service-worker.js` runs before `dev` and `build`; `VitePWA` autoUpdates and caches `/v1/*` with `NetworkFirst` (3s timeout, 24h TTL). Manifest hard-codes `start_url: /delivery/dashboard`.
- **Path alias**: Vite's `resolve.alias["@"] = src/` is configured but `vue-tsc` is the type checker, so the alias works in builds. (Backend's distroless `tsc`/Node mismatch does not apply here — Vite rewrites at bundle time.)

### v1 pain points (informs v2 scope)

See [[frontend-bloat-analysis]] for the full diagnosis. Headline drivers:

1. **Role-as-folder duplication** — same feature scattered across `admin/orders/`, `operator/orders/`, `delivery/orders/` instead of one orders module with role-aware variants.
2. **i18n for one deployment language** — 2 locales × 9 modules = ~2.4k LOC of translations + lazy-loader plumbing for a Spanish-only Peruvian operator. No real switcher in production.
3. **Two parallel transaction APIs** — `inventoryService.ts` carries the legacy "complex" API and the new "simplified" API in parallel; the deprecated paths still have callers.
4. **Five-state inventory workflow** — `InventoryStatusType` enum has `CREATED / ASSIGNED / CONSOLIDATED / VALIDATED / OBSERVED`. Backend v2 collapses to **three states** (open / closed / carried) per ADR-008; frontend types must follow.
5. **Firebase FCM (~940 LOC)** — `notificationService.ts` (460) + `notificationStore.ts` (478). Backend v2 dropped push out of MVP scope per [[../product/overview]].
6. **Per-role-per-feature stores** — three inventory stores (`adminInventoryStore`, `adminInventoryTransactionStore`, `deliveryInventoryStore`) where one module-scoped store would do.

### v2 plan (high level)
**Update 2026-05-08**: v2 skeleton landed via [[../specs/bootstrap/v2-skeleton/frontend]]. v1 is archived under `lpg-frontend-vue/legacy/` (read-only); a fresh `src/` is in place with auth + role-stub homes. Build is green, ~38 KB gzipped main bundle.

### v2 stack (current)

- Vue 3 + Vite 5, TypeScript strict
- Pinia, Vue Router 4
- Tailwind CSS + `@tailwindcss/forms` + `tailwindcss-animate`
- **shadcn-vue** (Reka UI primitives + lucide-vue-next icons + class-variance-authority + clsx + tailwind-merge) — components copied into `src/components/ui/` as needed
- `vite-plugin-pwa` (Workbox; auto-update SW; manifest `start_url` parameterized via `VITE_PWA_START_URL`)
- `@vueuse/core` for utility composables
- **Removed from v1**: `vue-i18n`, `firebase`, `@vuepic/vue-datepicker`, `@vite-pwa/assets-generator`, `dotenv`, `@heroicons/vue`, custom service-worker generator script

### v2 layout (current)

```
src/
  main.ts                       # composition root: ApiClient + createAuthModule + Pinia + router
  App.vue                       # <RouterView />
  style.css                     # Tailwind directives + shadcn HSL CSS variables (light + dark)
  vite-env.d.ts                 # ImportMetaEnv typings
  lib/
    apiClient.ts                # fetch wrapper; takes getToken callback; throws narrowed ApiError on non-OK
    errors.ts                   # ApiError + Auth/Validation/Permission/NotFound/Server/Network subclasses
    types.ts                    # ApiErrorBody envelope
    utils.ts                    # cn() helper for shadcn (clsx + tailwind-merge)
  layouts/
    AppLayout.vue               # one shared shell (header + nav prop + role-aware logout)
  router/
    index.ts                    # setupRouter() with single global guard (requiresAuth, roles[], developer bypass)
  modules/
    auth/                       # vertical slice
      types.ts                  # USER_ROLES const + UserRole + isUserRole guard, PublicUser, LoginResponse
      service.ts                # createAuthService(apiClient): login, getMe
      store.ts                  # Pinia auth store; localStorage read+write LIVES ONLY HERE
      routes.ts                 # /login route
      index.ts                  # createAuthModule({ apiClient }) → { routes, useAuthStore }
      views/
        LoginView.vue           # login form, hardcoded Spanish (no i18n)
    home/                       # role stub homes (placeholders until features port)
      routes.ts                 # /admin, /operator, /delivery (each renders AppLayout + role-home view)
      views/
        AdminHome.vue
        OperatorHome.vue
        DeliveryHome.vue
```

### Key v2 conventions

- **Module-by-domain** — see [[patterns/frontend-module-template]]. Role variants live *inside* a module (sibling components), not as a parallel folder tree.
- **Single ApiClient** instance, constructed in `main.ts` with `getToken: () => getStoredToken()`. The client never touches localStorage; `getStoredToken()` (defined in `modules/auth/store.ts`) is the only bridge.
- **`@/` alias** is kept (Vite rewrites at bundle time; shadcn-vue tooling expects it). The backend's distroless `tsc`/Node mismatch does not apply here.
- **Roles** (`developer | admin | operator | delivery`) are mirrored from `lpg-backend/src/modules/auth/schema.ts:userRoleEnum` as a `const` array with a runtime narrowing guard (`isUserRole`) that surfaces drift at the login boundary. `developer` aliases to admin's home route (matches backend's "developer always passes" middleware).
- **No i18n**, no Firebase, no datepicker. Reintroduce only when a real use case appears.
- **PWA**: `vite-plugin-pwa` only (custom SW generator dropped). `start_url` defaults to `/login`; override via `VITE_PWA_START_URL` env var.

### Related Files (frontend)
V2 (current):

- `lpg-frontend-vue/package.json` — slimmed deps (vue + vue-router + pinia + reka-ui + lucide-vue-next + tailwind + @vueuse/core; vite-plugin-pwa in devDeps)
- `lpg-frontend-vue/vite.config.ts` — VitePWA + manifest; `VITE_PWA_START_URL` env override
- `lpg-frontend-vue/tsconfig.app.json` — excludes `legacy/`; `@/` paths alias
- `lpg-frontend-vue/components.json` — shadcn-vue CLI config (slate base, lucide icons)
- `lpg-frontend-vue/tailwind.config.js` — shadcn theme tokens (HSL CSS vars) + animate plugin
- `lpg-frontend-vue/src/main.ts` — composition root
- `lpg-frontend-vue/src/lib/apiClient.ts` — fetch wrapper with `getToken` callback
- `lpg-frontend-vue/src/router/index.ts` — single global guard
- `lpg-frontend-vue/src/modules/auth/` — vertical-slice reference module
- `lpg-frontend-vue/src/modules/home/` — role-stub homes

V1 archive (reference only):

- `lpg-frontend-vue/legacy/src/` — full v1 source
- `lpg-frontend-vue/legacy/docs/` — v1 PRDs (delivery, orders, firebase, inventory)
- `lpg-frontend-vue/legacy/scripts/generate-service-worker.js` — old custom FCM SW generator
- `lpg-frontend-vue/legacy/CLAUDE.local.md` — v1 architecture notes
- `lpg-frontend-vue/legacy/README.md` — archive purpose + deletion criteria

## Bot (lpg-bot)

### Stack

Node + Express + axios + commander. Likely a chat-bot integration (channel TBD — likely WhatsApp or Telegram).

Architecture: not yet documented in the vault.

### Related Files

- `lpg-bot/package.json`
- `lpg-bot/README.md`
- `lpg-bot/src/`
