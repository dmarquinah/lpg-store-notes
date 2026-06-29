---
project: lpg-store
domain: specs
type: spec-track
spec: push-notifications
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-06-16
---

# Push Notifications — lpg-frontend-vue (Piloto) track

Shared spec: [[index]] · Backend contract: [[backend]]

## Technical Notes

A new **`notifications`** vertical module plus a small addition to the PWA
service worker. **No Firebase** — drop the entire v1 approach
([legacy/src/services/notificationService.ts](../../../../lpg-frontend-vue/legacy/src/services/notificationService.ts),
[legacy/src/services/firebase/config.ts](../../../../lpg-frontend-vue/legacy/src/services/firebase/config.ts),
[legacy/public/firebase-messaging-sw.template.js](../../../../lpg-frontend-vue/legacy/public/firebase-messaging-sw.template.js),
[legacy/scripts/generate-service-worker.js](../../../../lpg-frontend-vue/legacy/scripts/generate-service-worker.js)).
The native Web Push flow needs none of it.

### Env (publishable)

- `VITE_VAPID_PUBLIC_KEY` — the VAPID **public** key, base64url. Publishable;
  baked into the bundle. **This is the only key the client ever sees.** No
  private key, no Firebase config, no build-time injection script.
- Document it in `.env.example`.

### Module: `src/modules/notifications/`

Follow [[../../eng/patterns/frontend-module-template]] and the existing modules:

- `service.ts` — `subscribe(subscription)` → `POST /notifications/subscriptions`
  `{ endpoint, keys: { p256dh, auth }, platform }`; `unsubscribe(endpoint)` →
  `DELETE /notifications/subscriptions`. Use the shared `apiClient`.
- `store.ts` (Pinia) — permission state (`default | granted | denied`),
  `isSubscribed`, and actions: `enable()` (request permission **in the user
  gesture** → `pushManager.subscribe` → POST), `disable()` (local
  `subscription.unsubscribe()` + DELETE), `syncState()` (read current
  permission + existing `pushManager.getSubscription()` on load).
- `createNotificationsModule({ apiClient })` factory wired in `main.ts` (same as
  the other modules).

### Subscribe flow (the careful bits)

- **User gesture required.** v1 hit gesture violations by auto-initializing.
  Permission + `subscribe` must run **inside the click handler** of the "Activar
  notificaciones" control — never on mount/login.
- `applicationServerKey` must be a `Uint8Array` — implement the standard
  `urlBase64ToUint8Array(VITE_VAPID_PUBLIC_KEY)` helper.
- `subscribe({ userVisibleOnly: true, applicationServerKey })` on the existing
  SW registration. `vite-plugin-pwa` already registers the SW
  (`registerType: 'autoUpdate'`); get the registration via
  `navigator.serviceWorker.ready`.
- Serialize the `PushSubscription` with `.toJSON()` → `{ endpoint, keys:{ p256dh,
  auth } }` and POST it.
- **`pushsubscriptionchange`** — handle re-subscription (the SW can fire it; or
  re-check on app focus) and re-POST the new subscription.
- **Logout** — call `disable()` (unsubscribe + DELETE) so a shared device stops
  pushing to a signed-out driver. Hook into the auth store's logout.

### Enable surface (delivery-role only)

- An **"Activar notificaciones"** control gated to the **delivery** role
  (mirrors v1's `shouldEnableNotifications === DELIVERY`). Natural home: the
  delivery home card and/or the `/mi-dia` view header.
- Three visible states: not-enabled (CTA), enabled (subscribed — offer "Desactivar"),
  denied (explain the browser-settings path; can't re-prompt programmatically).
- Petrol+flame tokens only, ≥44px touch target (`eng/design-system.md`).
- Surface a hint for **iOS**: Web Push works only when the PWA is **installed to
  the home screen** (iOS 16.4+). If `Notification` / `PushManager` is absent
  (e.g. iOS Safari tab), show "Instala la app para activar las notificaciones"
  instead of a dead button.

### Service worker handlers

The SW needs `push` + `notificationclick`. Keep the existing precache + `/v1/`
`NetworkFirst` runtime cache intact:

- **Primary: `workbox.importScripts: ['push-sw.js']`** in
  [vite.config.ts](../../../../lpg-frontend-vue/vite.config.ts) — author a small
  `public/push-sw.js` that adds:
  - `self.addEventListener('push', e => { const p = e.data.json();
    e.waitUntil(self.registration.showNotification(p.title, { body: p.body,
    icon: '/images/pwa-192x192.png', badge: '/images/pwa-192x192.png',
    data: p.data })); })`
  - `self.addEventListener('notificationclick', e => { e.notification.close();
    const url = e.notification.data?.url || '/'; e.waitUntil(focus-or-openWindow(url)); })`
    — match an existing client and `client.navigate`/`focus`, else
    `clients.openWindow(url)`. (The v1 `firebase-messaging-sw.template.js`
    `handleViewAction` is a usable reference for the focus-or-open logic, minus
    Firebase.)
  - **No keys in the SW** — the public key is only needed at subscribe time, in
    the app.
- *Alternative:* switch `VitePWA` to `strategies: 'injectManifest'` with a custom
  `src/sw.ts` (call `precacheAndRoute(self.__WB_MANIFEST)` + re-express the
  `/v1/` `NetworkFirst` route + add the push handlers). More control, more code —
  only if `importScripts` proves limiting.

### Reuse / don't reinvent

- Module wiring, service/store shape: copy an existing small module
  (`customers`, `catalog`).
- Deep-link targets already exist: `/mi-dia` (driver day) and `/mis-entregas/:id`
  (delivery job detail) — the notification `data.url` just routes there.
- Do **not** add `firebase`, `@vuepic/vue-datepicker`, `vue-i18n`, or any
  deferred dependency (CLAUDE.md).

## Related Files

> Confirm exact paths at `/focus` time.

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)

To create:

- `src/modules/notifications/{service,store,routes?}.ts` + the enable
  component(s) — new vertical module.
- `public/push-sw.js` — the `push` + `notificationclick` handlers imported into
  the Workbox SW.

To modify:

- `src/main.ts` — `createNotificationsModule({ apiClient })`.
- `vite.config.ts` — `workbox.importScripts: ['push-sw.js']` (or switch to
  `injectManifest`).
- The delivery home + `/mi-dia` view (`src/modules/inventory/`) — host the
  "Activar notificaciones" control (delivery-gated).
- `src/modules/auth/store.ts` — call `notifications.disable()` on logout.
- `.env.example` — add `VITE_VAPID_PUBLIC_KEY`.

Read-only reference (to delete/ignore, not port):

- `legacy/src/services/notificationService.ts`, `legacy/src/services/firebase/`,
  `legacy/public/firebase-messaging-sw.template.js`,
  `legacy/scripts/generate-service-worker.js` — the v1 Firebase approach this
  spec deliberately replaces.

## Implementation Notes

<!-- Format: [YYYY-MM-DD] [repo-name] description of what was done -->

[2026-06-16] [lpg-frontend-vue] Frontend track **done**. Native Web Push (VAPID) end-to-end on the client — **no Firebase, zero client secrets**. **New `notifications` vertical module** (`src/modules/notifications/`): `service.ts` (`subscribe` → `POST /notifications/subscriptions`; `unsubscribe` → `DELETE /notifications/subscriptions` with the `{endpoint}` body forwarded via `apiClient.delete`'s `init`); `store.ts` (Pinia, provide-pattern like auth/customers) holding `supported`/`configured`/`permission`/`isSubscribed`/`busy`/`error` with `syncState()`, `enable()`, `disable()` + a `urlBase64ToUint8Array` helper reading `VITE_VAPID_PUBLIC_KEY`; `index.ts` `createNotificationsModule({ apiClient })` wired in `main.ts`. **Enable surface** `components/EnableNotifications.vue` rendered on `DeliveryHome.vue` (already `meta.roles:['delivery']`-gated → delivery-only by construction; component imported nowhere else): permission is requested **inside the click gesture** (`enable()` is the `@click`, no `await` before `Notification.requestPermission()`), then `pushManager.subscribe({ userVisibleOnly:true, applicationServerKey })` → POST of the `toJSON()`-serialized subscription (`{endpoint, keys:{p256dh,auth}, platform}`, exact backend shape). Reflects four states — unsupported/iOS-needs-install, default→"Activar notificaciones", granted+subscribed→"Desactivar", denied→browser-settings guidance — plus inline `error`; the whole card is hidden when no `VITE_VAPID_PUBLIC_KEY` is configured. **Service worker**: new `public/push-sw.js` (`push` → `showNotification`; `notificationclick` → focus an existing same-origin client + `navigate(data.url)` else `openWindow`; `pushsubscriptionchange` → re-subscribe, app re-POSTs on next `syncState()`), imported into the Workbox-generated SW via `workbox.importScripts:['/push-sw.js']` in `vite.config.ts` — the existing precache + `/v1/` `NetworkFirst` runtime cache are untouched (verified in the built `dist/sw.js`, which also precaches `push-sw.js` with a revision). **Logout** drops the device's subscription: `AppLayout.handleLogout` `await notifications.disable()` (best-effort, `.catch`) before `auth.logout()`, so a shared device stops pushing to the signed-out driver. **Housekeeping**: `VITE_VAPID_PUBLIC_KEY` typed in `vite-env.d.ts`; `.env.example` stripped of the stale Firebase block and given `VITE_VAPID_PUBLIC_KEY` (publishable) with a generate-keys note. No new npm deps (native API); no `firebase/*`. **Touch targets** ≥44px on phone (the project `Button` default is `h-11 md:h-10`). **Independent validation: all 6 acceptance criteria MET, no blocking bugs**, no design-system violations (tokens only — `text-muted-foreground`/`text-destructive-text`/`text-primary`, no raw palette). **Gates green:** `npm run typecheck` clean + `npm run build` (PWA `generateSW`, **69 precache entries / 777.48 KiB**, `dist/sw.js` + `dist/push-sw.js` emitted). Verified **no Firebase/FCM/gstatic and no private key in `dist/`**. Manual end-to-end smoke (install PWA on a phone → Activar → open a day / assign an order from the backend with VAPID keys set → receive + tap → lands on `/mi-dia` or `/mis-entregas/:id`) left to the operator — needs the backend running with a real VAPID pair.