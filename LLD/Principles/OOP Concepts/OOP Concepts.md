# LLD Foundations: OOP Concepts

*Where Design Principles taught you how to think about clean code, these four concepts are the actual language mechanisms you use to build it. Examples below are translated to Java (source material uses Python) and tied back to Parking Lot where possible.*

---

## 0. Why This Page Exists Separately from SOLID

You don't need to recite "polymorphism" during an interview. What matters is that when requirements say "support multiple payment methods," you know to reach for an interface — not that you can define the word. These four concepts show through in *how* you design, not in what you call it while you're doing it.

---

## 1. Encapsulation

**Keep an object's data private; let the object control how that data is used.** Callers interact through methods, not by reaching in and mutating fields directly.

**The benefit is predictability.** If `Account` owns its `balance` field and only exposes `deposit()`/`withdraw()`, you can enforce rules — no negative balances, transaction logging, related state updates — *inside* those methods. If `balance` were public, nothing guarantees those rules ever run.

**Interview hygiene check:** do your classes expose fields directly, or via methods? If a method returns a reference to an internal mutable collection, can the caller now silently corrupt your object's internal state from outside?

### Code smell
```java
// BAD: spots list is public and mutable — any caller can corrupt it directly
class ParkingLot {
    public List<ParkingSpot> spots = new ArrayList<>();
}
```

### Fix
```java
// GOOD: internal list is private; only exposed via controlled methods
class ParkingLot {
    private final List<ParkingSpot> spots = new ArrayList<>();

    public boolean parkVehicle(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle);
        if (spot == null) return false;
        spot.occupy(vehicle);
        return true;
    }

    private ParkingSpot findAvailableSpot(Vehicle vehicle) {
        return spots.isEmpty() ? null : spots.get(0);
    }

    // returns a copy, not the live internal list — caller can't mutate our state
    public List<ParkingSpot> getSpots() {
        return new ArrayList<>(spots);
    }
}
```

**Rule of thumb:** if you're debating whether to expose a field directly or write a getter, write the getter. If a method needs to return a collection, return a copy or an unmodifiable view — never the live internal reference.

---

## 2. Abstraction

**Expose only what's essential; hide implementation details behind a clear interface.** Define *what* something can do without revealing *how* it does it.

**The benefit is simplification.** If `OrderService` depends on a `PaymentMethod` interface instead of a concrete `StripeApi` class, you can swap implementations without touching `OrderService` at all. The caller doesn't need to know whether you're hitting Stripe's API or storing tokens in a database — it just calls `process()` and gets a result back.

**When to reach for it:** abstraction earns its keep where there's real complexity — tangled logic, multiple variations, messy details. If the requirements suggest several approaches to the same problem, that's your signal to define an interface and hide the variation behind it.

### Code smell
```java
// BAD: OrderService is tightly coupled to one concrete payment implementation.
// Changing payment providers means editing OrderService directly.
class OrderService {
    private final String apiKey;

    OrderService(String apiKey) { this.apiKey = apiKey; }

    void checkout(Order order) {
        StripeApi stripe = new StripeApi();
        stripe.setApiKey(apiKey);
        stripe.createCharge(order.getTotal(), order.getCardNumber());
    }
}
```

### Fix
```java
// GOOD: OrderService depends on the PaymentMethod abstraction, not a concrete provider
interface PaymentMethod {
    boolean process(double amount);
}

class CreditCardPayment implements PaymentMethod {
    public boolean process(double amount) { return true; }
}

class PayPalPayment implements PaymentMethod {
    public boolean process(double amount) { return true; }
}

class OrderService {
    private final PaymentMethod paymentMethod;

    OrderService(PaymentMethod paymentMethod) { this.paymentMethod = paymentMethod; }

    void checkout(Order order) {
        paymentMethod.process(order.getTotal());
    }
}
```

**The hard part is calibration.** Too abstract and the interface becomes meaningless (`doWork()`, `handleRequest()` — tells you nothing). Too specific and you haven't actually abstracted anything useful. Think about what operations the *caller* needs, not how those operations happen to be implemented today.

---

## 3. Polymorphism

**What replaces `if (type == "credit")` or `switch (vehicleType)` — call the same method, let each object handle itself.** Different objects respond to the same message in their own way, without the caller needing to know which concrete type it's holding.

**Polymorphism follows naturally from abstraction.** Once you've defined an interface (`PaymentMethod`, `Vehicle`), each implementation supplies its own behavior. Call a method on the interface, and the actual code that runs depends on the real underlying type — no type-checking required anywhere.

### Code smell
```java
// BAD: ParkingLot has to know about every vehicle type and branch on it
class ParkingLot {
    boolean parkVehicle(Vehicle vehicle) {
        if (vehicle.getType().equals("car")) {
            return findSpotBySize(SpotSize.REGULAR) != null;
        } else if (vehicle.getType().equals("motorcycle")) {
            return findSpotBySize(SpotSize.MOTORCYCLE) != null;
        } else if (vehicle.getType().equals("truck")) {
            return findSpotBySize(SpotSize.LARGE) != null;
        }
        return false;
    }
}
```

### Fix
```java
// GOOD: each Vehicle subtype knows its own required spot size — no branching
abstract class Vehicle {
    abstract SpotSize getRequiredSpotSize();
}

class Car extends Vehicle {
    SpotSize getRequiredSpotSize() { return SpotSize.REGULAR; }
}

class Motorcycle extends Vehicle {
    SpotSize getRequiredSpotSize() { return SpotSize.MOTORCYCLE; }
}

class Truck extends Vehicle {
    SpotSize getRequiredSpotSize() { return SpotSize.LARGE; }
}

class ParkingLot {
    boolean parkVehicle(Vehicle vehicle) {
        SpotSize required = vehicle.getRequiredSpotSize();
        return findSpotBySize(required) != null;
    }
}
```

Adding a new vehicle type is now just a new subclass — `ParkingLot.parkVehicle()` never changes again.

**Worth knowing for interview discussion — this isn't a free win.** Highly polymorphic code can be genuinely harder to trace and debug as the number of implementations grows; you lose the ability to just read top-to-bottom and see every branch in one place. Company culture varies here — some teams prefer explicit branches per type, others lean into polymorphism's extensibility. Be ready to name this tradeoff out loud if asked, rather than presenting polymorphism as strictly better.

**Trigger:** if you catch yourself writing type checks or a switch on an enum, that's usually your signal to reach for polymorphism instead.

---

## 4. Inheritance

**Lets one class be a more specific version of another, automatically inheriting the parent's data and behavior.** Powerful, but it comes with a real cost: tight coupling. A change to the parent can silently break every child — the classic "fragile base class" problem.

**The safer default is composition + interfaces.** An interface defines the behavior; each class implements it independently. You still get abstraction and polymorphism, without forcing unrelated classes into a rigid parent-child relationship or sharing state they shouldn't.

### When inheritance actually works
Use it when you have **stable, shared implementation** that multiple subclasses genuinely need — not just a shared label.

```java
// GOOD: SavingsAccount and CheckingAccount genuinely share stable balance
// logic — deposit/withdraw/balance tracking is identical across both.
class BankAccount {
    private double balance = 0.0;

    void deposit(double amount) { balance += amount; }

    boolean withdraw(double amount) {
        if (balance < amount) return false;
        balance -= amount;
        return true;
    }

    double getBalance() { return balance; }
}

class SavingsAccount extends BankAccount {
    private final double interestRate;
    SavingsAccount(double interestRate) { this.interestRate = interestRate; }
}

class CheckingAccount extends BankAccount {
    private final int overdraftLimit;
    CheckingAccount(int overdraftLimit) { this.overdraftLimit = overdraftLimit; }
}
```

Both subtypes are genuinely forms of `BankAccount`, and neither needs to override inherited behavior in a way that breaks the parent's contract (this is the same territory as Liskov Substitution — see `solid-notes.md`). Inheritance is the right tool here.

### When inheritance breaks down
**The classic interview mistake: using inheritance to model behavior differences, not shared implementation.**

```java
// BAD: ElectricCar doesn't share any real engine logic with Car —
// this is a behavior difference forced into a class hierarchy
class Car {
    void startEngine() { /* gasoline engine start logic */ }
}

class ElectricCar extends Car {
    @Override
    void startEngine() { /* completely different electric motor logic */ }
}
// Add a HybridCar next — does it extend Car or ElectricCar? Neither works cleanly.
```

### Fix — isolate the varying behavior into its own abstraction, and compose it
```java
// GOOD: Drivetrain captures the behavior that actually varies;
// Car composes whichever drivetrain it needs
interface Drivetrain {
    void start();
}

class GasEngine implements Drivetrain {
    public void start() { /* gas engine startup logic */ }
}

class ElectricMotor implements Drivetrain {
    public void start() { /* electric motor startup logic */ }
}

class Car {
    private final Drivetrain drivetrain;
    Car(Drivetrain drivetrain) { this.drivetrain = drivetrain; }
    void start() { drivetrain.start(); }
}
```

Now a hybrid is just a `Car` composing two `Drivetrain`s; a hydrogen car is just a new `Drivetrain` implementation. `Car` itself never changes.

**Default rule for interviews:** reach for interfaces + composition first. Only use inheritance when you genuinely need to share *stable* implementation across classes and the is-a relationship truly holds. In most LLD interviews, you won't need inheritance at all.

---

## 5. Putting It Together

| Concept | One-line rule | Trigger |
|---|---|---|
| **Encapsulation** | Hide state, expose behavior | Fields public, or returning live mutable references? → private + getter/copy |
| **Abstraction** | Define interfaces for variation | Complexity, tangled logic, multiple approaches? → extract an interface |
| **Polymorphism** | Let objects handle themselves | Writing type checks or a switch on an enum? → interface + override |
| **Inheritance** | Compose behavior, don't inherit it | Shared *stable* implementation? Inheritance OK. Shared *label*, different behavior? → composition instead |

---

## 6. Applying This to Parking Lot

Cross-reference back to `parking-lot-notes.md`:

| Concept | Where it shows up |
|---|---|
| Encapsulation | `ParkingSpot.isAvailable`/`vehicle` fields stay private; state only changes via `park()`/`unpark()` |
| Abstraction | `PricingStrategy` and `SpotAssignmentStrategy` interfaces hide the actual fee/assignment logic from `ParkingLot` |
| Polymorphism | `Vehicle` subtypes (`Car`, `Motorcycle`, `Truck`) each know their own spot-size requirement — `ParkingLot` never branches on type |
| Inheritance | Deliberately **avoided** for vehicle/spot type variation — that's a behavior difference, so it's modeled via polymorphism over an interface, not a class hierarchy. Inheritance would only make sense if, say, multiple ticket types shared identical, stable fee-calculation scaffolding. |

---

## 7. Suggested Resources

- HelloInterview — [OOP Concepts](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/oop-concepts) — source for the framing and examples above (their examples are in Python; this file translates them to Java to match your stack).
- Cross-reference `solid-notes.md` — LSP is really the same underlying concern as "when inheritance breaks down" here: a subclass that can't honestly fulfill its parent's contract is the same problem viewed from two different angles.

---

## 8. Self-Test — Multiple Choice

<details>
<summary>Q1: What does encapsulation primarily protect against?</summary>

**A)** Slow method calls
**B)** Callers bypassing an object's own validation/business rules by mutating its state directly
**C)** Memory leaks
**D)** Circular dependencies between classes

**Answer: B** — encapsulation's value is that internal rules (e.g. "balance can't go negative") only get enforced if all mutation goes through controlled methods. A public field has no way to enforce anything.
</details>

<details>
<summary>Q2: You're designing a `NotificationService` that needs to support Email, SMS, and Push — with more channels likely later. Which concept do you reach for first?</summary>

**A)** Inheritance — make `SmsNotification extends EmailNotification`
**B)** Abstraction — define a `NotificationChannel` interface, one implementation per channel
**C)** Encapsulation — just add a `channelType` field and branch internally
**D)** None of these — hardcode each channel as a separate method

**Answer: B** — this is the textbook "define an interface for variation" case; SMS and Email don't share real implementation, they share a contract (send a message).
</details>

<details>
<summary>Q3: What's the strongest sign you should replace a type-check/switch statement with polymorphism?</summary>

**A)** The switch statement is more than 10 lines long
**B)** Each branch calls a different method that could instead live on the type itself, and new types will keep getting added
**C)** The switch is missing a default case
**D)** You want to use fewer classes

**Answer: B** — length alone isn't the signal; the signal is that behavior varies *by type* and the type list is expected to grow, meaning every future addition requires editing the same central switch (an OCP violation, not just an aesthetic one).
</details>

<details>
<summary>Q4: In the Car/ElectricCar example, why does inheritance break down specifically?</summary>

**A)** Java doesn't support multiple inheritance
**B)** `ElectricCar` shares no genuinely reusable implementation with `Car` — `startEngine()` is a completely different behavior forced into a shared hierarchy, not shared logic
**C)** `ElectricCar` should be a private class
**D)** `Car` should have been declared `final`

**Answer: B** — inheritance is for sharing *stable, common implementation*, not for grouping things that merely sound related. When a subclass must override a method entirely rather than extend it, that's the tell.
</details>

<details>
<summary>Q5: When is inheritance actually the right call, per this framework?</summary>

**A)** Whenever two classes have similar names
**B)** Whenever you want to avoid writing an interface
**C)** When multiple subclasses genuinely need identical, stable shared implementation and a true is-a relationship holds (e.g. SavingsAccount/CheckingAccount both being BankAccounts)
**D)** Never — inheritance should always be avoided in modern code

**Answer: C** — inheritance isn't banned, it's just not the default. It's the right tool specifically when the shared logic is stable and the subclasses are genuinely more-specific versions of the same thing, not merely similar-sounding concepts with different behavior.
</details>
