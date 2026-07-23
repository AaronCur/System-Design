> [!TIP] 
> **A BankAccount class has a public balance field. What's the problem with this design?**
> * A) Callers can set balance to any value, bypassing validation
> * B) The class needs a constructor to initialize public fields
> * C) Public fields prevent the class from being serialized
> * D) Public fields require more memory than private fields at runtime
> 
> <details><summary><b>Answer</b></summary><b>A) Callers can set balance to any value, bypassing validation. </b>When balance is public, anyone can set it directly—including negative values or values that don't match transaction history. Encapsulation means keeping data private and controlling access through methods like deposit() and withdraw() that can enforce rules.</details>

> [!TIP] 
> **Returning a reference to an internal mutable collection from a getter violates encapsulation.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) True. </b>When you return a reference to a mutable internal collection, callers can modify it directly, bypassing your control. Return an unmodifiable view or a copy instead to maintain encapsulation.</details>

> [!TIP] 
> **Polymorphism replaces type-checking if/else statements by letting each object handle itself.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) True. </b>Polymorphism is what replaces if (type == "credit") or switch (vehicleType) statements. Instead of checking types, you call the same method and let each object handle itself. Different objects respond to the same action in their own way.</details>

> [!TIP] 
> **Inheritance creates tight coupling between parent and child classes.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) True. </b> When a subclass inherits the parent's fields and methods, any change in the parent can break every child. That's the 'fragile base class' problem, and it's why inheritance often creates more rigidity than it solves.</details>

> [!TIP] 
> **A Car needs different Engine types (gas, electric, hybrid). Which approach avoids the fragile base class problem?**
> * A) Make Engine an abstract base class
> * B) Use a switch statement on engine type
> * C) Have Car contain an Engine interface field
> * D) Create GasCar, ElectricCar, HybridCar subclasses
> 
> <details><summary><b>Answer</b></summary><b>C) Have Car contain an Engine interface field. </b> Composition avoids the fragile base class problem. Instead of inheritance hierarchies, Car holds a reference to an Engine interface. You can swap engine implementations without modifying Car, and changes to one engine don't affect others..</details>

> [!TIP] 
> **A ReportService class fetches data, calculates metrics, formats HTML, writes files, and sends emails. Why is this design hard to maintain?**
> * A) It violates encapsulation because all methods should be public.
> * B) It has low cohesion because one class is responsible for several unrelated tasks.
> * C) It should use inheritance so each method can be overridden separately.
> * D) It uses too much polymorphism because every method has a different purpose.
> 
> <details><summary><b>Answer</b></summary><b>b) It has low cohesion because one class is responsible for several unrelated tasks. </b> A cohesive class has responsibilities that belong together. When one class handles data access, business logic, formatting, file I/O, and messaging, changes in many different areas can force edits to the same class..</details>

> [!TIP] 
> **A Printer interface requires print(), scan(), and fax(). A basic printer can only print, so its scan() and fax() methods throw UnsupportedOperationException. What design change best addresses this?**
> * A) Split the interface into smaller capabilities like Printable, Scannable, and Faxable.
> * B) Make scan() and fax() private so callers cannot access them.
> * C) Move the unsupported methods into an abstract base class.
> * D) Keep one interface so all printer-like objects have the same API.
> 
> <details><summary><b>Answer</b></summary><b>A). </b> This applies the Interface Segregation Principle: clients should not be forced to depend on methods they do not use. Smaller, capability-focused interfaces let each class implement only the behavior it actually supports.</details>

> [!TIP] 
> **If CheckoutService repeatedly calls order.getCustomer().getAddress().getZipCode(), applying the Law of Demeter would encourage it to ask Order or a dedicated object for the needed shipping destination instead of navigating the whole object graph.**
> * A) True
> * B) False
> 
> <details><summary><b>Answer</b></summary><b>A) True. </b> The Law of Demeter suggests an object should avoid depending on the internal structure of several other objects. By hiding the navigation behind a method or dedicated abstraction, changes to Customer or Address are less likely to ripple into CheckoutService..</details>

> [!TIP] 
> **A PaymentProcessor class directly calls stripe.charge() throughout its code. You want to add PayPal support. What does abstraction suggest?**
> * A) Create a PaymentGateway interface both Stripe and PayPal implement
> * B) Copy the PaymentProcessor class and modify it for PayPal
> * C) Add if/else checks for PayPal wherever stripe.charge() is called
> * D) Make stripe.charge() also handle PayPal requests
> 
> <details><summary><b>Answer</b></summary><b>A). </b> Abstraction means defining what something does without exposing how it works. A PaymentGateway interface hides Stripe/PayPal details behind a common charge() method, letting you swap implementations without changing PaymentProcessor.</details>


> [!TIP] 
> **SavingsAccount and CheckingAccount both need balance tracking, deposits, withdrawals, and transaction history. What's an appropriate use of inheritance here?**
> * A) Have both extend BankAccount with shared implementation
> * B) Create an interface and duplicate the implementation
> * C) Never use inheritance—always prefer composition regardless of the situation
> * D) Copy the shared code into both classes
> 
> <details><summary><b>Answer</b></summary><b>A). </b> Inheritance works well when sharing stable implementation that multiple subclasses genuinely need. Bank accounts are a good example—the core account behavior is stable and truly shared, not just coincidentally similar.</details>

> [!TIP] 
> **You have code like if (vehicle.type == "car") { vehicle.drive() } else if (vehicle.type == "boat") { vehicle.sail() }. How does polymorphism improve this?**
> * A) Define a Vehicle interface with a move() method
> * B) Store the vehicle type as an enum with associated methods
> * C) Extract the if/else logic into a separate utility class
> * D) Replace if/else with a switch statement for clarity
> 
> <details><summary><b>Answer</b></summary><b>A). </b> Polymorphism replaces type-checking conditionals. Instead of checking types, you call vehicle.move() and let each class implement it appropriately. Adding a new vehicle type means adding a new class, not modifying existing code.</details>

> [!TIP] 
> **ElectricCar extends Car and overrides getEngine() to return null. What's wrong with this design?**
> * A) Electric cars should return a placeholder NoEngine object instead
> * B) It forces a behavior difference into a class hierarchy
> * C) The Car base class should have an abstract getEngine() method
> * D) Returning null forces callers to add null checks everywhere
> 
> <details><summary><b>Answer</b></summary><b>B). </b> This is the classic inheritance mistake. Electric cars don't have engines—you're forcing a behavior difference into a class hierarchy. The null checks and placeholder objects are symptoms, not the root problem. Use composition with a Drivetrain interface instead.</details>

> [!TIP] 
> **A mutable User object overrides equals() and hashCode() using its email. The object is inserted into a HashSet, then setEmail() changes the email. After that, set.contains(user) returns false. What is the best explanation?**
> * A) Hash sets compare objects only by reference identity, so overriding equals() is ignored.
> * B) Encapsulation prevents hash-based collections from reading private fields after mutation.
> * C) Changing a field used by hashCode() can move the object's logical bucket without rehashing the set, so lookup searches the wrong bucket.
> * D) The set automatically clones inserted objects, so user is no longer the same instance.
> 
> <details><summary><b>Answer</b></summary><b>C). </b> Hash-based collections assume that an object's hash-relevant state remains stable while it is stored. If equality depends on mutable fields, the object can become unreachable in the collection; use immutable keys or equality based on stable identity..</details>

> [!TIP] 
> **A PricingEngine computes discounts with a switch (customer.getType()) block that is copy-pasted into checkout(), renewal(), and refund(). Every new customer type forces edits to all three methods. Which redesign best follows OOP principles so that adding a customer type becomes a localized change?**
> * A) Replace the type string with an enum so the compiler flags unhandled cases in each switch.
> * B) Extract the switch into one shared helper that checkout, renewal, and refund all call.
> * C) Define a CustomerType interface each type implements, and have the engine dispatch through it.
> * D) Cache each type's computed discount so the three methods avoid recomputing it.
> 
> <details><summary><b>Answer</b></summary><b>C). </b> A repeated switch on an object's type is the classic signal to reach for polymorphism. Giving each customer type its own class behind a shared interface means adding a type is a single new class and the engine's methods never change. Centralizing the switch or converting it to an enum still forces edits everywhere a new case appears.</details>