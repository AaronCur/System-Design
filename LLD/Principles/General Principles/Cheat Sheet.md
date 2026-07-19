# General Software Design Principles Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## The Five, at a Glance

| Principle | One-line rule | Trigger question |
|---|---|---|
| **KISS** | Simplest solution that works, wins | Am I adding this pattern because the problem needs it, or to show I know it? |
| **DRY** | Don't repeat conceptually-identical logic | Is this duplication textual, or actually conceptual? |
| **YAGNI** | Build for stated requirements only | Did the interviewer ask for this, or am I imagining they might? |
| **Separation of Concerns** | Each part owns one concern, no reaching into others' internals | Could I test this piece in isolation? |
| **Law of Demeter** | Talk to immediate collaborators only | Am I chaining through multiple different object types to get one value? |

## If you remember only three: KISS, DRY, YAGNI

## Most Common LLD Interview Mistake
**Over-engineering (violates KISS)** — introducing Factory/Builder/Decorator/Strategy patterns where a plain class or conditional would work fine, purely to demonstrate pattern knowledge. Add complexity only when simplicity actually breaks down (e.g. a class hits 500 lines / 10 responsibilities, or a new type means editing 5 different places).

## DRY vs. KISS Tension — How to Talk About It Live
Good interview line when you notice duplication but aren't ready to abstract yet:
> "I'd expect this to show up elsewhere too, but I'll keep it local for now to avoid premature abstraction. If it shows up 3-4 times, that's my signal to extract it."

This shows you can hold competing principles at once — a senior-level signal.

## YAGNI Cheat: Extension Points vs. Extension Implementation
- **Fine:** designing with a seam (interface) that *could* support a future need
- **Not fine:** actually building out that future need before it's been asked for
- Your "out of scope" list (from the Delivery Framework's Requirements phase) is your YAGNI enforcement tool — write it down, refer back to it.

## Separation of Concerns — Quick Test
Can you swap the UI without touching business logic? Can you swap storage without touching business logic? Can you test the core logic without spinning up I/O? If any answer is no, concerns are tangled.

## Law of Demeter — Spot Check
| Pattern | OK? |
|---|---|
| `order.getCustomerZipCode()` | ✅ one call, hides internal navigation |
| `order.getCustomer().getAddress().getZipCode()` | ❌ tunnels through 3 object types |
| `builder.setName(x).setAge(y).build()` | ✅ fluent chain, same object type throughout |

## Relationship to SOLID
General principles apply to any code, any paradigm. SOLID is the object-oriented-specific subset — SRP is essentially Separation of Concerns applied to a single class. Apply general principles first; reach for SOLID once you're actually shaping class hierarchies.
