# LLD Foundations: Design Patterns

*HelloInterview's "Low Level Design in a Hurry" Design Patterns section, completing the Week 1 tracker deliverable ("pick 2 patterns, Strategy + Factory, and write toy implementations") and extending it to every pattern that has come up across the problem breakdowns in this repo.*

---

## 0. The Rule Before Any Pattern

> "Patterns only help when they match the problem you're solving. Most interview-ready designs use no patterns, or at most one or two. If you're reaching for three or more, you're probably forcing it and over-engineering." — HelloInterview

This is KISS and YAGNI (see `General Principles.md`) applied to patterns. The skill interviewers test isn't *naming* patterns (in the US and Europe they rarely ask directly). It's **recognising the problem shape a pattern solves, and recognising when it isn't there.**

**The one question to ask before reaching for any pattern:**
> "What concrete requirement changes, or varies at runtime, that this pattern absorbs?"

If you can't name one, don't use the pattern.

### Priority for interviews

| Tier | Patterns | Why |
|---|---|---|
| **Must know** | **Strategy**, **Observer**, **Factory** | "Interviewers love Strategy. It's the single most common pattern in LLD interviews." Observer is "top-tier." Factory "shows up regularly" |
| **Know well** | **State**, **Decorator**, **Facade** | State: less common but powerful for workflows. Decorator: stacking optional behaviour. Facade: you already build it in every orchestrator |
| **Recognise, rarely use** | **Builder**, **Singleton**, Template Method | Builder only for genuinely complex objects; Singleton usually the wrong answer; Template Method a lightweight inheritance tool |

---

## 1. Strategy (Behavioural) — *the* interview pattern

**Intent:** define a family of interchangeable algorithms behind one interface, and choose which one to use at runtime. It **replaces conditional logic with polymorphism.**

**Use when:** the *behaviour itself* differs between variants, and the choice is made by config or at runtime (payment methods, pricing rules, rate-limit algorithms, dispatch policies).
**Don't use when:** the variants are one algorithm with different constants, or the behaviour is fixed at design time.

```java
interface PaymentStrategy {
    boolean pay(BigDecimal amount);
}

class CreditCardPayment implements PaymentStrategy {
    public boolean pay(BigDecimal amount) { /* charge card */ return true; }
}

class PayPalPayment implements PaymentStrategy {
    public boolean pay(BigDecimal amount) { /* redirect to PayPal */ return true; }
}

class ShoppingCart {
    private final PaymentStrategy payment;               // injected: the cart never branches on type
    ShoppingCart(PaymentStrategy payment) { this.payment = payment; }
    boolean checkout(BigDecimal total) { return payment.pay(total); }
}
```

**Where it shows up in this repo (the most useful comparison you have):**

| Problem | Strategy? | Why |
|---|---|---|
| Parking Lot — `PricingStrategy` | ✅ | Hourly vs. flat vs. tiered are different fee calculations |
| Rate Limiter — `Limiter` | ✅ | Token bucket, sliding log and fixed window are different algorithms with different state, chosen per endpoint |
| Connect Four — `WinChecker` | ❌ | Four directions = one algorithm with different `(dr, dc)` constants, so parameters instead |
| Elevator — dispatch | ⏸ Not yet | One dispatch algorithm in the requirements. Extract `DispatchStrategy` when a second is asked for |

**The test:** *different behaviour, or the same behaviour with different numbers?*

---

## 2. Observer (Behavioural)

**Intent:** a subject keeps a list of observers and notifies them when its state changes, **without knowing what they do** with the news.

**Use when:** several independent components react to one event, and you want to add or remove reactions without touching the source (price alerts, order status notifications, "lot is full" signals, UI updates).
**Don't use when:** there's exactly one reaction that will never change. A direct method call is clearer.

```java
interface PriceObserver {
    void onPriceChanged(String symbol, BigDecimal price);
}

class Stock {
    private final String symbol;
    private final List<PriceObserver> observers = new ArrayList<>();
    private BigDecimal price;

    Stock(String symbol) { this.symbol = symbol; }

    void subscribe(PriceObserver o)   { observers.add(o); }
    void unsubscribe(PriceObserver o) { observers.remove(o); }

    void setPrice(BigDecimal newPrice) {
        this.price = newPrice;
        observers.forEach(o -> o.onPriceChanged(symbol, newPrice));   // Stock doesn't know what they do
    }
}

class PriceAlert implements PriceObserver {
    private final BigDecimal threshold;
    PriceAlert(BigDecimal threshold) { this.threshold = threshold; }
    public void onPriceChanged(String symbol, BigDecimal price) {
        if (price.compareTo(threshold) > 0) System.out.println(symbol + " above " + threshold);
    }
}
```

**Where it shows up:** Parking Lot's "lot full" notification follow-up (`ParkingLotObserver`). In your day job it's Spring's `ApplicationEventPublisher` / `@EventListener`, and at system scale it's RabbitMQ pub/sub, the same idea across processes.

**Interview gotchas worth a sentence:** observers are called synchronously on the publisher's thread, so a slow observer slows the publisher (make it async if needed). Forgetting to `unsubscribe` leaks memory. Notification order isn't guaranteed.

---

## 3. State (Behavioural)

**Intent:** an object changes its behaviour when its internal state changes. Each state is its own class, and the object delegates to its current state.

**Use when:** the *same action* does something genuinely different in each state and there are many transition rules. A vending machine is the classic case: "insert coin" or "press dispense" behave differently in every state.
**Don't use when:** the states share one algorithm and transitions are a few `if`s. Then an **enum** is enough.

```java
interface VendingState {
    VendingState insertCoin();
    VendingState pressDispense();
}

class NoCoinState implements VendingState {
    public VendingState insertCoin()    { return new HasCoinState(); }
    public VendingState pressDispense() { System.out.println("Insert a coin first"); return this; }
}

class HasCoinState implements VendingState {
    public VendingState insertCoin()    { System.out.println("Coin already inserted"); return this; }
    public VendingState pressDispense() { System.out.println("Dispensing"); return new NoCoinState(); }
}

class VendingMachine {
    private VendingState state = new NoCoinState();
    void insertCoin()    { state = state.insertCoin(); }       // no switch on state anywhere
    void pressDispense() { state = state.pressDispense(); }
}
```

**Where it shows up (as a deliberate *no*):** Connect Four's `GameState` and Elevator's `Direction` are both **enums, not the State pattern**. Their states don't carry different behaviour, so an enum makes invalid states unrepresentable without class ceremony. Knowing when State *isn't* needed is as important as knowing the pattern.

**Tip from HelloInterview:** *"Drawing a state diagram is one of the best ways to communicate a state machine design."* Sketch the boxes and arrows before writing classes.

---

## 4. Factory (Creational)

**Intent:** put object creation behind one method, so callers ask for "a limiter for this config" or "a notification for this channel" without knowing the concrete classes.

**Use when:** there are several implementations of one interface and the choice comes from data (a config string, an enum, user input), especially when paired with Strategy.
**Don't use when:** there's one implementation, or callers always know exactly which class they want. Then `new` is fine.

```java
enum Channel { EMAIL, SMS }

interface Notification { void send(String to, String message); }
class EmailNotification implements Notification { public void send(String to, String m) { /* ... */ } }
class SmsNotification   implements Notification { public void send(String to, String m) { /* ... */ } }

class NotificationFactory {
    static Notification create(Channel channel) {
        return switch (channel) {                   // the ONLY switch on channel in the codebase
            case EMAIL -> new EmailNotification();
            case SMS   -> new SmsNotification();
        };
    }
}
```

> "Update the factory. The rest of your code never changes." — HelloInterview

**Where it shows up:** Rate Limiter's `LimiterFactory` (config → `Limiter`), and the Parking Lot notes' spot-assignment option. **Strategy + Factory is the most common pairing in LLD interviews:** Strategy gives the interchangeable behaviours, Factory decides which one to build, and together they keep the orchestrator closed for modification (the O in SOLID).

In Spring, the container often *is* your factory: inject a `Map<String, Limiter>` of beans keyed by bean name and look one up. That's worth mentioning as the production-code equivalent.

---

## 5. Decorator (Structural)

**Intent:** wrap an object in another object with the same interface, adding one behaviour. Wrappers stack, in any order, at runtime.

**Use when:** you need optional features that combine freely (compression + encryption + buffering; logging + retry + caching around a client). Subclassing every combination would explode.
**Don't use when:** the set of behaviours is fixed and small. A normal subclass or a flag is simpler.

```java
interface DataSource {
    void write(byte[] data);
    byte[] read();
}

class FileDataSource implements DataSource { /* real I/O */
    public void write(byte[] data) { }
    public byte[] read() { return new byte[0]; }
}

abstract class DataSourceDecorator implements DataSource {
    protected final DataSource inner;
    DataSourceDecorator(DataSource inner) { this.inner = inner; }
}

class EncryptionDecorator extends DataSourceDecorator {
    EncryptionDecorator(DataSource inner) { super(inner); }
    public void write(byte[] data) { inner.write(encrypt(data)); }
    public byte[] read() { return decrypt(inner.read()); }
    private byte[] encrypt(byte[] d) { return d; } private byte[] decrypt(byte[] d) { return d; }
}

class CompressionDecorator extends DataSourceDecorator {
    CompressionDecorator(DataSource inner) { super(inner); }
    public void write(byte[] data) { inner.write(compress(data)); }
    public byte[] read() { return decompress(inner.read()); }
    private byte[] compress(byte[] d) { return d; } private byte[] decompress(byte[] d) { return d; }
}

// Stack them: compress, then encrypt, then write to file
DataSource source = new EncryptionDecorator(new CompressionDecorator(new FileDataSource()));
```

**You already use it:** `new BufferedReader(new InputStreamReader(in))` is Decorator. Spring's `@Transactional` / `@Cacheable` proxies are the same idea done by the framework (AOP).

---

## 6. Facade (Structural)

**Intent:** one simple entry point that coordinates several components, so callers don't have to.

> "You're probably already building facades in every LLD interview without calling them that." — HelloInterview

**Where it shows up:** every orchestrator in this repo, including `Game` (Connect Four), `ParkingLot`, `RateLimiter` and `ElevatorController`. There's no need to name it unless asked. If you are asked: "`Game` is a facade over `Board` and the players: callers make one `makeMove()` call and never touch the grid."

**Watch for:** a facade that starts holding business logic that belongs to the components behind it. It should coordinate, not do the work itself.

---

## 7. Builder (Creational) — recognise, rarely use

**Intent:** build a complex object step by step, with readable named setters, ending in `build()`.

**Use when:** an object has many optional fields and constructors would multiply (an HTTP request with headers, timeout, retries, body…).
**Don't use when:** the object has two to four required fields, which covers most interview domain objects.

> "If the interviewer didn't describe a complex object with lots of optional details, Builder probably isn't needed." — HelloInterview

```java
HttpRequest request = HttpRequest.newBuilder()          // the JDK's own java.net.http builder
        .uri(URI.create("https://api.example.com/orders"))
        .header("Authorization", "Bearer " + token)
        .timeout(Duration.ofSeconds(5))
        .POST(HttpRequest.BodyPublishers.ofString(json))
        .build();
```

In your stack, Lombok's `@Builder` gives you this for free, so in an interview you can say "I'd use `@Builder` in production" and move on.

---

## 8. Singleton (Creational) — usually the wrong answer

**Intent:** exactly one instance, reachable globally.

> "The answer is usually no unless they explicitly want a single shared instance." — HelloInterview

**Why it's usually wrong:** hidden global state, hard to replace with a mock in tests, and a trap for concurrency bugs. **Pass the object through constructors instead** (dependency injection). A Spring bean is already a singleton *per container* without any of those downsides.

If the interviewer insists, the safe Java answer is an enum:

```java
enum ConfigManager {
    INSTANCE;
    private final Map<String, String> settings = new ConcurrentHashMap<>();
    String get(String key) { return settings.get(key); }
}
```

It's thread-safe and serialisation-safe, with no double-checked locking to get wrong. Then **flag the testability trade-off** (as the Parking Lot notes do).

---

## 9. Template Method (Behavioural) — bonus

**Intent:** a base class defines the *skeleton* of an algorithm in a `final` method and lets subclasses fill in specific steps.

**Use when:** several variants share the same sequence of steps but differ in one or two of them (e.g. report exporters: load → format → write, where only `format` differs).
**Compared with Strategy:** Template Method varies steps through **inheritance** (fixed at compile time). Strategy varies the whole algorithm through **composition** (swappable at runtime). Prefer Strategy when the choice happens at runtime, and Template Method when the skeleton is fixed and the variation is small.

```java
abstract class ReportExporter {
    final void export(List<Row> rows) {           // the fixed skeleton
        List<Row> data = filter(rows);
        String body = format(data);               // the step that varies
        write(body);
    }
    protected List<Row> filter(List<Row> rows) { return rows; }   // optional hook
    protected abstract String format(List<Row> rows);
    private void write(String body) { /* ... */ }
}

class CsvExporter extends ReportExporter {
    protected String format(List<Row> rows) { return "csv"; }
}
```

You use it constantly without noticing: Spring's `JdbcTemplate` and `RestTemplate` are Template Method (the framework owns the skeleton, and you supply the callback).

---

## 10. Pattern → Problem Map (this repo)

| Pattern | Parking Lot | Connect Four | Amazon Locker | Elevator | Rate Limiter |
|---|---|---|---|---|---|
| Strategy | ✅ Pricing | ❌ WinChecker trap | — | ⏸ Dispatch (on request) | ✅ Algorithms |
| Factory | ✅ Spot assignment | — | — | — | ✅ `LimiterFactory` |
| Observer | ⏸ "Lot full" follow-up | — | — | — | — |
| State | — | ❌ enum instead | — | ❌ enum instead | — |
| Facade | ✅ `ParkingLot` | ✅ `Game` | ✅ orchestrator | ✅ `ElevatorController` | ✅ `RateLimiter` |
| Singleton | ⚠️ only if asked | — | — | — | — |

✅ used · ❌ deliberately not used (know why) · ⏸ the right extension point when a follow-up asks · ⚠️ with a trade-off

The ❌ and ⏸ cells are where senior signal comes from: they show you know when a pattern *isn't* needed.

---

## 11. Suggested Resources

- HelloInterview — [Design Patterns](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/patterns) — source for the tiering, the when/when-not guidance and the quoted advice above (examples translated to Java).
- HelloInterview — [Design Principles](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/design-principles) — the KISS/YAGNI reasoning that decides when *not* to use a pattern; pairs with `General Principles.md`.
- Cross-reference: `Rate Limiter.md` Section 3 and `Connect 4.md` Section 5, the same pattern right in one place and wrong in the other.
- *Head First Design Patterns* (Freeman & Robson) — the most readable full treatment, Java throughout.

---

## 12. Self-Test — Multiple Choice

<details>
<summary>Q1: What single question best decides whether to introduce a pattern?</summary>

**A)** "Is this pattern in the Gang of Four book?"
**B)** "What concrete requirement changes, or varies at runtime, that this pattern absorbs?"
**C)** "Will this impress the interviewer?"
**D)** "Does this problem have more than three classes?"

**Answer: B** — if you can't name the variation it absorbs, the pattern is ceremony (a KISS/YAGNI violation).
</details>

<details>
<summary>Q2: Why is Strategy right for rate-limiting algorithms but wrong for Connect Four's win directions?</summary>

**A)** Rate limiters have more code
**B)** Rate-limiting algorithms are genuinely different behaviour with different state, chosen at runtime; win directions are one algorithm with different (dr, dc) constants
**C)** Connect Four is a game, and patterns don't apply to games
**D)** Strategy needs at least five implementations

**Answer: B** — different behaviour → Strategy; different numbers → parameters.
</details>

<details>
<summary>Q3: Connect Four's GameState and Elevator's Direction are enums. When would the State pattern be the better choice?</summary>

**A)** Whenever there are three or more states
**B)** When the same action behaves genuinely differently in each state and there are many transition rules, like a vending machine
**C)** Whenever states are stored in a database
**D)** Never, because enums are always better

**Answer: B** — an enum is right when states share one algorithm. The State pattern earns its classes when behaviour differs per state.
</details>

<details>
<summary>Q4: Why do Strategy and Factory so often appear together?</summary>

**A)** Java requires factories for interfaces
**B)** Strategy supplies interchangeable behaviours; Factory turns config/data into the right one, keeping the only type switch in one place, so the orchestrator stays closed for modification
**C)** They're the same pattern with different names
**D)** Factory makes strategies thread-safe

**Answer: B** — e.g. `LimiterFactory` + `Limiter`. A new algorithm means one new class plus one factory case, and `RateLimiter` doesn't change.
</details>

<details>
<summary>Q5: An interviewer asks, "Should the ParkingLot be a Singleton?" What's the strongest answer?</summary>

**A)** Yes, always, because there's only one parking lot
**B)** No, Singletons are banned
**C)** Usually I'd pass one instance through constructors (DI) instead, because Singleton adds hidden global state and hurts testability; if a single global instance is a hard requirement, I'd use an enum Singleton and flag that trade-off
**D)** Yes, with double-checked locking for performance

**Answer: C** — it acknowledges the requirement, gives the safe implementation, and names the cost.
</details>

<details>
<summary>Q6: <code>new BufferedReader(new InputStreamReader(in))</code> is an example of which pattern?</summary>

**A)** Factory
**B)** Decorator: each wrapper has the same kind of interface and adds one behaviour, and they stack
**C)** Facade
**D)** Template Method

**Answer: B** — Java I/O streams are the textbook Decorator. A Facade would *hide* several different components behind a new, simpler interface.
</details>
