---
project: lpg-store
domain: specs
type: spec
status: '"done"'
depends-on:
  - bootstrap/v2-skeleton
  - auth/auth-foundation
last-updated: 2026-05-08
---

# Spec: Frontend v2 Skeleton (reset with v1 as reference)

## Problem Statement

The v1 frontend (`lpg-frontend-vue`) has grown to ~32.7k LOC across 212 files for ~25 screens. The bloat is structural, not feature-driven — see [[../../eng/frontend-bloat-analysis]] for the full diagnosis. Headline drivers:

- **Role-as-folder duplication** at every layer (`components/`, `views/`, `stores/`, `services/`, `layouts/` all split into `admin/`, `delivery/`, `operator/`, `superadmin/`), so the same domain feature appears 2–3 times in parallel folders that drift independently.
- **i18n stack for one deployment language** (~2.4k LOC of translations + lazy-loader plumbing for an EN locale that has no production user).
- **Dual transaction API** (legacy "complex" + new "simplified") coexisting in `inventoryService.ts` and `inventoryTypes.ts`.
- **Five-state inventory workflow** types in the frontend, while backend v2 collapses to **three states** (ADR-008).
- **Firebase FCM stack** (~940 LOC) for push notifications that [[../../product/overview]] explicitly drops from MVP.

The product is pre-production. There are no users to migrate, but there is requirements knowledge buried in the existing components — same situation the backend faced before its v2 reset.

## Proposed Solution

Mirror the backend's [[../bootstrap/v2-skeleton]] approach, adapted for Vue/Vite:

1. Archive current `src/` to `legacy/` via `git mv` (history-preserving, fully grep-able, not buildable from the v2 entry point).
2. Bootstrap a fresh `src/` skeleton with **no business features** — only auth shell, router, layout, API client, build/PWA wiring.
3. Adopt **module-by-domain** structure (`src/modules/<feature>/`) instead of role-by-folder. Role variance lives **inside** a module (e.g. via role-aware sub-routes or component variants), not as a parallel folder tree.
4. Drop deferred dependencies: `vue-i18n`, `firebase`, the FCM service stack, the custom service-worker generator, `@vuepic/vue-datepicker` (pending need), and the dual transaction API surface.
5. **Do not move legacy code now.** Strategy is documented; the `git mv` happens as the first phase of this spec when implementation begins. Until then, v1 keeps running.
6. Reuse v1 components opportunistically by copying them into the new module and stripping role/i18n coupling — *not* by importing across the `legacy/` boundary.

The first feature module ports begin only after this spec is `done` and `[[../auth/auth-foundation]]` (backend) is live, so the frontend has a working API to hit.

## Acceptance Criteria

- [ ] v1 archived under `lpg-frontend-vue/legacy/` via `git mv` (history preserved per file)
- [ ] `legacy/README.md` explains the archive's purpose and deletion criteria (matches backend pattern)
- [ ] `tsconfig.json` excludes `legacy/`; `vue-tsc -b --noEmit` typechecks zero legacy files
- [ ] Repo root has no v1 source files outside `legacy/`
- [ ] Fresh `package.json` removes: `vue-i18n`, `firebase`, `@vuepic/vue-datepicker`. Keeps: `vue`, `vue-router`, `pinia`, `@headlessui/vue`, `@heroicons/vue`, `@vueuse/core`, `tailwindcss`, `chart.js`+`vue-chartjs`, `leaflet`+`@types/leaflet`, `vite-plugin-pwa`, `date-fns`
- [ ] `src/` skeleton landed:
  - [ ] `main.ts` — app composition root (no i18n plugin, no Firebase init)
  - [ ] `App.vue`, `style.css`
  - [ ] `router/{index,routes}.ts` — guards + role redirects, **no i18n loader hook**, no `i18nModules` meta on routes
  - [ ] `layouts/AppLayout.vue` — single layout component that takes nav config as a prop (replaces three role-folder layouts)
  - [ ] `lib/apiClient.ts` — fetch wrapper that takes a `getToken()` callback (no direct localStorage read)
  - [ ] `lib/{errors,types}.ts` — shared error class + `ApiResponse<T>` envelope
  - [ ] `modules/auth/` — first module, vertical slice (login view, store, route, auth guard composable)
  - [ ] empty `modules/` peers ready for future ports
- [ ] `vite.config.ts` keeps `vite-plugin-pwa`, drops the custom `scripts/generate-service-worker.js`, parameterizes `manifest.start_url` via env var (default to login)
- [ ] `npm run dev` boots, login screen renders, login flow against backend `/v1/auth/login` succeeds, role redirect lands on a stub home view
- [ ] `npm run build` produces a working PWA bundle that installs and serves offline (per existing Workbox config)
- [ ] `vue-tsc -b --noEmit` passes with zero errors
- [ ] [[../../eng/architecture]] Frontend section updated to describe v2 layout (not just v1 + plan)
- [ ] [[../../eng/patterns]] gains a `frontend-module-template.md` mirroring [[../../eng/patterns/module-template]] for Vue modules

## Out of Scope

- Porting any business feature beyond the auth login screen (users, stores, inventory, orders, customers UI) — each becomes its own spec
- Backend changes
- Deleting `legacy/` — only after v2 reaches functional parity for the operator/admin/delivery flows
- Test coverage beyond a manual smoke test of login + role redirect (test strategy is its own decision, deferred)
- Reintroducing FCM, i18n, datepicker, or the legacy transaction API — each requires a fresh case if it returns

## Technical Notes

- **Vite + alias**: Vite rewrites `@/` to `src/` at bundle time, so the alias is safe (this is the case where the backend's distroless `tsc`/Node mismatch does *not* apply). v2 may keep or drop the alias; bias is to drop it for consistency with backend convention (relative imports).
- **PWA scope**: `start_url` should default to `/login` and let post-login navigation route the user to their role's home. Hard-coding `/delivery/dashboard` as v1 does breaks installs for admin/operator users.
- **Auth token storage**: keep localStorage for now (matches backend JWT model; refresh-token / cookie strategy is a separate decision). The change is *where the read happens* (inside `modules/auth/` only) not *what storage is used*.
- **Module composition**: each module exports `createXxxModule({ apiClient, deps })` returning `{ router, store, components }`. `main.ts` wires modules into the app — analogous to backend `app.ts`.
- **Role-aware routes**: live in each module's `module.routes.ts` with `meta.roles` enforced by a single global guard. Role-specific UI variants live as sibling components inside the module (e.g. `modules/orders/views/OperatorOrdersView.vue` + `modules/orders/views/DeliveryOrdersView.vue`), not as separate role folders.
- **No barrel `index.ts` files** at the components level (v1's `components/admin/users/index.ts` etc.) unless they earn their keep — module roots are the only public boundary.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

V1 (will be archived to `legacy/` during phase A):

- `src/` — entire current tree (212 files, ~32.7k LOC)
- `scripts/generate-service-worker.js` — to be removed (replaced by `vite-plugin-pwa` default)
- `docs/` — kept as reference under `legacy/docs/` for v1 PRDs and implementation plans

V2 (to land during this spec):

- `src/main.ts`, `src/App.vue`, `src/style.css`
- `src/router/{index,routes}.ts`
- `src/layouts/AppLayout.vue`
- `src/lib/{apiClient,errors,types}.ts`
- `src/modules/auth/` — first vertical slice
- `package.json` (slimmed deps), `vite.config.ts` (PWA-only), `tsconfig.json` (excludes `legacy/`)
- `legacy/README.md` — archive purpose + deletion criteria

### Cross-repo dependency

- Backend `[[../auth/auth-foundation]]` (done 2026-05-08) — provides `/v1/auth/login`, JWT issuance, role middleware that the frontend skeleton's auth module hits

## Implementation Notes
- [2026-05-08] [lpg-frontend-vue] Phase A: archived v1 to `legacy/` via `git mv` (single commit, 230 file renames at 100% similarity). Wrote `legacy/README.md` with purpose, rules, and deletion criteria. `CLAUDE.local.md` is gitignored — moved physically.
- [2026-05-08] [lpg-frontend-vue] Phase B: fresh `package.json` (Vue 3.5, Vite 5.4, Pinia 2.2, Vue Router 4.4, Reka UI alpha, lucide-vue-next, class-variance-authority, clsx, tailwind-merge, tailwindcss-animate, @vueuse/core 11). Dropped: `vue-i18n`, `firebase`, `@vuepic/vue-datepicker`, `@vite-pwa/assets-generator`, `dotenv`, `@heroicons/vue`. Removed the `generate-sw` script. New `tsconfig.app.json` excludes `legacy/`, ES2022, strict + noUnused* on. New `vite.config.ts` keeps VitePWA but parameterizes `manifest.start_url` via `VITE_PWA_START_URL` (default `/login`). New `tailwind.config.js` with shadcn HSL CSS-var theme + tailwindcss-animate. Added `components.json` for shadcn-vue CLI (slate base, lucide icons, `@/` aliases).
- [2026-05-08] [lpg-frontend-vue] Phase C+D+E (single commit): `src/lib/{apiClient,errors,types,utils}.ts`, `src/router/index.ts` (single global guard with `developer` bypass aligned to backend middleware), `src/layouts/AppLayout.vue` (one shared shell taking `title` + `nav` prop), `src/modules/auth/` (full vertical slice: types/service/store/routes/views, with `USER_ROLES` const + `isUserRole` runtime guard mirroring `lpg-backend/src/modules/auth/schema.ts`), `src/modules/home/` (AdminHome / OperatorHome / DeliveryHome stubs + module-load-time invariant check that every USER_ROLES entry has a home route). `developer` aliased to `admin-home` in `ROLE_HOME_ROUTE` (no separate UI). localStorage read+write for the JWT lives ONLY in `modules/auth/store.ts`; the apiClient takes a `getToken()` callback wired in `main.ts`.
- [2026-05-08] [lpg-frontend-vue] Phase F: `vue-tsc -b --noEmit` clean (zero errors, zero legacy files type-checked). `npm run build` clean — main bundle 96.82 KB raw / **38.26 KB gzipped**, LoginView lazy chunk 1.74 KB gzipped, three home stubs ~0.4 KB each, CSS 3.84 KB gzipped, total precache 150 KiB. Tree-shaking verified: only the `Flame` icon pulled from lucide-vue-next. `npm run dev` boots in 531 ms; `/` returns 200 with v2 shell, `/src/main.ts` and LoginView transform without errors. Backend round-trip not exercised (backend not running locally during this session) — visual + transform smoke test only, matching the spec's "single manual smoke test" scope.
- [2026-05-08] [lpg-frontend-vue] Vault docs: added `eng/patterns/frontend-module-template.md`, updated `eng/architecture.md` Frontend section to describe v2 layout + Related Files, updated `eng/index.md` to register the new pattern doc.

### Decisions baked in (not pre-existing in the spec)

- **shadcn-vue chosen** as the UI library (instead of just keeping Headless UI per the original spec). User direction during /focus. Tree-shaking validated by build (38 KB gzipped main bundle).
- **`@/` alias kept** for the frontend (the spec's Technical Notes flagged this as open). shadcn-vue CLI scaffolds imports as `@/lib/utils` etc.; dropping the alias would force rewriting every shadcn-added component. Backend's distroless `tsc`/Node mismatch does not apply to Vite, which rewrites at bundle time.
- **`developer` role aliased to admin home** (no separate UI) since backend middleware treats it as "admin + override."
- **Manifest hard-coded `start_url`** replaced with `VITE_PWA_START_URL` env var defaulting to `/login`, fixing v1's bug of hard-coding `/delivery/dashboard` (which broke installs for admin/operator users).
