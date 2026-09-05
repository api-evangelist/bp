---
name: bp-fleet-reconcile-fleet-spend
description: Pull bp fleet-card transactions and invoices for a period and reconcile spend by account.
api: BP Open Fleet — Transaction Management, Invoice Management, Card Management
generated: '2026-09-04'
method: generated
source: openapi/bp-fleet-transaction-management-openapi.json, openapi/bp-fleet-invoice-management-openapi.json, openapi/bp-fleet-card-management-openapi.json
operations:
  - GET /transactions
  - GET /invoices
  - GET /cards
---

# Reconcile bp fleet spend for a period

This is the marquee read flow on the platform: pull what was spent, pull what was billed, and
line them up against the cards that did the spending. All three APIs are **read-only** — nothing
in this skill can change state.

## Before you start

Get a bearer token (see `bp-fleet-authenticate`). All three APIs are limited to **10 requests per
minute** each, so plan your paging: a wide date range is many sequential pages at six seconds
apart.

## Steps

1. **Pull transactions for the window.**

   `GET /transactions`

   Filter with `StartDateTime`, `EndDateTime`, and optionally `AuthorityIds` and `ParentIds` to
   scope to the accounts you care about. Page with `Page` and `PageSize`.

   The response entity is wide — 77 properties — and carries `transactionId`,
   `transactionUniqueId`, `siteId`, `parentId`, `authorityId`, `vehicleDriverCode`,
   `productCode`, cost-centre names, tax and pricing-type fields.

2. **Pull invoices for the same window.**

   `GET /invoices`

   Same filter vocabulary: `StartDateTime`, `EndDateTime`, `AuthorityIds`, `ParentIds`, `Page`,
   `PageSize`. Returns `invoiceId`, `parentId`, `authorityId`, `summaryStatementId`,
   `invoiceCurrencyCode` and net values.

3. **Pull card state.**

   `GET /cards`

   Note the different requirement profile: on this operation `PageSize`, `Page`,
   `StartDateTime` and `EndDateTime` are **all required**. Optionally filter by `CardStatusId`,
   `AuthorityIds`, `ParentIds`. BP notes this API currently exposes only the "latest card
   updates" endpoint, with more under development.

4. **Join.** There is no invoice id on the transaction record, so you cannot navigate directly
   from a transaction to its invoice. Reconcile on the shared account hierarchy —
   `parentId` + `authorityId` — plus the date window. Join a transaction to a physical station
   with `siteId` against Retail Site Information.

## Conventions that apply throughout

- **Correlation id.** Send `x-correlation-id` (uuid) on every request. The response envelope
  echoes `correlationId`. BP's support page asks for it when you report a problem.
- **Casing is inconsistent across products.** These three use `Page`/`PageSize`; Retail Site
  Information uses lowercase `page`/`pageSize`. Do not share one parameter serialiser blindly.
- **Error envelope** is `ResultEntity`: `success`, `message`, `correlationId`, and
  `errorDetails { errorCode, errorMessage }`. It is **not** RFC 9457 problem+json.

## Errors

`400` bad request (usually a missing required parameter on `/cards`), `500` server error.
Retry `500` with backoff and keep the correlation id.
