# HLD Core Building Blocks Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## The Blocks, at a Glance

| Block | Solves | Costs | Default answer |
|---|---|---|---|
| Vertical scaling | More capacity, zero complexity | Hard ceiling, SPOF | Fine until the numbers say otherwise |
| Horizontal scaling | Throughput, redundancy | Needs stateless services + LB | Stateless pods + autoscaling |
| Load balancer | Spreads load, hides failures | Extra hop; must itself be HA | L7, round robin, health checks |
| Cache | Read latency, DB load | Staleness, invalidation, stampedes | Cache-aside, Redis, TTL, delete on write |
| CDN | Global latency for static/media | Invalidation, cost | Static assets and media |
| Replication | Read scale, availability, failover | Lag, stale reads, failover complexity | 1 leader + read replicas + sync standby |
| Sharding | Write scale, storage size | Cross-shard queries/transactions, resharding | Consistent hashing on the hot query's key |
| CAP choice | Correctness vs. uptime during a partition | Either errors or stale data | AP by default; CP for money, stock, uniqueness |

## The One Rule
Never add a block without naming the requirement + number it serves.
> "Reads are 100:1 and p99 must be < 200 ms, so cache-aside in Redis." ✅
> "I'll add Redis." ❌

## Numbers to Know
| Quantity | Rough value |
|---|---|
| Seconds/day | ~100k |
| 1M req/day | ~12 req/s (×2–3 at peak) |
| One DB primary | ~1,000s writes/s, ~10,000s reads/s |
| One Redis node | ~100k ops/s, < 1 ms |
| Same-DC / cross-region round trip | ~0.5 ms / ~50–150 ms |

## Scaling — What to Reach For
| Symptom | First | Then |
|---|---|---|
| Read-heavy | Cache | Read replicas |
| Write-heavy | Queue / batch | Shard |
| Spiky | Queue | Autoscale consumers |
| Global | CDN | Regional deployments |

## Load Balancing
| | L4 | L7 |
|---|---|---|
| Sees | IP + port | HTTP path, headers, cookies |
| Use for | WebSockets, raw TCP, max throughput | Path routing, auth, rate limiting, canaries |

- **Algorithm:** round robin by default; least-connections for long/uneven requests (WebSockets); consistent hash for affinity.
- **Always mention:** health checks, LB runs as an HA pair, prefer stateless over sticky sessions.

## Caching
| Pattern | Write path | Trade-off |
|---|---|---|
| **Cache-aside** (default) | Write DB → delete key | Brief staleness; first read slow |
| Write-through | Write cache + DB together | Always fresh; slower writes |
| Write-behind | Write cache; flush later | Fastest; data loss risk |

| Problem | Fix |
|---|---|
| Stampede (hot key expires) | Single-flight lock, serve stale, TTL jitter |
| Hot key | Key suffix replicas + in-process L1 |
| Penetration (missing keys) | Cache "not found" briefly / Bloom filter |
| Read-your-writes | Read DB for that user briefly after their write |

**Delete, don't update, on write** — avoids a race that leaves the older value cached.

## CAP & Consistency
- P is non-negotiable → choose **C or A during a partition**, **per feature**.
- **PACELC:** no partition → you're trading **latency vs. consistency**.
- **Quorum:** R + W > N → reads see latest write (N=3, W=2, R=2).

| CP | AP |
|---|---|
| Payments, inventory, seat booking, unique usernames/aliases | Feeds, likes, profiles, catalogue, search |

## Replication
| Topology | One-liner |
|---|---|
| Single leader | Default; leader caps writes |
| Multi-leader | Multi-region writes; conflict resolution needed |
| Leaderless | Quorums; highly available, eventually consistent |

- **Semi-sync** (one sync standby, rest async) = usual compromise.
- **Lag fixes:** read-your-writes → leader for that user briefly; monotonic reads → pin user to one replica.
- **Failover risks:** lost async writes, **split brain** → consensus (Raft/etcd, Patroni) + fencing.
- **Spring:** `AbstractRoutingDataSource` routes `readOnly` transactions to replicas.

## Sharding
| Strategy | Good | Bad |
|---|---|---|
| Range | Range scans | Time hot spots |
| Hash (`% N`) | Even spread | Adding a node moves ~all keys |
| **Consistent hashing** | Adding a node moves ~1/N keys | Needs virtual nodes |
| Directory | Flexible per-tenant moves | Lookup is a dependency |

**Shard key checklist:** high cardinality · hot queries stay on one shard · no time hot spot.
**Costs:** scatter–gather queries, cross-shard transactions (use sagas), hot partitions, resharding (start with many logical shards), global IDs (Snowflake/UUID).
**Java:** consistent hash ring = `TreeMap<Long, String>` + `ceilingEntry(hash(key))`, wrap to `firstEntry()`.

## One-Line Justifications (interview-ready)
- "Stateless service behind an L7 LB so any pod can serve any request and we scale by adding pods."
- "Cache-aside with a TTL — delete on write; TTL caps staleness if invalidation ever misses."
- "Browsing is AP, reservation is CP — consistency is a per-feature decision."
- "Replicas scale reads, not writes — for write load we shard."
- "Shard by `user_id` so a user's data is a single-shard read; consistent hashing so adding nodes moves ~1/N of keys."
- "At 40 writes/s one primary is plenty — I'd shard only for storage growth."

## Failure Modes to Name Unprompted
Cache stampede · hot key · replication lag · split brain · hot partition · scatter–gather tail latency

## Level Expectations
- **Mid:** LB + service + DB + cache; knows cache-aside, read replicas, what sharding is
- **Senior:** every block tied to a number; CAP per feature; names failure modes; knows when *not* to shard
- **Staff:** operational cost (resharding, multi-region failover, cache warm-up); explicit PACELC trade-offs

⚠️ Don't over-build — a single well-sized node is a valid answer when the numbers support it.
