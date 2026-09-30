# Elevator System LLD Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Default Requirements
| Question | Answer |
|---|---|
| Scale | 3 elevators, floors 0–9, fixed |
| Buttons | Hall calls (floor + UP/DOWN) + destination buttons inside each car |
| Multiple destinations | Yes, per car |
| Time model | **Simulation**: discrete ticks via `step()`, not hardware |
| Invalid input | Bad floor → reject; current floor → no-op |
| Out of scope | Weight, doors, emergency stop, dynamic config, UI |

## Entities
| Entity | Responsibility |
|---|---|
| `ElevatorController` | Orchestrator: dispatches hall calls (**which car**), steps all cars |
| `Elevator` | One car: floor, direction, `Set<Request>`, SCAN movement (**how it moves**) |
| `Request` | Record: `floor` + `RequestType` |

## Class Diagram (compressed)
```
ElevatorController ──has──> List<Elevator> (3)
Elevator ──has──> Set<Request>
Elevator.direction: Direction (UP | DOWN | IDLE)
Request.type: RequestType (PICKUP_UP | PICKUP_DOWN | DESTINATION)
```

## Key Design Decisions & One-Line Justifications
| Decision | Why |
|---|---|
| Hall call type carries direction (`PICKUP_UP/DOWN`) | Car travels towards every request but stops for a pickup only if its direction matches |
| `DESTINATION` always stops | Passenger is already in the car, so direction is irrelevant |
| `Direction` enum, **not** the State pattern | One shared movement algorithm, simple transitions (same judgement as Connect Four's `GameState`) |
| `Set<Request>` with record equality | Duplicate button presses are free no-ops |
| Controller dispatches, car moves | Dispatch and movement change independently and are tested separately |
| Destinations go straight to `Elevator.addRequest` | No car to choose, so there's nothing to dispatch |
| Dispatch is a private method, not a Strategy (yet) | One algorithm in the requirements; extract `DispatchStrategy` when a second is asked for |

## SCAN `step()` — Order of Operations
1. No requests → `IDLE`, return
2. `IDLE` → direction towards the **nearest** request
3. Remove `DESTINATION` + **direction-matching** `PICKUP` at this floor (use `|`, not `||`). If stopped: empty → `IDLE`; return
4. Nothing ahead → **reverse**, return
5. Move one floor

**Edge case to know cold:** going UP, only request is `PICKUP_DOWN` at the current top floor → no stop → reverse → stops on the next tick.

## Dispatch — Three Tiers
1. Moving **towards** the floor **in the requested direction** → closest
2. **Idle** → closest
3. Any → closest

❌ Naive "nearest car" can pick a car heading away with a queue of stops.
Also: if any car already has the same hall call, it's a no-op.

## Method Signatures (Java)
```java
enum Direction { UP, DOWN, IDLE }
enum RequestType { PICKUP_UP, PICKUP_DOWN, DESTINATION }
record Request(int floor, RequestType type) {}

class ElevatorController {
    boolean requestElevator(int floor, RequestType type); // hall calls only
    void step();
    Elevator getElevator(int index);
}

class Elevator {
    boolean addRequest(Request request);
    void step();
    boolean hasRequestsAhead(Direction dir);
    int getCurrentFloor();
    Direction getDirection();
}
```

## Extensibility Follow-Ups
| Ask | Answer shape |
|---|---|
| Swappable dispatch | Extract `DispatchStrategy` from `requestElevator`. `Elevator` unchanged |
| Express elevators | `servedFloors` on `Elevator` + a filter in dispatch |
| Cancel request | `Elevator.removeRequest()`: it's a Set of value objects |
| Concurrency | `synchronized` controller methods, or queue requests and drain at the start of each tick |
| Capacity | Full car skips `PICKUP`s, still serves `DESTINATION`s |
| Priority floors | Weighting inside the dispatch strategy |

## Level Expectations
- **Junior:** entities, nearest-car dispatch, basic up/down movement
- **Mid:** SCAN with correct transitions, non-naive dispatch, controller/car separation
- **Senior:** clarifies simulation vs. hardware, three-tier dispatch, explains the pickup-direction rule + reverse-then-stop edge case, extends to express cars/priority floors

⚠️ Don't model doors, buttons, displays or passengers. The system tracks **stops**, not people.
