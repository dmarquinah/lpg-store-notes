---
project: lpg-store
domain: specs
type: spec-track
spec: delivery-gps-location
repo: lpg-frontend-vue
kind: frontend
track-status: done
last-updated: 2026-07-15
---

# GPS delivery location (pin per customer) — lpg-frontend-vue track

Shared spec: [[index]]

## Technical Notes
- **Capture.** On the driver's deliver flow, add a **"Guardar ubicación GPS"** action
  that calls `navigator.geolocation.getCurrentPosition` and attaches
  `{ latitude, longitude }` to the deliver request. Requires HTTPS — already satisfied
  (Traefik TLS, `first-vps-deploy`). Handle the async permission prompt, timeout, and
  denial gracefully: show status ("Ubicación guardada" / "No se pudo obtener la
  ubicación"), never block the delivery submit. Phone-first, ≥44px touch target
  (design-system rules).
- **Show / navigate.** Wherever a pinned location exists (order detail, customer
  detail, driver delivery view), render an **"Abrir en Google Maps"** link
  (`https://www.google.com/maps?q=${lat},${lng}`) and, on driver-facing views, a
  **directions** link (`https://www.google.com/maps/dir/?api=1&destination=${lat},${lng}`).
  A small pure helper `mapsLink(lat,lng)` / `directionsLink(lat,lng)` keeps URL-building
  in one place.
- **Create wizard.** Today selecting a client prefills the free-text address. Keep that.
  Additionally, if the selected customer has a saved pin, surface it (e.g. a "Ubicación
  guardada ✓ — Ver en Maps" line) so the operator/driver knows a precise point exists.
  The backend prefills the order coords from the customer, so the frontend mainly needs
  to **display** the pin state, not re-send it.
- **Optional (respect scope):** an embedded Leaflet preview is explicitly out of scope
  for MVP (link-out only), even though Leaflet is already a dependency — do not add a map
  picker unless the Open Questions reopen it.

## Related Files

### lpg-frontend-vue (/home/diegomh/dev/personal/freelance/lpg-store/lpg-frontend-vue)
- `src/modules/orders/views/` — driver deliver view (`mis-entregas` / delivery detail): capture button + submit wiring.
- `src/modules/orders/views/OrderDetailView.vue` — show pin + "Abrir en Google Maps".
- `src/modules/orders/views/` — create-order wizard: display selected customer's saved-pin state.
- `src/modules/customers/views/` — customer detail: show pin + Maps link.
- `src/modules/orders/store` + `src/modules/customers/store` — pass/read the new coord fields.
- `src/lib/` — new `mapsLink`/`directionsLink` helper (+ small test).

## Implementation Notes
<!-- [YYYY-MM-DD] [lpg-frontend-vue] description -->

- [2026-07-15] [lpg-frontend-vue] Shipped the frontend track. New pure `src/lib/maps.ts` (`mapsLink`/`directionsLink` — Google Maps view + directions deep-links; no test file since the project has no test runner wired). Mirrored the backend read/write shapes in FE types: `OrderSummary.customerLatitude/Longitude` + `DeliverPayload.location`/`updateLocation` (orders), `PublicCustomer.latitude/longitude` + `Create/UpdateCustomerPayload` coords (customers) — stores pass payloads through untouched. **Capture:** `DeliverDialog` gained a registered-customer-only "Ubicación GPS" section — a ≥44px "Usar mi ubicación actual" button wrapping `navigator.geolocation.getCurrentPosition` (`enableHighAccuracy`, 15s timeout), unsupported-device guard, idle/capturing/captured/error states, never blocks submit; sticky rule mirrored client-side (no pin → send `location`; existing pin → send `updateLocation:true` only when the driver flips the "Actualizar la ubicación" switch). **Show/navigate:** "Abrir en Google Maps" + field-only "Cómo llegar" on `OrderDetailView` (Cliente card) and `CustomerDetailView` (Datos card, "Sin ubicación guardada" when null). **Office edit:** `CustomerFormDialog` single "lat, lng" paste field (chosen with owner) — `parseCoordinates` enforces both-or-neither + WGS84 ranges (matches backend Zod/DB), emptying clears both to null on update. **Create wizard:** `OrderCustomerChoice` threads lat/lng; `CustomerSelect` + `OrderCreateView` summary show a "Ubicación guardada ✓ — Ver en Maps" line for a pinned customer (display only; backend resolves order coords from the customer). Validated against the shared checklist (all frontend sub-points met) + typecheck + build green. All acceptance criteria met — track done.

- [2026-07-15] [lpg-frontend-vue] UX refinement after the backend finalized the write model (deliver `location` = **fill-when-empty only**, the `updateLocation` overwrite flag **removed**; overwrite moved to a dedicated `PUT /customers/:id/location` now allowed for **operator/admin AND delivery** — a driver only for a customer they're delivering to today, else 403). Frontend changes: (1) `DeliverDialog` GPS capture now shows **only when the customer has no pin yet** (`!isWalkIn && customerLatitude === null`) and is **hidden once a pin exists** — dropped the in-modal overwrite toggle; removed `updateLocation` from `DeliverPayload`. (2) New **`LocationDialog`** + an **"Actualizar/Agregar ubicación"** button on `OrderDetailView` (shown to any field user on a registered order) — GPS capture **or** paste feed one reviewable "lat, lng" field, submitted via a new `customers` service/store `updateLocation` → `PUT /customers/:id/location`; on save the order refetches so the Maps links update. A driver denied by the today-scope sees the backend 403 message. (3) Extracted a shared **`useGeolocationCapture`** composable (`src/lib/geolocation.ts`) used by both the deliver + location dialogs, and shared **`parseLatLng`/`formatLatLng`** in `src/lib/maps.ts` (also adopted by `CustomerFormDialog`). typecheck + build green; validated against the finalized backend contract.