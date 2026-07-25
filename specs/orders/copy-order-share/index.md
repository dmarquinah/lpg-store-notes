---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: '"done"'
depends-on: []
last-updated: 2026-07-15
---

# Spec: Copy order data to share

## Problem Statement
Workers (operators/drivers) are still learning the system. Until they're fluent,
the admin wants to relay a new order's details to them out-of-band (WhatsApp /
text) as a plain, readable message. Today there's no one-tap way to grab all of
an order's data — the admin would have to retype customer, address, items,
prices, payment method and notes by hand, which is slow and error-prone.

## Proposed Solution
A **"Copiar datos"** button on the order detail (and optionally the last step of
the create-order wizard / the create success toast) that formats the order into a
single human-readable Spanish text block and copies it to the clipboard, ready to
paste into WhatsApp.

Everything needed is already on the order-detail payload the frontend fetches — no
backend change. The work is: a formatting helper (order → text), a copy-to-clipboard
action with success feedback (reuse the existing `vue-sonner` toast), and placing
the button where the admin needs it.

The formatted block should include: order id, store, customer name + phone,
delivery address (+ location reference if present), each line item with type/qty/
unit price, total, payment method / on-credit flag, and notes. Keep it phone-legible.

## Acceptance Criteria
<!-- THE single shared checklist — source of truth across all tracks. -->
- [ ] Order detail shows a **"Copiar datos"** button that copies a formatted text block of the full order to the clipboard.
- [ ] The copied text includes: order #, store, customer name + phone, delivery address (and location reference when present), every line item (type · qty · unit price), total, payment method / on-credit status, and notes — labelled in Spanish, phone-legible.
- [ ] Copy uses the Clipboard API with a graceful fallback and shows a success toast (`vue-sonner`); a failure shows an error toast.
- [ ] Walk-in orders (no saved customer) render name/phone from the snapshot without blanks or "null".
- [ ] Formatting lives in one reusable helper (no duplicated string-building), unit-testable in isolation.
- [ ] No backend change; typecheck + build green.

## Tracks
<!-- Overall status becomes `done` only when EVERY track is done. -->

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope
- Any backend endpoint or schema change (all data is already on the detail payload).
- Sharing via a native share-sheet / deep WhatsApp integration — plain clipboard copy only for now.
- A print / PDF format.
- Copying from the orders **list** (queue) rows — detail (and optionally create success) only.

## Open Questions
- Besides the order detail, should the button also appear on the create-order
  **success** step/toast so the admin can copy immediately after creating? (Lean: yes, cheap.)
- Exact wording/emoji of the template — confirm with the owner once a draft exists.
