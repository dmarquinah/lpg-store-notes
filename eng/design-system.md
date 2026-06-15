---
project: lpg-store
domain: eng
last-updated: 2026-06-12
---

# Piloto Design System

The visual and interaction conventions for `lpg-frontend-vue`. **Every future UI change must follow this doc**; if a change needs to break a rule here, update this doc in the same spec. Introduced by spec [[../specs/ui-design/design-system/index|design-system]].

Audience constraint that drives everything: operators/drivers/admin are **middle-aged users with little web-app experience**. Legibility, large touch targets, low visual noise, and predictable patterns outrank density and flair.

## 1. Palette — "Petrol + flame"

Tokens are HSL CSS vars in `src/style.css` (`:root` = light, `.dark` = dark), mapped to Tailwind classes in `tailwind.config.js`. **Never use raw Tailwind palette classes** (`text-red-500`, `bg-amber-100`, …) in views or components — always tokens.

| Role | Token | Light | Dark | Rule |
|------|-------|-------|------|------|
| Primary action | `--primary` / `-foreground` | petrol `192 70% 26%` / white | `192 60% 34%` / white | White text on petrol in BOTH themes. |
| Brand flame | `--brand` | `18 85% 40%` | `22 90% 58%` | ONLY for the "Piloto" wordmark. Not for buttons, hovers, or surfaces. |
| Focus ring | `--ring` | flame `18 85% 40%` | flame `24 90% 60%` | Keyboard-focus outline keeps the flame identity. |
| Surfaces | `--background`, `--card`, `--popover` | cool petrol-tinted off-white (`200 25% 97%`, card `200 30% 99%`) | cool charcoal (`203 15% 9%`, card `203 14% 12%`) | All neutrals share the 200–205 hue family. No warm/brown neutrals. |
| Hover/active tint | `--accent` / `-foreground` | `200 25% 88%` | `202 14% 20%` | Ghost-button hover, nav active state. Cool, never flame. |
| Subdued | `--secondary`, `--muted` / `-foreground` | `200 20% 91%` / `205 12% 38%` | `202 12% 18%` / `200 10% 70%` | `muted-foreground` is AA on background — don't lighten it. |
| Borders/inputs | `--border`, `--input` | `200 15% 86%` / `82%` | `202 12% 21%` / `24%` | Input slightly darker than border so fields read as fields. |

### Status colors — fills vs text

Each status has **two shades** with different jobs:

- **Fill** (`--success`, `--warning`, `--info`, `--destructive` + `-foreground`): deep color + **white text in both themes**. For solid badges, buttons, filled chips. Values (light/dark): success `145 65% 27%`/`145 60% 30%`, warning `28 85% 36%`/`38%`, info `210 80% 38%`/`210 75% 42%`, destructive `0 72% 42%`/`46%`.
- **Text** (`--success-text`, `--warning-text`, `--info-text`, `--destructive-text`): text-grade shade readable directly on the page background — deep in light mode, **lightened in dark mode** (e.g. destructive-text dark `0 85% 72%`). For error messages, status icons, colored inline text. Tailwind classes: `text-destructive-text`, `text-warning-text`, etc.

**Rule:** never put `text-{status}` (the fill shade) on the page background in dark mode — use `text-{status}-text`. Never put white text on a `-text` shade.

Status colors carry **meaning only** (state, alerts, debt) — never decoration.

### Canonical status → badge variant mapping

| Domain | Status → variant |
|--------|------------------|
| Order | pending → `warning` · assigned → `secondary` · in_transit → `info` · delivered → `success` · failed → `destructive` · cancelled → `outline` |
| Payment | unpaid → `warning` · partial → `info` · paid → `success` |
| Inventory assignment | open → `info` · closed → `secondary` · carried → `outline` |
| Customer | active → `success` · inactive → `outline` · monetary debt → `destructive` · empty-tank debt → `warning` |

New domains extend this table here first, then in code (per-module `*_VARIANTS` records, e.g. `orders/orderLabels.ts`).

## 2. Typography

- **Font:** Public Sans Variable, self-hosted via `@fontsource-variable/public-sans` (imported in `main.ts` before `style.css`), precached by the PWA. Fallback stack: `system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif`. No external font requests, no italic subset.
- **14px floor:** the Tailwind scale is lifted globally — `text-xs` = 14px, `text-sm` = 15px, `text-base` = 16px. **Nothing may render below 14px**; don't add smaller custom sizes.
- Scale usage: page title `text-2xl font-semibold tracking-tight` (via PageHeader), card/section title `text-lg`, body `text-base`/`text-sm`, captions `text-xs text-muted-foreground`.

## 3. Touch targets & density

- Interactive controls are **≥44px tall on mobile**, stepping down on desktop: the sizing idiom is `h-11 md:h-10` (buttons default, inputs, select triggers). Don't hand-roll smaller controls.
- Tables: comfortable density — `text-sm` (15px) cells with `p-4` padding (~54px rows). Do not compact.
- Nav/drawer links: `min-h-11` touch height.

## 4. Theme system

- Both light and dark are first-class; **every change must be eyeballed in both**.
- Persistence: localStorage key **`piloto-theme`** (`"light" | "dark" | "auto"`; absent/auto ⇒ OS preference). Owned exclusively by `src/lib/theme.ts` (vueuse `useColorMode`); an inline FOUC-guard script in `index.html` reads the same key before first paint — keep the two in sync.
- Toggle: Sun/Moon ghost button in the AppLayout top bar (and login). Two-state flip only — no three-way menu.
- `<meta name="theme-color">` ×2 in `index.html` must match the two `--background` values.

## 5. Canonical components & patterns

Shared chrome lives in `src/components/app/` (flat, no barrels); shadcn-vue primitives stay pristine in `src/components/ui/`.

- **PageHeader** (`components/app/PageHeader.vue`): the ONLY page-title pattern — props `title`, `description?`, slot `#actions` for right-aligned buttons. No ad-hoc `<h2>` headers in views.
- **Table states:** loading row = `TableEmpty` + `Spinner` + "Cargando…"; empty row = `TableEmpty` + "No se encontraron …". No bare inline `TableCell` states.
- **EmptyState** (`components/app/EmptyState.vue`): non-table empty/loading surfaces (detail views).
- **Spinner** (`components/app/Spinner.vue`): lucide `LoaderCircle` + `animate-spin`; in buttons it accompanies the label while `:disabled` — never replace the label with only a spinner.
- **Badges:** solid fills (`bg-{status} text-{status}-foreground`, white text). Badges are not interactive — no hover effects.
- **Form dialogs:** DialogHeader (title + description) → destructive `Alert` for the request error → `form.space-y-5` with fields as `space-y-1.5` (Label, control, `text-sm text-destructive-text` field error) → DialogFooter with `Cancelar` (outline) + submit (primary, disabled while loading).
- **Dialog/form errors:** request-level → `Alert variant="destructive"`; field-level → `text-destructive-text` paragraph under the field.
- **Money:** always `formatMoney()` from `src/lib/format.ts` (`S/ 120.00`). No inline `toFixed` formatting.
- **Icons + labels:** every nav item and primary action shows a visible Spanish label; icons never appear alone for primary actions (icon-only is acceptable for tertiary/utility controls with `aria-label`, e.g. the theme toggle).
- **Row links:** clickable table rows use `Button as-child > RouterLink` ("Ver") — not bare `@click` rows.

## 6. Hard rules (review checklist)

1. No raw Tailwind palette colors — tokens only.
2. No text below 14px.
3. White text on petrol/status fills; `-text` shades for colored text on backgrounds.
4. Flame orange only in wordmark + focus ring.
5. Controls ≥44px on mobile (`h-11 md:h-10` idiom).
6. Both themes checked before shipping.
7. Spanish labels, plain language, no jargon.
8. New screens use PageHeader / TableEmpty / EmptyState / Spinner / formatMoney — no parallel patterns.
9. Status colors = meaning only; mapping table extended here first.
10. No new UI framework or component library; shadcn-vue + Tailwind + lucide only.

## Related Files

- `lpg-frontend-vue/src/style.css` — token source of truth
- `lpg-frontend-vue/tailwind.config.js` — token→class mapping, type scale, font
- `lpg-frontend-vue/src/lib/theme.ts` — theme persistence/toggle
- `lpg-frontend-vue/index.html` — FOUC guard, theme-color metas
- `lpg-frontend-vue/src/components/app/` — PageHeader, Spinner, EmptyState
- `lpg-frontend-vue/src/lib/format.ts` — formatMoney
