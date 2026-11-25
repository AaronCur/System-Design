# API Design Cheat Sheet

Quick-reference version of `API Design.md` — for interview warm-ups, not first-time learning.

## The One Rule
Design from the caller's side: every choice should make the customer's integration easier to get right and harder to get wrong.

## Resources
| Do | Don't |
|---|---|
| `POST /bookings` | `POST /createBooking` |
| `GET /customers/{id}/bookings` (one level of nesting) | Deep nesting beyond one level |
| `POST /payments/{id}/capture` (actions as sub-resources) | Generic `PATCH` with an `action` field |

Opaque, prefixed IDs: `bk_8f3a...`

## Methods
| Method | Idempotent | Retry blindly? |
|---|---|---|
| GET / PUT / DELETE | Yes | Yes |
| PATCH | Depends ("set" yes, "increment" no) | Only if it's a "set" |
| POST | No | Only with an Idempotency-Key |

## Status Codes → What the Caller Does
| Code | Do |
|---|---|
| 400 / 403 / 404 / 422 | Don't retry; fix the request or the data |
| 401 | Refresh the token, retry once |
| 409 | Re-read state, then decide |
| 429 | Wait for `Retry-After`, back off |
| 5xx | Retry with backoff + jitter + idempotency key |
| 202 | Poll the status URL or wait for a webhook |

**Errors:** one shape everywhere (RFC 9457): `type`, `title`, `status`, `detail`, stable `code`, `request_id`.

## Pagination
| | Offset | Cursor |
|---|---|---|
| Stable while data changes | No | **Yes** |
| Deep pages | Slow | Fast |
| Use for | Admin UIs | Public APIs, syncs |

Response: `data`, `has_more`, `next_cursor`. Order by a stable unique key. Cap `limit`.

## Idempotency Keys
1. Client sends `Idempotency-Key` per logical operation, same key on every retry.
2. Server: new → do it and store the response; seen + done → replay; in progress → 409; same key + different body → reject.
3. Unique constraint on `(account_id, key)`. Expire after ~24h.

**At-least-once + idempotency = effectively once.**

## Versioning
- Additive = safe: new endpoints, optional fields, new response fields.
- Breaking = new version: removing or renaming fields, new required fields, changed types or codes.
- Best for customers: pin each account to a version, opt-in upgrades, changelog, long deprecation window (Stripe's model).

## Style
| Style | Default for |
|---|---|
| REST + JSON | Public and partner APIs |
| Webhooks | Telling the customer something happened |
| gRPC | Internal service-to-service |
| GraphQL | Flexible frontend reads |

## Also Mention
Filtering and sorting · `expand` · rate-limit headers · UTC timestamps · money as integer minor units + currency.

## Customer Angle (what turns "solid" into "strong")
- Docs, sandbox, SDKs, examples.
- Say what the customer has to build and own.
- Request IDs and clear error codes cut support load.
