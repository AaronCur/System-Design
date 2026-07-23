
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


> [!TIP] 
> **A Report class handles content generation, PDF formatting, and file storage. According to the Single Responsibility Principle, how many separate responsibilities does this class have?**
> * A) Two - Content and Delivery
> * B) Zero - SRP doesn't apply to utility classes
> * C) One - It's all about reports
> * D) Three - Content generation, PDF formatting and file storage
> 
> <details><summary><b>Answer</b></summary><b>D) Three </b>SRP says each class should have one reason to change. This class has three distinct responsibilities: content generation (what the report says), PDF formatting (how it looks), and file storage (where it's saved). Changes to any one of these areas would require modifying this class.</details>

> [!TIP] 
> **DIP is achieved when NotificationService receives a MessageSender interface through its constructor rather than creating EmailSender directly.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) True </b>DIP says high-level modules should depend on abstractions, not concrete classes. Receiving the interface through the constructor enables dependency injection.</details>

> [!TIP] 
> **A checkout service must let the business choose between several discount calculation rules at runtime without changing the checkout flow. Which design best fits this need?.**
> * A) Put all the discount rules in one large "if / else" block inside "CheckoutService".
> * B) Make "CheckoutService" inherit from each discount type it might use.
> * C) Duplicate the checkout flow once per discount rule.
> * D) Create a seperate discount strategy classes behind a common interface and inject the selected one
> 
> <details><summary><b>Answer</b></summary><b>D). </b>Using a Strategy-style design separates the stable checkout workflow from interchangeable discount behavior. This keeps the main service simpler and makes adding or swapping rules less invasive than editing a large conditional.</details>

> [!TIP] 
> **Which principle most directly improves testability by enabling mock objects?**
> * A) OCP.
> * B) SRP.
> * C) LSP
> * D) DIP.
> 
> <details><summary><b>Answer</b></summary><b>D) DIP. </b>DIP enables testability by having code depend on interfaces rather than concrete classes. This lets you inject mock implementations during testing..</details>

> [!TIP] 
> **If callers need to write special-case logic like if (bird instanceof Penguin) before calling fly(), the design likely violates LSP.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>B) True. </b>LSP says subclasses must work wherever the base class works. If callers need instanceof checks, the subclass is breaking the contract the parent class established.</details>
