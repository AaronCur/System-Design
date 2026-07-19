# SOLID Principles Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## The Five, at a Glance

| Letter | Principle | One-line rule | Trigger question to ask yourself |
|---|---|---|---|
| **S** | Single Responsibility | One reason to change | Can I describe this class's job without saying "and"? |
| **O** | Open/Closed | Open for extension, closed for modification | If a new requirement lands tomorrow, do I edit this class or add a new one? |
| **L** | Liskov Substitution | Subtypes must honor the base type's contract | Would swapping any subclass in silently break behavior? |
| **I** | Interface Segregation | Don't force unused methods on implementers | Does any implementer have empty/stubbed methods just to comply? |
| **D** | Dependency Inversion | Depend on abstractions, not concretes | Is this dependency `new`'d directly, or injected via interface? |

## Canonical Examples (HelloInterview)

| Principle | Bad → Good, one-liner |
|---|---|
| **SRP** | `Report` doing content + PDF + file I/O → split into `Report`, `PDFPrinter`, `FileStorage` |
| **OCP** | `PaymentProcessor` with `if/else` per type → `PaymentMethod` interface + `CreditCardPayment`/`PayPalPayment`/`CryptoPayment` |
| **LSP** | `Penguin extends Bird`, `fly()` throws → split into `Bird` (eat) and `FlyingBird extends Bird` (fly); `Penguin implements Bird` only |
| **ISP** | `Worker` interface forces `Robot` to implement `eat()`/`sleep()` → split into `Workable`/`Feedable`/`Restable` |
| **DIP** | `NotificationService` hardcodes `new EmailSender()` → depends on `MessageSender` interface, injected via constructor |

## Common Smells → Fixes

| Smell | Violates | Fix |
|---|---|---|
| Class named `Manager`/`Handler`/`Processor` doing I/O + validation + business logic | SRP | Split by reason to change |
| `if/else` or `switch` on type, edited every time a new type is added | OCP | Interface + polymorphism (Strategy pattern) |
| Subclass overrides a method to throw `UnsupportedOperationException` or no-op it | LSP | Don't force the inheritance relationship; pull the divergent behavior into its own interface |
| Fat interface, some implementers have empty method bodies | ISP | Split into smaller, role-specific interfaces |
| `new ConcreteClass()` hardcoded inside a consumer class | DIP | Constructor-inject an interface instead |

## Live Interview Framework (run this when stuck mid-design)

1. Nouns in requirements → candidate classes
2. Each class → one reason to change? (S)
3. Behavior varies by type → interface + polymorphism, not conditionals (O, L)
4. Interface lean, or stub methods present? (I)
5. Dependency between classes → concrete or interface? Interface if it might vary or need mocking (D)

## Quick Pattern Associations

- **OCP** commonly implemented via → **Strategy pattern** (swap algorithm/behavior without touching the consumer)
- **DIP** commonly implemented via → **constructor injection** (same concept as Spring DI, no framework needed)
- **SRP** violations often fixed by → extracting a collaborator class (e.g., `Printer`, `Validator`, `Repository`)

## Calibration Reminder

SOLID comes out of Java/C#-era deep inheritance design — outside that world it's applied more loosely. Apply these principles when the problem actually calls for them; forcing an interface or a Strategy pattern where a simple class would do violates KISS in the name of demonstrating SOLID. Don't recite the acronym in an interview — apply it silently while designing, and only name-drop the principle if asked to justify a decision.


> [!question]- Why is `ParkingSpot` → `Vehicle` an association rather than composition?
> The vehicle exists before it parks and after it leaves — its lifecycle isn't owned by the spot. Composition would mean the vehicle is destroyed if the spot is.