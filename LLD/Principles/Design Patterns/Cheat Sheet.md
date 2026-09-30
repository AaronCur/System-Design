# Design Patterns Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## The Rule
Most good designs use **0–2 patterns**. Three or more usually means you're forcing it.
**Before any pattern, ask:** "What concrete requirement changes, or varies at runtime, that this absorbs?" No answer → no pattern.

## The Patterns, at a Glance
| Pattern | Type | One-line intent | Use when | Don't use when |
|---|---|---|---|---|
| **Strategy** ⭐ | Behavioural | Swap algorithms behind one interface | Behaviour genuinely differs, chosen at runtime | Same algorithm, different constants |
| **Observer** ⭐ | Behavioural | Notify many listeners of a change | Several independent reactions to one event | One fixed reaction; just call it |
| **Factory** ⭐ | Creational | One place turns data → concrete type | Many implementations chosen by config/input | One implementation; use `new` |
| **State** | Behavioural | Each state is a class; object delegates | Same action behaves differently per state, many transitions | States share one algorithm; use an **enum** |
| **Decorator** | Structural | Wrap to add one behaviour; stackable | Optional features that combine freely | Small fixed set; use a subclass/flag |
| **Facade** | Structural | One simple entry point over many parts | Every orchestrator (you already do this) | Don't let it absorb business logic |
| **Builder** | Creational | Named step-by-step construction | Many optional fields | 2–4 required fields (most interview objects) |
| **Singleton** | Creational | Exactly one global instance | Only if explicitly required | Almost always; use DI instead |
| Template Method | Behavioural | Fixed skeleton, subclass fills steps | Shared sequence, one step varies (compile-time) | Runtime choice; use Strategy |

⭐ = must know. "Interviewers love Strategy. It's the single most common pattern in LLD interviews."

## Strategy vs. Parameters (know this cold)
| Problem | Verdict | Reason |
|---|---|---|
| Rate Limiter algorithms | ✅ Strategy | Different algorithms + state, per-endpoint runtime choice |
| Parking Lot pricing | ✅ Strategy | Different fee calculations |
| Connect Four win directions | ❌ Parameters | Same algorithm, different `(dr, dc)` |
| Elevator dispatch | ⏸ Seam only | One algorithm today; extract `DispatchStrategy` when a second is asked for |

## State Pattern vs. Enum
- **Enum:** Connect Four `GameState`, Elevator `Direction`. Shared logic, simple transitions.
- **State pattern:** vending machine, document workflow. Each state reacts differently to the same action.
- Draw the state diagram first either way.

## Common Pairings
| Pairing | Why |
|---|---|
| **Strategy + Factory** | Factory picks the strategy from config; the orchestrator never switches on type (OCP) |
| Facade + Strategy | Orchestrator delegates the varying part to an injected strategy |
| Observer + Strategy | Each observer can be a different reaction policy |

## Java/Spring Equivalents to Name-Drop
| Pattern | Where you already use it |
|---|---|
| Strategy | Injected interface beans (`PaymentService` impls) |
| Factory | Spring container; `Map<String, Bean>` injection |
| Observer | `ApplicationEventPublisher` / `@EventListener`; RabbitMQ pub/sub at scale |
| Decorator | `BufferedReader(InputStreamReader(...))`; `@Transactional`/`@Cacheable` proxies |
| Builder | Lombok `@Builder`; `HttpRequest.newBuilder()` |
| Singleton | Spring beans (singleton per container, no global state) |
| Template Method | `JdbcTemplate`, `RestTemplate` |

## One-Line Justifications (interview-ready)
- "Strategy here because the algorithms genuinely differ and the choice comes from config."
- "Not Strategy here: it's one algorithm with different constants, so I'll parameterise it."
- "An enum is enough; the states don't behave differently, so the State pattern would be ceremony."
- "Factory keeps the only type `switch` in one place, so adding a type doesn't touch the orchestrator."
- "I'd avoid a Singleton and inject one instance; it keeps things testable."

## Singleton — if forced
`enum ConfigManager { INSTANCE; }`: thread-safe, serialisation-safe. **Flag the testability cost.**

⚠️ Name patterns only when it helps explain a decision. The signal is judgement, not vocabulary.
