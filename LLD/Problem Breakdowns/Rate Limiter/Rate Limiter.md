# LLD Problem Breakdown: Rate Limiter

*HelloInterview problem, and the Week 4 LLD deliverable. It pairs with this week's HLD notes (`HLD/Core Concepts/Core Building Blocks.md`): this is the in-memory, single-process version, and Redis is the distributed follow-up.*

---

## 1. Clarifying Questions to Ask First

Prompt you'd likely get: *"Build an in-memory rate limiter for an API gateway. Each endpoint has its own rule: which algorithm to use and its parameters. For each incoming request, decide whether to allow it."*

Work through the same four themes (core operations, error handling, scope boundaries, future extensions):

| Theme | Question | Typical answer |
|---|---|---|
| Core operations | What identifies a request? | `(clientId, endpoint)`, so each client gets its own quota per endpoint |
| Core operations | Which algorithms? One global choice or per endpoint? | **Per endpoint**, each with its own parameters (e.g. token bucket: `capacity`, `refillRatePerSecond`) |
| Core operations | What does the caller get back? | Not just a boolean: `allowed`, `remaining`, and `retryAfterMs` (null if allowed) |
| Error handling | Endpoint with no config? | **Fall back to a default limit**, don't reject |
| Scope boundary | Single process or distributed? | Single process, in memory. Distributed is out of scope |
| Scope boundary | Config changes at runtime? | No. It's loaded once at startup and static |
| Future extensions | Thread safety? | "Don't worry to start," but expect it as a follow-up (Section 7) |

**Final requirements to write down:**
```
Requirements:
1. Configuration is loaded once at startup (static)
2. A request arrives as (clientId, endpoint)
3. Each endpoint has a config: algorithm type + algorithm parameters
4. Return RateLimitResult(allowed, remaining, retryAfterMs | null)
5. Endpoints with no config use a default limit

Out of scope: distributed limiting, dynamic config reload, monitoring/metrics
```

**Why the return type matters as a question:** a boolean is enough to block a request, but an API gateway also needs to send `429 Too Many Requests` with `Retry-After` and `X-RateLimit-Remaining` headers. Asking up front changes the whole interface. `remaining` and `retryAfterMs` have to be computed by *each* algorithm, so they go on the `Limiter` contract, not in the orchestrator.

---

## 2. Identify Entities

| Entity | Responsibility |
|---|---|
| **RateLimiter** | Orchestrator and public entry point. Maps endpoint → `Limiter`, falls back to the default, delegates the decision. Knows nothing about any algorithm. |
| **Limiter** (interface) | The contract every algorithm implements: `RateLimitResult tryAcquire(String clientId)`. One instance per endpoint, holding **per-client state** inside it. |
| **TokenBucketLimiter / SlidingWindowLogLimiter / FixedWindowLimiter** | Concrete algorithms. Each owns its own per-client state type and maths. |
| **LimiterFactory** | Turns a config (`algorithm` + params) into the right `Limiter`. The one place that knows every algorithm by name. |
| **LimiterConfig** | Value object: the algorithm type plus its parameters. |
| **RateLimitResult** | Value object: `allowed`, `remaining`, `retryAfterMs`. |

**The design question worth pausing on: where does per-client state live?** You could keep one limiter per `(endpoint, client)` pair in a map inside `RateLimiter`. The cleaner choice is **one `Limiter` per endpoint, with a `Map<clientId, State>` inside it**. The endpoint's rule (capacity, window) is shared by every client, so it belongs to the limiter once. Each client's changing state (tokens, timestamps) is what varies, so that's what the map holds. `RateLimiter` then only deals with endpoints, and each algorithm fully owns its own state shape.

---

## 3. Class Design

### Why Strategy is *right* here (contrast with Connect Four)

In `Connect 4.md`, a `WinChecker` interface with four implementations was over-engineering: the four directions were the same algorithm with different numbers. **Here it's the opposite.** Token bucket, sliding window log and fixed window are **genuinely different algorithms** with different state (a token count vs. a deque of timestamps vs. a counter), and the choice is made **per endpoint from config at runtime**. That's the textbook case for Strategy.

**The test to say out loud:** *"Is the variation different behaviour, or the same behaviour with different constants?"* Different behaviour → Strategy. Different constants → parameters.

### Factory: keeping `RateLimiter` closed for modification

Without a factory, `RateLimiter`'s constructor would need a `switch` on algorithm type, and adding a new algorithm would mean editing the orchestrator. `LimiterFactory` isolates that `switch` in one place. A new algorithm then means one new class plus one case in the factory, and `RateLimiter` doesn't change (Open/Closed).

### RateLimitResult: immutable value object

```java
record RateLimitResult(boolean allowed, int remaining, Long retryAfterMs) {
    static RateLimitResult allow(int remaining) { return new RateLimitResult(true, remaining, null); }
    static RateLimitResult deny(long retryAfterMs) { return new RateLimitResult(false, 0, retryAfterMs); }
}
```

The static factories make the two valid shapes explicit, so callers can't build `allowed=true` with a `retryAfterMs`. It's the same "make invalid states unrepresentable" idea as `GameState` in Connect Four, within what Java allows.

### Injecting time: the testability decision

Every algorithm depends on "now." Calling `System.currentTimeMillis()` directly makes the code impossible to unit test without `Thread.sleep()`. **Inject a `java.time.Clock`** (or a `LongSupplier`) instead. Tests pass a fake clock and move it forward by exact amounts. This is Dependency Inversion in practice, and interviewers notice it.

---

## 4. Full Class Diagram (Java)

```java
enum AlgorithmType { TOKEN_BUCKET, SLIDING_WINDOW_LOG, FIXED_WINDOW }

record LimiterConfig(AlgorithmType algorithm, Map<String, Number> params) {}

record RateLimitResult(boolean allowed, int remaining, Long retryAfterMs) { /* allow()/deny() as above */ }

interface Limiter {
    RateLimitResult tryAcquire(String clientId);
}

class TokenBucketLimiter implements Limiter { /* Section 5 */ }
class SlidingWindowLogLimiter implements Limiter { /* Section 5 */ }
class FixedWindowLimiter implements Limiter { /* Section 5 */ }

class LimiterFactory {
    private final Clock clock;

    LimiterFactory(Clock clock) { this.clock = clock; }

    Limiter create(LimiterConfig config) {
        Map<String, Number> p = config.params();
        return switch (config.algorithm()) {
            case TOKEN_BUCKET -> new TokenBucketLimiter(
                    p.get("capacity").intValue(), p.get("refillRatePerSecond").doubleValue(), clock);
            case SLIDING_WINDOW_LOG -> new SlidingWindowLogLimiter(
                    p.get("maxRequests").intValue(), p.get("windowMs").longValue(), clock);
            case FIXED_WINDOW -> new FixedWindowLimiter(
                    p.get("maxRequests").intValue(), p.get("windowMs").longValue(), clock);
        };
    }
}

class RateLimiter {
    private final Map<String, Limiter> limitersByEndpoint;
    private final Limiter defaultLimiter;

    RateLimiter(Map<String, LimiterConfig> configs, LimiterConfig defaultConfig, LimiterFactory factory) {
        this.limitersByEndpoint = new HashMap<>();
        configs.forEach((endpoint, cfg) -> limitersByEndpoint.put(endpoint, factory.create(cfg)));
        this.defaultLimiter = factory.create(defaultConfig);
    }

    RateLimitResult allow(String clientId, String endpoint) {
        Limiter limiter = limitersByEndpoint.getOrDefault(endpoint, defaultLimiter);
        return limiter.tryAcquire(clientId);
    }
}
```

**Note:** all limiters are built **once at startup** (the config is static), so `allow()` is just a map lookup plus a delegation. The map is never written to after construction, which also matters for thread safety later.

**Note:** the default limiter is shared across all unconfigured endpoints, so a client's calls to `/a` and `/b` (both unconfigured) draw from the same quota. That's usually acceptable. If the interviewer wants a separate default quota per endpoint, create a default limiter lazily per endpoint instead. It's worth stating this as an explicit choice.

---

## 5. Implementation — the Three Algorithms

For each one: the per-client state it needs, then `tryAcquire`.

### Token bucket (the default answer)

**Idea:** each client has a bucket of up to `capacity` tokens that refills continuously at `refillRatePerSecond`. Each request spends one token; if none are left, the request is denied. This **allows bursts up to `capacity`** while enforcing the long-run average rate, which is why most API gateways use it.

**Key trick: lazy refill.** No background thread tops up buckets. On each request, work out how much time has passed since the last refill and add that many tokens (capped at capacity).

```java
class TokenBucketLimiter implements Limiter {
    private final int capacity;
    private final double refillPerMs;
    private final Clock clock;
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private static final class Bucket {
        double tokens;
        long lastRefillMs;
        Bucket(double tokens, long now) { this.tokens = tokens; this.lastRefillMs = now; }
    }

    TokenBucketLimiter(int capacity, double refillRatePerSecond, Clock clock) {
        this.capacity = capacity;
        this.refillPerMs = refillRatePerSecond / 1000.0;
        this.clock = clock;
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        long now = clock.millis();
        Bucket b = buckets.computeIfAbsent(clientId, id -> new Bucket(capacity, now)); // new clients start full

        synchronized (b) {                                   // per-client lock (Section 7)
            b.tokens = Math.min(capacity, b.tokens + (now - b.lastRefillMs) * refillPerMs);
            b.lastRefillMs = now;

            if (b.tokens >= 1) {
                b.tokens -= 1;
                return RateLimitResult.allow((int) b.tokens);
            }
            long retryAfter = (long) Math.ceil((1 - b.tokens) / refillPerMs); // time until 1 full token
            return RateLimitResult.deny(retryAfter);
        }
    }
}
```

**Edge cases:** new client → starts with a full bucket (allowing the initial burst is the point); long idle → `Math.min` caps the refill at capacity; `tokens` is a `double` so fractional refill between requests isn't lost.

### Sliding window log (exact, memory-heavy)

**Idea:** keep a timestamp for every accepted request in the last `windowMs`. Allow the request if fewer than `maxRequests` timestamps remain after dropping expired ones. It's **exact**: no bursts at window edges.

```java
class SlidingWindowLogLimiter implements Limiter {
    private final int maxRequests;
    private final long windowMs;
    private final Clock clock;
    private final Map<String, Deque<Long>> logs = new ConcurrentHashMap<>();

    SlidingWindowLogLimiter(int maxRequests, long windowMs, Clock clock) {
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
        this.clock = clock;
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        long now = clock.millis();
        Deque<Long> log = logs.computeIfAbsent(clientId, id -> new ArrayDeque<>());

        synchronized (log) {
            while (!log.isEmpty() && log.peekFirst() <= now - windowMs) {
                log.pollFirst();                              // evict expired timestamps
            }
            if (log.size() < maxRequests) {
                log.addLast(now);
                return RateLimitResult.allow(maxRequests - log.size());
            }
            return RateLimitResult.deny(log.peekFirst() + windowMs - now); // when the oldest expires
        }
    }
}
```

**Trade-off:** O(maxRequests) memory **per client**. That's fine for 10 requests/minute, but painful for 10,000 requests/hour across millions of clients.

### Fixed window counter (simplest, has a boundary flaw)

**Idea:** divide time into fixed windows (e.g. each minute) and count requests per client per window. Once the count reaches `maxRequests`, deny until the next window starts.

```java
class FixedWindowLimiter implements Limiter {
    private final int maxRequests;
    private final long windowMs;
    private final Clock clock;
    private final Map<String, Window> windows = new ConcurrentHashMap<>();

    private static final class Window { long start; int count; }

    FixedWindowLimiter(int maxRequests, long windowMs, Clock clock) {
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
        this.clock = clock;
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        long now = clock.millis();
        long currentStart = now - (now % windowMs);
        Window w = windows.computeIfAbsent(clientId, id -> new Window());

        synchronized (w) {
            if (w.start != currentStart) { w.start = currentStart; w.count = 0; } // new window
            if (w.count < maxRequests) {
                w.count++;
                return RateLimitResult.allow(maxRequests - w.count);
            }
            return RateLimitResult.deny(w.start + windowMs - now);
        }
    }
}
```

**The flaw to name:** with a limit of 100/minute, a client can send 100 requests at 00:59 and another 100 at 01:00, which is **200 in two seconds**. Sliding window log (exact) or a *sliding window counter* (weight the previous window's count by how much of it still overlaps) fixes this.

### Algorithm comparison

| Algorithm | State per client | Bursts | Accuracy | Use when |
|---|---|---|---|---|
| **Token bucket** | 2 numbers | Allowed up to capacity | Good long-run average | **Default**: API gateways, bursty clients |
| Sliding window log | Up to N timestamps | No | Exact | Low limits where precision matters |
| Sliding window counter | 2 counters | Smoothed | Close approximation | High limits, memory-constrained |
| Fixed window | 1 counter + start | 2× at window edges | Weakest | Simplest possible; coarse quotas |
| Leaky bucket | Queue / level | No, output is smoothed | Good | Shaping traffic to a constant outflow rate |

---

## 6. Verification (trace a scenario)

Token bucket, `capacity = 3`, `refillRatePerSecond = 1`, one client, fake clock starting at t = 0 ms:

| Time | Before refill | After refill | Result | Tokens after |
|---|---|---|---|---|
| t = 0 | new bucket: 3 | 3 | allow, remaining 2 | 2 |
| t = 0 | 2 | 2 | allow, remaining 1 | 1 |
| t = 0 | 1 | 1 | allow, remaining 0 | 0 |
| t = 0 | 0 | 0 | **deny, retryAfter = 1000 ms** | 0 |
| t = 500 | 0 | 0.5 | **deny, retryAfter = 500 ms** | 0.5 |
| t = 1000 | 0.5 | 1.0 | allow, remaining 0 | 0 |

This confirms the burst of 3 is allowed, then the rate settles to 1/s, and `retryAfterMs` shrinks correctly as tokens accumulate. Tracing one case like this out loud is usually enough in an interview.

---

## 7. Extensibility — Common Follow-Ups

| Follow-up | Where the change lives | Why it's clean |
|---|---|---|
| **New algorithm** (e.g. sliding window counter) | One new `Limiter` class + one `case` in `LimiterFactory` | `RateLimiter` and the other algorithms don't change (Open/Closed via Strategy + Factory) |
| **Thread safety** | Already built in: `ConcurrentHashMap.computeIfAbsent` for per-client state creation + `synchronized` on **that client's** state object | Locking per client means two different clients never block each other. A single global lock would be correct but serialise all traffic. `limitersByEndpoint` is read-only after startup, so it needs no locking |
| **Memory growth** (millions of one-off clients) | Evict idle client state: a Caffeine cache with `expireAfterAccess(windowMs)`, or a periodic sweep | An evicted token bucket just comes back full, which is exactly its correct state after a long idle period, so eviction is safe |
| **Dynamic config reload** | Build a new `Map<String, Limiter>` from the new config and swap it in with an `AtomicReference` / `volatile` field | Readers see either the old or the new map, never a half-built one. Trade-off: per-client state resets on swap (mention it; carry state over only if asked) |
| **Distributed (many gateway instances)** | Move per-client state to Redis: fixed window = `INCR` + `EXPIRE`; token bucket = a Lua script so read-refill-decrement is atomic | Same algorithms, shared state. See `HLD/Core Concepts/Core Building Blocks.md` Section 3 (Redis beyond caching). Trade-off: a network hop per request, and deciding whether to **fail open** (allow) or **fail closed** (deny) if Redis is down |
| **Limit by more than client** (per user *and* per IP, global) | Compose limiters: a `CompositeLimiter` that allows only if all children allow | Each rule stays a simple `Limiter`, so no algorithm changes |

**Concurrency check-and-act, in one line for the interview:** "Refill, check and decrement have to be atomic per client, otherwise two threads can both see 1 token and both proceed. I lock on the client's own state object, so contention stays per-client."

**Level expectations:**
- **Junior:** a working token bucket (or fixed window) for one endpoint; returns allow/deny correctly
- **Mid-level:** Strategy + Factory without prompting, per-endpoint config, default fallback, `remaining` + `retryAfterMs`, can compare two algorithms
- **Senior:** justifies *why* Strategy fits here (and didn't in Connect Four), injects a `Clock`, raises thread safety and memory growth unprompted, explains the fixed-window boundary burst, sketches the Redis/Lua distributed version and the fail-open vs. fail-closed choice

---

## 8. Suggested Resources

- HelloInterview — [Rate Limiter (LLD)](https://www.hellointerview.com/learn/low-level-design/problem-breakdowns/rate-limiter) — source for the requirements and entity framing above (translated to Java here).
- Cross-reference: `Connect 4.md` Section 5, the `checkWin` over-engineering trap. Read the two side by side: the same pattern (Strategy) is wrong there and right here, and being able to explain why is the strongest signal in this problem.
- Cross-reference: `HLD/Core Concepts/Core Building Blocks.md` Section 3 — Redis `INCR` + TTL is the distributed version of `FixedWindowLimiter`.
- Guava's `RateLimiter` (a smooth token bucket) and Bucket4j (token bucket for Java, with Redis/JCache backends) are good real-world references if asked "what would you use in production?"

---

## 9. Self-Test — Multiple Choice

<details>
<summary>Q1: Why is the Strategy pattern justified for rate-limiting algorithms when it was over-engineering for Connect Four's win directions?</summary>

**A)** Rate limiters are more performance-sensitive
**B)** Token bucket, sliding window and fixed window are genuinely different behaviours with different state, chosen per endpoint at runtime from config; Connect Four's four directions were one algorithm with different constants
**C)** Strategy is always correct once there are more than three implementations
**D)** Connect Four is too small a problem for design patterns

**Answer: B** — the test is "different behaviour, or the same behaviour with different parameters?" Different behaviour selected at runtime is exactly what Strategy is for.
</details>

<details>
<summary>Q2: Why does each endpoint's Limiter hold a Map&lt;clientId, State&gt;, rather than RateLimiter holding one limiter per (endpoint, client) pair?</summary>

**A)** Maps are faster than nested objects
**B)** The endpoint's rule (capacity, window) is shared by all clients and belongs to the limiter once; only each client's changing state varies, so each algorithm owns its own state shape and RateLimiter only deals with endpoints
**C)** Java doesn't allow composite map keys
**D)** It uses less memory in every case

**Answer: B** — it's separation of concerns. The orchestrator routes by endpoint, and each algorithm owns its rule and its per-client state.
</details>

<details>
<summary>Q3: How does the token bucket refill without a background thread?</summary>

**A)** It doesn't. Tokens only reset when the window ends
**B)** A scheduled executor adds one token per client every second
**C)** Lazy refill: on each request, it computes the elapsed time since the last refill × the refill rate, adds that many tokens (capped at capacity), then checks and decrements
**D)** The client sends a refill request

**Answer: C** — lazy refill costs O(1) per request, and idle clients cost no CPU at all.
</details>

<details>
<summary>Q4: With a fixed window limit of 100 requests/minute, what's the worst case a client can achieve?</summary>

**A)** Exactly 100 in any 60-second period
**B)** Around 200 in about two seconds, by sending 100 at the end of one window and 100 at the start of the next
**C)** Unlimited, because the counter never resets
**D)** 50, because windows overlap

**Answer: B** — this boundary burst is the known flaw of fixed windows. Sliding window log (exact) or sliding window counter (approximation) fixes it.
</details>

<details>
<summary>Q5: Two threads handle requests from the same client at the same moment, with 1 token left. What's the risk and the best fix?</summary>

**A)** No risk, because ConcurrentHashMap makes everything thread-safe
**B)** Both read 1 token and both proceed. Make refill-check-decrement atomic by synchronising on that client's state object, so different clients never contend
**C)** Put `synchronized` on RateLimiter.allow() so all requests are serialised
**D)** Use a volatile double for tokens

**Answer: B** — ConcurrentHashMap only makes the *map* safe, not the compound read-modify-write on the value. C is correct but serialises all traffic, and D doesn't make the compound action atomic.
</details>

<details>
<summary>Q6: Why inject a Clock into each limiter instead of calling System.currentTimeMillis()?</summary>

**A)** Clock is faster
**B)** Tests can use a fake clock and move time forward exactly, so refill and window logic can be verified deterministically without Thread.sleep()
**C)** System.currentTimeMillis() isn't thread-safe
**D)** The factory can't build limiters otherwise

**Answer: B** — time is a dependency like any other. Inverting it (the D in SOLID) makes time-based logic testable.
</details>
