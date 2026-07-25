---
project: lpg-store
domain: specs
type: spec
spec-layout: folder
status: done
depends-on: []
last-updated: 2026-07-15
---

# Spec: GPS delivery location (pin per customer)

## Problem Statement
Addresses in a small city are unreliable — a free-text `deliveryAddress` (or the
customer's `locationReference` like "portón rojo junto a la panadería") often isn't
enough for a driver to find the exact spot, especially a driver new to a route. The
business wants the driver to **pin the exact GPS point** where an order was
fulfilled, so that for **future orders to the same customer** any driver can see the
precise location and open it in **Google Maps** to navigate there. Today there is
**zero** geolocation in the system: orders and customers carry only free text, no
coordinates.

## Proposed Solution
**Chosen model: a single sticky pin on the customer** (confirmed with the owner).
The pin lives **only on `customers`** — orders carry no coordinates of their own;
the order read resolves the customer's pin through the existing customer join, so
the UI can build a Maps link without duplicating or snapshotting anything.

- Add nullable `latitude`/`longitude` (`numeric(9,6)`, ≈0.11 m — beyond any browser
  GPS) to **customers**, guarded by DB CHECKs (both-or-neither + WGS84 range).
- **The pin is sticky — written only when empty or on explicit intent:**
  - **Driver at delivery:** marking an order `delivered` can optionally attach the
    device GPS point. It is saved onto the customer **only when the customer has no
    pin yet** (fill-when-empty) — the delivery flow **never overwrites** an existing
    pin. Capture is optional and never blocks delivery; a walk-in saves nothing.
  - **Office & delivery:** operators/admins correct or replace an existing pin via a
    dedicated `PUT /customers/:id/location` (e.g. pasting a coordinate off Google
    Maps). A **delivery** driver may use the same endpoint, but only for a customer
    they have a live order for on one of **today's** assignments (no such order, or a
    different day → 403). This is the only way to overwrite an existing pin.
- **Reuse + navigate:** the order read (summary + detail) exposes the resolved
  customer pin (`customerLatitude`/`customerLongitude`); the customer read exposes
  `latitude`/`longitude`. Everywhere a pin exists the UI shows an **"Abrir en Google
  Maps"** link (`https://www.google.com/maps?q=lat,lng`) plus a directions link for
  drivers. The free-text address stays — the pin is additional structured data.

## Acceptance Criteria
<!-- THE single shared checklist — source of truth across all tracks. -->
- [x] `customers` gains nullable `latitude`+`longitude` (`numeric(9,6)`) with DB CHECKs (both-or-neither + WGS84 range), via migration `0018_customer_gps_location`. **No `orders` columns** — the order reads the customer's pin via the existing join.
- [x] A driver marking an order `delivered` can optionally attach a captured GPS point; it is saved onto the customer's pin **only when the pin is empty** (fill-when-empty) — the delivery flow never overwrites an existing pin.
- [x] Capturing GPS is optional and never blocks delivery: no fix / denied permission / walk-in still delivers normally (walk-in → nothing saved).
- [x] A customer's pin can be set/replaced via a dedicated `PUT /customers/:id/location` (both coordinates required, range-validated) — the only path that overwrites. Operators/admins may correct any customer; a **delivery** driver only one they have a live order for on **today's** assignment (composed from inventory + orders; else 403).
- [x] The order read (summary + detail) exposes the customer's saved pin (`customerLatitude`/`customerLongitude`) for the Maps link; the customer read exposes `latitude`/`longitude`.
- [x] Coordinates validated server-side (lat ∈ [-90,90], lng ∈ [-180,180]); invalid or half-pin input rejected (400), not stored.
- [x] Backend tests: deliver-capture fills empty pin; no-overwrite without flag; overwrite with flag; deliver-without-location unchanged; invalid coords rejected; office set/clear/half-pin; order read exposes pin. Full gate green.
- [x] Frontend: GPS capture uses the browser Geolocation API over HTTPS (PWA), with a clear permission prompt and a graceful fallback when unavailable; "Abrir en Google Maps" / directions links on the relevant views; typecheck + build green.

## Tracks
<!-- Overall status becomes `done` only when EVERY track is done. -->

| Track | Repo | Kind | Status |
|-------|------|------|--------|
| [[backend]] | lpg-backend | backend | done |
| [[frontend]] | lpg-frontend-vue | frontend | done |

## Out of Scope
- **Multiple saved locations per customer** (home vs shop). The single pin is
  sticky (set-once, then explicit edits). If the business needs several pins per
  customer, that's a follow-up spec introducing a `customer_locations` table —
  deliberately deferred to keep MVP small.
- **Per-order delivery-point history.** Coordinates live only on the customer, not
  on each order — we don't record where every individual delivery physically landed.
- Map **rendering** inside the app (embedded Leaflet picker/preview). MVP links out
  to Google Maps; an in-app map is a possible later enhancement.
- Reverse-geocoding coordinates back into a text address.

## Open Questions
_All resolved with the owner (2026-07-15):_
- **Manual edit** — **included now:** operators/admins correct/replace the pin via a
  dedicated `PUT /customers/:id/location`; a **delivery** driver may too, but only for
  a customer they have a live order for on today's assignment.
- **Re-capture at delivery** — **fill-when-empty only:** delivery saves the pin when
  the customer has none and never overwrites; correcting an existing pin is the
  separate office `PUT /customers/:id/location`, so the pin doesn't drift per delivery.
- **Capture point** — **`deliver` only** (where the driver is at the door).
