# MiniInvest – API guidelines

Rules for the HTTP APIs of MiniInvest. They apply to the public API of invest-core ([`contracts/invest-core/openapi.yaml`](../contracts/invest-core/openapi.yaml)) and to every new endpoint. When an endpoint has to break a rule, the reason is recorded in an ADR in [`docs/adr`](adr/).

The main client is an Android app that stays in users' phones for months and cannot be forced to update. Most rules below follow from that.

## 1. Contract first

- The contract is the source of truth. It is written in OpenAPI 3.1, lives in `contracts/<service>/openapi.yaml` and is owned by the backend.
- A change to the contract is reviewed before it is implemented, together with the mobile side.
- The contract must pass `npx @redocly/cli lint contracts/<service>/openapi.yaml`. Detection of breaking changes against `main` (oasdiff) is added to CI in module 8.
- Every response has an example in the contract, so a mock server ([Prism](https://github.com/stoplightio/prism)) can stand in for the backend.

## 2. Resources and methods

- Paths are plural nouns under a major version: `/api/v1/orders`, `/api/v1/orders/{orderId}`. A resource that exists once per user has no ID in the path: `/api/v1/portfolio`.
- IDs are UUIDs. Clients treat them as opaque strings.
- **The user, the account, the market and prices never come from the request.** They come from the access token, the account and the server's own sources. A request carries only what the user decided.

| Method | Use for | Safe | Idempotent |
|---|---|---|---|
| `GET` | Reading; never changes state | yes | yes |
| `POST` | Creating a resource whose ID the server assigns; commands that are not CRUD | no | no, unless it takes an `Idempotency-Key` |
| `PUT` | Replacing a whole resource at a URL the client knows | no | yes |
| `PATCH` | Partial change, `application/merge-patch+json` | no | not guaranteed |
| `DELETE` | Removing a resource | no | yes |

- A command that is not create, read, update or delete is a `POST` to a sub-resource, for example `POST /api/v1/orders/{orderId}/cancellation`. No verbs in paths (`/orders/42/update`).
- `GET` and `DELETE` have no request body.
- Work that finishes later returns `202 Accepted` with a `Location` header pointing to a resource whose state the client can read.

## 3. Status codes

| Code | When | What the client does |
|---|---|---|
| `200 OK` | Success with a body | Uses the body |
| `201 Created` | A resource was created synchronously; `Location` points to it | Uses the body |
| `202 Accepted` | Accepted, processing continues; `Location` points to the state | Shows the pending state and follows the resource |
| `204 No Content` | Success without a body | Nothing |
| `400 Bad Request` | The request is malformed or fails shape validation | Does not retry unchanged |
| `401 Unauthorized` | Missing, invalid or expired token; carries `WWW-Authenticate` | Refreshes the token and retries |
| `403 Forbidden` | The user is known but not allowed to do this | Does not retry |
| `404 Not Found` | The resource does not exist **or belongs to another user** | Does not retry |
| `409 Conflict` | Conflict with the current state, for example a request with the same `Idempotency-Key` in progress | Waits and retries with the same key |
| `412 Precondition Failed` | `If-Match` no longer matches | Reloads the resource and resolves the conflict |
| `422 Unprocessable Content` | The request is valid but breaks a business rule | Shows the error by `code` |
| `428 Precondition Required` | A conditional write was sent without `If-Match` | Sends the request with `If-Match` |
| `429 Too Many Requests` | Rate limit exceeded; carries `Retry-After` | Waits and retries |
| `500 Internal Server Error` | Unexpected server error | Retries with the same `Idempotency-Key`, then gives up |
| `503 Service Unavailable` | A dependency is temporarily down; carries `Retry-After` | Waits and retries with the same key |

**400 or 422.** `400` means the request itself is wrong: a missing field, a wrong type, a negative quantity, an invalid cursor. `422` means the request is fine but not allowed now: insufficient funds, an instrument not offered in the user's market, a quantity off the instrument's step. Business checks run after the idempotency check (see section 5).

**Errors are never `200`.** A failure always has a 4xx or 5xx status.

## 4. Errors

Every error is `application/problem+json` as defined in [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457):

```json
{
  "type": "https://api.miniinvest.example/problems/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Available cash is 1200.00 CZK, the order needs 2650.00 CZK.",
  "instance": "/api/v1/orders",
  "code": "INSUFFICIENT_FUNDS",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

- `type`, `title`, `status` and `code` are always present. `code` is the stable machine-readable identifier; `type` and `code` change together.
- Clients decide by `code`, never by `detail`. `detail` is for humans and may change at any time; its language follows `Accept-Language`.
- **Codes are an open set.** The app maps known codes to texts in the user's language and shows a generic message by `status` for unknown ones. Adding a code is a compatible change.
- Validation errors list the invalid fields in `errors`, each with a JSON Pointer as a URI fragment: `{ "pointer": "#/quantity", "detail": "must be greater than 0" }`.
- No internals: no stack traces, SQL, class names or hostnames. `traceId` links the error to the logs.
- Errors produced by the framework (unknown path, unsupported media type) use the same format.

Current codes:

| Code | Status | Meaning |
|---|---|---|
| `VALIDATION_FAILED` | 400 | Body or parameters are not valid; see `errors` |
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | `Idempotency-Key` is missing or not a UUID |
| `INVALID_CURSOR` | 400 | The pagination cursor is not valid |
| `UNAUTHORIZED` | 401 | Missing, invalid or expired access token |
| `ORDER_NOT_FOUND` | 404 | The order does not exist or belongs to another user |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | 409 | A request with the same key is still being processed |
| `IDEMPOTENCY_KEY_REUSED` | 422 | The key was used for a different request |
| `INSUFFICIENT_FUNDS` | 422 | Available cash does not cover the reservation |
| `INSTRUMENT_NOT_AVAILABLE` | 422 | The instrument is not offered in the user's market |
| `QUANTITY_NOT_ALLOWED` | 422 | The quantity is below the minimum or not a multiple of the step |
| `INTERNAL_ERROR` | 500 | Unexpected server error |
| `PRICE_UNAVAILABLE` | 503 | The price source does not respond |

The contract (`Problem.code`) is the authoritative list; this table is kept in sync with it.

## 5. Idempotency

Every `POST` whose effect must not happen twice requires an `Idempotency-Key` header. Today that is `POST /api/v1/orders`.

- The key is a UUID generated by the client **once per user intent**, for example when the user confirms a purchase, and reused for every retry of that intent. A new intent gets a new key, even with the same body.
- The server stores the key per user (unique on `(owner_id, key)`) together with a fingerprint of the parsed request and the response, **in the same database transaction as the effect**. The fingerprint uses normalized values, so `"5"` and `"5.0"` are the same quantity.
- Only successful results are stored. A request that fails with 4xx or 5xx stores nothing and a retry is evaluated again.

| Situation | Response |
|---|---|
| Key missing or not a UUID | `400`, `IDEMPOTENCY_KEY_REQUIRED` |
| New key | The request is processed and its response stored |
| Known key, same body, original finished | The stored response again; nothing is executed |
| Known key, same body, original still in progress | The stored response once the original finishes, or `409`, `IDEMPOTENCY_KEY_IN_PROGRESS` |
| Known key, different body | `422`, `IDEMPOTENCY_KEY_REUSED` |

- Keys expire **72 hours** after first use. After that, a request with the same key is a new request.
- The idempotency check runs before business checks, so a retry of an accepted request always gets its original response, even if the data it was checked against changed in between.
- Clients persist the key of an unfinished intent (for example in Room or DataStore) so a retry after the app was killed uses the same key.

## 6. Pagination

Lists that grow over time (order history, events) use **cursor pagination**:

- Query parameters `limit` (default 20, maximum 100) and `cursor`. The first page has no `cursor`.
- The response is `{ "items": [...], "nextCursor": "..." }`; `nextCursor` is `null` on the last page.
- The order is deterministic and includes a unique column, for example `created_at DESC, id DESC`.
- The cursor is opaque to the client. Inside it is Base64url-encoded JSON with the sort values of the last item; the server may change its content at any time. A tampered cursor is rejected with `400`, `INVALID_CURSOR`, and never bypasses authorization.
- Filters are part of the query; changing a filter starts again from the first page.
- No total count unless a screen really needs it.

Short lists that do not grow (instruments of a market) return the same envelope `{ "items": [...] }` without paging, so fields can be added later.

## 7. Data formats

- JSON with `camelCase` field names.
- **Response fields are always present; a field without a value is `null`.** The only exception is `Problem`, which follows RFC 9457 and omits empty members.
- **Money** is `{ "amount": "2650.00", "currency": "CZK" }`: a decimal string and an [ISO 4217](https://www.iso.org/iso-4217-currency-codes.html) code. Never a JSON number, never without a currency, no implicit default currency. Amounts have as many decimal places as the currency; unit prices may have more. The server uses `BigDecimal` (compared with `compareTo`, not `equals`) and `NUMERIC` in the database.
- **Quantities** are decimal strings with at most 6 decimal places, for example `"2.5"`. Each instrument states `minQuantity` and `quantityStep`; a step of `"1"` means whole units only. See [ADR 0001](adr/0001-order-quantity-format.md).
- **Time** is an RFC 3339 date-time in UTC (`2026-10-31T22:27:00Z`); a date without time is `2026-11-03`.
- **Codes** use standards: markets as ISO 3166-1 alpha-2 (`CZ`, `SK`), currencies as ISO 4217, languages as BCP 47 (`cs-CZ`), instruments by ISIN.
- Values are never formatted for display; the client formats them for the user's locale.

## 8. Markets and language

- The market is a property of the account. It is never a request parameter or header.
- The contract has the same shape for every market; only data differs. A new market is server-side data and configuration: its currency, instrument offer and rules, including whether fractional shares are allowed.
- Differences in features are expressed as data in responses (limits, steps, flags), not as different response shapes.
- `Accept-Language` only selects the language of texts the server sends (`Problem.detail`, instrument names). It never changes business logic.

## 9. Compatibility and versioning

Within a major version the contract only changes **additively**.

| Compatible | Breaking |
|---|---|
| New endpoint | Removing or renaming a field |
| New field in a response | Changing a field's type or meaning |
| New optional field or parameter in a request | New required field or parameter in a request |
| New value of an open set (status, code, market) | Making a response field nullable that was not |
| Looser validation | Stricter validation |
| | Different success status, for example `200` → `202` |

- **Open sets are not `enum`s.** Values that can grow (order status, order side, rejection codes, error codes, market codes) are `type: string` with a description that starts with "Extensible" and lists the known values. Generated clients then do not fail on a new value.
- **Clients are tolerant readers.** The Android app ignores unknown fields and maps unknown values to a fallback (kotlinx.serialization with `ignoreUnknownKeys = true`, `coerceInputValues = true` and an `UNKNOWN` enum value).
- A breaking change that cannot be avoided gets a new major version (`/api/v2`) that runs next to the old one until old app versions disappear. The old version announces its end with the `Deprecation` and `Sunset` headers. The app sends its version, so the backend can measure who still calls the old API.

## 10. Checklist for a new endpoint

- [ ] The contract is changed first and reviewed with the mobile side.
- [ ] The method matches the semantics in section 2; reads are `GET` and change nothing.
- [ ] No user ID, account ID, market or price is taken from the request.
- [ ] Every error response is `application/problem+json` with the shared `Problem` schema; new codes are added to section 4 and to the contract.
- [ ] A `POST` with an effect that must not repeat requires `Idempotency-Key`.
- [ ] Lists that grow use cursor pagination with a limit and a maximum.
- [ ] Money is `Money`, quantities are decimal strings, time is UTC.
- [ ] Values that can grow are open sets, not `enum`s.
- [ ] Every response has an example.
- [ ] `redocly lint` passes and the change is additive, or it comes with a new major version and an ADR.
