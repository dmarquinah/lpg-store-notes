---
project: lpg-store
domain: specs
category: ui-design
last-updated: 2026-06-12
---

# UI Design — lpg-store

Visual design, UX conventions, and design-system work for the Piloto frontend.
The audience constraint that drives everything here: **operators and drivers are
middle-aged users with limited web-app experience** — legibility, large touch
targets, low visual noise, and predictable navigation outrank density and flair.

## Context Documents

Read these vault docs before working on specs in this category:

- [[../../eng/architecture]] — Frontend section: module layout, shadcn-vue/Tailwind stack, theming file map
- [[../../eng/frontend-bloat-analysis]] — why v1 bloated; conventions here must not reintroduce per-role parallel UI
- [[../../eng/patterns/frontend-module-template]] — the vertical module convention new UI work must respect

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[design-system/index\|design-system]] | done | Design system + full UI overhaul: tokens (color scheme, type scale, spacing/touch targets), documented conventions, and a restyle pass over the app shell and all 7 modules. |
| [[mobile-layout-audit/index\|mobile-layout-audit]] | done | Phone-first responsive sweep over all pages: a shared **`ResponsiveTable`** (rows→cards <640px) replacing 17 raw tables, scroll/wrap tab strips, full-width filter controls, and an AppLayout header that stops cramming on 390px. Layout only — no token/behavior change. |
