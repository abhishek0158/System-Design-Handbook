# Parking Lot (LLD)

## Problem

Design a parking lot system. The system must support many vehicle types (car, bike, truck). It must support many parking spot sizes. It must support multiple floors. When a vehicle enters, the system must find a free spot and give a ticket. When a vehicle leaves, the system must calculate the fee and free the spot.

This is a common machine-coding interview question. The goal is not just "make it work." The goal is to show clean object-oriented design. The interviewer wants to see SOLID principles and the right design patterns.

## Requirements & Clarifying Questions

Before coding, ask these questions. In an interview, always clarify scope first.

**Functional requirements (assumed for this design):**

1. The parking lot has multiple floors. Each floor has multiple spots.
2. Spots come in sizes: `MOTORCYCLE`, `COMPACT`, `LARGE`.
3. Vehicles come in types: `Bike`, `Car`, `Truck`. Each vehicle type fits one or more spot sizes.
4. On entry, the system finds the nearest free spot that fits the vehicle. It issues a `Ticket` with entry time and spot info.
5. On exit, the system calculates the fee. It uses the ticket's entry time and a pricing rule. Then it frees the spot.
6. Multiple entries and exits can happen at the same time. The design must be thread-safe.

**Clarifying questions to ask the interviewer:**

- Can one spot size fit more than one vehicle type? (Assumption: yes. A `LARGE` spot can also fit a `Car` or `Bike` if no better spot is free.)
- Is pricing hourly, flat-rate, or does it change by vehicle type? (Assumption: pricing is pluggable. We support hourly and flat-rate. We may need day-based or slab-based pricing later.)
- Do we need payment processing? (Out of scope. We only compute the fee amount.)
- Do we need a display board showing free spot counts? (Out of scope for now. Mentioned as an extension.)
- Is there a limit on entry gates or exit gates? (Out of scope. We assume one entry point method and one exit point method.)
- What happens if the lot is full? (The system returns an error / throws an exception, no ticket is issued.)

## Design / Approach

### Key design decisions

1. **`Vehicle` is abstract.** `Car`, `Bike`, `Truck` extend it. Each vehicle reports the `VehicleType` it needs. This follows the **Open/Closed Principle**: adding a new vehicle type (say, `Bus`) does not change existing code.

2. **`ParkingSpot` knows its own `SpotSize`.** It does not know about pricing or vehicle rules. This is **Single Responsibility Principle** (SRP): a spot's only job is to hold or release one vehicle.

3. **Spot assignment uses a `SpotFindingStrategy` interface.** Today we use "nearest free spot on the lowest floor number." Tomorrow we could add "nearest to the exit" or "load-balanced across floors." This is the **Strategy Pattern**.

4. **Fee calculation uses a `PricingStrategy` interface.** `HourlyPricingStrategy` and `FlatRatePricingStrategy` implement it. This is also **Strategy Pattern**. It follows the **Dependency Inversion Principle** (DIP): `ParkingLot` depends on the `PricingStrategy` interface, not on a concrete pricing class.

5. **`ParkingLot` is the Facade.** Client code calls only two methods: `parkVehicle(vehicle)` and `unparkVehicle(ticketId)`. Internally it coordinates floors, spots, tickets, and pricing. This is the **Facade Pattern**. It also follows **SRP**: `ParkingLot` orchestrates; it does not do spot-matching logic or fee math itself.

6. **`ParkingLot` is a Singleton.** A building has exactly one parking lot object in memory. We use a thread-safe singleton (enum-based or double-checked locking).

7. **A simple factory method `VehicleFactory`** builds vehicle objects from a type code. This keeps object creation in one place, which helps if construction logic grows (for example, reading vehicle data from a database row).

8. **Each `ParkingSpot` has its own lock.** This gives fine-grained thread-safety. Two threads parking in two different spots do not block each other. Only threads competing for the *same* spot block each other.

### Class relationships (ASCII sketch)

```
                         +-------------------+
                         |     Vehicle       |  (abstract)
                         +-------------------+
                         | licensePlate      |
                         | getType()         |
                         +---------^---------+
                    +----------+---------+----------+
                    |                    |           |
             +------+-----+      +-------+---+  +----+------+
             |    Car     |      |    Bike   |  |   Truck   |
             +------------+      +-----------+  +-----------+

  +------------------+        finds spot via         +------------------------+
  |   ParkingLot     |------------------------------->|  SpotFindingStrategy   |<<interface>>
  |  (Singleton,     |                                +------------------------+
  |   Facade)        |                                | findSpot(lot, vehicle) |
  +------------------+                                +-----------^------------+
  | floors: List     |                                            |
  | parkVehicle()    |                                +-----------+------------+
  | unparkVehicle()  |                                | NearestSpotStrategy    |
  +---+----------+---+                                +------------------------+
      | has many         computes fee via
      v                  +------------------------+
  +------------+         |   PricingStrategy      |<<interface>>
  | ParkingFloor|         +------------------------+
  +------------+          | calculateFee(ticket)   |
  | spots: List|          +-----------^------------+
  +---+--------+                      |
      | has many        +-------------+--------------+
      v                 |                             |
  +--------------+  +----------------------+  +--------------------------+
  | ParkingSpot  |  | HourlyPricingStrategy|  | FlatRatePricingStrategy  |
  +--------------+  +----------------------+  +--------------------------+
  | size         |
  | vehicle      |
  | occupy/free()|         +---------------+
  +--------------+         |    Ticket     |
                            +---------------+
                            | id            |
                            | vehicle       |
                            | spot          |
                            | entryTime     |
                            +---------------+

  +------------------+
  |  VehicleFactory   |  (Factory Pattern - creates Car/Bike/Truck)
  +------------------+
```

**Patterns used:** Singleton (`ParkingLot`), Facade (`ParkingLot`), Strategy (`PricingStrategy`, `SpotFindingStrategy`), Factory Method (`VehicleFactory`).

## Java Solution

```java
// ===== Enums =====

public enum VehicleType {
    BIKE, CAR, TRUCK
}

public enum SpotSize {
    MOTORCYCLE, COMPACT, LARGE
}

// ===== Vehicle hierarchy =====

public abstract class Vehicle {
    private final String licensePlate;

    protected Vehicle(String licensePlate) {
        this.licensePlate = licensePlate;
    }

    public String getLicensePlate() {
        return licensePlate;
    }

    // Each vehicle knows which spot size it fits into at minimum.
    public abstract VehicleType getType();

    public abstract SpotSize minimumSpotSize();
}

public class Bike extends Vehicle {
    public Bike(String licensePlate) { super(licensePlate); }
    public VehicleType getType() { return VehicleType.BIKE; }
    public SpotSize minimumSpotSize() { return SpotSize.MOTORCYCLE; }
}

public class Car extends Vehicle {
    public Car(String licensePlate) { super(licensePlate); }
    public VehicleType getType() { return VehicleType.CAR; }
    public SpotSize minimumSpotSize() { return SpotSize.COMPACT; }
}

public class Truck extends Vehicle {
    public Truck(String licensePlate) { super(licensePlate); }
    public VehicleType getType() { return VehicleType.TRUCK; }
    public SpotSize minimumSpotSize() { return SpotSize.LARGE; }
}

// ===== Factory =====

public class VehicleFactory {
    private VehicleFactory() { }

    public static Vehicle create(VehicleType type, String licensePlate) {
        switch (type) {
            case BIKE:  return new Bike(licensePlate);
            case CAR:   return new Car(licensePlate);
            case TRUCK: return new Truck(licensePlate);
            default:    throw new IllegalArgumentException("Unknown vehicle type: " + type);
        }
    }
}

// ===== ParkingSpot =====

public class ParkingSpot {
    private final String spotId;
    private final int floorNumber;
    private final SpotSize size;
    private volatile Vehicle parkedVehicle; // null when free
    private final Object lock = new Object(); // fine-grained lock per spot

    public ParkingSpot(String spotId, int floorNumber, SpotSize size) {
        this.spotId = spotId;
        this.floorNumber = floorNumber;
        this.size = size;
    }

    // Returns true if this spot's size can hold the given vehicle.
    public boolean canFit(Vehicle vehicle) {
        return size.ordinal() >= vehicle.minimumSpotSize().ordinal();
    }

    public boolean isFree() {
        return parkedVehicle == null;
    }

    // Atomically claims the spot. Returns false if another thread got it first.
    public boolean tryOccupy(Vehicle vehicle) {
        synchronized (lock) {
            if (parkedVehicle != null) {
                return false;
            }
            parkedVehicle = vehicle;
            return true;
        }
    }

    public void free() {
        synchronized (lock) {
            parkedVehicle = null;
        }
    }

    public String getSpotId() { return spotId; }
    public int getFloorNumber() { return floorNumber; }
    public SpotSize getSize() { return size; }
}

// ===== ParkingFloor =====

public class ParkingFloor {
    private final int floorNumber;
    private final List<ParkingSpot> spots;

    public ParkingFloor(int floorNumber, List<ParkingSpot> spots) {
        this.floorNumber = floorNumber;
        this.spots = spots;
    }

    public int getFloorNumber() { return floorNumber; }
    public List<ParkingSpot> getSpots() { return spots; }
}

// ===== Ticket =====

public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;

    public Ticket(String ticketId, Vehicle vehicle, ParkingSpot spot, LocalDateTime entryTime) {
        this.ticketId = ticketId;
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = entryTime;
    }

    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getSpot() { return spot; }
    public LocalDateTime getEntryTime() { return entryTime; }
}

// ===== Spot-finding strategy =====

public interface SpotFindingStrategy {
    // Returns an empty Optional if no spot fits.
    Optional<ParkingSpot> findSpot(List<ParkingFloor> floors, Vehicle vehicle);
}

// Picks the smallest fitting spot on the lowest floor number first.
public class NearestFitSpotStrategy implements SpotFindingStrategy {
    @Override
    public Optional<ParkingSpot> findSpot(List<ParkingFloor> floors, Vehicle vehicle) {
        return floors.stream()
                .sorted(Comparator.comparingInt(ParkingFloor::getFloorNumber))
                .flatMap(floor -> floor.getSpots().stream())
                .filter(ParkingSpot::isFree)
                .filter(spot -> spot.canFit(vehicle))
                .sorted(Comparator.comparingInt(spot -> spot.getSize().ordinal())) // smallest fit first
                .findFirst();
    }
}

// ===== Pricing strategy =====

public interface PricingStrategy {
    double calculateFee(Ticket ticket, LocalDateTime exitTime);
}

public class HourlyPricingStrategy implements PricingStrategy {
    private final Map<VehicleType, Double> ratePerHour;

    public HourlyPricingStrategy(Map<VehicleType, Double> ratePerHour) {
        this.ratePerHour = ratePerHour;
    }

    @Override
    public double calculateFee(Ticket ticket, LocalDateTime exitTime) {
        long minutes = Duration.between(ticket.getEntryTime(), exitTime).toMinutes();
        long billableHours = Math.max(1, (long) Math.ceil(minutes / 60.0)); // round up, min 1 hour
        double rate = ratePerHour.getOrDefault(ticket.getVehicle().getType(), 0.0);
        return billableHours * rate;
    }
}

public class FlatRatePricingStrategy implements PricingStrategy {
    private final double flatFee;

    public FlatRatePricingStrategy(double flatFee) {
        this.flatFee = flatFee;
    }

    @Override
    public double calculateFee(Ticket ticket, LocalDateTime exitTime) {
        return flatFee;
    }
}

// ===== ParkingLot (Singleton + Facade) =====

public class ParkingLot {
    private static volatile ParkingLot instance;

    private final List<ParkingFloor> floors;
    private final SpotFindingStrategy spotFindingStrategy;
    private final PricingStrategy pricingStrategy;
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();
    private final AtomicLong ticketSequence = new AtomicLong(0);

    private ParkingLot(List<ParkingFloor> floors,
                        SpotFindingStrategy spotFindingStrategy,
                        PricingStrategy pricingStrategy) {
        this.floors = floors;
        this.spotFindingStrategy = spotFindingStrategy;
        this.pricingStrategy = pricingStrategy;
    }

    public static ParkingLot initialize(List<ParkingFloor> floors,
                                         SpotFindingStrategy spotFindingStrategy,
                                         PricingStrategy pricingStrategy) {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot(floors, spotFindingStrategy, pricingStrategy);
                }
            }
        }
        return instance;
    }

    public static ParkingLot getInstance() {
        if (instance == null) {
            throw new IllegalStateException("ParkingLot not initialized yet");
        }
        return instance;
    }

    // Entry flow: find a spot, claim it, issue a ticket.
    public Ticket parkVehicle(Vehicle vehicle) {
        // Retry once if another thread grabs the same spot between "find" and "claim".
        for (int attempt = 0; attempt < 3; attempt++) {
            Optional<ParkingSpot> candidate = spotFindingStrategy.findSpot(floors, vehicle);
            if (candidate.isEmpty()) {
                throw new IllegalStateException("Parking lot is full for vehicle type: " + vehicle.getType());
            }
            ParkingSpot spot = candidate.get();
            if (spot.tryOccupy(vehicle)) {
                String ticketId = "T-" + ticketSequence.incrementAndGet();
                Ticket ticket = new Ticket(ticketId, vehicle, spot, LocalDateTime.now());
                activeTickets.put(ticketId, ticket);
                return ticket;
            }
            // Someone else took it first; loop and try again.
        }
        throw new IllegalStateException("Could not assign a spot after retries; try again");
    }

    // Exit flow: look up the ticket, calculate fee, free the spot.
    public double unparkVehicle(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) {
            throw new NoSuchElementException("Invalid or already closed ticket: " + ticketId);
        }
        double fee = pricingStrategy.calculateFee(ticket, LocalDateTime.now());
        ticket.getSpot().free();
        return fee;
    }
}
```

## How It Works

**Entry (parking a vehicle):**

1. Client calls `ParkingLot.getInstance().parkVehicle(new Car("KA-01-1234"))`.
2. `ParkingLot` asks the `SpotFindingStrategy` for a free spot that fits the car.
3. `NearestFitSpotStrategy` scans floors in order, filters free spots, filters spots that fit, and picks the smallest fitting spot.
4. `ParkingLot` calls `tryOccupy()` on that spot. This is an atomic check-and-set inside a `synchronized` block, so two threads cannot both claim the same spot.
5. If the claim succeeds, a `Ticket` is created with a unique ID and the current time. It is stored in `activeTickets`.
6. If the claim fails (another thread won the race), the loop retries and finds a different spot.

**Exit (unparking a vehicle):**

1. Client calls `ParkingLot.getInstance().unparkVehicle(ticketId)`.
2. The ticket is removed from `activeTickets`. `ConcurrentHashMap.remove` is atomic, so two threads cannot both process the same ticket.
3. `PricingStrategy.calculateFee()` computes the amount using entry time, exit time, and vehicle type.
4. The spot is freed, so it becomes available for the next vehicle.

**Why Strategy for pricing:** `ParkingLot` never checks "if hourly, do X, else do Y." It just calls `pricingStrategy.calculateFee(...)`. Swapping `HourlyPricingStrategy` for `FlatRatePricingStrategy` needs no change inside `ParkingLot`. This is the Open/Closed Principle in action.

**Why Strategy for spot-finding too:** Same idea. If the business wants "assign spots near the elevator first," we write a new class that implements `SpotFindingStrategy`. `ParkingLot` code does not change.

## How to Extend (Follow-ups)

Interviewers often ask "what if we add X?" right after the base design. Here is how this design handles common follow-ups:

- **Multiple entry/exit gates:** Add an `EntryGate` and `ExitGate` class. Each gate calls the same `ParkingLot` methods. Since `ParkingLot` is a thread-safe singleton, multiple gates work concurrently without extra change.
- **Reserved / handicapped spots:** Add a `SpotCategory` enum (`REGULAR`, `HANDICAPPED`, `EV_CHARGING`). Add a filter step in a new `SpotFindingStrategy` implementation.
- **Display board (free spot counts per floor):** Add an `Observer` pattern. `ParkingSpot` or `ParkingFloor` notifies a `DisplayBoard` listener whenever a spot is occupied or freed. `DisplayBoard` keeps a running count; no need to scan all spots on every query.
- **Slab-based or peak-hour pricing:** Add a new class implementing `PricingStrategy`, for example `SlabPricingStrategy` (first hour 50, next hours 30 each) or `PeakHourPricingStrategy` (checks entry time against peak windows). Plug it in at `ParkingLot.initialize()`.
- **Payment integration:** Add a `PaymentProcessor` interface with implementations like `CardPaymentProcessor` and `CashPaymentProcessor`. Call it inside `unparkVehicle()` after computing the fee, but keep fee calculation and payment as two separate concerns (SRP).
- **Vehicle-specific spot preference (e.g., trucks only on ground floor):** Encode this as a rule inside the `SpotFindingStrategy` filter chain, or make it configurable via a `SpotEligibilityRule` list passed into the strategy.
- **Lost ticket handling:** Add a fallback fee, for example `LostTicketPricingStrategy`, chosen when the client cannot supply a valid `ticketId` and instead scans by license plate.

## Complexity & Thread-Safety Notes

**Time complexity:**

- `parkVehicle()`: `O(S)` where `S` is the total number of spots, because `NearestFitSpotStrategy` scans all spots in the worst case (lot almost full). This can be improved to near `O(1)` average case by keeping a `TreeMap<SpotSize, Queue<ParkingSpot>>` of free spots per floor, updated on occupy/free. That trade-off is worth mentioning in an interview: simple linear scan first, then optimize with an index if asked.
- `unparkVehicle()`: `O(1)`, since it is a hash map lookup plus a fee formula.

**Thread-safety:**

- `ParkingLot` uses **double-checked locking** for singleton creation. This is safe only because `instance` is `volatile`. Without `volatile`, a thread could see a partially constructed object.
- `activeTickets` is a `ConcurrentHashMap`. Multiple threads can insert and remove tickets concurrently without a global lock.
- Each `ParkingSpot` has **its own lock** (an object monitor via `synchronized`). This is fine-grained locking: two threads parking in two different spots never block each other. Only threads racing for the exact same spot block each other, and only for a very short time (the size of `tryOccupy`).
- `ticketSequence` uses `AtomicLong` so ticket IDs never collide, even under concurrent entries.
- There is a known race in `parkVehicle()`: between "find a candidate spot" (a read-only scan) and "claim it" (`tryOccupy`), another thread might occupy the same spot first. We handle this with an optimistic retry loop (try up to 3 times), rather than locking the whole floor. This keeps throughput high, because we do not serialize all entries behind one big lock.
- We deliberately avoid a single global lock on the whole `ParkingLot` for `parkVehicle()`. A global lock would be simpler to reason about, but it would force every entry across every floor to wait in line, even when they target different spots. Fine-grained locking scales much better under real concurrent load.

## Interview Tips & Common Mistakes

- **Do not hardcode pricing logic inside `ParkingLot`.** This is the single most common mistake. If you write `if (vehicleType == CAR) fee = hours * 10;` inside `ParkingLot`, you fail the Open/Closed Principle and the interviewer will ask "how do you add a new pricing model without touching this class?" Use `PricingStrategy` from the start.
- **Do not let `Vehicle` know about `ParkingSpot` size mapping in a giant `if-else`.** Keep `minimumSpotSize()` as a method on each vehicle subclass. This is polymorphism doing the work instead of conditionals.
- **Show that you understand the "check-then-act" race condition** on spot assignment. Many candidates find a free spot and just mark it occupied without discussing what happens if two threads pick the same spot. Mention `synchronized`, `compareAndSet`-style logic, or a retry loop.
- **Do not use one big lock on the whole parking lot object** for every operation, and say so explicitly. Interviewers reward candidates who discuss lock granularity trade-offs (global lock = simple but slow, per-spot lock = fast but more code).
- **Keep `Ticket` immutable.** Its entry time and assigned spot should never change after creation. Mutable tickets are a common source of subtle bugs, especially with concurrent exits.
- **Be ready to draw the class diagram from memory** and explain, in one sentence each, why `ParkingLot` is a Facade, why pricing and spot-finding are Strategies, and why vehicle creation is a Factory. Interviewers care more about *why* you picked a pattern than about reciting its name.
- **Do not over-engineer up front.** Do not add `Observer` for a display board, or slab pricing, unless asked. Mention these as "How to Extend" ideas instead. A focused, correct core design scores higher than a bloated one with half-finished features.
- **Clarify before coding.** Spend the first two minutes on the clarifying questions listed above. This shows structured thinking and avoids wasted code for requirements the interviewer did not want.
