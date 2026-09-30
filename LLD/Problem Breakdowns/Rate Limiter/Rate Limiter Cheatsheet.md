# Rate Limiter LLD Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Default Requirements
| Question | Answer |
|---|---|
| Request identity | `(clientId, endpoint)`, a separate quota per client per endpoint |
| Algorithm | Per endpoint, from config (type + params) |
| Return | `RateLimitResult(allowed, remaining, retryAfterMs \| null)` |
| Unconfigured endpoint | Fall back to a default limit, don't reject |
| Config | Loaded once at startup, static |
| Scope | Single process, in memory. No distribution, reload or metrics |

## Entities
| Entity | Responsibility |
|---|---|
| `RateLimiter` | Orchestrator: endpoint → `Limiter`, default fallback, delegates |
| `Limiter` (interface) | `tryAcquire(clientId)`. One per endpoint, holds `Map<clientId, State>` |
| `TokenBucketLimiter` / `SlidingWindowLogLimiter` / `FixedWindowLimiter` | Concrete algorithms, each owning its own state shape |
| `LimiterFactory` | Config → `Limiter`. The only place with the algorithm `switch` |
| `LimiterConfig`, `RateLimitResult` | Value objects (records) |

## Class Diagram (compressed)
```
RateLimiter ──has──> Map<endpoint, Limiter> + defaultLimiter
RateLimiter ──uses──> LimiterFactory (at startup only)
Limiter <|── TokenBucketLimiter | SlidingWindowLogLimiter | FixedWindowLimiter
each Limiter ──has──> Map<clientId, State> + Clock
```

## Key Design Decisions & One-Line Justifications
| Decision | Why |
|---|---|
| Strategy for algorithms | Genuinely different behaviour + state, chosen per endpoint at runtime |
| Factory for creation | Keeps the `switch` out of `RateLimiter`, so a new algorithm doesn't touch the orchestrator (OCP) |
| One `Limiter` per endpoint, per-client map inside | The rule is shared per endpoint; only client state varies |
| Build all limiters at startup | Config is static, so `allow()` is a lookup + delegate, and the map is read-only (thread-safe) |
| `remaining` + `retryAfterMs` on the result | Gateway needs `429` + `Retry-After` / `X-RateLimit-Remaining` headers |
| Inject `Clock` | Deterministic tests with a fake clock (DIP) |
| `RateLimitResult.allow()` / `deny()` factories | Only valid shapes can be built |

## Strategy: Right Here, Wrong in Connect Four (know this cold)
✅ **Rate limiter:** token bucket vs. sliding log vs. fixed window = **different behaviour, different state, runtime choice** → Strategy.
❌ **Connect Four `WinChecker`:** four directions = **same algorithm, different `(dr, dc)` constants** → parameters, not classes.
**The test:** "Different behaviour, or the same behaviour with different numbers?"

## Algorithms
| Algorithm | State/client | Bursts | Accuracy | Use for |
|---|---|---|---|---|
| **Token bucket** | tokens + lastRefill | Up to capacity | Good average | **Default** |
| Sliding window log | Deque of timestamps | No | Exact | Low limits, precision needed |
| Sliding window counter | 2 counters | Smoothed | Approximate | High limits, low memory |
| Fixed window | count + windowStart | **2× at edges** | Weakest | Simplest coarse quotas |

## Token Bucket — Order of Operations
1. `now = clock.millis()`; `computeIfAbsent` bucket (new = full)
2. `synchronized (bucket)`
3. `tokens = min(capacity, tokens + elapsed × rate)`; `lastRefill = now`
4. `tokens ≥ 1` → decrement, `allow(floor(tokens))`
5. else → `deny(ceil((1 − tokens) / ratePerMs))`

## Method Signatures (Java)
```java
record RateLimitResult(boolean allowed, int remaining, Long retryAfterMs) {}
record LimiterConfig(AlgorithmType algorithm, Map<String, Number> params) {}

interface Limiter {
    RateLimitResult tryAcquire(String clientId);
}

class LimiterFactory {
    Limiter create(LimiterConfig config);
}

class RateLimiter {
    RateLimiter(Map<String, LimiterConfig> configs, LimiterConfig defaultConfig, LimiterFactory factory);
    RateLimitResult allow(String clientId, String endpoint);
}
```

## Extensibility Follow-Ups
| Ask | Answer shape |
|---|---|
| New algorithm | New `Limiter` class + one factory `case`. Nothing else changes |
| Thread safety | `computeIfAbsent` + `synchronized` on **that client's** state. Per-client locks, not a global lock |
| Memory growth | Evict idle clients (Caffeine `expireAfterAccess`). An evicted bucket returns full, which is correct |
| Dynamic config | Build a new map and swap it via `AtomicReference`; client state resets (say so) |
| Distributed | Redis: `INCR` + `EXPIRE` (fixed window) or a Lua script (token bucket, atomic). Choose fail-open vs. fail-closed |
| Multiple rules | `CompositeLimiter`: allow only if every child allows |

## Level Expectations
- **Junior:** working token bucket or fixed window, correct allow/deny
- **Mid:** Strategy + Factory unprompted, default fallback, `remaining` + `retryAfterMs`, compares two algorithms
- **Senior:** explains why Strategy fits here and not in Connect Four, injects `Clock`, raises thread safety + memory unprompted, names the fixed-window boundary burst, sketches Redis/Lua + fail-open/closed

⚠️ Don't build distributed limiting, config reload or metrics unless the interviewer asks.
