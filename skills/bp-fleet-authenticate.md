---
name: bp-fleet-authenticate
description: Obtain a bearer access token for the bp Open Fleet APIs using client credentials.
api: BP Open Fleet — Authentication
generated: '2026-09-04'
method: generated
source: openapi/bp-fleet-authentication-openapi.json
operations:
  - POST /token
---

# Authenticate against the bp Open Fleet API

Every bp Open Fleet API requires a bearer token. There is exactly one way to get one.

## Before you start

You need a `client_id` and `client_secret` provisioned through the bp Open Fleet developer
portal. Credentials are bound to **one environment**. BP states this plainly: sandbox
credentials only reach sandbox endpoints, production credentials only production endpoints.
Nothing in the credential itself tells you which environment it belongs to — track it yourself.

| Environment | Host |
|---|---|
| Production | `https://api.fleet.bp.com` |
| Sandbox | `https://api.sandbox.fleet.bp.com` |

## Steps

1. **POST to the token endpoint.**

   `POST https://api.fleet.bp.com/authentication/v1.0/token`

   Send `client_id` and `client_secret` as form data. Both are required.

2. **Read the response.** A `200` returns:

   - `access_token` — the bearer token
   - `token_type`
   - `expires_in` (seconds, int32)
   - `ext_expires_in` (seconds, int32)

3. **Cache the token until it expires.** This matters more here than on most platforms: the
   Authentication API is rate limited to **10 requests per minute**, the tightest limit on the
   platform. Fetching a token per API call will throttle you before anything else does.

4. **Send it on every other call** as `Authorization: Bearer <access_token>`.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| `401` | Invalid client secret | The secret is wrong, or belongs to the other environment. |
| `403` | Invalid client id | The client is not recognised in this environment. |

The error envelope is OAuth-shaped: `error`, `error_description`, `error_uri`, `error_codes[]`,
plus `correlation_id` and `trace_id`. Keep the `correlation_id` — BP's support process asks for it.

## Cautions

- There is **no published revocation endpoint** for Open Fleet tokens. Treat a leaked token as
  valid until `expires_in` elapses.
- BP publishes **no `429` response and no rate-limit headers**. You cannot observe your remaining
  budget; you must self-throttle to the published ceiling.
