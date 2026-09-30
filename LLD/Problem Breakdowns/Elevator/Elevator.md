# LLD Problem Breakdown: Elevator System

*Free HelloInterview problem and the Week 3 LLD deliverable. It's the third of the three free problems (with Connect Four and Amazon Locker). Rated "medium": the classes are simple, and the difficulty is in the movement and dispatch logic.*

---

## 1. Clarifying Questions to Ask First

Prompt you'd likely get: *"Design an elevator control system for a building. The system should handle multiple elevators, floor requests, and move elevators efficiently to service requests."*

Work through the same four themes (core operations, error handling, scope boundaries, future extensions):

| Theme | Question | Typical answer |
|---|---|---|
| Scale | How many elevators and floors? Fixed or configurable? | **3 elevators, 10 floors (0–9)**, fixed |
| Core operations | What buttons exist? | **Hall calls** on each floor with an up/down direction, plus **destination buttons** inside each car |
| Core operations | Can a passenger pick several floors? | Yes, a car can hold many destinations at once |
| Error handling | Non-existent floor? Request for the floor the car is already on? | Reject invalid floors; a request for the current floor is a no-op |
| Scope boundary | Real hardware control, or a simulation? | **Simulation**: time advances in discrete ticks through `step()` |
| Scope boundary | Weight limits, doors, emergency stop? | Out of scope |

**Final requirements to write down:**
```
Requirements:
1. System manages 3 elevators serving floors 0-9
2. Hall calls (floor + UP/DOWN) are dispatched to an elevator by the system
3. Passengers inside a car can select multiple destination floors
4. Time advances in discrete ticks via step()
5. Two request kinds: hall calls (have a direction) and destinations (no direction)
6. Invalid floors rejected; requests for the current floor are no-ops

Out of scope: weight limits, door mechanics, emergency stops,
dynamic configuration, UI/rendering
```

**Why "simulation vs. hardware" matters as a question:** a hardware framing pulls you towards threads, sensors and events. A tick-based simulation lets you make movement a pure, testable `step()` method: one call moves each car at most one floor. Asking this up front keeps the design small, and HelloInterview lists it explicitly as a senior signal.

---

## 2. Identify Entities

| Entity | Responsibility |
|---|---|
| **ElevatorController** | Orchestrator and public entry point. Receives hall calls, **chooses which elevator** takes each one, and advances time by stepping every car. |
| **Elevator** | One car. Owns its own floor, direction and set of pending stops, and decides **how it moves** (SCAN). Doesn't know other cars exist. |
| **Request** | Value object: a floor plus a `RequestType` (`PICKUP_UP`, `PICKUP_DOWN`, `DESTINATION`). |

**Deliberately small:** three entities, the same shape as Connect Four (orchestrator + worker + value object). The key split is **dispatch vs. movement**. The controller decides *which* car goes; each car decides *how* it gets there. Keep those apart and each piece stays simple to test.

**Not entities:** `Floor` (just an `int`), `Button` / `Door` / `Display` (hardware and UI concerns that are out of scope), `Passenger` (the system tracks stops, not people).

---

## 3. Class Design

### Why the request type encodes direction

A hall call and a destination aren't the same kind of request:

- Someone on floor 5 pressing **down** only wants a car that's **going down**. If an up-bound car stops, they can't get on.
- Someone inside the car pressing **5** wants to stop at 5 **whichever way** the car is going.

So a hall call is stored as `PICKUP_UP` or `PICKUP_DOWN`, and a car button as `DESTINATION`. **The car travels towards every request but only stops for the ones that match its direction** (plus all destinations). One enum encodes the whole rule.

### Direction: an enum, not the State pattern

It's tempting to build `IdleState`, `MovingUpState` and `MovingDownState` classes, but don't. The three states share one movement algorithm, and the transitions are simple (idle → pick a direction; nothing ahead → reverse; no requests → idle). Those fit in a few `if`s inside `step()`. **The State pattern pays off when each state has genuinely different behaviour and many transition rules**, as in a vending machine where inserting a coin does something different in every state. This is the same judgement as `GameState` in Connect Four: an enum makes invalid states unrepresentable without adding class ceremony.

### Why `requests` is a `Set`

Pressing the same button twice shouldn't create two stops. A `Set<Request>`, where `Request` is a record with value equality, removes duplicates for free. Removing "everything at this floor that matches my direction" is then just a couple of `remove()` calls.

---

## 4. Full Class Diagram (Java)

```java
enum Direction { UP, DOWN, IDLE }
enum RequestType { PICKUP_UP, PICKUP_DOWN, DESTINATION }

record Request(int floor, RequestType type) {}

class Elevator {
    static final int MIN_FLOOR = 0, MAX_FLOOR = 9;

    private int currentFloor = 0;
    private Direction direction = Direction.IDLE;
    private final Set<Request> requests = new HashSet<>();

    boolean addRequest(Request request) { /* Section 5 */ return false; }
    void step() { /* Section 5 */ }
    boolean hasRequestsAhead(Direction dir) { /* Section 5 */ return false; }
    boolean hasRequest(Request request) { return requests.contains(request); }
    int getCurrentFloor() { return currentFloor; }
    Direction getDirection() { return direction; }
}

class ElevatorController {
    private final List<Elevator> elevators;

    ElevatorController(int count) {
        elevators = new ArrayList<>();
        for (int i = 0; i < count; i++) elevators.add(new Elevator());
    }

    boolean requestElevator(int floor, RequestType type) { /* Section 5 */ return false; }
    void step() { elevators.forEach(Elevator::step); }
    Elevator getElevator(int index) { return elevators.get(index); } // car buttons: getElevator(i).addRequest(...)
}
```

**Note the two entry points:** hall calls go through `ElevatorController.requestElevator()`, because the system chooses the car. Destination buttons go straight to a specific `Elevator.addRequest()`, because the passenger is already in a particular car and there's nothing to dispatch.

---

## 5. Implementation — the Interesting Methods

### `Elevator.addRequest`

```java
boolean addRequest(Request request) {
    if (request.floor() < MIN_FLOOR || request.floor() > MAX_FLOOR) return false;
    if (request.floor() == currentFloor && direction == Direction.IDLE) return true; // already here: no-op
    requests.add(request);                                                            // Set ignores duplicates
    return true;
}
```

### `Elevator.step` — SCAN (the core of the problem)

**SCAN (the "elevator algorithm"):** keep going in the current direction until nothing is left ahead, then reverse. It avoids bouncing between floors and guarantees every request is eventually served.

**Order of operations for each tick:**
1. No requests → `IDLE`, done.
2. `IDLE` → pick a direction towards the **nearest** request.
3. Stop here? Remove this floor's `DESTINATION` and the `PICKUP` that **matches the current direction**. If anything was removed, the stop uses this tick.
4. Nothing ahead in the current direction → **reverse** (uses this tick).
5. Otherwise → move one floor.

```java
void step() {
    if (requests.isEmpty()) { direction = Direction.IDLE; return; }

    if (direction == Direction.IDLE) {
        Request nearest = requests.stream()
                .min(Comparator.comparingInt(r -> Math.abs(r.floor() - currentFloor)))
                .orElseThrow();
        if (nearest.floor() == currentFloor) {               // request at our own floor
            requests.removeIf(r -> r.floor() == currentFloor);
            return;
        }
        direction = nearest.floor() > currentFloor ? Direction.UP : Direction.DOWN;
    }

    RequestType matchingPickup = (direction == Direction.UP) ? RequestType.PICKUP_UP : RequestType.PICKUP_DOWN;
    boolean stopped = requests.remove(new Request(currentFloor, RequestType.DESTINATION))
                    | requests.remove(new Request(currentFloor, matchingPickup)); // non-short-circuit: remove both
    if (stopped) {
        if (requests.isEmpty()) direction = Direction.IDLE;
        return;
    }

    if (!hasRequestsAhead(direction)) {
        direction = (direction == Direction.UP) ? Direction.DOWN : Direction.UP;
        return;
    }

    currentFloor += (direction == Direction.UP) ? 1 : -1;
}

boolean hasRequestsAhead(Direction dir) {
    return requests.stream().anyMatch(r ->
            dir == Direction.UP ? r.floor() > currentFloor : r.floor() < currentFloor);
}
```

**The subtle case to be ready for:** a car going **up** reaches floor 8, where the only request is `PICKUP_DOWN`. It doesn't stop, because the direction doesn't match. With nothing above, step 4 reverses it to `DOWN`. On the next tick the `PICKUP_DOWN` at floor 8 now matches, so it stops. The passenger boards a car that's genuinely heading down, which is exactly the behaviour the `RequestType` split exists for.

**Why `|` and not `||`:** `||` short-circuits, so if the destination removal succeeds the pickup removal never runs, and a waiting passenger at the same floor would be left behind.

### `ElevatorController.requestElevator` — dispatch

**Three-tier priority** (from HelloInterview):
1. A car **already moving towards the floor in the requested direction**, closest first. It picks the passenger up on the way.
2. An **idle** car, closest first.
3. Fallback: the **closest car of any kind**.

```java
boolean requestElevator(int floor, RequestType type) {
    if (floor < Elevator.MIN_FLOOR || floor > Elevator.MAX_FLOOR) return false;
    if (type == RequestType.DESTINATION) return false;           // car buttons go to a specific Elevator

    Request request = new Request(floor, type);
    for (Elevator e : elevators) {
        if (e.hasRequest(request)) return true;                  // someone already pressed it
    }

    Direction wanted = (type == RequestType.PICKUP_UP) ? Direction.UP : Direction.DOWN;
    Comparator<Elevator> byDistance = Comparator.comparingInt(e -> Math.abs(e.getCurrentFloor() - floor));

    Elevator chosen = elevators.stream()
            .filter(e -> e.getDirection() == wanted && isOnTheWay(e, floor))
            .min(byDistance)
            .or(() -> elevators.stream().filter(e -> e.getDirection() == Direction.IDLE).min(byDistance))
            .orElseGet(() -> elevators.stream().min(byDistance).orElseThrow());

    return chosen.addRequest(request);
}

private boolean isOnTheWay(Elevator e, int floor) {
    return e.getDirection() == Direction.UP ? e.getCurrentFloor() <= floor : e.getCurrentFloor() >= floor;
}
```

**The naive version to avoid:** "send the nearest car." A car one floor away but heading the other way with five stops queued is a worse choice than an idle car three floors away. Tier 1 is the insight interviewers look for.

**Why dispatch isn't a Strategy interface (yet):** there's one dispatch algorithm in the requirements. A private method is the KISS/YAGNI answer. Extracting a `DispatchStrategy` interface is the **first move** if the interviewer asks for a second algorithm (Section 7). Say that out loud: you're designing the seam without building it.

---

## 6. Verification (trace a scenario)

One car at floor 3, direction `UP`, requests `{Request(5, PICKUP_UP), Request(7, DESTINATION)}`:

| Tick | Floor at start | What happens | Floor after | Direction after |
|---|---|---|---|---|
| 1 | 3 | No stop; requests ahead → move | 4 | UP |
| 2 | 4 | No stop; requests ahead → move | 5 | UP |
| 3 | 5 | `PICKUP_UP` at 5 matches → **stop**, remove it | 5 | UP |
| 4 | 5 | No stop; 7 ahead → move | 6 | UP |
| 5 | 6 | Move | 7 | UP |
| 6 | 7 | `DESTINATION` at 7 → **stop**, remove it; set now empty | 7 | **IDLE** |

Now a hall call `(2, PICKUP_DOWN)` arrives. Tick 7: `IDLE` → nearest request is below → direction `DOWN`; no stop at 7; request ahead → move to 6. The car keeps moving down and stops at 2, because `PICKUP_DOWN` matches `DOWN`.

---

## 7. Extensibility — Common Follow-Ups

| Follow-up | Where the change lives | Why it's clean |
|---|---|---|
| **Smarter / swappable dispatch** (predictive, least-loaded, zoning) | Extract a `DispatchStrategy` interface from `requestElevator`'s selection logic; inject it into the controller | Movement (`Elevator.step`) doesn't change at all, so dispatch and movement were already separate. Strategy is now justified: there are two real algorithms |
| **Express elevators** (serve only some floors) | `Set<Integer> servedFloors` on `Elevator`; `addRequest` rejects others; dispatch filters by `servesFloor(floor)` | One new field and one filter. SCAN is untouched |
| **Cancel a request** | `Elevator.removeRequest(request)` | Requests are a `Set` of value objects, so removal is trivial |
| **Concurrency** (hall calls arrive while `step()` runs) | Make `requestElevator` and `step` `synchronized` on the controller, or queue incoming requests (`ConcurrentLinkedQueue`) and drain them at the start of each tick | The single-tick model makes a single lock simple and correct; draining a queue keeps `step()` lock-free |
| **Capacity / weight limit** | `load` on `Elevator`; dispatch skips full cars; a full car ignores `PICKUP`s but still serves `DESTINATION`s | The pickup-vs-destination split already makes "stop to let people off but not on" easy |
| **Priority floors** (lobby at rush hour, VIP) | Pass a weighting into the dispatch strategy | Lives entirely in dispatch |

**Level expectations (HelloInterview's framing):**
- **Junior:** basic entities, nearest-car dispatch, simple up/down movement
- **Mid-level:** SCAN with correct state transitions, non-naive dispatch, clean split between controller and car
- **Senior:** clarifies simulation vs. hardware up front, three-tier dispatch, explains the pickup-direction rule and the reverse-then-stop edge case, discusses trade-offs and shows the design extends to express cars and priority floors

---

## 8. Suggested Resources

- HelloInterview — [Elevator System](https://www.hellointerview.com/learn/low-level-design/problem-breakdowns/elevator) — free problem breakdown and source for the requirements, entities, dispatch tiers and SCAN logic above (translated to Java here).
- Cross-reference: `Connect 4.md` Section 3 (`GameState` enum) — the same "enum, not State pattern" judgement as `Direction` here.
- Cross-reference: `Rate Limiter.md` Section 3 — Strategy is justified there because two real algorithms exist on day one. Here dispatch has one algorithm, so the seam is designed but not built until a second one is asked for.
- Cross-reference: `Design Patterns.md` — the State pattern section shows what problem *does* need it (vending machine) for contrast.

---

## 9. Self-Test — Multiple Choice

<details>
<summary>Q1: Why are hall calls stored as PICKUP_UP / PICKUP_DOWN instead of just a floor number?</summary>

**A)** It makes the Set faster
**B)** A waiting passenger only wants a car going their way, so the car travels towards every request but stops for a pickup only when its direction matches; destinations stop regardless
**C)** The display needs to show arrows
**D)** Java requires enums for Set elements

**Answer: B** — without the direction, an up-bound car would stop for someone who wants to go down, and they couldn't get on.
</details>

<details>
<summary>Q2: A car going UP reaches floor 8. The only remaining request is PICKUP_DOWN at floor 8. What happens?</summary>

**A)** It stops immediately and picks the passenger up
**B)** It ignores the request forever
**C)** It doesn't stop (direction mismatch); with nothing ahead it reverses to DOWN, and on the next tick the request matches so it stops
**D)** It continues to floor 9 and then comes back

**Answer: C** — this edge case is where most SCAN implementations go wrong, and walking through it shows the direction rule is deliberate.
</details>

<details>
<summary>Q3: Which dispatch choice is best for a PICKUP_UP call at floor 6?</summary>

**A)** Car X at floor 5, direction DOWN, with several stops queued below
**B)** Car Y at floor 2, direction UP
**C)** Car Z idle at floor 9
**D)** Always the nearest car, which is X

**Answer: B** — tier 1: Y is already moving up towards floor 6 and picks the passenger up on the way. X is nearest but heading away (the naive-dispatch trap).
</details>

<details>
<summary>Q4: Why is Direction an enum rather than the State pattern (IdleState, MovingUpState, MovingDownState)?</summary>

**A)** Java enums can't hold behaviour
**B)** All three states share one movement algorithm and the transitions are a few simple rules; the State pattern pays off when each state behaves very differently with many transitions (like a vending machine)
**C)** The State pattern is deprecated
**D)** Enums use less memory

**Answer: B** — the same judgement as Connect Four's `GameState`: use the simplest type that makes invalid states unrepresentable.
</details>

<details>
<summary>Q5: Why do destination buttons call Elevator.addRequest directly instead of going through ElevatorController.requestElevator?</summary>

**A)** The controller is too slow
**B)** The passenger is already inside a specific car, so there's no choice of elevator to make; the controller exists to dispatch hall calls
**C)** Destinations don't need validation
**D)** It's a mistake; everything should go through the controller

**Answer: B** — the controller owns *which car*, each car owns *how it moves*. A destination has already answered "which car."
</details>
