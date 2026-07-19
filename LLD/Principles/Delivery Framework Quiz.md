
> [!TIP] 
> **What does KISS stand for??**
> * A) Keep it Short and Simple
> * B) Keep it Simple, Stupid
> * C) Keep Interfaces Small and Separate
> * D) Keep it Structured and Stable
> 
> <details><summary><b>Answer</b></summary><b>B) Keep it Simple, Stupid.</b>The principle advises choosing the simplest solution that works rather than over-engineering with complex patterns.</details>

> [!TIP] 
> **A PaymentProcessor has a switch statement that handles credit cards, PayPal, and crypto differently. To add Apple Pay, you must modify this class. Which principle does this violate?**
> * A) Liskov Substitution
> * B) Open/Closed
> * C) Interface Segregation
> * D) Single Responsibility
> 
> <details><summary><b>Answer</b></summary><b>B) Open / Closed</b> OCP says classes should be open for extension but closed for modification. Using a PaymentMethod interface lets you add Apple Pay without touching PaymentProcessor.</details>

> [!TIP] 
> **The code order.getCustomer().getAddress().getZipCode() couples your code to the internal structure of multiple objects.**
> * A) True
> * B) False
>
> 
> <details><summary><b>Answer</b></summary><b>A) True.</b> This violates the Law of Demeter. Your code now knows the internal structure of Order, Customer, and Address. If any of them change how they organize data, your code breaks.</details>

> [!TIP] 
> **In a chess game design, what's the benefit of separating Board, Display, and InputHandler into different classes?**
> * A) It makes the code run faster
> * B) Each change is isolated to one class
> * C) It's required by SOLID principles
> * D) It reduces the total lines of code
> 
> <details><summary><b>Answer</b></summary><b>B) Each change is isolated to one class </b> Separation of Concerns means each responsibility is handled by a different part of the code. If you want to switch from console to GUI input, you only touch InputHandler.</details>

> [!TIP] 
> **SOLID principles originated from Java's era of deep inheritance hierarchies, and modern languages often favor simpler approaches like composition over class hierarchies.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) True </b> SOLID comes from Java's heyday of interface-heavy design. Modern languages favor simpler approaches—composition over class hierarchies, functions over interfaces. Apply SOLID when it helps, but don't break KISS to force it.</details>

> [!TIP] 
> **Two modules both need currency amounts with validation, rounding, and formatting rules. Which approach best reduces bugs while keeping the domain concept explicit?.**
> * A) Put currency helper functions randomly in whichever module needs them first.
> * B) Store amounts as raw double values everywhere and remember to apply the rules manually.
> * C) Create a shared Money value object that owns the currency-related rules.
> * D) Represent all amounts as formatted strings so they are ready for display.
> 
> <details><summary><b>Answer</b></summary><b>C) Create a shared Money value object that owns the currency-related rules. </b> A value object such as Money centralizes rules for a domain concept and prevents inconsistent handling across modules. It also makes invalid states harder to represent than using raw numbers or strings throughout the codebase.</details>

> [!TIP] 
> **An order service that directly calls EmailService, AnalyticsService, and LoyaltyService after an order ships is a good fit when you want the order service to stay loosely coupled to future post-shipment actions.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>B) False </b>Direct calls make the order service depend on each concrete follow-up action, so adding or changing listeners often requires modifying the order service. An event-driven or observer-style design can let new subscribers react to an order-shipped event without changing the publisher, though it may add complexity around delivery and failures.</details>

> [!TIP] 
> **An order service that directly calls EmailService, AnalyticsService, and LoyaltyService after an order ships is a good fit when you want the order service to stay loosely coupled to future post-shipment actions.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>B) False </b>Direct calls make the order service depend on each concrete follow-up action, so adding or changing listeners often requires modifying the order service. An event-driven or observer-style design can let new subscribers react to an order-shipped event without changing the publisher, though it may add complexity around delivery and failures.</details>