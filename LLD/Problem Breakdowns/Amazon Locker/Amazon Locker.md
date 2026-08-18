# LLD Problem Breakdown: Amazon Locker

*Free HelloInterview problem. Rated "easy" — but like Connect Four, the "easy" label is about domain familiarity, not about the design decisions being trivial.*

---

## 1. Clarifying Questions to Ask First

Prompt you'd likely get: *"Design a locker system like Amazon Locker where delivery drivers can deposit packages and customers can pick them up using a code."*

Work through the same four themes (core operations, error handling, scope boundaries, future extensions):

| Theme | Question | Typical answer |
|---|---|---|
| Core operations | Different compartment sizes? Exact match or fallback? | Small/medium/large; **exact match only** — reject if no compartment of that size |
| Scope boundary | Full delivery flow, or just locker operations? | Just the locker — assume package already arrived; delivery routing out of scope |
| Notification | Does the system send the code (SMS/email)? | No — **return the code**, notification is someone else's problem |
| Error handling | Wrong code entered repeatedly — lockout? | No lockout logic — just validate, reject if wrong |
| Multiplicity | Multiple packages per customer? Shared codes? | One access token **per package**, 1:1 — no bulk pickup |
| Expiry | How long do codes last? What if never picked up? | **7-day expiry**; package stays physically in compartment until staff manually removes it |
| Capacity | All compartments of a size full? | Return an error — no queueing, no reservations |

**Final requirements to write down:**
```
Requirements:
1. Carrier deposits a package by size (small/medium/large)
   - System assigns matching-size compartment, opens it, returns access token
   - Error if no space
2. One access token per package
3. Customer enters token to retrieve package
   - Valid → opens compartment
   - Invalid/expired → specific error
4. Tokens expire after 7 days
   - Expired token rejected on pickup attempt
   - Package stays in compartment until staff removes it
5. Staff can open all expired compartments to manually retrieve packages
6. Invalid tokens (wrong/expired/already-used) rejected with clear errors

Out of scope: delivery logistics, notification delivery (SMS/email),
lockout after failed attempts, UI, multiple locker stations, payment/pricing
```

**Worth doing explicitly in the interview:** call out the two tradeoffs you deliberately scoped out (lockout logic, notification delivery) rather than silently ignoring them. Naming a feature you considered and chose not to build signals forward-thinking without over-engineering — and gives you something concrete to reach for later if the interviewer asks about extensibility.

---

## 2. Identify Entities — Including the One That *Isn't* One

This problem has a genuinely instructive trap: **Package looks like an obvious entity, and isn't.**

| Candidate | Entity? | Reasoning |
|---|---|---|
| **Package** | ❌ **Not an entity** | Packages are external to this system — Amazon's fulfillment system owns package IDs, shipping info, customer details. Our locker system cares about exactly one thing: the package's *size*. That's an **input parameter** to `depositPackage(size)`, not a class of its own. |
| **Compartment** | ✅ Entity | A real physical thing — has an ID, a size, tracks its own occupancy |
| **Locker** | ✅ Entity (orchestrator) | Something has to scan compartments, find one available, generate a code, tie it all together — the system's entry point |
| **AccessToken** | ✅ Entity | Easy to underrate as "just a string field" — but it's actually a bearer token with an expiration and a specific compartment reference. Worth modeling on its own so it can own its own expiry logic. |

**The general lesson here, not just for this problem:** not every noun in the requirements deserves a class. Ask what the "entity" would actually *do* in your system — if the answer is "nothing, we just need one field from it," it's a parameter or a field on something else, not an entity. This is the same discipline as the entity-filter step in the Delivery Framework (`delivery-framework-notes.md`), applied to a case where the obvious answer is wrong.

| Entity | Responsibility |
|---|---|
| **Locker** | Orchestrator. Owns all compartments + the access-token lookup map. Handles deposit/pickup. |
| **AccessToken** | Bearer token — code, expiration timestamp, reference to the compartment it unlocks. Owns its own expiry check. |
| **Compartment** | Physical slot — ID, size, tracks its own occupancy. |

---

## 3. Class Design

### Locker — deriving state

| Requirement | What Locker must track |
|---|---|
| "System assigns an available compartment of matching size" | The collection of compartments |
| "User retrieves package by entering access token" | A map from token code → `AccessToken`, for fast lookup |

```java
class Locker {
    private List<Compartment> compartments;
    private Map<String, AccessToken> accessTokenMapping;
}
```

**Key design decision — where does "occupied" state live?** Two reasonable options:
- **Physical state on `Compartment`** (the choice made here) — an `occupied` boolean lives directly on the compartment, since physical presence is intrinsic to the compartment itself.
- **Relational state in `Locker`** — an `occupiedCompartments` Set tracked by the orchestrator, treating occupancy as a system-managed relationship rather than a property of the thing itself.

**Both are defensible — what matters is having a rationale, not picking the "correct" one.** (Notably, the Parking Lot problem makes the *opposite* choice — tracking occupancy via a Set in the orchestrator rather than on `ParkingSpot` directly. Worth comparing against `parking-lot-notes.md` once you've done both problems — same underlying decision, reasonably answered two different ways.) A useful general heuristic: **physical state** (contains a package, is broken, needs maintenance) tends to live on the entity, since it describes the entity's own condition; **relational state** (assigned to this token, reserved by this user) tends to live in the orchestrator, since it describes a relationship the system is managing. The heuristic isn't a hard rule — just a reasonable starting point to justify your call either way.

**Deriving Locker's methods:**

| Need | Method |
|---|---|
| "Carrier deposits a package by size" | `depositPackage(size)` — opens compartment, returns token |
| "Customer retrieves package with token" | `pickup(tokenCode)` — opens compartment or throws |
| "Staff opens all expired compartments" | `openExpiredCompartments()` |

```java
class Locker {
    private List<Compartment> compartments;
    private Map<String, AccessToken> accessTokenMapping;

    Locker(List<Compartment> compartments) { /* ... */ }
    String depositPackage(Size size) throws LockerException { return null; }
    void pickup(String tokenCode) throws LockerException { }
    void openExpiredCompartments() { }
}
```

**Two API design decisions worth being able to justify out loud:**
- **`depositPackage` returns only the token code, not the compartment.** The compartment physically opens as a side effect of the call — the driver just walks up and sees which door opened. They don't need the compartment ID returned to them; the token is what gets forwarded to the customer.
- **`pickup` returns `void`.** Same logic — the physical door opening *is* the feedback. No need to return a compartment reference to the caller. If the code is invalid, that's communicated via a thrown error with a specific message instead.

### AccessToken

| Requirement | What AccessToken must track |
|---|---|
| "Access token is generated and returned" | The code string |
| "Tokens expire after 7 days" | An expiration timestamp |
| "System validates code and opens compartment" | A reference to the compartment it unlocks |

```java
class AccessToken {
    private String code;
    private Instant expiration;
    private Compartment compartment;

    AccessToken(String code, Instant expiration, Compartment compartment) { /* ... */ }
    boolean isExpired() { return Instant.now().isAfter(expiration); }
    Compartment getCompartment() { return compartment; }
    String getCode() { return code; }
}
```

Deliberately minimal — expose the data, provide the one bit of derived logic (`isExpired()`) that only `AccessToken` itself has enough information to compute correctly, and let callers decide what to do with that information.

### Compartment

| Requirement | What Compartment must track |
|---|---|
| "Assign an available compartment of matching size" | `size` |

```java
class Compartment {
    private Size size;
    private boolean occupied;

    Compartment(Size size) { this.size = size; }
    Size getSize() { return size; }
    boolean isOccupied() { return occupied; }
    void markOccupied() { occupied = true; }
    void markFree() { occupied = false; }
    void open() { /* triggers physical unlock */ }
}

enum Size { SMALL, MEDIUM, LARGE }
```

**Design principle at work across all three classes: Information Expert** — the class that owns a piece of data is also the class responsible for the logic that uses it. `AccessToken` enforces expiry because it owns the expiration timestamp. `Compartment` manages its own occupancy because it represents the physical thing being occupied. Neither `Locker` nor any other class reaches in to compute these things externally — that's Locker's job kept minimal, and each entity kept self-contained.

---

## 4. Full Class Diagram (Java)

```java
enum Size { SMALL, MEDIUM, LARGE }

class Compartment {
    private Size size;
    private boolean occupied;

    Compartment(Size size) { this.size = size; }
    Size getSize() { return size; }
    boolean isOccupied() { return occupied; }
    void markOccupied() { occupied = true; }
    void markFree() { occupied = false; }
    void open() { /* physical unlock */ }
}

class AccessToken {
    private String code;
    private Instant expiration;
    private Compartment compartment;

    AccessToken(String code, Instant expiration, Compartment compartment) {
        this.code = code;
        this.expiration = expiration;
        this.compartment = compartment;
    }
    boolean isExpired() { return Instant.now().isAfter(expiration); }
    Compartment getCompartment() { return compartment; }
    String getCode() { return code; }
}

class Locker {
    private List<Compartment> compartments;
    private Map<String, AccessToken> accessTokenMapping = new HashMap<>();

    Locker(List<Compartment> compartments) { this.compartments = compartments; }

    String depositPackage(Size size) { /* see Section 5 */ return null; }
    void pickup(String tokenCode) { /* see Section 5 */ }
    void openExpiredCompartments() { /* see Section 5 */ }
}
```

---

## 5. Implementation — the Two Interesting Methods

### `depositPackage` — and the compartment-lookup decision that matters most

**Core logic:** find an available compartment of the requested size → open it → generate a token → mark occupied → store token in the map.
**Edge case:** no compartment of that size available → error.

```java
String depositPackage(Size size) {
    Compartment compartment = getAvailableCompartment(size);
    if (compartment == null) {
        throw new LockerException("No available compartment of size " + size);
    }

    compartment.open();
    compartment.markOccupied();
    AccessToken token = generateAccessToken(compartment);
    accessTokenMapping.put(token.getCode(), token);

    return token.getCode();
}
```

**This is the single most instructive design decision in the whole problem — three approaches to `getAvailableCompartment`, worth understanding all three:**

**❌ Approach 1 — derive availability from the access-token map (don't do this).** Tempting because deriving state instead of storing it usually prevents duplication bugs. But it breaks here for a subtle reason: an *expired* token still has to stay in the map (so `pickup()` can say "expired" instead of "invalid"), but the package is still physically sitting in the compartment. If you derive occupancy purely from "does a token reference this compartment," you can't distinguish "still validly occupied" from "expired token, package still physically there." **Token validity and physical occupancy are genuinely different facts that can diverge — you can't reliably compute one from the other.**

**⚠️ Approach 2 — index available compartments by size in a `Map<Size, Queue<Compartment>>` for O(1) lookup.** Real performance win — dequeue on deposit, enqueue on pickup/removal, all O(1) instead of scanning. The real cost: state now lives in *two* places (the compartments list and the size-indexed queues), which creates a synchronization risk — forget to enqueue on pickup and a compartment silently vanishes from availability forever; enqueue twice and it looks available while still occupied. **Worth it for hundreds/thousands of compartments; overkill for a typical 20-50 compartment locker bank**, where O(n) linear scan and O(1) indexed lookup are practically indistinguishable.

**✅ Approach 3 (the one actually used) — track occupancy directly on `Compartment`, scan linearly.**
```java
private Compartment getAvailableCompartment(Size size) {
    for (Compartment c : compartments) {
        if (c.getSize() == size && !c.isOccupied()) {
            return c;
        }
    }
    return null;
}
```
Single source of truth, no synchronization risk, correct by construction. Simpler wins here specifically because compartment counts in this domain are small — this is KISS in action: don't pay a complexity cost for a performance problem you don't actually have at this scale.

**Interview instinct worth internalizing:** when you spot an "obviously more efficient" data structure, it's worth naming out loud *and* weighing against the added state-synchronization risk — don't just reach for it because it's faster. Say the tradeoff, then pick based on realistic scale for the problem as scoped.

### `pickup`

**Core logic:** look up token by code → check expiry → if valid, open + clean up → if invalid, throw a specific error.
**Edge cases:** token doesn't exist; token exists but expired; empty/null code.

```java
void pickup(String tokenCode) {
    if (tokenCode == null || tokenCode.isEmpty()) {
        throw new LockerException("Invalid access token code");
    }

    AccessToken token = accessTokenMapping.get(tokenCode);
    if (token == null) {
        throw new LockerException("Invalid access token code");
    }
    if (token.isExpired()) {
        throw new LockerException("Access token has expired");
    }

    Compartment compartment = token.getCompartment();
    compartment.open();
    clearDeposit(token);
}

private void clearDeposit(AccessToken token) {
    token.getCompartment().markFree();
    accessTokenMapping.remove(token.getCode());
}
```

**Notice the error-message decision:** a code that *never existed* and a code that was *already used* both get the same "Invalid access token code" message. That's intentional — once a token is used, it's removed from the map, so a second pickup attempt with the same code is indistinguishable from a random invalid code. Distinguishing "already used" from "never existed" would need tracking used codes separately (a `usedTokens` set, or an `isUsed` flag kept before removal) — extra state for marginal UX benefit. **A genuinely reasonable place to draw the scope line, and worth stating as a deliberate choice rather than an oversight if asked.** Expired codes *do* get a specific message ("Access token has expired") because that's actionable — the user knows to contact support rather than assume they mistyped.

### `openExpiredCompartments` (staff-facing)

```java
void openExpiredCompartments() {
    for (AccessToken token : accessTokenMapping.values()) {
        if (token.isExpired()) {
            token.getCompartment().open();
        }
    }
}
```

Deliberately does **not** call `clearDeposit()` — the compartment stays "occupied" in the system's bookkeeping until staff physically remove the package and (out of scope here) call a separate cleanup method. Opening the door and clearing the deposit are two different real-world events; conflating them would let the system think a compartment is free before a human has actually emptied it.

---

## 6. Verification (trace a scenario)

```
Locker: [A(SMALL), B(MEDIUM), C(LARGE)], all occupied=false, map={}

depositPackage(MEDIUM):
  getAvailableCompartment(MEDIUM) → B (matches, not occupied)
  B.open(), B.markOccupied() → B.occupied = true
  generateAccessToken(B) → AccessToken("ABC123", now+7days, B)
  map.put("ABC123", token)
  → returns "ABC123"

pickup("ABC123"), same day:
  map.get("ABC123") → found
  isExpired() → false
  B.open()
  clearDeposit → B.markFree() (occupied=false), map.remove("ABC123")
  → void (door opened)

pickup("ABC123"), 8 days later (if not already removed above):
  map.get("ABC123") → still found (never deleted on expiry, only on successful pickup)
  isExpired() → true
  → throws "Access token has expired"
  → B.occupied stays true, token stays in map — staff must intervene
```

---

## 7. Extensibility — Common Follow-Ups

| Follow-up | Where the change lives | Key idea |
|---|---|---|
| **Size fallback** (small package can use a medium/large compartment if exact size is full) | `getAvailableCompartment` — iterate sizes from requested size upward (`[requestedSize, ..., LARGE]`) instead of exact match only | Fallback order is implicit in the size ordering; never falls back to something *smaller* than requested |
| **Broken/maintenance compartments** | Replace `occupied: boolean` with a `CompartmentStatus` enum (`AVAILABLE`, `OCCUPIED`, `OUT_OF_SERVICE`) | `isAvailable()` becomes `status == AVAILABLE`; allocation logic automatically skips out-of-service compartments with no other changes needed |
| **Verify package is actually deposited before issuing a token** | Split `depositPackage` into `reserveCompartment()` + `confirmDeposit()` (two-phase), add a `RESERVED` status, add reservation timeout handling | Current design is "fire and forget" — door opens, token issued immediately, assumes the driver follows through. Two-phase commit adds real complexity (new state, timeout logic) — reasonable to name as a production concern, but explicitly out of scope for the interview as originally defined |

**Level expectations:**
- **Junior:** identifies the 3 core entities, happy-path deposit/pickup works, aware edge cases exist even if not fully handled
- **Mid:** arrives at the 3-entity model through reasoning (specifically: correctly excludes `Package`) without hand-holding, handles key edge cases (invalid/expired/full), can explain *why* occupancy lives where it does
- **Senior:** proactively rules out `Package` with justification, names Information Expert explicitly, proactively raises tradeoffs (occupancy placement, lazy vs. eager token cleanup, when a `Package` entity *would* become justified — e.g. multiple packages per compartment), anticipates extensibility questions before being asked

---

## 8. Suggested Resources

- HelloInterview — [Amazon Locker](https://www.hellointerview.com/learn/low-level-design/problem-breakdowns/amazon-locker) — free problem breakdown; source for the framework above (original in Python/pseudocode, translated to Java here).
- Cross-reference `parking-lot-notes.md` — the occupancy-placement decision (entity vs. orchestrator) is made the *opposite* way there; comparing the two is a genuinely good exercise once you've done both problems.
- Cross-reference `general-principles-notes.md` — the three-approaches comparison for `getAvailableCompartment` is a clean real-world example of weighing KISS against a genuine performance optimization, not just avoiding complexity reflexively.

---

## 9. Self-Test — Multiple Choice

<details>
<summary>Q1: Why isn't Package modeled as its own entity in this system?</summary>

**A)** Packages are too small to matter
**B)** The locker system only needs one fact about a package — its size — to do its job; everything else about the package (ID, shipping info, customer details) is owned by an external fulfillment system, so size is just an input parameter, not a class
**C)** Java doesn't support modeling physical objects well
**D)** Package would create a circular dependency with Compartment

**Answer: B** — the test for "does this noun deserve to be an entity" is whether it would actually hold meaningful state or behavior *within this system's boundary*. Package fails that test here; Compartment and AccessToken pass it.
</details>

<details>
<summary>Q2: Why can't you reliably derive compartment occupancy purely from "is there an access token referencing it"?</summary>

**A)** Access tokens are stored as strings, not objects, so you can't check references
**B)** Token validity and physical occupancy are genuinely different facts that can diverge — an expired token must stay in the map (so pickup can report "expired" vs "invalid"), but the package is still physically in the compartment, so "has a token" and "is occupied" disagree during that window
**C)** HashMap lookups are too slow for this use case
**D)** It would violate the Single Responsibility Principle

**Answer: B** — this is the core reason the "derive it" approach fails, and it's a good general lesson: derived state only works when the thing you're deriving from is guaranteed to always represent what you think it represents. Here it isn't (expired-but-still-occupied breaks the assumption).
</details>

<details>
<summary>Q3: Why does depositPackage return only the access token code, not the compartment ID?</summary>

**A)** Compartment IDs are internal implementation details, not because of the physical UX
**B)** The compartment door physically opens as a side effect of the call — the driver just sees which door unlocked, so returning a compartment reference would be redundant with what's already visibly happening in front of them
**C)** Returning two values isn't possible in most languages
**D)** Compartment doesn't have a public getId() method

**Answer: B** — this ties back to a broader interview signal: your API design should reflect what information the *caller* actually needs, not everything you technically have available to return.
</details>

<details>
<summary>Q4: When would the indexed-by-size Queue approach (Approach 2) actually be worth its added complexity over the simple linear scan?</summary>

**A)** Never — always prefer the simple approach
**B)** When the number of compartments is in the hundreds/thousands and deposit operations happen constantly enough that O(n) scanning becomes a real bottleneck — not for a typical 20-50 compartment locker bank
**C)** Whenever Queue is available as a data structure
**D)** Only when using a language without built-in Maps

**Answer: B** — this is the KISS/performance tradeoff made concrete: added complexity should be justified by an actual, scale-appropriate need, not adopted preemptively because it's theoretically faster.
</details>

<details>
<summary>Q5: Why do both "code never existed" and "code already used" return the identical "Invalid access token code" error?</summary>

**A)** It's a bug in the design that should be fixed
**B)** Once a token is used, it's removed from the map entirely — so a reused code and a code that was never generated are, from the system's point of view, indistinguishable without adding extra state (like a separate "used tokens" set) purely for marginal UX benefit
**C)** Java can't differentiate between these two error cases
**D)** Security requirements mandate identical error messages for all invalid codes

**Answer: B** — this is a deliberate, defensible scope decision (unlike expired codes, which *do* get a distinct message because it's actionable information), not an oversight — worth being able to state as a conscious tradeoff if an interviewer probes it.
</details>
