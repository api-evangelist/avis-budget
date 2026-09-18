---
generated: '2026-09-18'
method: generated
name: View, modify or cancel a reservation
description: Retrieve an existing Avis/Budget/Payless reservation by confirmation number and last name, change it (full or partial update), or cancel it.
api: openapi/avis-budget-rental-cars-openapi.yml
operations: ['GET /cars/reservation/v2', 'PATCH /cars/reservation/v2', 'PATCH /cars/reservation/v2/partial-modify', 'PUT /cars/reservation/v2']
source: >-
  Grounded in openapi/avis-budget-rental-cars-openapi.yml (OpenAPI 3.0.2). None of these four operations
  carries a provider operationId; each is cited by method + path verbatim, with the operationId proposed in
  overlays/avis-budget-rental-cars-overlay.yaml in brackets. Auth per authentication/, errors per errors/,
  reversibility per conventions/avis-budget-conventions.yml.
---

# View, modify or cancel a reservation

A reservation is addressed by `confirmation_number` + the renter's `last_name` (business code 150004 fires when they do not match) and always carries `brand`.

## Auth
- Bearer token from `https://stage.abgapiservices.com/oauth/token/v2` plus the `client_id` header on every call. See `authentication/avis-budget-authentication.yml`.
- Base URL `https://stage.abgapiservices.com` (staging — see `sandbox/avis-budget-sandbox.yml`).

## Steps
1. **View** — `GET /cars/reservation/v2` [overlay: `viewReservation`] with `brand`, `confirmation_number` and `last_name`. Returns `view_reservation_v2_response`: `status`, `transaction`, `product` (brand) and the full `reservation` (customer, vehicle with `sipp_code`/`vehicle_class_code`, pickup/dropoff, rate, totals, payment). Always do this first — it is the source of truth and the safe way to confirm what a previous, possibly timed-out, write actually did.
2. **Modify** — choose ONE of:
   - `PATCH /cars/reservation/v2/partial-modify` [overlay: `partialUpdateReservation`] with body `partial_modify_request` when changing a subset of fields (dates, vehicle, contact details). Prefer this.
   - `PATCH /cars/reservation/v2` [overlay: `updateReservation`] with the full `create_modify_request` body when re-specifying the whole reservation (same schema as create).
   Both return `200` with the updated reservation and re-price it; check `totals` in the response and show the user any change before treating the modification as done.
3. **Cancel** — `PUT /cars/reservation/v2` [overlay: `cancelReservation`] with body `cancel_reservation_v2_request` (brand + confirmation). `200` returns `cancel_reservation_v2_response` with `status.success[]` ("The reservation was successfully cancelled.") and the `transaction_id`. Note the provider models cancel as PUT, not DELETE, despite its own design guide.

## Rules
- **Cancellation is not undoable and no cancellation window or fee schedule is published in the API** (`conventions/` reversibility grade: documented, not verified). Prepaid reservations carry payment amounts; before cancelling one, read the location's terms via `GET /terms/v2/location/{brand}/{country_code}/{location_code}/{locale}` and confirm with the user.
- No idempotency key: if a PATCH/PUT times out, re-run step 1 to see the current state before repeating it.
- Errors arrive as `status.errors[]` `{code, message, reason, details}`; `404` on view means no reservation for that confirmation/last-name pair (or `resource_failure` for a bad URI); `400` business codes name the offending field. See `errors/avis-budget-problem-types.yml`.
- `401` -> refresh the token; `403` -> the application is not approved for this API.
