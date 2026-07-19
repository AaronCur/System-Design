# LLD Delivery Framework

*How to structure the ~35 minutes of a low-level design interview so you don't run out of time or drown in edge cases before the interviewer even sees your structure.*

---

## 0. Why This Exists

Two failure modes dominate LLD interviews:
- **Diving straight into code** — get bogged down in edge cases before the interviewer understands your structure.
- **Over-engineering the setup** — defining every tiny detail up front, then running out of time before showing any real design.

The fix is a fixed sequence with rough timings, so you always know what to do next instead of freezing or wandering. If your interviewer pulls you off-framework to explore something, follow their lead — don't fight it, just steer back gently once the tangent's resolved.

**The five phases, with rough timing (~35 min total):**
1. Requirements (~5 min)
2. Entities & Relationships (~3 min)
3. Class Design (~10-15 min)
4. Implementation (~10 min)
5. Extensibility (~5 min, if time/level allows)

---

## 1. Requirements (~5 minutes)

Every prompt starts intentionally vague ("Design a parking lot system," "Design Tic Tac Toe"). Your job is to convert that one sentence into a concrete spec before touching any design — spend the first minute or two turning ambiguity into a written spec via clarifying questions.

**Four question themes to work through** (these apply to any domain — games, devices, workflows, transactional systems):

| Theme | What you're asking |
|---|---|
| **Primary capabilities** | What operations must this system support? |
| **Rules & completion** | What defines success, failure, or a state transition? |
| **Error handling** | How should invalid inputs/actions be handled? |
| **Scope boundaries** | What's explicitly in scope vs. out (UI, storage, networking, concurrency, extensibility)? |

Write the final answers as a numbered requirements list plus an explicit out-of-scope list, and confirm it with the interviewer before moving on. This is exactly the "clarifying questions" step already in `parking-lot-notes.md` — same idea, now named and timeboxed.

**Example shape (Parking Lot):**
```
Requirements:
1. System supports Motorcycle, Car, and Large vehicles.
2. Vehicles are auto-assigned to an available spot on entry.
3. A ticket with a unique ID is issued on entry and required on exit.
4. Fee is calculated hourly, varying by vehicle type.
5. Multi-level structure; system can report when full.

Out of Scope:
- Reservations
- Full payment processing (only fee calculation)
- Valet parking / EV charging
- Concurrency handling (unless it comes up later)
```

---

## 2. Entities & Relationships (~3 minutes)

Translate the confirmed requirements into a small number of core entities with clean ownership — you're shaping the system's skeleton before worrying about any single class's details.

**Identify entities** — scan requirements for meaningful nouns, then filter each one:
- Maintains changing state or enforces rules → it's an entity.
- Just information attached to something else → it's a field on another class, not its own entity.

**Define relationships** — once entities are named, work out:
- Which entity is the **orchestrator** (drives the main workflow)?
- Which entities own durable state?
- How do they depend on each other (has-a / uses / contains)?
- Where should each rule logically live?

**On the whiteboard:** don't overthink notation. A simple entity list plus a few labeled arrows is enough — you're communicating structure to a person, not producing a formal UML diagram. Don't get hung up on strict notation; clarity beats formality.

**Example shape (Parking Lot):**
```
Entities:
- ParkingLot (orchestrator)
- Level
- ParkingSpot
- Vehicle
- Ticket

Relationships:
- ParkingLot -> Level (owns, composition)
- Level -> ParkingSpot (owns, composition)
- ParkingSpot -> Vehicle (references, association)
- Ticket -> ParkingSpot, Vehicle (references both)
```

This matches the entity table and composition/association distinction already worked out in `parking-lot-notes.md`.

---

## 3. Class Design (~10-15 minutes)

Now turn each entity into an actual class outline — go **top-down**, starting with the orchestrator, then supporting entities.

For each entity, answer two questions:
1. **State** — what must this class remember to enforce the requirements?
2. **Behavior** — what operations/queries must it expose?

**Deriving state:** for each entity, trace back to the requirements it owns and ask what it needs to hold in memory to satisfy them. Building a mental (or whiteboard) table — *requirement → what this class must track* — keeps you from guessing or over-adding fields.

**Deriving behavior:** for each entity, ask what the outside world needs from it, and map each need to one method. Aim for a small, focused API — one method per real action or question implied by the requirements, not a kitchen-sink interface.

**Guiding principle — keep rules with the entity that owns the relevant state** (a form of encapsulation, sometimes called "Tell, Don't Ask"): objects should manage their own state and expose behavior, not just getters for others to make decisions on their behalf.
- **Workflow/lifecycle rules** ("can this operation run right now?") → belong in the orchestrator.
- **Data-specific rules** ("is this spot free? does this vehicle fit?") → belong in the entity that owns that data.

This is exactly why `canFit()` lives on `ParkingSpot` rather than `ParkingLot` in the existing notes — that wasn't arbitrary, it's this principle in action.

**Example shape (ParkingLot class):**
```
class ParkingLot:
  - levels: List<Level>
  - pricingStrategy: PricingStrategy

  + parkVehicle(vehicle) -> Ticket
  + unparkVehicle(ticketId) -> fee
  + isFull() -> bool
```

**On UML:** don't use it. It's largely gone from modern production workflows (Microsoft dropped UML tooling from Visual Studio back in 2016 due to near-zero usage) — engineers now design in code or simplified class notation because it's faster to write and easier to iterate on live. If an interviewer explicitly asks for UML, it's fine to ask whether simplified notation is acceptable instead — usually is.

---

## 4. Implementation (~10 minutes)

Implement the most interesting methods — the ones that define system behavior — not every method in the design.

- **Ask the interviewer their preference first.** Some want pseudocode, some want near-complete code in a specific language, some just want you to talk through logic.
- **Default to pseudocode** unless told otherwise, and default to your own strongest language if the interviewer doesn't specify one — don't switch languages just to seem impressive; it usually backfires under pressure.

**Two-pass structure per method:**
1. **Happy path first** — walk the normal flow: inputs, steps, calls to other classes, return value/state change. Let the interviewer see the shape before edge cases muddy it.
2. **Then edge cases** — invalid inputs, illegal operations, out-of-range values, calls that violate current state. This signals you think like someone shipping production code, not toy logic.

**Example shape (parkVehicle, pseudocode):**
```
parkVehicle(vehicle)
    if isFull()
        return null   // or throw — confirm behavior with interviewer

    spot = findAvailableSpotAcrossLevels(vehicle)
    if spot == null
        return null

    spot.park(vehicle)
    ticket = new Ticket(generateId(), spot, vehicle)
    activeTickets[ticket.id] = ticket
    return ticket
```

**Verification step (1-2 min, don't skip this):** trace a concrete scenario through your own pseudocode, tick by tick — initial state, what happens on each call, how state changes, where transitions occur. This isn't about syntax; it's catching logical errors (forgot to update a flag, a check in the wrong order, a missed transition) before the interviewer does, and it's often explicitly part of the grading rubric.

**On patterns (Strategy, Factory, Singleton, etc.):** fine to use if they add real value (e.g., `PricingStrategy` in the Parking Lot design), but the more common failure is *forcing* a pattern where it isn't needed rather than missing one where it was required. Don't reach for a pattern just to demonstrate you know it.

---

## 5. Extensibility (~5 minutes, if time/level allows)

Usually interviewer-led — they propose a twist ("what if we added reservations?" / "what if two entry gates operate concurrently?") to see if your design absorbs the change cleanly, not whether you can bolt on a hack.

- **Junior:** may get little or no extensibility discussion.
- **Mid-level:** one or two small follow-ups.
- **Senior:** several "what if we..." questions in a row — and you're expected to anticipate some of these yourself rather than waiting to be asked.

**How to answer well:** point to the specific part of your design that makes the change clean, and explain *why* — stay high-level here, you're not rewriting code live, you're demonstrating the design already has the right seams.

**Example (Parking Lot):** *"If we needed reserved/VIP spots, I'd add a `ReservationPolicy` check inside `Level.findAvailableSpot()` — since spot-selection logic already lives there, not in `ParkingLot`, extending it doesn't touch the orchestrator at all."*

This is the same reasoning already captured under "Follow-ups / Extensions" in `parking-lot-notes.md` — the Delivery Framework is just the formal name for why that section exists and where it fits in the interview's timeline.

---

## 6. Quick Reference — The Five Phases

| Phase | Time | Output |
|---|---|---|
| 1. Requirements | ~5 min | Numbered spec + explicit out-of-scope list |
| 2. Entities & Relationships | ~3 min | Entity list + ownership arrows (not full UML) |
| 3. Class Design | ~10-15 min | State + behavior per class, top-down from orchestrator |
| 4. Implementation | ~10 min | Happy path → edge cases → verification trace, for 1-2 key methods |
| 5. Extensibility | ~5 min (if reached) | Point to design seams, explain why the change stays clean |

---

## 7. Self-Test

1. Why does the framework put Requirements before Entities, and Entities before Class Design — what breaks if you skip straight to classes?
2. What's the difference between a "workflow rule" and a "data-specific rule," and where does each belong?
3. Why avoid UML in a live interview even if you know it well?
4. In the Parking Lot example, why is verification (tracing a scenario) a separate step from writing the pseudocode itself?
5. If an interviewer asks "how would you support multiple entry gates?" during Extensibility — which existing design decision (from `parking-lot-notes.md`) would you point to, and why?

---

*Source: HelloInterview — [Delivery Framework](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/delivery). Examples above are adapted to Parking Lot rather than their Tic Tac Toe walkthrough, so this file lines up directly with `parking-lot-notes.md` in this repo.*
