# Cab Booking (LLD)

## Problem

Design a cab booking system, like Uber or Ola. The system has three kinds of users and data:

- **Riders.** A person who wants a ride. A rider has a name and a current location.
- **Drivers.** A person who drives a cab. A driver has a name, a cab, a current location, and an availability status (free or busy).
- **Cabs.** A vehicle. A cab has a type (mini, sedan, SUV) and a plate number.
- **Trips.** One ride, from a source location to a destination location.

A location is a point on a map. We store it as latitude and longitude.

The system must support:

1. A driver updates their current location and their availability (online/offline).
2. A rider requests a ride from a source to a destination.
3. **Matching.** The system finds nearby, available drivers and picks one. This is the core part of the problem. We need a **matching strategy** — the simple rule is "pick the nearest driver." A real system uses a spatial index (a **geohash** or a **quadtree**) to find nearby drivers fast, instead of scanning every driver in the city. We will keep our index simple, but the design must allow swapping in a real geo index later.
4. The chosen driver accepts, the trip starts, and the trip ends.
5. **Fare calculation.** The fare uses a **pricing strategy**: a base fare, plus a rate per kilometer, plus a rate per minute, times a surge multiplier during high demand.
6. **Concurrency.** Two riders must never get assigned the same driver at the same time. Claiming a driver must be an atomic step — one operation that either fully succeeds or fully fails, with no in-between state another thread can see.
7. A trip moves through a clear state machine: `REQUESTED → ASSIGNED → STARTED → COMPLETED` or `CANCELLED`.

This is a classic LLD (low-level design) interview question. It tests object modeling, the Strategy pattern (used twice — once for matching, once for pricing), a state machine for the trip lifecycle, and correct handling of a shared, mutable resource (the driver) under concurrent access.

## Requirements & Clarifying Questions

Ask these before coding. They show the interviewer that you scope the problem first.

1. **Do we need a real map and real road distance?** No. We use straight-line ("as the crow flies") distance between latitude/longitude points, using the haversine formula. Real road distance and ETA would come from a routing service — out of scope, noted as a follow-up.
2. **How do we find "nearby" drivers — do we need a real geo index?** Not for the base solution. We scan all available drivers and filter by straight-line distance. We will design the driver store behind an interface, so a geohash or quadtree index can replace the linear scan later without changing the matching strategy or the booking service.
3. **What matching rule do we use?** "Nearest available driver," as a pluggable `DriverMatchingStrategy`. The interviewer may later ask for a different rule — for example, "prefer drivers rated above 4.5 stars," or "balance load across drivers." A strategy interface makes that a small change, not a rewrite.
4. **Does a driver explicitly accept or reject a ride, with a timeout?** In a real app, yes — the driver's phone shows a request and a countdown timer. For this design, we simplify: the matching step directly tries to **claim** the nearest driver. If the claim succeeds, that driver is assigned. If it fails (another request claimed the same driver a moment earlier), the system tries the next-nearest candidate. We note the explicit accept/reject/timeout flow as a follow-up.
5. **Can a trip be cancelled?** Yes, before it starts (`REQUESTED` or `ASSIGNED`) by either side. Once `STARTED`, it can only reach `COMPLETED` (in the base design — cancellation mid-trip is a follow-up).
6. **How is surge pricing decided?** We keep it simple: a `SurgePricingProvider` returns a multiplier for a given area. The real logic (ratio of open ride requests to available drivers in a zone) is out of scope, but the interface makes it swappable.
7. **Is this single-threaded or concurrent?** Concurrent. Many riders request rides at the same time, from different threads (think of many API requests hitting the booking service at once). The design must guarantee one driver never serves two trips at once.

## Design / Approach

```
                    ┌───────────────────────────────────────────┐
                    │               BookingService                │  (Facade)
                    │ - DriverRegistry registry                    │
                    │ - DriverMatchingStrategy matchingStrategy     │  (Strategy)
                    │ - FareStrategy fareStrategy                   │  (Strategy)
                    │ - Map<String, Trip> trips                     │
                    │ + requestRide(rider, source, dest): Trip      │
                    │ + startTrip(tripId)                           │
                    │ + endTrip(tripId, endLocation)                │
                    │ + cancelTrip(tripId)                          │
                    └───────────────┬───────────────┬─────────────┘
                                    │               │
                    uses            │               │ uses
           ┌────────────────────────┘               └────────────────────────┐
           ▼                                                                  ▼
┌───────────────────────────┐                                  ┌───────────────────────────┐
│      DriverRegistry         │                                  │   DriverMatchingStrategy    │ (interface)
│ - Map<id, Driver> drivers    │                                  │ + rankCandidates(pickup,    │
│ + register(driver)           │                                  │     drivers, limit): List   │
│ + updateLocation(id, loc)    │                                  └──────────────┬─────────────┘
│ + findAvailableNear(loc, r)  │◄── linear scan (swap for                        │ implemented by
└───────────────────────────┘     geohash/quadtree at scale)                    ▼
                                                                     ┌───────────────────────────┐
┌───────────────────────────┐                                       │   NearestDriverStrategy     │
│           Driver             │                                       └───────────────────────────┘
│ - AtomicReference<Status>    │
│ + tryClaim(): boolean  ◄──── atomic CAS: AVAILABLE -> ASSIGNED (the concurrency guard)
│ + release()                  │
└───────────────────────────┘
                                                                  ┌───────────────────────────┐
┌───────────────────────────┐                                    │        FareStrategy          │ (interface)
│            Trip              │                                    │ + calculateFare(trip): double│
│ - AtomicReference<TripStatus>│                                    └──────────────┬─────────────┘
│ + transitionTo(target): bool │  <- state machine, guarded by                     │ implemented by
│     an allowed-transitions map│                                                  ▼
└───────────────────────────┘                                       ┌───────────────────────────┐
                                                                      │   StandardFareStrategy       │
     Location (lat, lng)                                             │ base + perKm + perMinute,     │
     + distanceKm(other)                                              │ x SurgePricingProvider        │
                                                                      └───────────────────────────┘
```

**Patterns used:**

- **Strategy (matching).** `DriverMatchingStrategy` separates "how we rank candidate drivers" from the booking flow. `NearestDriverStrategy` is one rule; a future `HighestRatedDriverStrategy` plugs in without touching `BookingService`.
- **Strategy (pricing).** `FareStrategy` separates "how we compute the fare" from the trip lifecycle. `StandardFareStrategy` uses base + per-km + per-minute, then applies a surge multiplier from a `SurgePricingProvider` (also a small strategy interface).
- **State machine.** `Trip` holds its status in an `AtomicReference<TripStatus>` and only changes it through `transitionTo`, which checks a static map of allowed transitions. No code outside `Trip` can push it into an invalid state (for example, `COMPLETED` directly from `REQUESTED`).
- **Facade.** `BookingService` is the single entry point. Riders and drivers never touch `Driver`, `Trip`, or `DriverRegistry` internals directly — they call `requestRide`, `startTrip`, `endTrip`, `cancelTrip`.
- **Atomic claim (the key concurrency idea).** `Driver.tryClaim()` uses `AtomicReference.compareAndSet(AVAILABLE, ASSIGNED)`. This is a single hardware-level atomic operation: it reads the current status and writes the new status as one indivisible step. Two threads racing to claim the same driver will never both succeed — exactly one `compareAndSet` call returns `true`.

**Why not just "find nearest driver, then assign"?** If matching and assignment are two separate steps ("read driver's status, see it is free" then later "mark it busy"), two threads can both read "free" before either writes "busy." Both riders then get the same driver. This is a classic **check-then-act race condition**. The fix is to make "check and act" one atomic operation — `compareAndSet` does exactly this.

## Java Solution

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicReference;
import java.util.stream.Collectors;

// ---------- Location: a point on the map ----------

final class Location {
    final double latitude;
    final double longitude;

    Location(double latitude, double longitude) {
        this.latitude = latitude;
        this.longitude = longitude;
    }

    /** Straight-line distance in kilometers, using the haversine formula. */
    double distanceKm(Location other) {
        final double earthRadiusKm = 6371.0;
        double dLat = Math.toRadians(other.latitude - this.latitude);
        double dLon = Math.toRadians(other.longitude - this.longitude);
        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2)
                + Math.cos(Math.toRadians(this.latitude)) * Math.cos(Math.toRadians(other.latitude))
                * Math.sin(dLon / 2) * Math.sin(dLon / 2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return earthRadiusKm * c;
    }
}

// ---------- People and vehicles ----------

enum CabType { MINI, SEDAN, SUV }

final class Cab {
    final String id;
    final String plateNumber;
    final CabType type;

    Cab(String id, String plateNumber, CabType type) {
        this.id = id;
        this.plateNumber = plateNumber;
        this.type = type;
    }
}

final class Rider {
    final String id;
    final String name;
    private volatile Location currentLocation;

    Rider(String id, String name, Location currentLocation) {
        this.id = id;
        this.name = name;
        this.currentLocation = currentLocation;
    }

    Location getCurrentLocation() { return currentLocation; }
    void updateLocation(Location location) { this.currentLocation = location; }
}

enum DriverStatus { OFFLINE, AVAILABLE, ASSIGNED, ON_TRIP }

class Driver {
    private final String id;
    private final String name;
    private final Cab cab;
    // volatile: many threads read location for matching, one thread (this driver's
    // app) writes it. We only need visibility here, not atomicity.
    private volatile Location currentLocation;
    // AtomicReference: the atomic claim primitive. This is the field that must
    // never let two concurrent requests both "win" the same driver.
    private final AtomicReference<DriverStatus> status = new AtomicReference<>(DriverStatus.OFFLINE);

    Driver(String id, String name, Cab cab, Location startLocation) {
        this.id = id;
        this.name = name;
        this.cab = cab;
        this.currentLocation = startLocation;
    }

    String getId() { return id; }
    String getName() { return name; }
    Cab getCab() { return cab; }
    Location getCurrentLocation() { return currentLocation; }
    DriverStatus getStatus() { return status.get(); }

    void updateLocation(Location location) { this.currentLocation = location; }

    void goOnline() { status.compareAndSet(DriverStatus.OFFLINE, DriverStatus.AVAILABLE); }
    void goOffline() { status.compareAndSet(DriverStatus.AVAILABLE, DriverStatus.OFFLINE); }

    /**
     * The atomic claim. Succeeds only if this driver is currently AVAILABLE.
     * This single call is what stops two riders from getting the same driver:
     * compareAndSet is one indivisible hardware-level operation.
     */
    boolean tryClaim() {
        return status.compareAndSet(DriverStatus.AVAILABLE, DriverStatus.ASSIGNED);
    }

    void markOnTrip() { status.compareAndSet(DriverStatus.ASSIGNED, DriverStatus.ON_TRIP); }

    /** Trip ended or was cancelled: driver is free again. */
    void release() {
        DriverStatus current = status.get();
        if (current == DriverStatus.ASSIGNED || current == DriverStatus.ON_TRIP) {
            status.set(DriverStatus.AVAILABLE);
        }
    }
}

// ---------- Trip: the state machine ----------

enum TripStatus { REQUESTED, ASSIGNED, STARTED, COMPLETED, CANCELLED }

class Trip {
    private static final Map<TripStatus, Set<TripStatus>> ALLOWED_TRANSITIONS = Map.of(
            TripStatus.REQUESTED, EnumSet.of(TripStatus.ASSIGNED, TripStatus.CANCELLED),
            TripStatus.ASSIGNED, EnumSet.of(TripStatus.STARTED, TripStatus.CANCELLED),
            TripStatus.STARTED, EnumSet.of(TripStatus.COMPLETED),
            TripStatus.COMPLETED, EnumSet.noneOf(TripStatus.class),
            TripStatus.CANCELLED, EnumSet.noneOf(TripStatus.class)
    );

    private final String id;
    private final Rider rider;
    private final Location source;
    private final Location destination;
    private volatile Driver driver;
    private final AtomicReference<TripStatus> status = new AtomicReference<>(TripStatus.REQUESTED);
    private volatile long startedAtMillis;
    private volatile long endedAtMillis;
    private volatile double distanceKm;
    private volatile double fare;

    Trip(String id, Rider rider, Location source, Location destination) {
        this.id = id;
        this.rider = rider;
        this.source = source;
        this.destination = destination;
    }

    String getId() { return id; }
    Rider getRider() { return rider; }
    Location getSource() { return source; }
    Location getDestination() { return destination; }
    Driver getDriver() { return driver; }
    TripStatus getStatus() { return status.get(); }
    double getDistanceKm() { return distanceKm; }
    double getFare() { return fare; }
    void setFare(double fare) { this.fare = fare; }

    double getDurationMinutes() {
        if (startedAtMillis == 0 || endedAtMillis == 0) return 0.0;
        return (endedAtMillis - startedAtMillis) / 60000.0;
    }

    void assignDriver(Driver driver) { this.driver = driver; }

    void markStarted() { this.startedAtMillis = System.currentTimeMillis(); }

    void markEnded(Location endLocation) {
        this.endedAtMillis = System.currentTimeMillis();
        this.distanceKm = source.distanceKm(endLocation);
    }

    /**
     * Move to a new state only if the transition is allowed from the current
     * state. Uses a compare-and-set loop so a concurrent transition (for
     * example, a cancel racing a start) cannot corrupt the state — exactly
     * one caller wins, and losers get a clean "false" instead of a crash.
     */
    boolean transitionTo(TripStatus target) {
        while (true) {
            TripStatus current = status.get();
            if (!ALLOWED_TRANSITIONS.getOrDefault(current, Set.of()).contains(target)) {
                return false;
            }
            if (status.compareAndSet(current, target)) {
                return true;
            }
            // Another thread changed the status between our read and our
            // compareAndSet. Loop and re-check against the new current state.
        }
    }
}

// ---------- Matching strategy: pluggable "how do we pick a driver" rule ----------

interface DriverMatchingStrategy {
    /** Returns candidate drivers ordered best-first (nearest first, here). */
    List<Driver> rankCandidates(Location pickup, Collection<Driver> availableDrivers, int limit);
}

class NearestDriverStrategy implements DriverMatchingStrategy {
    @Override
    public List<Driver> rankCandidates(Location pickup, Collection<Driver> availableDrivers, int limit) {
        return availableDrivers.stream()
                .sorted(Comparator.comparingDouble(d -> d.getCurrentLocation().distanceKm(pickup)))
                .limit(limit)
                .collect(Collectors.toList());
    }
}

// ---------- Driver store: today a linear scan, tomorrow a geo index ----------

class DriverRegistry {
    // ConcurrentHashMap: safe for many threads to read/update driver entries
    // at once (drivers pinging their location every few seconds, plus
    // booking requests reading the whole set).
    private final Map<String, Driver> drivers = new ConcurrentHashMap<>();

    void register(Driver driver) { drivers.put(driver.getId(), driver); }

    void updateLocation(String driverId, Location location) {
        Driver driver = drivers.get(driverId);
        if (driver != null) driver.updateLocation(location);
    }

    Optional<Driver> find(String driverId) {
        return Optional.ofNullable(drivers.get(driverId));
    }

    /**
     * Finds available drivers within radiusKm of pickup.
     * NOTE: this is a linear scan over every registered driver — fine for a
     * few thousand drivers, but a real system indexes drivers by geohash or
     * a quadtree, so this lookup becomes "look up nearby grid cells" instead
     * of "check every driver in the city." Swapping the implementation here
     * does not require any change to DriverMatchingStrategy or BookingService.
     */
    List<Driver> findAvailableNear(Location pickup, double radiusKm) {
        return drivers.values().stream()
                .filter(d -> d.getStatus() == DriverStatus.AVAILABLE)
                .filter(d -> d.getCurrentLocation().distanceKm(pickup) <= radiusKm)
                .collect(Collectors.toList());
    }
}

// ---------- Pricing strategy: pluggable "how do we price a trip" rule ----------

interface SurgePricingProvider {
    double getSurgeMultiplier(Location area);
}

/** Fixed multiplier for this example. A real provider reads live supply/demand per zone. */
class SimpleSurgePricingProvider implements SurgePricingProvider {
    private final double multiplier;
    SimpleSurgePricingProvider(double multiplier) { this.multiplier = multiplier; }
    @Override public double getSurgeMultiplier(Location area) { return multiplier; }
}

interface FareStrategy {
    double calculateFare(Trip trip);
}

class StandardFareStrategy implements FareStrategy {
    private final double baseFare;
    private final double perKmRate;
    private final double perMinuteRate;
    private final SurgePricingProvider surgeProvider;

    StandardFareStrategy(double baseFare, double perKmRate, double perMinuteRate,
                          SurgePricingProvider surgeProvider) {
        this.baseFare = baseFare;
        this.perKmRate = perKmRate;
        this.perMinuteRate = perMinuteRate;
        this.surgeProvider = surgeProvider;
    }

    @Override
    public double calculateFare(Trip trip) {
        double distanceCost = perKmRate * trip.getDistanceKm();
        double timeCost = perMinuteRate * trip.getDurationMinutes();
        double surge = surgeProvider.getSurgeMultiplier(trip.getSource());
        return (baseFare + distanceCost + timeCost) * surge;
    }
}

// ---------- BookingService: the facade that ties everything together ----------

class BookingService {
    private final DriverRegistry driverRegistry;
    private final DriverMatchingStrategy matchingStrategy;
    private final FareStrategy fareStrategy;
    private final double searchRadiusKm;
    private final Map<String, Trip> trips = new ConcurrentHashMap<>();

    BookingService(DriverRegistry driverRegistry, DriverMatchingStrategy matchingStrategy,
                   FareStrategy fareStrategy, double searchRadiusKm) {
        this.driverRegistry = driverRegistry;
        this.matchingStrategy = matchingStrategy;
        this.fareStrategy = fareStrategy;
        this.searchRadiusKm = searchRadiusKm;
    }

    /**
     * Requests a ride. Finds nearby available drivers, ranks them, and tries
     * to atomically claim each in order until one succeeds. This loop is
     * what makes matching safe under concurrency: a candidate that another
     * thread claimed a moment earlier is simply skipped, not double-booked.
     */
    Trip requestRide(Rider rider, Location source, Location destination) {
        Trip trip = new Trip(UUID.randomUUID().toString(), rider, source, destination);
        trips.put(trip.getId(), trip);

        List<Driver> candidates = driverRegistry.findAvailableNear(source, searchRadiusKm);
        List<Driver> ranked = matchingStrategy.rankCandidates(source, candidates, 5);

        for (Driver candidate : ranked) {
            if (candidate.tryClaim()) {
                trip.assignDriver(candidate);
                trip.transitionTo(TripStatus.ASSIGNED);
                return trip;
            }
            // tryClaim failed: another request claimed this driver first.
            // Fall through and try the next-nearest candidate.
        }

        trip.transitionTo(TripStatus.CANCELLED); // no driver could be claimed
        return trip;
    }

    void startTrip(String tripId) {
        Trip trip = getTripOrThrow(tripId);
        if (!trip.transitionTo(TripStatus.STARTED)) {
            throw new IllegalStateException("Cannot start trip in state " + trip.getStatus());
        }
        trip.markStarted();
        trip.getDriver().markOnTrip();
    }

    void endTrip(String tripId, Location endLocation) {
        Trip trip = getTripOrThrow(tripId);
        if (!trip.transitionTo(TripStatus.COMPLETED)) {
            throw new IllegalStateException("Cannot end trip in state " + trip.getStatus());
        }
        trip.markEnded(endLocation);
        trip.setFare(fareStrategy.calculateFare(trip));

        Driver driver = trip.getDriver();
        driver.updateLocation(endLocation);
        driver.release(); // free for the next rider
    }

    void cancelTrip(String tripId) {
        Trip trip = getTripOrThrow(tripId);
        if (!trip.transitionTo(TripStatus.CANCELLED)) {
            throw new IllegalStateException("Cannot cancel trip in state " + trip.getStatus());
        }
        Driver driver = trip.getDriver();
        if (driver != null) driver.release();
    }

    private Trip getTripOrThrow(String tripId) {
        Trip trip = trips.get(tripId);
        if (trip == null) throw new NoSuchElementException("Unknown trip: " + tripId);
        return trip;
    }
}

// ---------- Demo ----------

public class CabBookingDemo {
    public static void main(String[] args) throws InterruptedException {
        DriverRegistry registry = new DriverRegistry();
        Driver asha = new Driver("D1", "Asha", new Cab("C1", "KA01AB1234", CabType.MINI),
                new Location(12.9716, 77.5946));
        Driver ravi = new Driver("D2", "Ravi", new Cab("C2", "KA01CD5678", CabType.SEDAN),
                new Location(12.9352, 77.6146));
        asha.goOnline();
        ravi.goOnline();
        registry.register(asha);
        registry.register(ravi);

        BookingService bookingService = new BookingService(
                registry,
                new NearestDriverStrategy(),
                new StandardFareStrategy(50.0, 12.0, 1.5, new SimpleSurgePricingProvider(1.2)),
                5.0 // search radius in km
        );

        // --- Normal flow: one rider, one trip, start to finish ---
        Rider meera = new Rider("R1", "Meera", new Location(12.9700, 77.5950));
        Trip trip = bookingService.requestRide(meera, meera.getCurrentLocation(),
                new Location(12.9279, 77.6271));
        System.out.println("Trip " + trip.getId() + " status=" + trip.getStatus()
                + " driver=" + (trip.getDriver() != null ? trip.getDriver().getName() : "none"));

        bookingService.startTrip(trip.getId());
        Thread.sleep(1500); // simulate travel time
        bookingService.endTrip(trip.getId(), new Location(12.9279, 77.6271));
        System.out.printf("Trip completed. distance=%.2f km, fare=Rs.%.2f%n",
                trip.getDistanceKm(), trip.getFare());

        // --- Concurrency demo: two riders race for the one remaining driver ---
        Driver kiran = new Driver("D3", "Kiran", new Cab("C3", "KA01EF9999", CabType.SUV),
                new Location(12.9716, 77.5946));
        kiran.goOnline();
        registry.register(kiran);

        Rider anu = new Rider("RA", "Anu", new Location(12.9716, 77.5946));
        Rider bala = new Rider("RB", "Bala", new Location(12.9720, 77.5950));

        ExecutorService pool = Executors.newFixedThreadPool(2);
        pool.submit(() -> bookingService.requestRide(anu, anu.getCurrentLocation(),
                new Location(12.99, 77.60)));
        pool.submit(() -> bookingService.requestRide(bala, bala.getCurrentLocation(),
                new Location(12.98, 77.61)));
        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);

        // Exactly one of the two requests claims Kiran; the loser gets CANCELLED
        // (no other driver was in range in this small example).
        System.out.println("Kiran's status after the race: " + kiran.getStatus());
    }
}
```

## How It Works

Walk through the normal flow first, then the race condition.

1. **Registering drivers.** `asha` and `ravi` call `goOnline()`, moving from `OFFLINE` to `AVAILABLE`. `registry.register(...)` puts them in the `ConcurrentHashMap`.
2. **Requesting a ride.** Rider `meera` calls `bookingService.requestRide(meera, source, destination)`. This creates a `Trip` in state `REQUESTED` and stores it in the `trips` map, keyed by a generated UUID.
3. **Finding candidates.** `driverRegistry.findAvailableNear(source, 5.0)` scans registered drivers, keeps only `AVAILABLE` ones within 5 km, using `Location.distanceKm` (haversine formula). `NearestDriverStrategy.rankCandidates` sorts that list by distance and returns the top 5.
4. **Claiming.** `requestRide` loops through the ranked list and calls `candidate.tryClaim()` on each, in order, until one returns `true`. `tryClaim` is `status.compareAndSet(AVAILABLE, ASSIGNED)` — one atomic operation. Since `asha` is nearer, she is tried first and succeeds. The trip is assigned to her, and `trip.transitionTo(ASSIGNED)` moves the trip from `REQUESTED` to `ASSIGNED` (an allowed transition, so it succeeds).
5. **Starting the trip.** `bookingService.startTrip(tripId)` calls `trip.transitionTo(STARTED)`. The transition map allows `ASSIGNED → STARTED`, so it succeeds. `trip.markStarted()` records the start time, and `driver.markOnTrip()` moves Asha from `ASSIGNED` to `ON_TRIP`.
6. **Ending the trip.** `bookingService.endTrip(tripId, endLocation)` calls `trip.transitionTo(COMPLETED)`, which succeeds because `STARTED → COMPLETED` is allowed. `trip.markEnded` records the end time and computes `distanceKm` from source to the actual end location. `fareStrategy.calculateFare(trip)` computes `(base + perKm * distance + perMinute * duration) * surge` and stores it on the trip. Finally, `driver.release()` moves Asha back to `AVAILABLE`, so she can be matched to the next rider.
7. **The race condition, made safe.** Two riders, `anu` and `bala`, call `requestRide` from two different threads at nearly the same instant. Both searches see `kiran` as the (only) nearby available driver, so both candidate lists contain him. Both threads call `kiran.tryClaim()`. Internally, `compareAndSet(AVAILABLE, ASSIGNED)` is backed by a single atomic CPU instruction (compare-and-swap). Exactly one thread's call sees the field still `AVAILABLE` and flips it to `ASSIGNED`, returning `true`. The other thread's call sees the field is no longer `AVAILABLE` (it is now `ASSIGNED`), so `compareAndSet` returns `false` — no exception, no corrupted state, just a clean "you lost the race." The losing request's loop moves to the next candidate in its ranked list; since no other driver is in range in this example, that rider's trip ends up `CANCELLED`. Kiran is assigned to exactly one rider, never both.
8. **Why the `Trip` state machine also needs this pattern.** Suppose one thread calls `cancelTrip` at almost the same moment another calls `startTrip` for the same trip. Both read `status.get()` as `ASSIGNED`. `transitionTo` checks the allowed-transitions map and then calls `compareAndSet(ASSIGNED, target)`. Only one of the two calls wins the compare-and-set; the other sees the field already changed and its `while` loop re-reads the new current status, finds the new target is not an allowed transition from it (for example, `STARTED → CANCELLED` is not in our transition map), and returns `false`. The caller then throws `IllegalStateException`, which is exactly the right behavior — you cannot cancel a trip that already started.

## How to Extend (Follow-ups)

Interviewers often push into these directions once the base design works:

1. **Real geo index.** Replace the linear scan in `DriverRegistry.findAvailableNear` with a **geohash** (encode latitude/longitude into a short string; nearby points share a string prefix) or a **quadtree** (a tree that splits the map into four quadrants recursively, so a search only visits a few nodes). Because `DriverMatchingStrategy` and `BookingService` only depend on the `List<Driver> findAvailableNear(...)` method signature, the swap is contained to `DriverRegistry`.
2. **Explicit driver accept/reject with a timeout.** Instead of `tryClaim` succeeding immediately, put the trip in a `PENDING_DRIVER_RESPONSE` state, push a notification to the driver's app, and start a timer. If the driver rejects or the timer expires, release the claim (`driver.release()`) and try the next candidate. This adds one more state to the machine and one more strategy-like collaborator (a notification service), but does not change the atomic-claim idea underneath.
3. **Driver ratings and rider ratings.** Add a `rating` field to `Driver` and `Rider`. Extend `DriverMatchingStrategy` with a rule such as `HighestRatedNearbyDriverStrategy`, or combine distance and rating into a single weighted score. This is a good spot to discuss the **Strategy** pattern's benefit again: no change needed to `BookingService`.
4. **Cab types and fare by type.** Let `requestRide` accept a requested `CabType`, filter candidates by matching cab type, and let `FareStrategy` implementations vary rates by type (for example, `SUV` costs more per km than `MINI`). This could become a `Map<CabType, FareStrategy>` chosen by the booking service.
5. **Mid-trip cancellation and cancellation fees.** Allow `STARTED → CANCELLED` in the transition map, with a `CancellationFeeStrategy` that charges a partial fare based on distance covered so far.
6. **Notifications.** Add an **Observer** pattern: `Trip` publishes events (`driverAssigned`, `tripStarted`, `tripCompleted`) to listeners such as a push-notification service or an ETA-tracking service, instead of `BookingService` calling them directly.
7. **Where this becomes a distributed system.** The design above works inside one process, one `DriverRegistry`, one JVM. A real ride-hailing platform runs many booking service instances behind a load balancer, with drivers spread across them. At that point:
    - `AtomicReference.compareAndSet` (in-memory, single-JVM) is no longer enough — the atomic claim must happen in a shared store, for example a Redis `SETNX`/compare-and-swap, or a database row update with `WHERE status = 'AVAILABLE'` and checking the row count changed.
    - Driver location updates arrive from many drivers, continuously; these are usually streamed through a message queue (Kafka) into a geo-indexed store (Redis geo commands, or Elasticsearch with geo queries) instead of an in-memory map.
    - Trip state needs to survive a server crash, so it lives in a database, with the state machine enforced by a `WHERE status = :current` conditional update — the same compare-and-set idea, just expressed as SQL instead of a Java atomic.
    - Matching, pricing, and trip management often become separate services communicating over the network, each independently scalable.

## Complexity & Thread-Safety Notes

- **`findAvailableNear`**: O(n) where n is the number of registered drivers — a full scan. With a geohash or quadtree index, this drops to roughly O(log n) or better, since only nearby grid cells or tree nodes are visited.
- **`rankCandidates`**: O(k log k) where k is the number of candidates returned by the search (sorting by distance).
- **`tryClaim`**: O(1) — one `compareAndSet` call.
- **`transitionTo`**: O(1) amortized — the `while` loop retries only if another thread changed the status in the tiny window between the read and the compare-and-set; this is rare and the loop is short (at most a few enum values to check).
- **Thread-safety choices:**
    - `Driver.status` and `Trip.status` are `AtomicReference`, not plain fields with `synchronized` blocks. This gives lock-free, non-blocking atomic updates — no thread ever waits to acquire a lock just to check or change a status, which matters when many riders request rides at once.
    - `Driver.currentLocation` is `volatile`. Many threads read it (every matching search); one source (the driver's own app, or the location-update endpoint) writes it. `volatile` guarantees every reader sees the latest write — visibility — which is all we need, since only one logical writer exists per driver.
    - `DriverRegistry`'s backing map is a `ConcurrentHashMap`, safe for concurrent reads and writes without external locking — needed because driver registration, location updates, and matching searches all happen concurrently from different threads.
    - `BookingService.trips` is also a `ConcurrentHashMap`, since many trips are created, started, and ended concurrently.
    - The core safety guarantee — "one driver never serves two trips at once" — comes entirely from `Driver.tryClaim()`'s `compareAndSet`. No external lock object, no `synchronized` block, is needed, because `AtomicReference` already provides the needed atomicity for this single-field check-and-set.

## Interview Tips & Common Mistakes

- **Say the words "check-then-act race condition" out loud.** This is the single most important concept in this problem. A common mistake is writing `if (driver.getStatus() == AVAILABLE) { driver.setStatus(ASSIGNED); }` — two separate steps, with a gap between them where another thread can act. Naming this gap, and showing `compareAndSet` as the fix, is what separates a strong answer from a weak one.
- **Do not reach for a giant `synchronized` block around the whole matching loop as your first answer.** It works, but it serializes every ride request in the entire system through one lock — a severe bottleneck. `compareAndSet` on the individual driver is a much finer-grained, more scalable guard: it only blocks two requests that actually want the *same* driver, not all requests everywhere.
- **Keep the two Strategy interfaces separate.** Matching ("which driver?") and pricing ("how much?") are different concerns with different inputs and different reasons to change. Merging them into one "ride logic" class violates the Single Responsibility Principle and makes both harder to test in isolation.
- **Model the trip state machine explicitly, with a transition map or an enum-per-state class — not scattered `if` checks.** A frequent bug is allowing `endTrip` to run on a trip that was never started, because nothing validated the previous state. The `ALLOWED_TRANSITIONS` map in `Trip` makes invalid transitions impossible by construction, not by convention.
- **Mention the retry-on-claim-failure loop.** A shallow answer stops at "use `compareAndSet` for the top driver." A stronger answer explains that the *losing* request should not simply fail — it should fall back to the next-nearest candidate, which is why `rankCandidates` returns a ranked list, not a single driver.
- **Be ready to discuss what breaks at scale**, and say so before being asked: in-memory `AtomicReference` and `ConcurrentHashMap` only work within one process. A real system needs the same atomic-claim idea expressed against a shared store (database row lock, Redis atomic command) once there is more than one booking service instance. Interviewers view this awareness as a strong signal, even if implementing it fully is out of scope for the interview.
- **Do not forget to release the driver.** A common bug: `endTrip` computes the fare but forgets `driver.release()`, silently leaking drivers into permanent `ON_TRIP` status. Always pair a claim with a guaranteed release path (trip completion or cancellation).
