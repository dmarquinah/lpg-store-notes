# Notifications — lpg-store

---
project: lpg-store
domain: specs
category: notifications
last-updated: 2026-06-16
---

## Context Documents

Read these vault docs before working on specs in this category:
- [[../../eng/frontend-bloat-analysis]] — FCM was one of v1's named bloat drivers (Firebase SDK, gstatic `importScripts`, build-time config-injection script). This category deliberately drops Firebase for **native Web Push + VAPID**.
- [[../../eng/architecture]] — backend + frontend v2 module architecture (vertical modules, service composition).
- [[../../eng/decisions]] — **ADR-012** (explicit service composition over events): the triggering services (inventory/orders) call an **injected notifier port** wired in `app.ts`, mirroring the accounting↔inventory guard injection — notifications is never imported by the modules that fire it.
- [[../../eng/design-system]] — the "Activar notificaciones" enable surface follows the petrol+flame tokens + ≥44px touch rules.

## Specs

| Slug | Status | Summary |
|------|--------|---------|
| [[push-notifications/index\|push-notifications]] | done (backend ✓ · frontend ✓) | Native Web Push (VAPID) PWA notifications — notify a delivery driver of a new daily assignment or a new delivery job. **No Firebase, zero client secrets.** |
