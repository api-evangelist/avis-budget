---
generated: '2026-09-18'
method: generated
name: Shop and book a rental car
description: Find an Avis/Budget/Payless location by keyword, shop available vehicles and a detailed rate for the dates, read the location's terms, then create the reservation.
api: openapi/avis-budget-rental-cars-openapi.yml
operations: ['GET /cars/locations/v2/keyword', 'Car Availability', 'POST /cars/catalog/v2/vehicles/rates', 'GET /terms/v2/location/{brand}/{country_code}/{location_code}/{locale}', 'POST /cars/reservation/v2']
source: >-
  Grounded in openapi/avis-budget-rental-cars-openapi.yml (OpenAPI 3.0.2). "Car Availability" is the only
  operationId the provider declares; the other steps cite method + path verbatim. Auth per
  authentication/avis-budget-authentication.yml, errors per errors/avis-budget-problem-types.yml, entity
  graph per data-model/avis-budget-data-model.yml, environment per sandbox/avis-budget-sandbox.yml.
---

# Shop and book a rental car

The Rental Cars API is a shop-then-book flow: location -> availability -> rate -> (terms) -> reservation. Every request needs `brand` (Avis, Budget or Payless).

## Auth
- Exchange your approved application's Client ID/Secret for a Bearer token at `https://stage.abgapiservices.com/oauth/token/v2` (the docs send `client_id` and `client_secret` as request headers). Tokens expire (documented example `expires_in: 7140`).
- Send BOTH `Authorization: Bearer <token>` and the `client_id` header on every call — `client_id` is a required header parameter on all nine operations.
- Base URL: `https://stage.abgapiservices.com`. This is the staging environment: reservations made here are not live bookings. See `sandbox/avis-budget-sandbox.yml`.

## Steps
1. **Find the pickup (and dropoff) location** — `GET /cars/locations/v2/keyword` [overlay: `searchRentalLocationsByKeyword`]. Pass `brand`, `country_code` (ISO 3166) and `keyword` (a city, airport or address, e.g. "Boston"). Capture the `location_code` (e.g. EWR) of the location the user wants; it is the key for every later step.
2. **Shop availability** — `Car Availability` (`GET /cars/catalog/v2/vehicles`). Pass `brand`, `pickup_location_code`, `dropoff_location_code`, `pickup_date`, `dropoff_date` (ISO 8601) and, if known, `age` or `date_of_birth` (or a `membership_code`, whose profile DOB takes precedence: membership_code -> age -> date_of_birth). Optional filters: `sipp_codes` (ACRISS/SIPP, e.g. CMDR), `vehicle_class_codes`, `discount_code`, `coupon_code`. The response `vehicles[]` gives each category with `sipp_code`, `vehicle_class_code`, the base rate and any discount/coupon applied. Pick a vehicle and keep its `vehicle_class_code` + `rate_code`.
3. **Get the detailed rate** — `POST /cars/catalog/v2/vehicles/rates` [overlay: `getVehicleRate`] with the chosen vehicle/rate qualifiers in the body (schema `car_availability_rate_request`). This is a POST that only prices — it creates nothing. The response (`rate_request_response`) carries distance limits, coverages, ancillary products, `totals` (`vehicle_total`, `taxes_fees_total`, `reservation_total`), insurance excess and, if requested, a secondary-currency price.
4. **Read the location's terms before committing** — `GET /terms/v2/location/{brand}/{country_code}/{location_code}/{locale}` [overlay: `getLocationTermsAndConditions`]. This free-text is where cancellation and prepay conditions live; the API publishes no structured cancellation window (see `conventions/avis-budget-conventions.yml` -> reversibility). Surface it to the user for any prepaid booking.
5. **Create the reservation** — `POST /cars/reservation/v2` [overlay: `createReservation`] with body schema `create_modify_request`: customer (first/last name, email, address, DOB/age), pickup/dropoff location codes and times, `vehicle_class_code` + `rate_code`, and any discount/coupon/membership/IATA or payment/prepay fields. A `201` returns `create_modify_reservation_v2_response` with the `confirmation_number` — store it together with the renter's `last_name`; both are needed to view, modify or cancel.

## Rules
- **No idempotency key exists** (`conventions/` idempotency.coverage: none). If the POST in step 5 times out or 500s, do NOT blindly retry — call `GET /cars/reservation/v2` with the intended `confirmation_number`/`last_name` (if any was returned) or ask the user before re-posting, or you can double-book.
- Business validation comes back as `400` with `status.errors[]` carrying a numeric ABG `code` (e.g. 14528 invalid rate_code, 231 pickup_date in the past, 221 more than 330 rental days, 1112 coupon needs an IATA number). Read `code` + `details`, fix the named field, resubmit. `404` can be a business "no results" (223 pickup location sold out, 500708 no vehicles for the filter) — treat as empty, not as an error.
- `401` means the token expired or credentials are wrong (`authentication_failure`); refresh and retry. `403` means the application is not approved for this API — stop.
- Send `Accept: application/json` and `Content-Type: application/json` (`406`/`415` otherwise).
- No rate limit is enforced today per ABG's Postman docs; a `429` with reason `throttled` is the only signal if that changes. Back off exponentially.
