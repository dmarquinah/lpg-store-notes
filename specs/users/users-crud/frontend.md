---
project: lpg-store
domain: specs
type: spec-track
spec: users-crud
repo: lpg-frontend-vue
kind: frontend
track-status: '"done"'
last-updated: 2026-06-04
---

# Users CRUD — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

Not started. A `users` vertical module under `src/modules/users/` (mirroring the frontend module convention from [[../../bootstrap/v2-skeleton/frontend]] and [[../../../eng/patterns/frontend-module-template]]) that drives the backend endpoints below.

- **Backend contract** (done — see [[backend]]): `GET /api/v1/users` (`?role=`/`?active=`), `GET /api/v1/users/:id`, `PATCH /api/v1/users/:id` (`name`/`phone`/`role`/`active`). Admin/developer only.
- **Screens**: a list view with role/active filters + an edit form. Role select must reflect the escalation rule (a non-developer can't assign `developer` or edit a developer); the form must block self `role`/`active` changes and surface API `403`s inline.
- **Reuse**: the auth module's session/token-aware `apiClient` and the single global router guard (`meta.roles: ['admin','developer']`) gate the route.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To be created:

- `src/modules/users/` — vertical slice (types/service/store/routes/views: a list view + an edit form)
- route registration wiring in `src/main.ts` / router

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-06-04] [lpg-frontend-vue] Implemented the `users` vertical module (`src/modules/users/{types,service,store,routes,index}.ts` + `views/{UsersListView,UserEditView}.vue`) driving the shipped backend endpoints. List view with role/active `Select` filters (active serialized as `"true"`/`"false"`); edit form patches name/phone/role/active sending **only changed** fields (phone `''` → `null` clear), blocks empty/no-change submits client-side, and mirrors the backend developer-escalation + self-lockout guards (disables role/active for self, hard-blocks editing a developer for non-developers, hides `developer` from the role options) while still surfacing the API's Spanish 403/400 messages inline via `Alert`. `PublicUser` re-exported from `modules/auth/types` (single source of truth) — `password_hash` is never in the shape.
- [2026-06-04] [lpg-frontend-vue] Route `/users` gated by `meta.roles: ['admin','developer']` through the single global guard (developer bypass); registered in `src/router/index.ts` and composed via `createUsersModule({ apiClient })` in `src/main.ts`. Admin shell nav links Inicio/Usuarios (plain string paths, no cross-module import).
- [2026-06-04] [lpg-frontend-vue] UI built from **scaffolded shadcn-vue** components (button/input/label/select/table/badge/card/switch/alert under `src/components/ui/`). Tooling had to be aligned to the current shadcn-vue CLI (2.7.4): upgraded `reka-ui` 1.0.0-alpha.11 → 2.9.9 (no prior usages), removed invalid `tsConfigPath`/`framework` keys from `components.json`, and added `baseUrl`+`paths` to root `tsconfig.json` so the CLI resolves the `@/` alias. See [[../../../eng/decisions#ADR-011 — Adopt shadcn-vue components; upgrade reka-ui to 2.x (2026-06-04)|ADR-011]]. Also refactored `LoginView` + `AppLayout` to use the new components (user request).
- [2026-06-04] [lpg-frontend-vue] Gates: `vue-tsc -b --noEmit` clean; `npm run build` clean (main bundle 64 KB gzip — up from 38 KB due to reka-ui primitives; views are code-split, Select primitives in a shared 28.5 KB gzip lazy chunk). Independent validation agent confirmed all 4 frontend acceptance criteria met, no functional bugs. Manual smoke test deferred (requires the backend running locally).
- [2026-06-04] [lpg-frontend-vue] All frontend acceptance criteria for this repo met.

- [2026-06-04] [lpg-frontend-vue] Post-merge UI polish (operator feedback after first run): (1) fixed transparent `Select` dropdown — the scaffolded components reference `bg-popover`, but the skeleton's `style.css`/`tailwind.config.js` never defined the `popover` token; added `--popover`/`--popover-foreground` (light + dark) and the `popover` color mapping. (2) Edit moved from a `/users/:id` page to a **modal dialog** (`components/UserEditDialog.vue` via scaffolded shadcn-vue `dialog`); removed the `user-edit` route + `UserEditView.vue` (a dedicated detail page can come later if needed). (3) `AppLayout` now centers content in a `max-w-6xl` container with roomier padding. Gates green (`vue-tsc` + build).

- [2026-06-04] [lpg-frontend-vue] Theming fixes (operator feedback): removed `@tailwindcss/forms` (its base form styles fought shadcn-vue's own input/focus styling — the source of the harsh "white" border on focused inputs); inputs now rely solely on shadcn classes (border-input + a proper blue `--ring` focus ring). Fixed dark `--primary-foreground` (was dark navy `222.2 47.4% 11.2%` → light `210 40% 98%`) so primary buttons (`Guardar cambios`) and `default` badges are readable on the blue primary. CSS bundle dropped ~28.7 → 23.3 KB.

- [2026-06-04] [lpg-frontend-vue] Correction to the note above: the real cause of the "white" border on the modal/dropdown in dark mode was a **missing shadcn base rule** in `src/style.css` — `@layer base { * { @apply border-border; } }`. Without it, bare `border` utilities (DialogContent, SelectContent set width but no color) fall back to Tailwind Preflight's `gray-200` default, which looks white on the dark bg. Added the rule (per https://www.shadcn-vue.com/docs/theming); borders now use the `--border` slate token. (Removing `@tailwindcss/forms` + the `--primary-foreground` contrast fix from the previous note still stand — both are correct improvements, just not the border cause.)