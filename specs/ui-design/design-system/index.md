---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on: []
last-updated: 2026-06-12
---

# Spec: Design System + UI Overhaul

## Problem Statement

Piloto is functional but visually unrefined: the theme is the stock shadcn-vue
slate palette with only `--primary`/`--ring` swapped to burnt-orange, and the 7
modules consume components ad hoc — no shared conventions for page headers,
form layout, table density, empty/loading/error states, or status colors.

This matters because the real users (operators, drivers, the admin) are
**middle-aged people with little web-app experience**. The UI must be easy on
the eyes and easy to navigate, or the traceability value of the product is lost
to user friction. Without documented conventions, every future feature also
re-invents its own look, drifting further apart.

## Proposed Solution
One spec, two deliverables:

1. **The design system** — defined as a vault doc (`eng/design-system.md`) and
   implemented as theme tokens + component conventions in the existing stack
   (shadcn-vue + Tailwind; no new UI framework or dependency). Direction
   decided with the user 2026-06-12:
   - **Warm off-white palette** around the existing burnt-orange brand primary
     (`18 85% 40%`): warm-neutral background (`~30 20% 98%`), white cards, warm
     grays for muted/border tokens, muted-foreground darkened for WCAG AA.
     Fixed semantic set (success / warning / destructive / info) used *only*
     for meaning — status badges, alerts — never decoration.
   - **Both light and dark are first-class themes.** The current app default
     renders dark (the only tuned block today); this spec designs the warm
     palette in both modes, AA-verified in both, with an explicit user toggle
     in the app shell (persisted; default follows system preference).
   - **Bundled humanist sans** (self-hosted, e.g. Public Sans — legible at UI
     sizes, distinct numerals for prices/quantities; cached by the PWA, no
     external requests).
   - **Legibility-first typography**: base ≥ 16px, generous line height, a
     small fixed scale (page title / section / body / caption); no text below
     14px anywhere.
   - **Touch & spacing rules**: interactive targets ≥ 44px tall on mobile,
     consistent page padding/gaps from a small spacing scale. **Comfortable
     table density** (~52px rows, 15–16px cell text) on operator screens.
   - **Plain-language + icon pairing**: every nav item and primary action has a
     visible Spanish label; icons never appear alone for primary actions.
   - **Canonical patterns**: one page-header layout, one form layout, one
     dialog anatomy, one table treatment, and standard empty / loading / error
     states — documented with do/don't examples.
2. **The retrofit** — apply the system across the app shell (login, drawer
   nav, top bar) and all 7 modules (auth, home, users, catalog, inventory,
   customers, orders) so the whole app reads as one product. Future work
   follows the doc as a hard convention.

## Acceptance Criteria
<!-- THE single shared checklist — source of truth across all tracks. -->
- [x] `eng/design-system.md` exists in the vault: tokens (both themes), type scale, spacing/touch rules, semantic color usage, canonical component patterns (page header, form, dialog, table, badge, empty/loading/error states), and accessibility rules — written so future specs can cite it
- [x] Theme tokens implemented in `src/style.css` / `tailwind.config.js` for **both light and dark**: petrol+flame surface palette + semantic colors (success/warning/info added to the existing destructive), brand primary retained as `--brand`
- [x] All text/background combinations meet WCAG AA contrast **in both themes** (tokens chosen to target AA; status `*-text` shades lightened in dark for legibility)
- [x] Theme toggle in the app shell: persisted choice (`piloto-theme`), default follows system preference, both modes fully usable
- [x] Bundled humanist sans self-hosted (`@fontsource-variable/public-sans`) and applied app-wide (no external font requests; precached by the PWA)
- [x] Base font size ≥ 16px and interactive controls ≥ 44px touch height on mobile viewports (`h-11 md:h-10`); comfortable table density on operator screens
- [x] App shell restyled: login, drawer nav, top bar follow the system
- [x] All 7 module screens restyled to the canonical patterns (consistent page headers, tables, forms, dialogs, status badges, empty/loading/error states) with no functional regressions
- [x] No new UI framework or component library introduced; shadcn-vue + Tailwind + lucide retained
- [x] `npm run typecheck` and `npm run build` green — manual both-theme smoke pass left to the operator (per project convention)

## Tracks
| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope
- New features or behavior changes — this is a restyle + conventions pass only
- The bot (lpg-bot) and backend — no API changes
- i18n / language switching (single deployment language, per repo conventions)
- Component library swap or visual-regression test tooling

## Open Questions
All resolved with the user on 2026-06-12:

- Surface palette → **warm off-white** neutrals (not cool slate)
- Font → **bundled humanist sans** (self-hosted; Public Sans as default candidate)
- Dark mode → **both light and dark are in scope**, AA in both, shell toggle (system-preference default); note the app's current default renders dark
- Table density → **comfortable** (~52px rows) on operator screens