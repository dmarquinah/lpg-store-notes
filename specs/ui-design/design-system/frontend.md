---
project: lpg-store
domain: specs
type: spec-track
spec: design-system
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-12
---

# Design System + UI Overhaul — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes

- Tokens live as HSL CSS vars in `src/style.css` (`@layer base :root` / `.dark`)
  mapped by `tailwind.config.js` — the shadcn-vue convention. New semantic
  colors (success/warning/info) follow the same `--token` + `--token-foreground`
  pair pattern so badge/alert variants can consume them.
- Component variants are CVA-based (`class-variance-authority`) inside each
  `src/components/ui/<name>/`. Extend variants (e.g. badge `success|warning`)
  in place — do **not** fork components per module or per role
  (see [[../../eng/frontend-bloat-analysis]]).
- Canonical patterns (page header, empty/loading/error states) may become small
  shared components under `src/components/` (app-level, not module-level) —
  they are cross-module chrome, like `AppLayout`.
- Restyle is per-module view work; module boundaries and routes do not change.
  Nav config is centralized in `AppLayout.vue` (`ROLE_NAV`) — label+icon rules
  apply there once.
- Touch-target rule lands in the `button`/`input`/`select` size variants
  (default size ≥ 44px on mobile breakpoints), so modules inherit it for free.
- Gates: `npm run typecheck`, `npm run build`; no test runner is wired (manual
  smoke per module).

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

- `src/style.css` — theme tokens (HSL CSS vars), base layer rules
- `tailwind.config.js` — token→utility mapping, fonts, animate plugin
- `components.json` — shadcn-vue CLI config (affects future scaffolds)
- `src/components/ui/` — 15 shadcn-vue components (alert, badge, button, calendar, card, date-picker, dialog, input, label, popover, select, sheet, switch, table, tabs)
- `src/layouts/AppLayout.vue` — app shell: drawer nav (`ROLE_NAV`), top bar
- `src/modules/auth/views/LoginView.vue` — login screen
- `src/modules/{home,users,catalog,inventory,customers,orders}/views/` — module screens to retrofit
- `index.html` — font loading (if a bundled font is chosen)

## Implementation Notes
<!-- Claude appends progress for THIS repo here during implementation -->
<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->
- [2026-06-12] [lpg-frontend-vue] Phases 1–2 done, palette iterated live with the owner. Final direction: **"petrol + flame"** — deep petrol primary (`192 70% 26%` light / `192 60% 34%` dark, white fg in both), cool petrol-tinted neutrals (owner rejected the initial warm/brown neutrals and the dark-mode light-fill/dark-text treatment — wants white text in fills everywhere), flame orange only in `--brand` (wordmark) + `--ring`; status fills deep+white both themes plus text-grade `--{status}-text` shades for dark legibility. Type scale lifted (xs→14px, sm→15px), Public Sans Variable bundled (`@fontsource-variable/public-sans`), theme system shipped (`src/lib/theme.ts` vueuse useColorMode, key `piloto-theme`, FOUC guard + theme-color metas in index.html, Sun/Moon toggle in AppLayout); hardcoded dark default removed. Lone raw palette class (`text-amber-500` in AssignDialog) → `text-warning-text`. Wrote `eng/design-system.md` (canonical conventions + hard-rule checklist) and indexed it. Gates green (typecheck + build, precache ~650 KiB incl. font).- [2026-06-12] [lpg-frontend-vue] Phases 3–7 done. **Component layer:** button/input/select → `h-11 md:h-10` (≥44px touch on mobile); badge gained solid `success/warning/info` variants (white fg, hover dropped); alert destructive → `text-destructive-text` (deleted the `dark:*-red-*` hack); TabsTrigger `px-4 py-2`; CardTitle `text-2xl`→`text-lg`. **Shared chrome** (`src/components/app/`): `PageHeader`, `Spinner`, `EmptyState`; `src/lib/format.ts` `formatMoney` (deleted `orderLabels.money` + 2 customers dupes + catalog's unitless toFixed; ~11 import sites moved). **Retrofit** of all 7 modules + shell: PageHeader everywhere, `TableEmpty`+`Spinner` for every table loading/empty state, semantic status badges via the canonical mapping (orders/payment/inventory/customers). **DeliveryListView** rebuilt as a phone-first **Card list** (full address, `tel:` link, full-width default-size action buttons) — script untouched. Home placeholders → greeting + large shortcut Cards. Login: flame `text-brand` wordmark + theme toggle. AppLayout: `text-brand` wordmark, Spanish role-badge labels, `min-h-11` nav links. PWA `theme_color`/`background_color` refreshed (`#BD4310`/`#F5F8F9`). **Independent validation: all 10 acceptance criteria PASS** (zero raw palette classes, no leftover `money()`, both themes tokenized). Gates green: typecheck + build (precache 656 KiB, +~10 KiB). Manual both-theme smoke left to the operator (dev server verified to boot).