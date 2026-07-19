# Parking Lot LLD Cheat Sheet

Quick-reference version of the deep-topic notes — for interview warm-ups, not first-time learning.

## Default Assumptions (state these, then confirm with interviewer)
| Question | Default assumption |
|---|---|
| Vehicle types | Motorcycle, Car, Large/SUV |
| Spot assignment | System auto-assigns |
| Entry/exit | Ticket with unique ID issued on entry, required on exit |
| Pricing | Hourly, varies by vehicle type |
| Structure | Multi-level |
| Out of scope | Reservations, full payments, valet, EV charging |

## Entities
| Entity | Key fields | Key methods |
|---|---|---|
| `Vehicle` (abstract) | licensePlate, type | — |
| `ParkingSpot` | spotId, isAvailable, vehicle | `canFit()`, `park()`, `unpark()` |
| `Level` | floorNumber, spots | `findAvailableSpot()` |
| `ParkingLot` | levels | `parkVehicle()`, `unparkVehicle()`, `isFull()` |
| `Ticket` | ticketId, entryTime, spot, vehicle | `calculateFee()` |

## Class Diagram (compressed)
```
ParkingLot ──1..*──> Level ──1..*──> ParkingSpot
Vehicle <──ref── ParkingSpot
Vehicle + ParkingSpot ──ref──> Ticket
```
- `ParkingLot`→`Level`→`ParkingSpot` = **composition** (owned lifecycle)
- `ParkingSpot`→`Vehicle` = **association** (car outlives the parking event)

## Patterns Used & Why
| Pattern | Where | Why |
|---|---|---|
| Strategy | `PricingStrategy` | Swap fee logic without touching `Ticket` (OCP) |
| Strategy/Factory | `SpotAssignmentStrategy` | Swap nearest-fit vs. first-fit without touching `Level` |
| Composition over inheritance | `Vehicle`/`ParkingSpot` type handling | Avoids fragile inheritance chains for type variants |

## One-Line Design Justifications (interview-ready)
- "`canFit()` lives on `ParkingSpot` so adding a new spot/vehicle type doesn't touch `ParkingLot` — that's Open/Closed."
- "Spot references vehicle, not owns it — it's association, not composition, since the vehicle's lifecycle is independent."
- "Pricing is a Strategy so we can swap hourly/flat/tiered pricing without editing `Ticket`."
- "Concurrent spot assignment is a real race condition — I'd guard it with a lock or atomic compare-and-swap on spot state."

## Common Follow-Up Asks
- Reserved/VIP spots → new `ReservationPolicy` or spot attribute + filter in `findAvailableSpot()`
- Multiple entry gates → each gate independently calls `ParkingLot.parkVehicle()`; concurrency safety becomes critical
- "Lot full" notification → Observer pattern on `ParkingLot`
- Single global instance expected → Singleton (flag the testability trade-off if you use it)

⚠️ Don't over-build — no payments/reservations/valet/EV charging unless the interviewer explicitly asks.
