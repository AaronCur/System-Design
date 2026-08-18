# Amazon Locker LLD Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Default Requirements
| Question | Answer |
|---|---|
| Compartment sizes | Small/Medium/Large — **exact match only**, no fallback (baseline; fallback is an extension) |
| Scope | Locker operations only — package already arrived, delivery logistics out of scope |
| Notification | System **returns the code only** — SMS/email delivery is downstream, out of scope |
| Lockout on wrong code | **No** — just validate, reject on wrong code |
| Tokens per package | 1:1, no bulk pickup |
| Expiry | **7 days**; package physically stays until staff manually removes it |
| Full compartment bank | Return error — no queueing/reservations |

## Entities — Including the Trap
| Candidate | Entity? | Why |
|---|---|---|
| **Package** | ❌ | System only needs its *size* — that's a parameter, not a class. Package details live in an external system. |
| `Compartment` | ✅ | Physical slot — ID, size, own occupancy |
| `Locker` | ✅ | Orchestrator — compartments + token map |
| `AccessToken` | ✅ | Bearer token — code, expiry, compartment ref. Not "just a string." |

## Class Diagram (compressed)
```
Locker ──has──> Compartment[]
Locker ──has──> Map<code, AccessToken>
AccessToken ──refs──> Compartment
```

## Key Design Decisions & One-Line Justifications
| Decision | Why |
|---|---|
| Package excluded as an entity | Only its size matters to this system — everything else is external |
| `occupied` boolean lives on `Compartment` (not a Set in `Locker`) | Physical state → lives on the entity. (Parking Lot makes the opposite call — both are valid, know your rationale.) |
| `depositPackage` returns only the token code | Door opens as a visible side effect — driver doesn't need a compartment ID back |
| `pickup` returns `void` | Door opening IS the feedback; errors thrown with specific messages instead |
| Same "Invalid access token code" for expired-and-removed vs. never-existed | Once used, token is removed from the map — indistinguishable without extra state for marginal UX gain |
| Expired codes get a distinct error message | Actionable — tells user to contact support, unlike a generic invalid code |
| `openExpiredCompartments()` doesn't clean up state | Opening ≠ physically emptied — cleanup happens after staff manually confirm removal |

## getAvailableCompartment — Three Approaches (know all three)
| Approach | Verdict | Why |
|---|---|---|
| Derive from access-token map ("has a token = occupied") | ❌ | Expired tokens must stay mapped (for error messaging) but package is still physically present — token validity ≠ physical occupancy, can't derive one from the other |
| Index by size: `Map<Size, Queue<Compartment>>`, O(1) | ⚠️ Situational | Real perf win, but state lives in 2 places → sync risk (double-enqueue, forgot-to-enqueue bugs). Worth it only at hundreds/thousands of compartments. |
| **Linear scan + `occupied` flag on Compartment** | ✅ Chosen | Single source of truth, correct by construction. O(n) is fine at typical 20-50 compartment scale — KISS wins here. |

## Method Signatures (Java)
```java
class Locker {
    String depositPackage(Size size);       // throws if no compartment available
    void pickup(String tokenCode);          // throws if invalid/expired
    void openExpiredCompartments();         // staff-facing
}

class AccessToken {
    boolean isExpired();
    Compartment getCompartment();
    String getCode();
}

class Compartment {
    Size getSize();
    boolean isOccupied();
    void markOccupied();
    void markFree();
    void open();
}
```

## depositPackage — Order of Operations
1. `getAvailableCompartment(size)` → error if null
2. `compartment.open()`
3. `compartment.markOccupied()`
4. Generate `AccessToken` (code + now+7days + compartment ref)
5. Store in map, return code

## pickup — Order of Operations
1. Reject null/empty code
2. Look up token → reject if not found ("Invalid access token code")
3. Check `isExpired()` → reject if true ("Access token has expired")
4. Open compartment, `clearDeposit()` (mark free + remove from map)

## Extensibility Follow-Ups
| Ask | Answer shape |
|---|---|
| Size fallback | Iterate sizes upward from requested (`[requested...LARGE]`) — never fall back smaller |
| Broken/maintenance compartments | Replace `occupied: boolean` with `CompartmentStatus` enum (AVAILABLE/OCCUPIED/OUT_OF_SERVICE) |
| Verify package actually deposited | Split into `reserveCompartment()` + `confirmDeposit()` (two-phase), add RESERVED status + timeout — flag as added complexity, out of interview scope by default |

## Level Expectations
- **Junior:** 3 entities identified, happy-path works, aware edge cases exist
- **Mid:** excludes Package through reasoning (not told), explains occupancy placement choice, handles key edge cases unprompted
- **Senior:** names Information Expert explicitly, proactively discusses occupancy tradeoffs + cleanup timing + when Package *would* become justified (e.g. multi-package-per-compartment), raises extensibility unprompted

## Cross-Reference
Compare occupancy placement (`Compartment.occupied` here vs. a Set-based approach in `parking-lot-notes.md`) — same underlying decision, two valid answers, good exercise once you've done both problems.
