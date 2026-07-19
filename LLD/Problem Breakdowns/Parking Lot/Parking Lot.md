# LLD Problem Breakdown: Parking Lot

---

## 1. Clarifying Questions to Ask First

Every LLD interview starts with a vague prompt — turn it into something concrete before touching design. Ask around these themes:

- **Scope of operations:** What vehicle types are supported? (assume: Motorcycle, Car, Large/SUV)
- **Assignment behavior:** Is spot assignment automatic, or does the driver choose? (assume: system auto-assigns)
- **Entry/exit flow:** What happens on entry — ticket, code, something else? (assume: ticket issued with a unique ID on entry, required on exit)
- **Pricing:** In scope? Hourly, flat, or type-based? (assume: simple hourly rate, varies by vehicle type)
- **Structure:** Single level or multi-level? (assume: multi-level — slightly more realistic and gives you a natural `Level` entity)
- **Explicitly out of scope for v1:** reservations, full payment processing, valet, EV charging — **don't build these unless the interviewer asks**. Over-building for imagined future requirements is a common way candidates run out of time.

---

## 2. Identify Entities

Scan the requirements for meaningful nouns, then filter each one: *does it maintain changing state or enforce rules?* If yes, it's probably an entity. If it's just data attached to something else, it's a field.

| Candidate noun | Entity or field? | Reasoning |
|---|---|---|
| Vehicle | Entity | Has identity (license plate), type — referenced by tickets/spots |
| ParkingSpot | Entity | Has state (free/occupied) that changes over time |
| ParkingLot | Entity | Orchestrates levels, enforces top-level rules (is full, assign spot) |
| Level | Entity | Owns a collection of spots, has its own state |
| Ticket | Entity | Has entry time, references spot + vehicle, calculates fee |
| Payment/Fee | Value object | Calculated from ticket data — not usually a full entity for v1 |

---

## 3. Class Diagram

```
┌─────────────────────┐
│     ParkingLot        │
│----------------------│
│ - levels: List<Level>│
│----------------------│
│ + parkVehicle(v)     │
│ + unparkVehicle(t)   │
│ + isFull(): bool     │
└──────────┬───────────┘
           │ 1..*
           ▼
┌─────────────────────┐
│       Level           │
│----------------------│
│ - floorNumber        │
│ - spots: List<Spot>  │
│----------------------│
│ + findAvailableSpot()│
└──────────┬───────────┘
           │ 1..*
           ▼
┌─────────────────────┐        ┌───────────────────┐
│    ParkingSpot        ◄──────┤   SpotType (enum)   │
│ (abstract/interface) │        │ MOTORCYCLE/CAR/...  │
│----------------------│        └───────────────────┘
│ - spotId             │
│ - isAvailable        │
│ - vehicle: Vehicle   │
│----------------------│
│ + park(Vehicle)      │
│ + unpark()           │
│ + canFit(Vehicle)    │
└──────────────────────┘

┌─────────────────────┐        ┌────────────────────┐
│       Vehicle          ◄──────┤  VehicleType (enum) │
│ (abstract/interface) │        │ MOTORCYCLE/CAR/...  │
│----------------------│        └────────────────────┘
│ - licensePlate       │
│ - type               │
└──────────────────────┘

┌─────────────────────┐
│        Ticket          │
│----------------------│
│ - ticketId           │
│ - entryTime          │
│ - spot: ParkingSpot  │
│ - vehicle: Vehicle   │
│----------------------│
│ + calculateFee()     │
└──────────────────────┘
```

**Relationship types, and why they matter:**
- `ParkingLot` **has** `Level`s, which **have** `ParkingSpot`s → **composition**. If the lot is torn down, the spots go with it — they don't outlive their owner.
- `ParkingSpot` **references** a `Vehicle` → **association**, not composition. The car existed before it parked and will exist after it leaves; the spot doesn't own its lifecycle.

---

## 4. Key Design Decisions (be ready to justify these out loud)

- **Composition vs. association** (above) — a common thing interviewers probe on, since it signals whether you actually understand OOD or are just drawing boxes.
- **`canFit(Vehicle)` lives on `ParkingSpot`, not `ParkingLot`.** Avoids a giant conditional in the orchestrating class and keeps fit-logic next to the thing it concerns — this is Open/Closed in action (see SOLID notes).
- **Strategy pattern for pricing.** `PricingStrategy` interface with `HourlyPricingStrategy`, `FlatRatePricingStrategy`, etc. New pricing models plug in without touching `Ticket`.
- **Strategy/Factory for spot assignment.** A `SpotAssignmentStrategy` decides *which* spot to assign (nearest-fit vs. first-fit) — keeps `Level.findAvailableSpot()` swappable and testable in isolation.
- **Concurrency note (bonus points):** two vehicles could race for the same spot. Worth mentioning synchronizing spot assignment, or an atomic compare-and-swap on spot state, even if you don't fully implement it — shows you're thinking beyond the happy path.

---

## 5. Where SOLID Shows Up in This Design

| Principle | How it shows up here |
|---|---|
| **S** | `Ticket` tracks ticket data + delegates fee calc; it doesn't do payment processing itself |
| **O** | New vehicle types extend `Vehicle`; new spot sizes extend `ParkingSpot` — no existing code edited |
| **L** | Any `ParkingSpot` subtype must honestly support `canFit()`/`park()`/`unpark()` — no throwing "unsupported" on a core method |
| **I** | Don't force `Vehicle` to implement irrelevant methods (e.g. a `Motorcycle` shouldn't implement a car-only concern) — model as data/type instead |
| **D** | `ParkingLot` depends on `PricingStrategy` / `SpotAssignmentStrategy` interfaces, not concrete classes — easy to unit test with mocks |

---

## 6. Code Skeleton (Java)

Interviewers often want to see actual code, not just a diagram — practice writing this out, not just describing it.

```java
enum VehicleType { MOTORCYCLE, CAR, LARGE }

abstract class Vehicle {
    private final String licensePlate;
    private final VehicleType type;

    protected Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }

    public VehicleType getType() { return type; }
    public String getLicensePlate() { return licensePlate; }
}

class Motorcycle extends Vehicle {
    public Motorcycle(String plate) { super(plate, VehicleType.MOTORCYCLE); }
}
class Car extends Vehicle {
    public Car(String plate) { super(plate, VehicleType.CAR); }
}

enum SpotType { MOTORCYCLE, CAR, LARGE }

class ParkingSpot {
    private final String spotId;
    private final SpotType type;
    private boolean isAvailable = true;
    private Vehicle parkedVehicle;

    public ParkingSpot(String spotId, SpotType type) {
        this.spotId = spotId;
        this.type = type;
    }

    public boolean canFit(Vehicle v) {
        // simplest possible rule: exact type match.
        // real interviews may want size-based compatibility (e.g. motorcycle fits in any spot).
        return isAvailable && matchesType(v);
    }

    private boolean matchesType(Vehicle v) {
        return type.name().equals(v.getType().name());
    }

    public void park(Vehicle v) {
        if (!canFit(v)) throw new IllegalStateException("Spot cannot fit this vehicle");
        this.parkedVehicle = v;
        this.isAvailable = false;
    }

    public void unpark() {
        this.parkedVehicle = null;
        this.isAvailable = true;
    }

    public String getSpotId() { return spotId; }
}

class Level {
    private final int floorNumber;
    private final List<ParkingSpot> spots;

    public Level(int floorNumber, List<ParkingSpot> spots) {
        this.floorNumber = floorNumber;
        this.spots = spots;
    }

    public Optional<ParkingSpot> findAvailableSpot(Vehicle v) {
        return spots.stream().filter(s -> s.canFit(v)).findFirst();
    }
}

interface PricingStrategy {
    double calculateFee(VehicleType type, Duration duration);
}

class HourlyPricingStrategy implements PricingStrategy {
    private static final Map<VehicleType, Double> RATES = Map.of(
        VehicleType.MOTORCYCLE, 1.0,
        VehicleType.CAR, 2.0,
        VehicleType.LARGE, 3.5
    );

    public double calculateFee(VehicleType type, Duration duration) {
        long hours = Math.max(1, duration.toHours());
        return RATES.get(type) * hours;
    }
}

class Ticket {
    private final String ticketId;
    private final Instant entryTime;
    private final ParkingSpot spot;
    private final Vehicle vehicle;

    public Ticket(String ticketId, ParkingSpot spot, Vehicle vehicle) {
        this.ticketId = ticketId;
        this.entryTime = Instant.now();
        this.spot = spot;
        this.vehicle = vehicle;
    }

    public double calculateFee(PricingStrategy strategy) {
        Duration parked = Duration.between(entryTime, Instant.now());
        return strategy.calculateFee(vehicle.getType(), parked);
    }
}

class ParkingLot {
    private final List<Level> levels;
    private final PricingStrategy pricingStrategy;
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();

    public ParkingLot(List<Level> levels, PricingStrategy pricingStrategy) {
        this.levels = levels;
        this.pricingStrategy = pricingStrategy;
    }

    public Ticket parkVehicle(Vehicle vehicle) {
        for (Level level : levels) {
            Optional<ParkingSpot> spot = level.findAvailableSpot(vehicle);
            if (spot.isPresent()) {
                spot.get().park(vehicle);
                Ticket ticket = new Ticket(UUID.randomUUID().toString(), spot.get(), vehicle);
                activeTickets.put(ticket.getTicketId(), ticket);
                return ticket;
            }
        }
        throw new IllegalStateException("Parking lot is full for this vehicle type");
    }

    public double unparkVehicle(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) throw new IllegalArgumentException("Invalid ticket");
        double fee = ticket.calculateFee(pricingStrategy);
        ticket.getSpot().unpark();
        return fee;
    }
}
```

*(Skeleton omits a few getters/imports for brevity — flesh out `Ticket.getSpot()`/`getTicketId()` etc. when actually coding this live.)*

---

## 7. Follow-ups / Extensions to Practice Once Core Is Solid

- Multi-floor navigation — shortest path to the assigned spot
- `ParkingLotObserver` for a "lot full" notification (Observer pattern)
- `Singleton` for `ParkingLot` if the interviewer expects exactly one instance system-wide — be ready to discuss the testability trade-off this introduces
- Thread-safety hardening — the skeleton above uses `ConcurrentHashMap` for tickets but doesn't yet fully guard against a race on `spot.park()` between two threads calling `parkVehicle()` simultaneously; worth talking through a `synchronized` block or an atomic `compareAndSet`-style spot state.

---

## 8. Suggested Resources

- HelloInterview — [Parking Lot Low Level Design](https://www.hellointerview.com/learn/low-level-design/problem-breakdowns/parking-lot) — full walkthrough including the interviewer Q&A dialogue; good to compare your own requirements-gathering against theirs.
- HelloInterview — [How to Prepare for a Low-Level Design Interview](https://www.hellointerview.com/blog/how-to-prepare-lld) — the general requirements → entities → design framework this file follows.
- Cross-reference: this same requirements → entities → class diagram → design decisions → code skeleton flow applies directly to Elevator (Week 3) and Rate Limiter (Week 4).

---

## 9. Self-Test Questions

1. Why is `ParkingSpot` → `Vehicle` an association rather than composition?
2. Where would you extract an interface if the interviewer asked you to support reserved/VIP spots?
3. What would break if `canFit()` lived on `ParkingLot` instead of `ParkingSpot`, once a new spot type is added?
4. How would you change the design to support multiple simultaneous entry gates?
5. What's the concurrency risk in the current `parkVehicle()` implementation, and how would you fix it?
