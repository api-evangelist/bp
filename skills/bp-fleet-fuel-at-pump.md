---
name: bp-fleet-fuel-at-pump
description: Find a bp/Aral station, authorize a fuelling session at a pump, and cancel it if needed.
api: BP Open Fleet — Aral AppConnect (Pay@Pump)
generated: '2026-09-04'
method: generated
source: openapi/bp-fleet-aral-appconnect-openapi.json
operations:
  - GET /sites
  - GET /sites/{siteId}
  - POST /payment-method
  - POST /fueling
  - GET /fueling
  - PUT /fueling/cancel
  - DELETE /payment-method
---

# Authorize fuelling at a bp / Aral pump

This is the one **write** surface on the bp Open Fleet platform, and it moves real money at a
physical pump. Read the cautions before automating any of it.

## Before you start

Get a bearer token (see `bp-fleet-authenticate`). This product is limited to
**100 requests per second** — the most generous limit on the platform, because it backs an
interactive driver flow.

Every operation accepts three headers: `api-version`, `X-APP-Client` and `X-Correlation-Id`.

## Steps

1. **Find nearby stations.**

   `GET /sites` with **required** `Lat`, `Lng` and `Radius`. Returns `NearbySite` records
   carrying `siteId`.

2. **Get the station detail.**

   `GET /sites/{siteId}` — returns address, location, fuel grades, pump configuration
   (`pumpId`), site features and operating hours.

3. **Register a payment method** (if not already on file).

   `POST /payment-method` — returns `entityId`, the handle you use when starting a session.

4. **Authorize fuelling.**

   `POST /fueling` with the site, pump and `entityId` in
   `InitiateFuelingRequest` (`siteId`, `pumpId`, `entityId`).

   Returns `InitiateFuelingResponse` with `transactionId` and `pumpId`.

   **Keep the `transactionId`.** It is the only handle to the session, and it is passed as the
   `X-Transaction-Id` header — not in the body — on the next two calls.

5. **Check the session.**

   `GET /fueling` with **required** header `X-Transaction-Id`.

6. **Cancel if needed.**

   `PUT /fueling/cancel` with **required** header `X-Transaction-Id`.

## Cautions — read before automating

- **No idempotency.** There is no `Idempotency-Key` on `POST /fueling`. A retry after a timeout
  is a **second authorization**, not a replay of the first. If a call times out, use
  `GET /fueling` to establish state before retrying anything.
- **The cancellation window is unknown.** `PUT /fueling/cancel` exists, but BP publishes no
  statement of when it stops working — not before nozzle release, not before settlement, not
  some number of minutes. Do not assume a cancellation will succeed, and do not tell a user it
  will.
- **Thin error surface.** Every operation on this API declares only `200` and `500`. There is no
  documented `400`, `401`, `403` or `404`, so a validation failure has no published shape. Treat
  any non-200 defensively and read the `errors[]` array
  (`{ "errors": [ { "errorCode": …, "errorMessage": … } ] }`).
- **Reversal for payment methods** is `DELETE /payment-method` (by `entityId`); no window is
  published for that either.
