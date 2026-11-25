# Integration Design: API Design

*HelloInterview "System Design in a Hurry": Core Concepts, API Design. Week 1 of the combined plan: resources, HTTP semantics, error contracts, pagination, idempotency and versioning. In a general system design round the API is a 5-minute step. In SE, Integration and FDE rounds the API is often the whole conversation, because the customer is the one calling it.*

---

## 0. Where This Fits in the Interview

The HLD delivery framework (see `Core Building Blocks.md` section 0) has an API step between core entities and the high-level design. How much weight it carries depends on the role:

| Round type | How much the API matters | What the interviewer is listening for |
|---|---|---|
| General system design | ~5 minutes, then move on | A reasonable set of endpoints; don't gold-plate it |
| Integration design (Integration Engineer, SE) | Most of the round | Idempotency, errors, pagination, versioning, auth, what the customer has to build |
| Customer troubleshooting | The API contract *is* the evidence | Can you read a status code, a header or a payload and explain what went wrong? |
| FDE practical build | You consume someone else's API | Do you page correctly, retry safely and handle errors? |

**The rule for this file:** design the API from the caller's side. Every choice should make the customer's integration easier to get right and harder to get wrong.

---

## 1. Resource Modelling

Resources are nouns; HTTP methods are the verbs.

| Do | Don't |
|---|---|
| `POST /bookings` | `POST /createBooking` |
| `GET /customers/{id}/bookings` | `GET /getBookingsForCustomer?id=` |
| `POST /payments/{id}/capture` | `PATCH /payments/{id}` with `{"action":"capture"}` |

- **Nest only one level deep** when there's a real ownership relationship (`/customers/{id}/bookings`). Beyond that, use top-level resources with filters (`/bookings?customer_id=`).
- **State transitions are sub-resources.** `capture`, `refund` and `cancel` are actions with their own rules and audit trail, so give them their own endpoints rather than hiding them in a generic update.
- **IDs are opaque strings** with a type prefix where you can (`bk_8f3a...`). Callers can't guess them, and support can tell what an ID is at a glance.

---

## 2. HTTP Methods and Idempotency

An operation is **idempotent** if doing it twice leaves the server in the same state as doing it once. This matters because networks fail *after* the server has done the work, and the client can't tell.

| Method | Use for | Safe | Idempotent | Retry blindly? |
|---|---|---|---|---|
| GET | Read | Yes | Yes | Yes |
| PUT | Replace a whole resource at a known URL | No | Yes | Yes |
| DELETE | Remove | No | Yes | Yes (second call returns 404 or 204) |
| PATCH | Partial update | No | Depends (`set x=5` yes, `increment x` no) | Only if you know it's a "set" |
| POST | Create, or trigger an action | No | **No** | **Only with an idempotency key** (section 5) |

---

## 3. Status Codes and the Error Contract

Status codes tell the caller **whether to retry**. That's the most useful thing an error can do for an integrator.

| Code | Meaning | Caller should |
|---|---|---|
| 200 / 201 / 204 | OK / created / no content | Carry on |
| 202 | Accepted, processing async | Poll the status URL or wait for a webhook |
| 400 | Malformed request | Fix the request; don't retry |
| 401 | Not authenticated | Refresh the token, then retry once |
| 403 | Authenticated but not allowed | Don't retry; check permissions or scopes |
| 404 | Not found | Don't retry |
| 409 | Conflict (state clash, or duplicate idempotency key with a different body) | Re-read state, then decide |
| 422 | Valid JSON, but breaks a business rule | Fix the data; don't retry |
| 429 | Rate limited | Wait for `Retry-After`, then retry with backoff |
| 500 / 502 / 503 / 504 | Server or upstream problem | Retry with exponential backoff and jitter, **with an idempotency key** |

**One error shape for every endpoint.** RFC 9457 (Problem Details) is a good default:

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "The card was declined for insufficient funds.",
  "code": "card_declined",
  "request_id": "req_7Hk2..."
}
```

- `code` is a stable, machine-readable string the customer can branch on. Messages change; codes shouldn't.
- `request_id` is the single most useful field for support: the customer quotes it and you find the logs.

---

## 4. Pagination

| | Offset (`?offset=100&limit=50`) | Cursor (`?starting_after=bk_123&limit=50`) |
|---|---|---|
| How it works | Skip N rows | Start after a known item |
| Jump to page 7 | Yes | No |
| Stable when rows are added or removed | **No**: items are skipped or duplicated | Yes |
| Cost on deep pages | Grows with the offset (DB still scans skipped rows) | Constant (index seek) |
| Good for | Admin UIs, small tables | Public APIs, syncs, anything a machine pages through |

**Default to cursor pagination for integration APIs.** A customer syncing data page by page will otherwise miss or double-process records whenever data changes mid-sync.

Response shape (similar to Stripe's list endpoints):

```json
{
  "data": [ { "id": "bk_101" }, { "id": "bk_102" } ],
  "has_more": true,
  "next_cursor": "bk_102"
}
```

- Order by something unique and stable (creation time plus ID), or the cursor is ambiguous.
- Cap `limit` (e.g. max 100) so one caller can't ask for everything.

---

## 5. Idempotency Keys

**Problem:** the client sends `POST /payments`, the server charges the card, and the response is lost in a timeout. The client retries. Without protection, the customer is charged twice.

**Fix:** the client generates a unique key per *logical* operation and sends it on every retry.

```http
POST /payments
Idempotency-Key: 4f1c2a9e-6b2d-4c1e-9f3a-...
```

**Server flow:**

1. Look up the key (scoped to the caller's account).
2. **Not seen** → store the key as "in progress", do the work, store the response, return it.
3. **Seen and complete** → return the stored response. Don't run the operation again.
4. **Seen and in progress** → return 409 so the client backs off and retries later.
5. **Same key, different request body** → reject with 409 or 422. That's a client bug, and silently returning the old result would hide it.

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> create(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody PaymentRequest request,
        @AuthenticationPrincipal Account account) {

    return idempotencyStore.find(account.id(), key)
        .map(saved -> saved.matches(request)
                ? ResponseEntity.status(saved.status()).body(saved.body())
                : ResponseEntity.status(409).<PaymentResponse>build())
        .orElseGet(() -> {
            idempotencyStore.markInProgress(account.id(), key, request); // unique constraint on (account, key)
            PaymentResponse result = paymentService.charge(request);
            idempotencyStore.complete(account.id(), key, 201, result);
            return ResponseEntity.status(201).body(result);
        });
}
```

- The **unique constraint** on `(account_id, key)` is what makes this safe under concurrency, not the `find` check.
- Keep keys for a bounded time (Stripe documents 24 hours), then expire them.
- This is the same idea as the consumer-side dedupe in the async and events notes: **at-least-once delivery plus idempotency** is how real systems get "effectively once".

---

## 6. Versioning

**Breaking changes need a new version. Additive ones don't.**

| Non-breaking (ship any time) | Breaking (needs a version) |
|---|---|
| New endpoint | Removing or renaming a field |
| New optional request field | Making an optional field required |
| New field in a response | Changing a field's type or meaning |
| New enum value *if* clients were told to expect unknown values | Changing error codes or status codes |

| Approach | Example | Trade-off |
|---|---|---|
| URL path | `/v1/bookings` | Obvious and easy to route; big-bang jumps between versions |
| Header | `Api-Version: 2026-11-01` | Clean URLs; easy to forget in a client |
| Account-pinned dated versions (Stripe's model) | Each account is pinned to the version it first used; it can override per request | Customers never break by surprise; you maintain transformation layers between versions |

**Interview line:** "Customers integrate once and then don't touch the code for years, so I'd make breaking changes opt-in. Pin each account to a version, publish a changelog, and give a long deprecation window with warnings in a response header."

---

## 7. The Smaller Things Worth Mentioning

- **Filtering and sorting:** `?status=confirmed&created_after=2026-01-01&sort=-created_at`.
- **Expansion:** `?expand=customer` returns the related object inline, which saves the caller N extra requests.
- **Rate-limit headers:** `RateLimit-Limit`, `RateLimit-Remaining` and `Retry-After` on 429s, so well-behaved clients can slow down before they're blocked.
- **Timestamps:** ISO 8601 in UTC, or Unix seconds. Never a local time without an offset.
- **Money:** integer minor units plus a currency (`"amount": 1999, "currency": "eur"`). Never floats.

---

## 8. REST vs RPC vs GraphQL vs gRPC

| Style | Good for | Watch out for |
|---|---|---|
| REST + JSON | Public and partner APIs; the default for integrations | Over- or under-fetching; lots of round trips for nested data |
| GraphQL | Frontends that need flexible, nested reads | Caching, rate limiting by cost, N+1 resolvers |
| gRPC | Internal service-to-service calls with low latency and strict contracts | Not browser-friendly; harder for customers to debug |
| Webhooks (outbound) | Telling the customer something happened | Delivery, retries and ordering: see the async and events notes |

**Default answer for a customer-facing integration:** REST + JSON for requests, webhooks for events.

---

## 9. The Customer-Facing Angle (SE and Integration Rounds)

The technical answer gets you to "solid". This is what gets you to "strong" in a customer-facing role:

- **Developer experience is part of the design.** Mention docs, a sandbox with test keys, SDKs in the customer's language, and copy-pasteable examples.
- **Say what the customer has to own.** "You'll need to store our booking ID against your order ID, generate an idempotency key per checkout attempt, and handle 429s with backoff."
- **Think about debugging from day one.** Request IDs, a request log in the dashboard, and clear error codes reduce support tickets more than anything else.
- **Bring real experience.** Connect each choice to your work: errors and contracts between DAFM services, and versioning during the WebLogic to OpenShift migration.

---

## 10. Worked Example: Bookings API

**Prompt:** "A partner wants to create and manage bookings on our platform from their own system."

**Clarify first:** How many partners and how much traffic? Do they need to be told about changes we make (e.g. a cancellation by us)? Do they need to sync all bookings, or only the ones they create?

**Endpoints:**

```
POST   /v1/bookings                    create (Idempotency-Key required)
GET    /v1/bookings/{id}               read
GET    /v1/bookings?status=&cursor=    list, cursor-paginated
POST   /v1/bookings/{id}/cancel        state transition (Idempotency-Key required)
```

**Events (webhooks):** `booking.confirmed`, `booking.cancelled`, `booking.updated`.

**Deep dives to offer:** idempotency on create and cancel; cursor pagination for the partner's nightly reconciliation sync; auth with OAuth2 client credentials per partner (Week 2); per-partner rate limits (Week 5).

---

## 11. Level Expectations

- **Mid-level:** sensible resource names and methods; knows the main status codes.
- **Senior:** idempotency keys on POSTs unprompted; cursor pagination with a reason; a consistent error contract; a versioning strategy; names what the customer must build.
- **Staff / strong SE:** designs for the customer's whole lifecycle (onboarding, sandbox, debugging, deprecation) and ties API choices to support load and churn risk.

---

## 12. Suggested Resources

- HelloInterview: [Core Concepts: API Design](https://hellointerview.com/learn/system-design/core-concepts/api-design). This file follows it, with an integration slant.
- HelloInterview: [Delivery Framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction). Where the API step sits.
- Stripe API reference: the pagination, errors, idempotent requests and versioning sections. The best public example of an integration-first API.
- RFC 9457: Problem Details for HTTP APIs. The error shape in section 3.
- Cross-reference: `Core Building Blocks.md` section 0 for the HLD delivery framework.
- Cross-reference: `LLD/Problem Breakdowns/Rate Limiter`, where the algorithms behind 429s live (Week 5).

---

## 13. Self-Test — Multiple Choice

<details>
<summary>Q1: A client's POST /payments times out. The server may or may not have charged the card. What makes a retry safe?</summary>

**A)** Retrying with a longer timeout
**B)** Switching the endpoint to PUT
**C)** Sending the same Idempotency-Key on the retry, so the server returns the stored result if it already did the work
**D)** Calling GET /payments first to check

**Answer: C** — B changes the semantics without solving the problem, and D races with the in-flight request. The idempotency key lets the server detect the retry and replay the original response.
</details>

<details>
<summary>Q2: A partner syncs all bookings nightly with ?offset=&limit=. They report occasional missing and duplicated bookings. What's the cause and fix?</summary>

**A)** Their network is unreliable; add retries
**B)** Rows inserted or deleted during the sync shift the offsets; switch to cursor pagination ordered by a stable unique key
**C)** The page size is too small; raise the limit to 1,000
**D)** The database needs a read replica

**Answer: B** — offset pagination isn't stable while data changes. A cursor anchors each page to a specific item.
</details>

<details>
<summary>Q3: Which change is breaking?</summary>

**A)** Adding a new optional field to a response
**B)** Adding a new endpoint
**C)** Renaming `customer_id` to `client_id` in a response
**D)** Adding a new optional query parameter

**Answer: C** — every existing client reading `customer_id` breaks. The others are additive.
</details>

<details>
<summary>Q4: A request is well-formed JSON, but the booking date is in the past. Best status code?</summary>

**A)** 400
**B)** 404
**C)** 422
**D)** 500

**Answer: C** — the request parsed fine but breaks a business rule. 400 is for malformed requests. (Some APIs use 400 for both; what matters most is a clear, stable error `code`.)
</details>

<details>
<summary>Q5: A client retries with the same Idempotency-Key but a different amount. What should the server do?</summary>

**A)** Process the new amount
**B)** Return the original response silently
**C)** Reject it (409 or 422), because reusing a key with a different body is a client bug
**D)** Refund the first payment and charge the new amount

**Answer: C** — silently replaying hides a real bug, and processing it defeats the purpose of the key.
</details>

<details>
<summary>Q6: What is the main reason to include a request_id in every response?</summary>

**A)** It makes responses cacheable
**B)** It lets the customer quote one value so support can find the exact request in the logs
**C)** It is required by HTTP
**D)** It replaces the idempotency key

**Answer: B** — it's the cheapest way to cut debugging time on both sides of an integration.
</details>

<details>
<summary>Q7: Which approach best avoids surprising existing customers with breaking changes?</summary>

**A)** Ship changes to /v1 and email customers
**B)** Pin each account to the version it integrated against, let it opt in to newer versions, and publish a changelog with a deprecation window
**C)** Never change the API
**D)** Use GraphQL so versioning isn't needed

**Answer: B** — customers rarely revisit integration code, so changes should be opt-in. This is Stripe's model.
</details>
