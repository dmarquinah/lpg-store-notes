---
project: lpg-store
domain: specs
type: spec-track
spec: v2-skeleton
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-05-08
---

# v2 Skeleton — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

- **Vite + alias**: Vite rewrites `@/` to `src/` at bundle time, so the alias is safe (the backend's distroless `tsc`/Node mismatch does *not* apply). The frontend keeps `@/`.
- **PWA scope**: `start_url` defaults to `/login` and lets post-login navigation route the user to their role's home. Hard-coding `/delivery/dashboard` as v1 did breaks installs for admin/operator users.
- **Auth token storage**: localStorage for now (matches backend JWT model). The change vs v1 is *where the read happens* — inside `modules/auth/` only — not *what storage is used*.
- **Module composition**: each module exports `createXxxModule({ apiClient, deps })` returning `{ router, store, components }`. `main.ts` wires modules into the app — analogous to backend `app.ts`.
- **Role-aware routes**: live in each module's routes with `meta.roles` enforced by a single global guard. Role-specific UI variants live as sibling components inside the module, not as separate role folders.
- See [[../../../eng/frontend-bloat-analysis]] for the ten v1 inflation drivers this skeleton kills.

### Decisions baked in (beyond the original spec)

- **shadcn-vue chosen** as the UI library (instead of just keeping Headless UI). User direction during `/focus`. Tree-shaking validated by build (38 KB gzipped main bundle).
- **`@/` alias kept** for the frontend — shadcn-vue CLI scaffolds imports as `@/lib/utils`; dropping it would force rewriting every shadcn-added component.
- **`developer` role aliased to admin home** (no separate UI) since backend middleware treats it as "admin + override."
- **Manifest `start_url`** replaced with `VITE_PWA_START_URL` env var defaulting to `/login`, fixing v1's hard-coded `/delivery/dashboard` bug.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

V1 (archived to `legacy/` during phase A):

- `legacy/src/` — entire v1 tree (212 files, ~32.7k LOC)
- `legacy/README.md` — archive purpose + deletion criteria

V2 (landed during this spec):

- `src/main.ts`, `src/App.vue`, `src/style.css`
- `src/router/index.ts` — single global guard with `developer` bypass aligned to backend middleware
- `src/layouts/AppLayout.vue` — one shared shell taking `title` + `nav` prop
- `src/lib/{apiClient,errors,types,utils}.ts`
- `src/modules/auth/` — first vertical slice (types/service/store/routes/views; `USER_ROLES` + `isUserRole` mirror `lpg-backend/src/modules/auth/schema.ts`)
- `src/modules/home/` — AdminHome / OperatorHome / DeliveryHome stubs + module-load invariant check
- `package.json` (slimmed deps), `vite.config.ts` (PWA-only), `tsconfig.app.json` (excludes `legacy/`), `tailwind.config.js`, `components.json`

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-05-08] [lpg-frontend-vue] Phase A: archived v1 to `legacy/` via `git mv` (single commit, 230 file renames at 100% similarity). Wrote `legacy/README.md` with purpose, rules, and deletion criteria. `CLAUDE.local.md` is gitignored — moved physically.
- [2026-05-08] [lpg-frontend-vue] Phase B: fresh `package.json` (Vue 3.5, Vite 5.4, Pinia 2.2, Vue Router 4.4, Reka UI alpha, lucide-vue-next, class-variance-authority, clsx, tailwind-merge, tailwindcss-animate, @vueuse/core 11). Dropped: `vue-i18n`, `firebase`, `@vuepic/vue-datepicker`, `@vite-pwa/assets-generator`, `dotenv`, `@heroicons/vue`. Removed the `generate-sw` script. New `tsconfig.app.json` excludes `legacy/`, ES2022, strict + noUnused* on. New `vite.config.ts` keeps VitePWA but parameterizes `manifest.start_url` via `VITE_PWA_START_URL` (default `/login`). New `tailwind.config.js` with shadcn HSL CSS-var theme. Added `components.json` for shadcn-vue CLI.
- [2026-05-08] [lpg-frontend-vue] Phase C+D+E (single commit): `src/lib/{apiClient,errors,types,utils}.ts`, `src/router/index.ts` (single global guard with `developer` bypass), `src/layouts/AppLayout.vue` (one shared shell), `src/modules/auth/` (full vertical slice with `USER_ROLES` const + `isUserRole` guard mirroring backend schema), `src/modules/home/` (role-stub homes + module-load invariant check). localStorage read+write for the JWT lives ONLY in `modules/auth/store.ts`; the apiClient takes a `getToken()` callback wired in `main.ts`.
- [2026-05-08] [lpg-frontend-vue] Phase F: `vue-tsc -b --noEmit` clean (zero legacy files type-checked). `npm run build` clean — main bundle 96.82 KB raw / **38.26 KB gzipped**, LoginView lazy chunk 1.74 KB gzipped, total precache 150 KiB. Tree-shaking verified. `npm run dev` boots in 531 ms; `/` returns 200 with v2 shell. Backend round-trip not exercised (backend not running locally during this session) — visual + transform smoke test only, matching the spec's "single manual smoke test" scope.
- [2026-05-08] [lpg-frontend-vue] Vault docs: added `eng/patterns/frontend-module-template.md`, updated `eng/architecture.md` Frontend section to describe v2 layout, updated `eng/index.md` to register the new pattern doc.
- [2026-05-08] [lpg-frontend-vue] All frontend acceptance criteria for this repo met.
