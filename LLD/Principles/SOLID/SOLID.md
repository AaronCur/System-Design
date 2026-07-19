# LLD Foundations: SOLID Principles

---

## 0. Why This Matters in Interviews

SOLID isn't a checklist to recite — interviewers care that you **apply** the reasoning while designing, not that you can name the acronym. Use these as a live decision framework: every time you're deciding "should this be a separate class?" or "should I use inheritance here?", one of these five principles is usually the thing answering the question for you.

---

## 1. Single Responsibility Principle (SRP)

**A class should have one reason to change.**

Not "a class should do one thing" (too vague to apply) — think instead about *who* would ask for a change to this class, and whether there's more than one possible answer.

### Code smell
```java
// BAD: TicketService has two reasons to change —
// (1) business rules for issuing/validating tickets
// (2) how tickets get formatted/printed
class TicketService {
    public Ticket issueTicket(Vehicle v, ParkingSpot spot) {
        Ticket t = new Ticket(v, spot, Instant.now());
        // ...validation logic...
        System.out.println("=== TICKET ===");
        System.out.println("Spot: " + spot.getId());
        System.out.println("Time: " + t.getEntryTime());
        return t;
    }
}
```

### Fix
```java
// GOOD: split by reason to change
class TicketService {
    public Ticket issueTicket(Vehicle v, ParkingSpot spot) {
        Ticket t = new Ticket(v, spot, Instant.now());
        // ...validation logic only...
        return t;
    }
}

class TicketPrinter {
    public void print(Ticket t) {
        System.out.println("=== TICKET ===");
        System.out.println("Spot: " + t.getSpotId());
        System.out.println("Time: " + t.getEntryTime());
    }
}
```

**Interview tell:** if you can't describe a class's job in one sentence without using "and," it's doing too much. Names like `Manager`, `Handler`, `Processor` are often (not always) a red flag.

---

## 2. Open/Closed Principle (OCP)

**Software entities should be open for extension, but closed for modification.**

When a new requirement shows up, you should be able to *add* code, not *edit* existing, already-tested code.

### Code smell
```java
// BAD: adding a new vehicle type means editing this method again
class FeeCalculator {
    public double calculateFee(Vehicle v, long hours) {
        if (v.getType() == VehicleType.MOTORCYCLE) return hours * 1.0;
        else if (v.getType() == VehicleType.CAR) return hours * 2.0;
        else if (v.getType() == VehicleType.LARGE) return hours * 3.5;
        throw new IllegalArgumentException("Unknown type");
    }
}
```

### Fix
```java
// GOOD: new vehicle types just implement the interface — no existing code touched
interface PricingStrategy {
    double calculateFee(long hours);
}

class MotorcyclePricing implements PricingStrategy {
    public double calculateFee(long hours) { return hours * 1.0; }
}

class CarPricing implements PricingStrategy {
    public double calculateFee(long hours) { return hours * 2.0; }
}
// Adding LargeVehiclePricing later requires zero edits above.
```

**Interview tell:** ask yourself "if a new requirement dropped tomorrow, would I edit this class or just add a new one?" If the answer is "edit," look for a seam to extract an interface.

---

## 3. Liskov Substitution Principle (LSP)

**Subtypes must be substitutable for their base type without breaking correctness.**

Any code written against the base type/interface should keep working, unmodified, no matter which subtype is actually passed in.

### Code smell
```java
// BAD: classic Rectangle/Square violation
class Rectangle {
    protected int width, height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int getArea() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int w) { width = w; height = w; } // silently changes height too!
    @Override
    public void setHeight(int h) { height = h; width = h; }
}
// Code that does rect.setWidth(5); rect.setHeight(10); expects area == 50.
// If rect is actually a Square, area == 100 — silent correctness break.
```

### Fix
Don't model `Square` as a `Rectangle` subtype at all — they don't actually share behavioral contract, only superficial data shape. Prefer composition, or a shared narrower interface (`Shape` with just `getArea()`), over forcing an inheritance relationship that can't honor the parent's contract.

**Interview tell:** would swapping any subclass into code that expects the base class silently break behavior or throw where the base type wouldn't? If a subclass overrides a method to throw `UnsupportedOperationException` or leaves it as a no-op, that's usually LSP being violated.

---

## 4. Interface Segregation Principle (ISP)

**Don't force a class to implement methods it doesn't use.**

Many small, role-specific interfaces beat one large, general-purpose one.

### Code smell
```java
// BAD: fat interface forces irrelevant methods on some implementers
interface Vehicle {
    void refuel();
    void charge();
    void park();
}

class Bicycle implements Vehicle {
    public void refuel() { /* doesn't apply — empty or throws */ }
    public void charge() { /* doesn't apply — empty or throws */ }
    public void park() { /* actual behavior */ }
}
```

### Fix
```java
// GOOD: split into role-specific interfaces, implement only what applies
interface Parkable { void park(); }
interface Refuelable { void refuel(); }
interface Chargeable { void charge(); }

class Bicycle implements Parkable { public void park() { /* ... */ } }
class GasCar implements Parkable, Refuelable { /* ... */ }
class ElectricCar implements Parkable, Chargeable { /* ... */ }
```

**Interview tell:** if a class has empty or stubbed-out method bodies just to satisfy an interface it was forced to implement, that interface needs splitting.

---

## 5. Dependency Inversion Principle (DIP)

**Depend on abstractions, not concrete implementations. High-level modules shouldn't depend on low-level modules — both should depend on interfaces.**

### Code smell
```java
// BAD: ParkingLot is hard-wired to one concrete payment implementation
class ParkingLot {
    private CreditCardPaymentProcessor processor = new CreditCardPaymentProcessor();

    public void processPayment(double amount) {
        processor.charge(amount);
    }
}
```

### Fix
```java
// GOOD: depend on an interface; concrete implementation is injected
interface PaymentProcessor {
    void charge(double amount);
}

class ParkingLot {
    private final PaymentProcessor processor;

    public ParkingLot(PaymentProcessor processor) { // constructor injection
        this.processor = processor;
    }

    public void processPayment(double amount) {
        processor.charge(amount);
    }
}
```

**Interview tell:** this is the principle that maps most directly onto Spring DI experience — same idea, just without the framework doing the wiring for you. Any dependency between classes that might vary (different payment types, different pricing strategies) or need mocking in a unit test is a candidate for an interface + injection rather than a `new SomeConcreteClass()` inside the consuming class.

---

## 6. Live Application Framework (use this during the actual interview)

When you're mid-design and unsure how to structure something, run through this in order:

1. **Identify nouns** from the requirements → candidate classes.
2. For each class: **"one reason to change?"** → if no, split it (SRP).
3. For each behavior that **varies by type**: interface + polymorphism instead of a switch/if-else chain (OCP, LSP).
4. For each interface: is it lean, or are implementers forced to stub unused methods? → split if needed (ISP).
5. For each dependency between two classes: concrete reference or interface? → should be an interface if it might vary, get swapped, or needs mocking in tests (DIP).

---

## 7. Suggested Resources

- HelloInterview — [Design Principles](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/design-principles) — reframes SOLID/OOD principles specifically for interview application rather than academic definition; good complement to this file.
- Refactoring.Guru — solid overview of each principle with UML-style before/after diagrams, useful if the code examples above aren't clicking.
- Your own codebase — the most useful "resource" for this topic is a deliberate pass through 2-3 real files you've written, checking each against the five checks in Section 6.

---

## 8. Self-Test — Spot the Violation

Before moving on, try to name which principle is violated (some may have more than one issue) and how you'd fix it:

1. A `ReportGenerator` class that fetches data from a database, formats it as HTML, and emails it out.
2. An `Employee` interface with `calculateSalary()` and `calculateBonus()`, implemented by both `FullTimeEmployee` and `Contractor` — but `Contractor.calculateBonus()` just returns 0 because contractors never get bonuses.
3. A `NotificationService` class that directly instantiates `EmailSender` inside its `send()` method.
4. A `Bird` base class with a `fly()` method, extended by `Penguin`, which overrides `fly()` to throw `UnsupportedOperationException`.
5. Adding a new discount type to an e-commerce checkout requires adding another `else if` branch to an existing 200-line `calculateTotal()` method.

*(Answers, roughly: 1 = SRP — split fetch/format/send. 2 = ISP — `Contractor` shouldn't be forced to implement a bonus method that doesn't apply; split into a narrower interface or make bonus opt-in. 3 = DIP — inject an `EmailSender`/`NotificationSender` interface. 4 = LSP — `Penguin` can't honor the `Bird` contract; don't model flight as a base-class guarantee. 5 = OCP — extract a `DiscountStrategy` interface.)*
