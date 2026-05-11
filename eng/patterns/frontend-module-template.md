# Module Template — frontend vertical modules

---
project: lpg-store
domain: eng
last-updated: 2026-05-08
---

Convention for `lpg-frontend-vue/src/modules/<feature>/`. Every feature ported from v1 follows this shape. Adapted from [[module-template]] (backend) for Vue 3 + Pinia + Vue Router.

## Folder layout

```
src/modules/<feature>/
  types.ts              # interfaces + (later) zod schemas; mirrors backend module's PublicX shapes
  service.ts            # createXxxService(apiClient) — calls api endpoints, returns typed data
  store.ts              # Pinia store (defineStore); the ONLY file that touches localStorage / persistence
  routes.ts             # RouteRecordRaw[] for this module
  index.ts              # createXxxModule({ apiClient }) — composition root; re-exports useXxxStore
  views/                # SFCs reachable from routes
    XxxView.vue         # role-aware variants live here as siblings, e.g. OperatorOrdersView.vue
  components/           # internal components used only by this module's views
  composables/          # module-internal composables
```

## Boundaries

- **`types.ts`** is the only file other modules can import to share types. Cross-module deps go *only* through types.
- **`service.ts`** takes an `ApiClient` instance; service methods call `apiClient.get/post/...` and return typed payloads. Throws narrowed errors (`ApiError` subclasses from `@/lib/errors`) on non-OK responses.
- **`store.ts`** holds reactive state. Only this file may import `localStorage` / persistence APIs for the module. Stores depend on services via a module-scoped `provideXxxService()` setter populated by `createXxxModule`.
- **`routes.ts`** exports a `RouteRecordRaw[]` and never mounts anything itself; the central router stitches together each module's routes.
- **`index.ts`** exports `createXxxModule({ apiClient })` returning whatever the rest of the app needs (`{ routes, useXxxStore, ROLE_X }`). Wires service → store internally.

## Composition root

Modules are composed in `src/main.ts`:

```ts
import { ApiClient } from "@/lib/apiClient";
import { createAuthModule, getStoredToken } from "@/modules/auth";

const apiClient = new ApiClient({
  baseUrl: import.meta.env.VITE_API_URL ?? "http://localhost:3000",
  apiVersion: "v1",
  getToken: () => getStoredToken(),
});

createAuthModule({ apiClient });
// future modules wire in the same way
```

Cross-module dependencies (e.g. orders module reads `useAuthStore` for the current user) go through the consuming module's `index.ts` re-exports — never reach into another module's internals.

## Role variants live inside the module

Per [[frontend-bloat-analysis]] driver 1, do **not** create role-folder duplicates (`components/admin/orders/`, `components/operator/orders/`). Role-specific UI variants live as sibling components inside the module:

```
src/modules/orders/views/
  OperatorOrdersView.vue
  DeliveryOrdersView.vue
```

Routes carry `meta.roles` and render the right variant.

## State (Pinia) conventions

- One store per module by default. Splitting (e.g. `useOrdersStore` + `useOrdersFiltersStore`) is allowed when state lifecycles genuinely diverge — not because v1 had a separate role-scoped store.
- The store **never** receives an `apiClient` directly; it consumes a service injected via `provideXxxService()` set up by `createXxxModule`. This keeps stores testable with a fake service.
- localStorage / sessionStorage / IndexedDB usage is allowed *only* in the module that owns the data (e.g. `modules/auth/store.ts` for the JWT). Don't leak persistence keys across modules.

## API access

- Use the singleton `ApiClient` injected via `createXxxModule({ apiClient })`. Do not construct fresh `ApiClient` instances per module.
- Backend response shapes: success returns the resource directly; errors return `{ error: { code, message, issues? } }` with non-2xx status. The client throws narrowed `ApiError`s on non-OK — services do not need to inspect `response.error`.

## Source of truth for cross-repo types

When a frontend type mirrors a backend type (e.g. `PublicUser`, `UserRole`), include a comment at the top of the type definition pointing to the backend file. If the backend changes the shape, the comment is the audit trail.

```ts
// Mirrors backend PublicUser (lpg-backend/src/modules/auth/types.ts:55).
export interface PublicUser { ... }
```

For enums where drift is most painful, define them as a `const` array and derive both type and runtime guard:

```ts
export const USER_ROLES = ["developer", "admin", "operator", "delivery"] as const;
export type UserRole = (typeof USER_ROLES)[number];
export const isUserRole = (s: string | null | undefined): s is UserRole =>
  !!s && (USER_ROLES as readonly string[]).includes(s);
```

The runtime guard surfaces drift at API boundaries (login, /me) instead of letting unknown roles silently misroute.

## Tests

Test framework decision is deferred to its own spec. When tests land, place them under `__tests__/` next to the code they test (matches backend convention).

## Reference for porting from v1

Each feature spec should list:
1. The v1 implementation files in `legacy/src/...` for prior-art reference.
2. The expected v2 paths under `src/modules/<feature>/`.
3. The backend module the frontend depends on (so the port lands on a working API).
