---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on:
  - "[[../../auth/auth-foundation/index]]"
  - "[[../../inventory/inventory-foundation/index]]"
  - "[[../../orders/orders-foundation/index]]"
last-updated: 2026-06-16
---

# Spec: Push Notifications (native Web Push + VAPID, no Firebase)

## Problem Statement

A delivery driver has no way to learn — without manually opening the app and
refreshing — that work has been routed to them. Two moments matter:

1. **A new daily assignment** — an inventory day is opened/assigned to the driver
   (the tanks they carry for the day). Today they find out only by opening
   `/mi-dia`.
2. **A new delivery job** — an order is assigned (and/or dispatched) to the
   driver. Today they find out only by opening `/mis-entregas`.

The business wants the driver's installed PWA to surface a **push notification**
for each, so the driver acts promptly even when the app isn't open.

**The v1 lesson to avoid.** v1 implemented this with Firebase Cloud Messaging,
which [[../../eng/frontend-bloat-analysis|the bloat analysis]] flags as a real
driver of complexity: the `firebase/messaging` SDK, `importScripts` from
`gstatic`, a `firebase-messaging-sw.template.js` with `__VITE_FIREBASE_*__`
placeholders, and a `generate-service-worker.js` build step that injected config
from `.env.local`. The owner remembers this as "a lot of configs to securely
hide the Firebase token." **Two facts reframe that:**

- The Firebase web config (`apiKey`, `projectId`, `messagingSenderId`, `appId`,
  the VAPID *public* key) was **never secret** — Firebase publishes those as
  client identifiers, meant to ship in the bundle. The build-time
  template-injection ceremony was treating publishable values as secrets.
- The **only** true secret in any push setup is the **server credential**, which
  lives on the backend and never ships to the client either way.

So v1 was not leaking a real secret — but the ceremony was wasted, and the whole
Firebase dependency is avoidable.

## Proposed Solution

Use the browser's **native Web Push API with VAPID** — no Firebase, no
third-party SDK, **zero secrets in the client build by construction**. (Transport
confirmed with the owner 2026-06-16.)

**Key handling (the whole point of this spec):**

- The client bundle holds **only the VAPID public key**
  (`VITE_VAPID_PUBLIC_KEY`) — public by design, like a TLS public cert; safe to
  ship.
- The backend holds the **VAPID private key** (`VAPID_PRIVATE_KEY`) — the single
  secret, server-side env only, used to sign each push. Generated once
  (`web-push generate-vapid-keys`).
- No build-time injection script, no `.env.local` template dance, no
  `firebase/*` dependency, no `gstatic` `importScripts`.

**Flow:**

1. A **delivery** user explicitly taps **"Activar notificaciones"** (permission
   must follow a user gesture — v1's auto-init hit gesture violations). The app
   requests `Notification.permission`, then
   `registration.pushManager.subscribe({ userVisibleOnly: true,
   applicationServerKey: <VAPID public key> })`.
2. The resulting `PushSubscription` (`endpoint` + `p256dh`/`auth` keys) is
   **POSTed to the backend** and stored against the user.
3. When inventory opens a day for that driver, or an order is assigned to them,
   the triggering service calls an **injected notifier port**; the notifications
   service loads that user's subscriptions and sends a push via the `web-push`
   library, signing with the private key.
4. The PWA **service worker**'s `push` handler calls
   `self.registration.showNotification(...)`; `notificationclick` focuses an open
   window (or opens one) and navigates to the relevant deep link
   (`/mi-dia` for an assignment, `/mis-entregas/:id` for a delivery job).

**Service worker integration.** v2 already builds the SW via `vite-plugin-pwa`
(Workbox, `registerType: 'autoUpdate'`, with a `NetworkFirst` `/v1/` runtime
cache). Add the `push` / `notificationclick` handlers **without rewriting the
existing config**: the least-invasive path is `workbox.importScripts:
['push-sw.js']` (a small hand-authored `public/push-sw.js` imported into the
generated SW); the alternative is switching to the `injectManifest` strategy with
a custom `src/sw.ts` (more control, but the existing precache + runtime-caching
config must be re-expressed by hand). The track file recommends `importScripts`
as primary. Either way **no secret enters the SW** — the public key is only
needed at subscribe time, in the app, not the SW.

**Scope: delivery role only** (mirrors v1's `shouldEnableNotifications` ===
DELIVERY). Operators/admins are out of scope for this spec.

Detailed backend design (subscription store, VAPID, send + triggers) in
[[backend]]; the enable surface, subscribe flow, and SW handlers in [[frontend]].

## Acceptance Criteria

<!-- THE single shared checklist — source of truth across both tracks. -->

**Cross-cutting (the security goal):**

- [x] **No secret ships to the client.** The production frontend bundle contains
      only the VAPID **public** key (`VITE_VAPID_PUBLIC_KEY`); the private key
      exists only in the backend env. Grep the built `dist/` for the private key
      / any Firebase config → none present. No `firebase/*` dependency is added.
- [x] End-to-end: a delivery user enables notifications on an installed PWA →
      backend stores the subscription → a triggering event delivers a visible
      notification → tapping it opens the correct deep link.

**Backend (lpg-backend):**

- [x] New `notifications` vertical module + migration: a `push_subscriptions`
      table keyed by **unique `endpoint`** with `user_id`, `p256dh`, `auth`,
      optional `user_agent`/`platform`, `created_at`, `last_seen_at`.
- [x] `POST /api/v1/notifications/subscriptions` — authenticated; **upserts** by
      endpoint (re-subscribe is idempotent), binding the subscription to the
      caller. `DELETE /api/v1/notifications/subscriptions` — removes a
      subscription by endpoint (logout / disable).
- [x] VAPID config read from env (`VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`,
      `VAPID_SUBJECT`); private key never logged or returned by any endpoint.
      Sends use the `web-push` library.
- [x] `NotificationsService.notifyUser(userId, payload)` loads the user's
      subscriptions and sends to each; **prunes** subscriptions that return
      `404`/`410` (Gone) so dead endpoints don't accumulate. Send failures never
      break the triggering request (best-effort, fire-and-forget).
- [x] **Triggers via an injected notifier port** (ADR-012 — wired in `app.ts`,
      so inventory/orders never import the notifications module, no dependency
      cycle):
      - inventory **opens/assigns a day** for a driver → "Nueva asignación del
        día" → data deep-link `/mi-dia`;
      - orders **assigns (and/or dispatches)** an order to a driver → "Nuevo
        pedido para entregar" → data deep-link `/mis-entregas/:id`.
- [x] Scoping: a user may only create/delete subscriptions bound to themselves;
      no endpoint discloses another user's subscriptions or the private key.
- [x] Tests: subscribe upsert is idempotent on repeated endpoint; a pruned
      `410` endpoint is deleted; `notifyUser` with no subscriptions is a no-op; a
      triggering event calls the notifier port. Existing tests stay green;
      typecheck / lint / build green.

**Frontend (lpg-frontend-vue / Piloto):**

- [x] New `notifications` vertical module (`createNotificationsModule`) wired in
      `main.ts`, exposing a service (subscribe/unsubscribe) + a small store
      (permission + subscription state).
- [x] An explicit **"Activar notificaciones"** control, **delivery-role only**
      (surfaced on the delivery home / `/mi-dia`), that on tap requests
      permission **in the user gesture**, subscribes via `pushManager.subscribe`
      with the `VITE_VAPID_PUBLIC_KEY` (base64url → `Uint8Array`), and POSTs the
      subscription. It reflects the three states (default / granted+subscribed /
      denied-with-guidance) and lets the user **turn it off** (DELETE + local
      `unsubscribe`).
- [x] The PWA service worker gains `push` and `notificationclick` handlers (via
      `workbox.importScripts: ['push-sw.js']` in `vite.config`, or
      `injectManifest`) — `push` shows the notification; `notificationclick`
      focuses an existing client and navigates, else `openWindow`s the deep link.
      The existing precache + `/v1/` `NetworkFirst` runtime cache keep working.
- [x] `pushsubscriptionchange` is handled (re-subscribe + re-POST); logout
      unsubscribes/deletes the subscription so a shared device doesn't keep
      pushing to a signed-out driver.
- [x] Uses the petrol+flame design system (tokens only, ≥44px touch target,
      `formatMoney` n/a here) per `eng/design-system.md`.
- [x] typecheck + build green; a built `dist/` contains no private key / Firebase
      config. Manual smoke (install PWA on a phone → enable → trigger an
      assignment/order → receive + tap → lands on the deep link) left to the
      operator.

## Tracks

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope

- **Operator / admin notifications** (new order in store, failed delivery) — this
  spec is delivery-role only. A later spec can add the operator triggers reusing
  the same `notifyUser` port.
- **In-app / foreground toast handling beyond the basics** — the service worker
  shows the system notification; rich foreground UX (live badge counts,
  notification center) is a later concern.
- **Notification preferences** (per-category opt-in like v1's
  `NotificationPreferences`) — a single on/off enable is enough for MVP.
- **iOS quirks beyond documenting them** — iOS Safari delivers Web Push **only
  to a home-screen-installed PWA** (iOS 16.4+), never an in-Safari tab. The spec
  documents this; it does not add an iOS-specific transport.
- **Native app / store distribution, SMS, email** — Web Push only.
- **Reintroducing Firebase / FCM** — explicitly rejected; the transport is
  native Web Push + VAPID.

## Open Questions (resolved 2026-06-16)

- **Transport = native Web Push + VAPID, not Firebase** (owner decision). The
  client carries only the public VAPID key; the private key is the lone secret
  and stays on the backend — meeting the owner's "no secret in the client build"
  goal by construction, and dropping the Firebase dependency entirely.
- **Recipients/triggers = delivery driver only**, on (a) a new daily inventory
  assignment and (b) a new delivery-job order assignment (owner selection). The
  operator-side "order needs attention" trigger was offered and **not** chosen —
  deferred to a future spec (Out of Scope).
- **Public VAPID key delivery = build-time env (`VITE_VAPID_PUBLIC_KEY`)**, not a
  runtime fetch — it's publishable, so baking it into the bundle is simplest. An
  optional `GET …/vapid-public-key` endpoint is allowed but not required.
- **SW integration = `workbox.importScripts` (primary)** over `injectManifest`,
  to avoid re-expressing the existing precache + `/v1/` runtime-cache config by
  hand. `injectManifest` remains an acceptable alternative if richer SW control
  is wanted.

## Notes

- **Build order:** the backend track lands the subscription endpoint + VAPID +
  triggers first; the frontend track can be built in parallel against the
  documented contract but can't be smoke-tested end-to-end until the backend is
  up. (The owner is currently working in **lpg-frontend-vue**; the backend track
  is in the separate **lpg-backend** repo.)
- The overall spec is `done` only when **both** tracks are done.
