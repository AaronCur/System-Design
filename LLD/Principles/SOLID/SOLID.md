# LLD Foundations: SOLID Principles

*Updated with canonical examples from HelloInterview's Design Principles page.*

---

## 0. Why This Matters in Interviews

SOLID isn't a checklist to recite — interviewers care that you **apply** the reasoning while designing, not that you can name the acronym. Use these as a live decision framework: every time you're deciding "should this be a separate class?" or "should I use inheritance here?", one of these five principles is usually the thing answering the question for you.

**Worth knowing going in:** SOLID grew out of Java's era of deep inheritance hierarchies and interface-heavy design. Outside Java/C#, strict SOLID application is somewhat falling out of fashion — modern languages lean more on composition and functions than class hierarchies and interfaces for everything. Since your stack is Java/Spring, SOLID stays highly relevant for you, but the calibration still matters: apply these principles when the problem actually calls for them, and don't force complexity just to demonstrate you know the acronym. That's really just KISS applied to SOLID itself — see `general-principles-notes.md` if you want the non-OOP-specific principles (KISS, DRY, YAGNI, Separation of Concerns, Law of Demeter) that HelloInterview treats as prerequisites to SOLID.

---

## 1. Single Responsibility Principle (SRP)

**A class should have one reason to change.**

Not "a class should do one thing" (too vague to apply) — think instead about *who* would ask for a change to this class, and whether there's more than one possible answer.

### Code smell (HelloInterview's canonical example)
```java
// BAD: Report mixes content generation, PDF formatting, and file storage —
// three unrelated reasons to change, all in one class.
class Report {
    String generateContent() { return "content"; }
    void printToPDF() { /* PDF formatting */ }
    void saveToFile() { /* file I/O */ }
}
```

### Fix
```java
// GOOD: split by reason to change
class Report {
    String generateContent() { return "content"; }
}

class PDFPrinter {
    void print(Report report) { /* PDF formatting */ }
}

class FileStorage {
    void save(String content) { /* file I/O */ }
}
```

When the PDF library changes, you only touch `PDFPrinter`. When you switch from files to a database, you only touch `FileStorage`. When report content logic changes, you only touch `Report`. Each change is isolated to exactly one class.

**Interview tell:** if you can't describe a class's job in one sentence without using "and," it's doing too much. Names like `Manager`, `Handler`, `Processor` are often (not always) a red flag.

---

## 2. Open/Closed Principle (OCP)

**Classes should be open for extension but closed for modification** — you should be able to add new behavior without changing existing, already-tested code.

### Code smell (HelloInterview's canonical example)
```java
// BAD: adding a new payment type means modifying this method again
class PaymentProcessor {
    void process(String type, double amount) {
        if (type.equals("credit")) {
            // credit card logic
        } else if (type.equals("paypal")) {
            // paypal logic
        }
        // Adding crypto means modifying this method
    }
}
```

### Fix
```java
// GOOD: new payment types are new classes — existing code never touched
interface PaymentMethod {
    void process(double amount);
}

class CreditCardPayment implements PaymentMethod {
    public void process(double amount) { /* credit card logic */ }
}

class PayPalPayment implements PaymentMethod {
    public void process(double amount) { /* paypal logic */ }
}

class CryptoPayment implements PaymentMethod {
    public void process(double amount) { /* crypto logic */ }
}

class PaymentProcessor {
    void process(PaymentMethod method, double amount) {
        method.process(amount);
    }
}
```

Adding cryptocurrency support is now just a new `CryptoPayment` class — `PaymentProcessor` itself never changes.

**Interview tell:** ask yourself "if a new requirement dropped tomorrow, would I edit this class or just add a new one?" If the answer is "edit," look for a seam to extract an interface.

---

## 3. Liskov Substitution Principle (LSP)

**Subclasses must work wherever the base class works.** If your code uses a parent class or interface, it should work with any subclass without knowing which specific one it got. A subclass can add behavior, but it can't remove or break behavior the parent promised.

**Red flags:** a subclass throws an exception for a method the parent provides; or callers need special-case logic like `if (bird instanceof Penguin)` to work around a subclass's limitations.

### Code smell (HelloInterview's canonical example)
```java
// BAD: Penguin extends Bird but breaks the "all birds can fly" expectation
class Bird {
    void fly() { /* flying logic */ }
}

class Penguin extends Bird {
    void fly() {
        throw new UnsupportedOperationException("Penguins can't fly");
    }
}
```

### Fix
```java
// GOOD: flying is pulled into its own interface — only birds that actually
// fly need to implement it
interface Bird {
    void eat();
}

interface FlyingBird extends Bird {
    void fly();
}

class Sparrow implements FlyingBird {
    public void eat() { /* eating */ }
    public void fly() { /* flying */ }
}

class Penguin implements Bird {
    public void eat() { /* eating */ }
}
```

**Interview tell:** this comes up when you're designing class hierarchies — think carefully about what really belongs in the base type versus what should be a narrower, more specific interface.

---

## 4. Interface Segregation Principle (ISP)

**Prefer small, focused interfaces over large, general-purpose ones.** Don't force a class to implement methods it doesn't need — if a class only needs two methods from a ten-method interface, that interface is too big.

### Code smell (HelloInterview's canonical example)
```java
// BAD: Robot is forced to implement eat()/sleep() it will never use
interface Worker {
    void work();
    void eat();
    void sleep();
}

class Robot implements Worker {
    public void work() { /* working */ }
    public void eat() { /* robots don't eat */ }
    public void sleep() { /* robots don't sleep */ }
}
```

### Fix
```java
// GOOD: split into small, cohesive interfaces; implement only what applies
interface Workable {
    void work();
}

interface Feedable {
    void eat();
}

interface Restable {
    void sleep();
}

class Human implements Workable, Feedable, Restable {
    public void work() { /* working */ }
    public void eat() { /* eating */ }
    public void sleep() { /* sleeping */ }
}

class Robot implements Workable {
    public void work() { /* working */ }
}
```

**Interview tell:** if a class has empty or stubbed-out method bodies just to satisfy an interface it was forced to implement, that interface needs splitting.

---

## 5. Dependency Inversion Principle (DIP)

**Depend on abstractions, not concrete implementations.** The "inversion" refers to *who defines the contract*: normally your business logic conforms to whatever the implementation provides; with DIP you flip this — define an interface based on what your business logic actually needs, and have implementations conform to *that*. The implementation adapts to the business logic, not the other way around.

### Code smell (HelloInterview's canonical example)
```java
// BAD: NotificationService is hard-wired to one concrete email implementation
class EmailSender {
    void send(String message) { /* send email */ }
}

class NotificationService {
    EmailSender emailSender = new EmailSender();

    void notify(String message) {
        emailSender.send(message);
    }
}
```

### Fix
```java
// GOOD: define the interface around what NotificationService needs,
// inject the implementation through the constructor
interface MessageSender {
    void send(String message);
}

class EmailSender implements MessageSender {
    public void send(String message) { /* send email */ }
}

class NotificationService {
    private final MessageSender sender;

    NotificationService(MessageSender sender) {
        this.sender = sender;
    }

    void notify(String message) {
        sender.send(message);
    }
}
```

Now `NotificationService` depends on `MessageSender` (an abstraction), and `EmailSender` also depends on `MessageSender` (it implements it) — neither the high-level nor low-level module knows about the other directly. Swap email for SMS by injecting a different implementation; unit test with a mock `MessageSender` that never sends a real message.

**Note:** DIP is the *design principle*; dependency injection (passing dependencies through the constructor) is the *technique* used to achieve it. Related, not identical.

**Interview tell:** this is the principle that maps most directly onto your Spring DI experience — same idea, just without the framework doing the wiring for you. Any dependency between classes that might vary, get swapped, or need mocking in a test is a candidate for interface + injection rather than a `new SomeConcreteClass()` inside the consumer.

---

## 6. Applying SOLID to Parking Lot

Quick cross-reference back to the problem you're actually working through — where each principle shows up in `parking-lot-notes.md`:

| Principle | How it shows up in Parking Lot |
|---|---|
| **S** | `Ticket` tracks ticket data + delegates fee calc; it doesn't do payment processing itself |
| **O** | New vehicle types extend `Vehicle`; new spot sizes extend `ParkingSpot` — no existing code edited (same shape as the `PaymentMethod` fix above) |
| **L** | Any `ParkingSpot` subtype must honestly support `canFit()`/`park()`/`unpark()` — no throwing "unsupported" on a core method (same trap as `Penguin.fly()`) |
| **I** | Don't force `Vehicle` to implement irrelevant methods (e.g. a `Motorcycle` shouldn't implement a car-only concern) — same fix shape as `Worker` → `Workable`/`Feedable`/`Restable` |
| **D** | `ParkingLot` depends on `PricingStrategy` / `SpotAssignmentStrategy` interfaces, not concrete classes — same shape as `NotificationService` → `MessageSender` |

---

## 7. Live Application Framework (use this during the actual interview)

When you're mid-design and unsure how to structure something, run through this in order:

1. **Identify nouns** from the requirements → candidate classes.
2. For each class: **"one reason to change?"** → if no, split it (SRP).
3. For each behavior that **varies by type**: interface + polymorphism instead of a switch/if-else chain (OCP, LSP).
4. For each interface: is it lean, or are implementers forced to stub unused methods? → split if needed (ISP).
5. For each dependency between two classes: concrete reference or interface? → should be an interface if it might vary, get swapped, or needs mocking in tests (DIP).

---

## 8. Suggested Resources

- HelloInterview — [Design Principles](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/design-principles) — source for the canonical examples above; also covers KISS/DRY/YAGNI/Separation of Concerns/Law of Demeter, which sit above SOLID as more general principles.
- Refactoring.Guru — solid overview of each principle with UML-style before/after diagrams, useful if the code examples above aren't clicking.
- Your own codebase — the most useful "resource" for this topic is a deliberate pass through 2-3 real files you've written, checking each against the five checks in Section 7.

---

## 9. Self-Test — Spot the Violation

Before moving on, try to name which principle is violated (some may have more than one issue) and how you'd fix it:

1. A `ReportGenerator` class that fetches data from a database, formats it as HTML, and emails it out.
2. An `Employee` interface with `calculateSalary()` and `calculateBonus()`, implemented by both `FullTimeEmployee` and `Contractor` — but `Contractor.calculateBonus()` just returns 0 because contractors never get bonuses.
3. A `NotificationService` class that directly instantiates `EmailSender` inside its `send()` method.
4. A `Bird` base class with a `fly()` method, extended by `Penguin`, which overrides `fly()` to throw `UnsupportedOperationException`.
5. Adding a new discount type to an e-commerce checkout requires adding another `else if` branch to an existing 200-line `calculateTotal()` method.

*(Answers, roughly: 1 = SRP — split fetch/format/send, same shape as the `Report` fix. 2 = ISP — `Contractor` shouldn't be forced to implement a bonus method that doesn't apply; split into a narrower interface, same shape as `Worker`. 3 = DIP — inject a `MessageSender`-style interface. 4 = LSP — `Penguin` can't honor the `Bird` contract; pull `fly()` into a `FlyingBird` interface instead. 5 = OCP — extract a `DiscountStrategy` interface, same shape as `PaymentMethod`.)*
