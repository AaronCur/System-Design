# HLD Foundations: Core Building Blocks

*HelloInterview "System Design in a Hurry" — Core Concepts. Week 4 deep dive: scaling, load balancing, caching, CAP theorem, replication and sharding. These are the tools you reach for during the high-level design and deep-dive stages, not topics you present on their own.*

---

## 0. How These Fit Into the Interview

The HLD delivery framework mirrors the LLD one you already know (see `Delivery Framework.md`). The difference is that building blocks come in at stages 4 and 5:

| Stage | What you do | Where building blocks show up |
|---|---|---|
| 1. Requirements | Functional (what users do) + non-functional (scale, latency, availability vs consistency) | Non-functionals decide **which** blocks you'll need |
| 2. Core entities | The nouns: User, Post, Order | — |
| 3. API | Endpoints or events between client and system | — |
| 4. High-level design | Simple, working box diagram that satisfies the functional requirements | Load balancer + stateless service + one database. Keep it boring |
| 5. Deep dives | Revisit each non-functional and fix the bottleneck | Cache, replicas, shards, queues, consistency choices. **Most senior-level signal comes from here** |

**The rule that ties this whole file together:** never add a block without naming the requirement it serves.

> "Reads outnumber writes 100:1 and p99 has to be under 200 ms, so I'll add cache-aside in Redis."

beats

> "I'll add Redis."

This is the HLD version of KISS/YAGNI from `General Principles.md` — complexity is earned by a number, not front-loaded to show you know the tool.

---

## 1. Scaling

**Scale the simple thing vertically until the numbers force you horizontal — and keep the app tier stateless so going horizontal is cheap when you get there.**

### Vertical vs. horizontal

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| What | Bigger box: more CPU, RAM, faster disk | More boxes behind a load balancer |
| Pros | No code change, no distributed-systems problems | Near-unlimited ceiling, redundancy, rolling deploys |
| Cons | Hard ceiling, single point of failure, cost curve steepens | Needs statelessness, a load balancer, coordination; the data tier gets hard |
| Good for | Databases early on, stateful components | App/API tier, workers, caches |

### Stateless services

A service is stateless when **any instance can serve any request**. Push state out of the JVM:

- **Sessions** → JWT, or a shared store (Redis via Spring Session) — not the in-memory HTTP session.
- **Files** → object storage (S3 or equivalent) — not local disk on the pod.
- **In-memory caches** → fine as an L1 if slightly stale data is acceptable; otherwise use a shared cache.

Once stateless, the app tier scales by adding pods (an HPA on OpenShift/Kubernetes). The hard problem then moves to the **data tier** — which is exactly why replication (Section 5) and sharding (Section 6) exist.

### Back-of-envelope numbers worth memorising

| Quantity | Rough value |
|---|---|
| Seconds in a day | ~100,000 (86,400) |
| 1M requests/day | ~12 req/s average; plan for 2–3× at peak |
| Single well-tuned app instance | ~1,000s of simple req/s |
| Single Postgres/MySQL primary | ~1,000s of writes/s, ~10,000s of simple reads/s |
| Single Redis node | ~100,000 ops/s, reads well under 1 ms |
| Memory read / SSD read / same-DC round trip | ~100 ns / ~100 µs / ~0.5 ms |
| Cross-region round trip | ~50–150 ms |

These are order-of-magnitude figures for interview reasoning, not benchmarks. Their job is to justify decisions: *"10k writes/s is past one primary, so we shard."* Modern hardware is large — **saying out loud that a single node is enough is a senior answer when the numbers support it.**

### When to scale what

| Symptom | First move | Second move |
|---|---|---|
| Read-heavy | Cache | Read replicas |
| Write-heavy | Batch / queue writes | Shard |
| Spiky traffic | Queue to absorb bursts | Autoscale consumers |
| Global users | CDN for static content | Regional deployments |

---

## 2. Load Balancing

**A load balancer spreads traffic across stateless instances and routes around dead ones.** In interviews, draw one in front of every horizontally scaled tier and spend your words on L4 vs. L7 and the algorithm — not on whether to have one.

### L4 vs. L7

| | Layer 4 (transport) | Layer 7 (application) |
|---|---|---|
| Sees | IP + port, TCP/UDP | HTTP method, path, headers, cookies |
| Routing unit | Per connection | Per request — `/api/orders` → order service |
| Speed | Very fast, low overhead | Slower: terminates TLS, parses requests |
| Use when | Long-lived connections (WebSockets, raw TCP), max throughput | Path/host routing, auth, rate limiting, canary releases |
| Examples | AWS NLB, HAProxy in TCP mode | AWS ALB, Nginx, Envoy, OpenShift Router, API gateways |

**You already run both:** an OpenShift Route is L7 (HAProxy terminating TLS and routing by host/path to a Service); the Service then balances at L4 across pods.

### Algorithms

| Algorithm | How | Best for |
|---|---|---|
| Round robin | Next server in turn | Uniform, short requests — **the default answer** |
| Weighted round robin | More turns for bigger boxes | Mixed instance sizes, canary at 5% |
| Least connections | Fewest open connections | Long or uneven requests, WebSockets |
| Least response time | Fastest recent responder | Latency-sensitive, heterogeneous backends |
| IP / consistent hash | Hash of client or key → server | Stickiness, cache affinity, WebSocket servers |

### What to mention in a deep dive

- **Health checks** — active (LB polls `/actuator/health`) and passive (eject after N failed requests). Remove unhealthy instances fast; add them back slowly.
- **Sticky sessions** — pin a client to one server. Works, but breaks even load distribution and failover. Prefer stateless + a shared session store.
- **The LB isn't a single point of failure** — run it as an HA pair (active–passive with a floating IP) or use a managed multi-AZ LB. DNS can spread traffic across LBs or regions.
- **Where LBs sit** — client → (DNS/CDN) → edge LB / API gateway → service instances. Internal service-to-service calls can use another LB, a service mesh (Envoy sidecars) or client-side balancing (Spring Cloud LoadBalancer).
- **Real-time connections** — WebSocket/SSE servers need L4 or connection-aware L7, least-connections, and a way to find which server holds a given user's socket (Redis pub/sub or a connection registry). See HelloInterview's *Realtime Updates* pattern.

**Interview line:**
> "An L7 load balancer in front of stateless API instances, round robin with health checks. I'd switch to least-connections for the WebSocket tier."

---

## 3. Caching

**Cache when reads dominate and slightly stale data is acceptable.** The default answer is **cache-aside in Redis with a TTL** — and the interesting discussion is invalidation, stampedes and hot keys, not the cache itself.

### Where to cache

| Layer | Example | Notes |
|---|---|---|
| Client/browser | HTTP `Cache-Control`, ETags | Free, but you can't invalidate it |
| CDN | CloudFront, Akamai | Static assets, media, public pages close to users |
| In-process (L1) | Caffeine, Spring `@Cacheable` | Fastest (ns); each pod has its own copy, so pods can drift |
| Distributed (L2) | Redis, Memcached | Shared across pods; ~1 ms network hop |
| Database | Buffer pool, materialised views | Already there — tune before adding layers |

### Read/write patterns

| Pattern | Read path | Write path | Trade-off |
|---|---|---|---|
| **Cache-aside** (lazy loading) | App checks cache → on miss reads DB and populates cache | App writes DB, then **deletes** the cache key | **Default.** Only caches what's read; first read is slow; brief staleness |
| Read-through | Cache library loads from DB on miss | — | Same as cache-aside, but the loading logic lives in the cache layer |
| **Write-through** | Read from cache | Write cache and DB synchronously | Cache always fresh; slower writes; caches data that may never be read |
| **Write-behind** (write-back) | Read from cache | Write cache; flush to DB asynchronously | Fastest writes; **data loss** if the cache dies before flushing |
| Write-around | Cache-aside reads | Write DB only | Keeps write-once data from polluting the cache |

### Cache-aside in Java (Spring + Redis)

```java
@Service
class ProfileService {
    private static final Duration TTL = Duration.ofMinutes(5);
    private final StringRedisTemplate redis;
    private final ProfileRepository repo;
    private final ObjectMapper mapper;

    Profile getProfile(long userId) {
        String key = "profile:" + userId;
        String cached = redis.opsForValue().get(key);
        if (cached != null) return fromJson(cached);          // hit

        Profile profile = repo.findById(userId).orElseThrow(); // miss → DB
        redis.opsForValue().set(key, toJson(profile), TTL);    // populate with TTL
        return profile;
    }

    @Transactional
    void updateProfile(long userId, ProfileUpdate update) {
        repo.save(apply(update, repo.findById(userId).orElseThrow()));
        redis.delete("profile:" + userId);                     // delete, don't update
    }
}
```

**Why delete on write rather than update the cache?** Two concurrent writers can race: writer A updates DB, writer B updates DB, writer B updates cache, writer A updates cache → the cache now holds A's *older* value until the TTL expires. A delete forces the next read to load whatever the DB currently has.

**Spring shorthand:** `@Cacheable("profiles")` on the read and `@CacheEvict("profiles")` on the write give you the same pattern declaratively — worth mentioning, but write it out by hand in an interview so the logic is visible.

### Eviction

| Policy | Evicts | When to use |
|---|---|---|
| **LRU** | Least recently used | Sensible default (Redis `allkeys-lru`) |
| **LFU** | Least frequently used | Popularity is stable over time |
| **TTL** | Anything older than N | **Always set one** — bounds staleness and caps the damage from invalidation bugs |

### Problems interviewers probe

- **Invalidation / staleness** — delete on write + TTL as a safety net. For cross-service invalidation, publish change events (CDC, or a message on RabbitMQ/Kafka) and have consumers evict.
- **Cache stampede** (thundering herd) — a hot key expires and thousands of requests hit the DB at once. Fixes: a lock/single-flight so only one request rebuilds; serve stale while refreshing; jitter TTLs; refresh ahead of expiry.
- **Hot keys** — one key (a celebrity's profile) overloads one Redis shard. Fixes: replicate the key with suffixes (`user:42#1` … `#10`) and read a random copy; put an in-process L1 in front.
- **Cache penetration** — requests for keys that don't exist always miss and always hit the DB. Fix: cache the "not found" with a short TTL, or front it with a Bloom filter.
- **Read-your-writes** — caches are eventually consistent by nature. If a user edits their profile and must see the change, read from the DB for that user briefly, or update the cache on their own write.

### Redis beyond caching

It recurs across problems, so know its other uses: sorted sets for leaderboards, `INCR` + TTL for rate limiting (**direct link to this week's Rate Limiter LLD problem**), pub/sub for fan-out, distributed locks (carefully), geospatial indexes.

**Interview line:**
> "Read-heavy at 100:1, so cache-aside in Redis with a 5-minute TTL, invalidating on write by deleting the key. For hot celebrity profiles I'd add a short-lived Caffeine L1 to spread the load."

---

## 4. CAP Theorem & Consistency

**Network partitions will happen, so the real choice is consistency vs. availability *during a partition* — and you make it per feature, not per system.** Default to availability unless a stale or conflicting read would cause real harm.

### The three letters

- **Consistency** — every read sees the most recent write (linearisability). *Not* the same "C" as in ACID.
- **Availability** — every request to a non-failed node gets a non-error response, even if it's stale.
- **Partition tolerance** — the system keeps working when nodes can't talk to each other.

You can't give up P in a distributed system, so "CA" isn't a real option once you have more than one node. When a partition happens, a node either **refuses to answer (CP)** or **answers with what it has (AP)**.

### Choosing per feature

| Choose consistency (CP) when… | Choose availability (AP) when… |
|---|---|
| Double-spend or overselling is unacceptable: payments, inventory, seat/ticket booking | A briefly stale view is fine: feeds, like counts, profiles, search results |
| Uniqueness must hold: usernames, short-URL aliases | Uptime matters more than freshness: product catalogue, recommendations |
| Coordination: locks, leader election (ZooKeeper, etcd) | High write availability across regions: carts, presence, metrics |
| Tools: Postgres/MySQL primary, Spanner, etcd | Tools: Cassandra, DynamoDB (default settings), Redis replicas, DNS |

**Mixing within one system is normal — and saying so is strong senior signal.** In ticket booking, browsing events is AP; reserving a seat is CP.

### PACELC — the more useful version

If there's a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose **L**atency or **C**onsistency.

Most of the time there's no partition, so the trade-off you actually live with day to day is latency: waiting for replicas to confirm a write makes it slower.

### Consistency models (strongest → weakest)

| Model | Guarantee | Example |
|---|---|---|
| Strong / linearisable | Reads always see the latest write | Single primary DB, consensus stores |
| Causal | Causally related writes are seen in order | A reply never appears before its post |
| Read-your-writes | You always see your own writes | Edit profile → refresh → see the change |
| Monotonic reads | You never see time go backwards | Sticky reads to one replica |
| Eventual | Replicas converge if writes stop | DNS, Cassandra, async replicas |

*"Eventual consistency is fine here, but users must see their own writes"* is often exactly the right level of precision for an interview.

### Tuneable consistency (quorums)

In leaderless stores with **N** replicas, write to **W** and read from **R**. If **R + W > N**, every read overlaps at least one replica that has the latest write.

- Typical: N = 3, W = 2, R = 2.
- Lower W or R → faster and more available. Raise them → more consistent.

---

## 5. Replication

**Replication copies the *same* data to several nodes for availability and read throughput. It does not raise write capacity.** Default answer: single leader with async read replicas, plus a synchronous standby for failover.

### Topologies

| Topology | How writes flow | Pros | Cons | Seen in |
|---|---|---|---|---|
| **Single leader** (primary–replica) | All writes to the leader; replicas follow its log | Simple, no write conflicts | Leader caps write throughput; failover needed | Postgres, MySQL, MongoDB, Redis |
| **Multi-leader** | A leader per region; leaders sync with each other | Local writes everywhere; survives region loss | Write conflicts need resolution (last-write-wins, CRDTs) | Multi-region MySQL, CouchDB, collaborative editors |
| **Leaderless** | Client writes to W of N replicas | No failover step; highly available | Quorum tuning, read repair, eventual consistency | Cassandra, DynamoDB |

### Sync vs. async

| Mode | Behaviour | Trade-off |
|---|---|---|
| Synchronous | Leader waits for replica to confirm | No data loss on failover; slower writes; one slow replica stalls writes |
| Asynchronous | Leader confirms immediately | Fast writes; replicas lag; recent writes can be lost if leader dies |
| **Semi-sync** | Wait for one replica, rest async | **The usual production compromise** |

### Replication lag — what it breaks and how to fix it

With async read replicas, a read may hit a replica that hasn't caught up yet.

| Symptom | Fix |
|---|---|
| User doesn't see their own write | Route that user's reads to the leader for a few seconds after a write, or track their last-write position and wait for a replica that has it |
| Data appears, then disappears across refreshes (different replicas) | Pin each user to one replica (monotonic reads) |

**In Spring:** an `AbstractRoutingDataSource` can send `@Transactional(readOnly = true)` work to replicas and everything else to the primary:

```java
class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "replica"
                : "primary";
    }
}
```

Override it (force `primary`) on read-after-write paths such as "show the profile I just saved."

### Failover

Detect the leader is down (heartbeat timeouts) → promote the most up-to-date replica → repoint clients. Two risks worth naming unprompted:

- **Lost writes** — with async replication, anything the old leader hadn't shipped is gone.
- **Split brain** — two nodes both believe they're leader and accept conflicting writes. Prevented with consensus (Raft via etcd, Patroni for Postgres) and fencing tokens.

**Interview line:**
> "Postgres primary with a synchronous standby for failover and two async read replicas. Reads go to replicas, except immediately after a user's own write."

---

## 6. Sharding (Partitioning)

**Sharding splits *different* data across nodes so writes and storage scale.** It's the last resort because it makes queries, transactions and operations harder. Everything hinges on the **shard key**.

### Replication vs. sharding

They're complementary, not alternatives — each shard is usually itself replicated.

| | Replication | Sharding |
|---|---|---|
| Each node holds | The same data | A different slice of the data |
| Scales | Reads, availability | Writes, storage size |
| Main cost | Replication lag | Cross-shard queries and transactions |

### Strategies

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Range** | Key ranges per shard (A–F, G–M…, or by date) | Efficient range scans | Hot spots — all of today's writes hit one shard |
| **Hash** | `hash(key) % N` | Even spread | Range queries hit every shard; changing N moves almost every key |
| **Consistent hashing** | Keys and nodes on a hash ring; key goes to the next node clockwise | Adding/removing a node moves only ~1/N of keys | Needs virtual nodes to balance |
| **Directory / lookup** | A service maps key → shard | Flexible; can move tenants individually | Lookup service is a dependency and potential bottleneck |
| **Geo / tenant** | Shard by region or customer | Data locality, compliance, isolation | Uneven tenant sizes |

### Consistent hashing in Java

Place each physical node at many points on a ring (**virtual nodes**), hash each key onto the ring, and walk clockwise to the first node. A `TreeMap` makes "walk clockwise" a one-liner:

```java
class ConsistentHashRing {
    private static final int VIRTUAL_NODES = 150;
    private final TreeMap<Long, String> ring = new TreeMap<>();

    void addNode(String node) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            ring.put(hash(node + "#" + i), node);
        }
    }

    void removeNode(String node) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            ring.remove(hash(node + "#" + i));
        }
    }

    String nodeFor(String key) {
        if (ring.isEmpty()) throw new IllegalStateException("No nodes");
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash(key)); // next point clockwise
        return (entry != null ? entry : ring.firstEntry()).getValue(); // wrap around
    }

    private long hash(String s) { /* e.g. first 8 bytes of MD5 / Murmur3 */ return 0L; }
}
```

**Why this beats `hash(key) % N`:** going from 4 to 5 nodes with modulo remaps ~80% of keys. With the ring, the new node only takes over the arcs next to its own points — ~1/N of keys move. Virtual nodes smooth out the arc sizes so no physical node gets an unlucky large slice. Used by Cassandra, DynamoDB and many cache clients.

### Choosing a shard key — three questions

1. **Does it spread load?** High cardinality, no single value dominating. `country` is poor; `user_id` is usually good.
2. **Do the hot queries stay on one shard?** Shard messages by `conversation_id` so loading a chat is a single-shard read.
3. **Does it avoid time hot spots?** Auto-increment IDs or raw timestamps concentrate new writes on one shard; hash them, or prefix with a hashed bucket.

### Costs to acknowledge (unprompted, if you can)

| Cost | Mitigation |
|---|---|
| Cross-shard queries (scatter–gather, slow at the tail) | Denormalise, or maintain a secondary index / search store (Elasticsearch fed by CDC) |
| Cross-shard transactions | Avoid by design; otherwise sagas with compensating steps rather than two-phase commit |
| Hot partitions (a celebrity, a giant tenant) | Split that key further (key + random suffix) or give it a dedicated shard |
| Resharding | Start with many **logical** shards mapped to fewer physical nodes — growth moves logical shards instead of rehashing keys |
| Global unique IDs | No single auto-increment — use UUIDs or Snowflake-style IDs (timestamp + machine + sequence) |

**Interview line:**
> "At ~50k writes/s we're past one primary, so I'll shard posts by `user_id` with consistent hashing — a user's timeline is a single-shard read. Global search goes to Elasticsearch via CDC rather than scatter–gather."

---

## 7. Putting It Together — Worked Example: URL Shortener

A URL shortener exercises every block in this file and previews Week 5.

**Assumed requirements:**
```
Functional:
1. Create a short link for a long URL (optional custom alias)
2. Redirect short link → long URL

Non-functional:
- 100M new links/month; reads outnumber writes 100:1
- Redirect p99 < 100 ms
- Redirects must stay available; custom aliases must be unique

Out of scope: analytics, link expiry, user accounts
```

**Numbers first:**
- 100M/month ≈ **40 writes/s**
- 100:1 → ~4,000 redirects/s average, ~10,000 at peak
- 5 years ≈ 6B rows × ~500 bytes ≈ **3 TB**

**Design:**
```
Clients
   │
   ▼
L7 load balancer (round robin, health checks)
   │
   ▼
URL service (stateless pods, autoscaled)
   │                          │
   │ reads: cache first       │ misses + writes
   ▼                          ▼
Redis (cache-aside,       Postgres, sharded by short code
 24h TTL, LRU)              ├─ Shard A: primary + sync standby + 2 async replicas
                            └─ Shard B: primary + sync standby + 2 async replicas
```

| Block | Decision | Why (tie to a requirement) |
|---|---|---|
| Scaling | Stateless URL service, autoscaled horizontally | 10k req/s at peak; any pod can serve any redirect |
| Load balancing | L7, round robin, health checks on `/actuator/health` | Short, uniform requests; drop dead pods fast |
| Caching | Cache-aside in Redis, 24h TTL, LRU | 100:1 reads + p99 < 100 ms; popular links are a small hot set |
| CAP | Redirects **AP**; alias creation **CP** | A redirect must always work; two users must never get the same alias |
| Replication | Primary + sync standby per shard, async read replicas | Failover without losing new links; replicas absorb cache misses |
| Sharding | Hash of short code, consistent hashing, many logical shards | Every lookup is by short code → single-shard reads; growth to ~3 TB |

**Senior nuance to say out loud:** 40 writes/s is trivial for one primary. Sharding here is justified by **storage growth, not throughput** — and starting on a single primary with a plan to shard later is a perfectly reasonable answer. Interviewers reward that honesty more than reflexive sharding.

**Uniqueness without cross-shard coordination:** hand out ID ranges from a CP store (each pod reserves a block of 1,000 IDs from Postgres or etcd) and base62-encode them. Custom aliases do an insert-if-absent on the shard that owns that alias.

---

## 8. Level Expectations

- **Mid-level:** draws a sensible LB + service + DB + cache design; knows cache-aside and read replicas; can say what sharding is.
- **Senior:** ties every block to a requirement and a number; picks consistency per feature; names failure modes unprompted (stampede, hot key, replication lag, split brain, hot partition); says when *not* to shard.
- **Staff:** reasons about operational cost (resharding, multi-region failover, cache warm-up), and trades off latency vs. consistency (PACELC) explicitly.

---

## 9. Suggested Resources

- HelloInterview — [System Design in a Hurry: Core Concepts](https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts) — the overview this file follows.
- HelloInterview deep dives — [Caching](https://www.hellointerview.com/learn/system-design/core-concepts/caching), [Sharding](https://www.hellointerview.com/learn/system-design/core-concepts/sharding), [Consistent Hashing](https://www.hellointerview.com/learn/system-design/core-concepts/consistent-hashing), [CAP Theorem](https://www.hellointerview.com/learn/system-design/core-concepts/cap-theorem), [Numbers to Know](https://www.hellointerview.com/learn/system-design/core-concepts/numbers-to-know).
- HelloInterview patterns — [Scaling Reads](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads), [Scaling Writes](https://www.hellointerview.com/learn/system-design/patterns/scaling-writes), [Realtime Updates](https://www.hellointerview.com/learn/system-design/patterns/realtime-updates) — how these blocks combine in real problems.
- Cross-reference: `General Principles.md` Sections 1 and 3 — "don't shard/cache until the numbers say so" is KISS and YAGNI at system scale.
- Cross-reference: this week's Rate Limiter LLD — the Redis `INCR` + TTL counter is the distributed version of that design.

---

## 10. Self-Test — Multiple Choice

<details>
<summary>Q1: Your API has 8 stateless pods and one Postgres primary at 90% CPU, almost entirely from reads. What should you do first?</summary>

**A)** Shard the database by user ID
**B)** Add more API pods
**C)** Add cache-aside in Redis for the hot reads, then add read replicas if the DB is still under pressure
**D)** Switch to a leaderless database like Cassandra

**Answer: C** — the bottleneck is read load on the database, not the app tier (so B does nothing) and not writes (so sharding is premature). Caching is the cheapest, biggest win for read-heavy load; replicas come next.
</details>

<details>
<summary>Q2: In cache-aside, why delete the cache key on write instead of writing the new value into the cache?</summary>

**A)** Redis doesn't support overwriting existing keys
**B)** Deleting is faster than writing
**C)** Two concurrent writers can race so the older value ends up in the cache; deleting forces the next read to load whatever the database currently holds
**D)** Updating the cache would bypass the TTL

**Answer: C** — delete-on-write turns a race that corrupts the cache into, at worst, one extra cache miss.
</details>

<details>
<summary>Q3: A popular key expires and thousands of requests simultaneously hit the database. What is this, and what's a good fix?</summary>

**A)** Cache penetration — fix with a Bloom filter
**B)** Cache stampede — fix with single-flight/locking on rebuild, serving stale while refreshing, or jittered TTLs
**C)** Replication lag — fix by routing reads to the leader
**D)** Hot partition — fix by resharding

**Answer: B** — penetration is about keys that *don't exist*; a stampede is many requests rebuilding the *same* hot key at once.
</details>

<details>
<summary>Q4: In a ticket-booking system, which is the strongest CAP answer?</summary>

**A)** The whole system should be CP, because money is involved
**B)** The whole system should be AP, because uptime matters most
**C)** Event browsing and search are AP; seat reservation and payment are CP
**D)** CAP doesn't apply because we use a relational database

**Answer: C** — consistency is chosen per feature. Stale event descriptions are harmless; double-selling a seat is not.
</details>

<details>
<summary>Q5: A user edits their bio, refreshes, and sees the old one. Reads go to async read replicas. What's the cause and fix?</summary>

**A)** Cache stampede — add TTL jitter
**B)** Replication lag — give that user read-your-writes by routing their reads to the leader briefly after a write
**C)** Split brain — add fencing tokens
**D)** The write failed — retry it

**Answer: B** — the write succeeded on the leader, but the read hit a replica that hadn't caught up. Read-your-writes fixes it for the user who cares without giving up replicas for everyone else.
</details>

<details>
<summary>Q6: Why don't read replicas help a write-bound system?</summary>

**A)** Replicas can't be promoted to leader
**B)** Every replica still has to apply every write — replication copies data, it doesn't split the write load. You need sharding
**C)** Replicas only work with NoSQL databases
**D)** Replicas increase write latency so much that they make it worse

**Answer: B** — replication scales reads and availability; sharding scales writes and storage.
</details>

<details>
<summary>Q7: Why is <code>hash(key) % N</code> painful when adding a node, and what fixes it?</summary>

**A)** Modulo is slow to compute — use a faster hash
**B)** Changing N remaps nearly every key; consistent hashing with virtual nodes moves only ~1/N of keys
**C)** It causes range queries to fail — use range sharding
**D)** It only works with an even number of nodes

**Answer: B** — with a ring, a new node only takes over the arcs next to its own points; everything else stays where it was.
</details>

<details>
<summary>Q8: You shard an orders table by <code>order_id</code>. What query just became expensive, and what's a better shard key if that query is hot?</summary>

**A)** "Get order by ID" — shard by timestamp instead
**B)** "All orders for a user" is now scatter–gather across every shard — shard by <code>user_id</code> if that's the hot query
**C)** Nothing — hashing makes every query single-shard
**D)** "Count all orders" — shard by country

**Answer: B** — the shard key should match the field your hottest queries filter on. `order_id` spreads load well but scatters one user's orders everywhere.
</details>

<details>
<summary>Q9: With N = 3 replicas in a leaderless store, which R/W setting guarantees reads see the latest write?</summary>

**A)** R = 1, W = 1
**B)** R = 1, W = 2
**C)** R = 2, W = 2
**D)** R = 3, W = 0

**Answer: C** — you need R + W > N (2 + 2 = 4 > 3) so every read set overlaps every write set. W = 3, R = 1 also works, at the cost of write availability.
</details>

<details>
<summary>Q10: The URL shortener handles ~40 writes/s. What's the most senior answer about sharding?</summary>

**A)** Shard from day one — it's always needed at scale
**B)** Never shard a URL shortener
**C)** 40 writes/s is easy for one primary; shard only for storage growth (~3 TB over 5 years), and it's fine to start unsharded with a plan to shard by short code
**D)** Use multi-leader replication instead of sharding

**Answer: C** — tying the decision to the actual numbers, and saying when *not* to add complexity, is exactly the signal interviewers look for.
</details>
