# OOP Concepts Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## The Four, at a Glance

| Concept | One-line rule | Trigger question |
|---|---|---|
| **Encapsulation** | Hide state, expose behavior | Are fields public, or am I returning a live mutable reference? |
| **Abstraction** | Define interfaces for variation | Is there tangled logic or multiple approaches to the same problem? |
| **Polymorphism** | Objects handle themselves, no type checks | Am I writing `if/switch` on a type or enum? |
| **Inheritance** | Compose behavior, don't inherit it | Is this *shared stable implementation*, or just a shared label with different behavior? |

## Canonical Examples (translated to Java from HelloInterview)

| Concept | Bad → Good, one-liner |
|---|---|
| **Encapsulation** | Public mutable `spots` list on `ParkingLot` → private list, exposed via `parkVehicle()` + defensive copy in getter |
| **Abstraction** | `OrderService` hardcodes `new StripeApi()` → depends on `PaymentMethod` interface, injected |
| **Polymorphism** | `ParkingLot.parkVehicle()` branches on `vehicle.getType()` string → `Vehicle` subtypes each implement `getRequiredSpotSize()` |
| **Inheritance (good)** | `SavingsAccount`/`CheckingAccount extends BankAccount` — genuinely shared, stable deposit/withdraw logic |
| **Inheritance (bad)** | `ElectricCar extends Car`, overrides `startEngine()` entirely → extract `Drivetrain` interface, `Car` composes it instead |

## Encapsulation — Quick Check
- Field private? ✅ needed
- Getter returns the live internal collection? ❌ return a copy or unmodifiable view instead
- Rule of thumb: "if in doubt, write the getter"

## Abstraction — Calibration Warning
- Too abstract → interface is meaningless (`doWork()`, `handleRequest()`)
- Too specific → you haven't actually abstracted anything
- Right level → interface reflects what the **caller needs**, not how it's implemented today

## Polymorphism — Known Tradeoff (say this out loud if asked)
Pro: new types = new class, zero edits to existing code (OCP in action).
Con: harder to trace/debug as implementations multiply — no single place to read every branch. Company culture varies on tolerance for this; know both sides.

## Inheritance — Decision Rule
```
Shared implementation genuinely stable + true "is-a" relationship?
  → YES: inheritance is fine (BankAccount example)
  → NO (shared label, different behavior): extract an interface + compose (Drivetrain example)
```
**Default for interviews: reach for interfaces + composition first.** Most LLD interviews don't need inheritance at all.

## Cross-Reference to SOLID
- Polymorphism replacing type-checks = same motivating problem as **OCP**
- "When inheritance breaks down" = same underlying issue as **LSP** (subclass can't honor the parent's contract)
- Abstraction via interface + injected implementation = same shape as **DIP**

## One-Line Reminders
- "If you're debating field vs. getter — write the getter."
- "Type check or switch on an enum → that's polymorphism's cue."
- "Inheritance for implementation, never for behavior variation."
- "Interfaces + composition first. Inheritance only when truly stable and truly is-a."
