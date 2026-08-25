# Elevator System (LLD)

## Problem

Design an elevator system for a building. The building has many floors. It has multiple elevators (also called "cars"). People request an elevator in two ways:

1. **Hall call.** A person stands on a floor and presses "up" or "down" on the wall panel. This is an **external** request. The person is not inside any elevator yet. The system must pick one elevator to serve this call.
2. **Car call.** A person is already inside an elevator. They press a floor button on the car panel. This is an **internal** request. It always belongs to one specific elevator — the one the person is standing in.

The system must:
- Track the state of each elevator: current floor, direction of travel, and door state.
- Decide which elevator answers each hall call.
- Move each elevator floor by floor, stopping at the right floors, opening and closing doors.
- Handle many hall calls and car calls arriving at the same time, from many floors, possibly from many threads (many people pressing buttons at once).

This is a classic LLD (low-level design) interview question. It tests state machine design, the Strategy pattern for a pluggable dispatch rule, and safe handling of shared, mutable data under concurrency.

## Requirements & Clarifying Questions

Ask these before coding. They show the interviewer you scope the problem before diving in.

1. **How many elevators and floors?** We will support a configurable number of elevators (`N`) and floors (`1..F`), fixed at system startup. Dynamic add/remove of elevators is out of scope, but the design should not block it later.
2. **Do we model weight or passenger capacity limits?** No, not in the base design. We will note it as a follow-up.
3. **Do we model door sensors (obstruction) and exact timing in seconds?** No. We will use a simplified, discrete-time model: the system "ticks" and each tick moves an elevator one floor, or opens/closes a door. This keeps the logic testable without real clocks.
4. **What scheduling rule decides which elevator answers a hall call?** This must be **pluggable**. We will implement one concrete rule — "nearest suitable elevator" — behind an interface, so the interviewer can ask for a different rule later without a rewrite.
5. **Should an elevator serve calls in one sweep, or jump around?** It should sweep: serve all stops in its current direction first, then reverse if needed. This is the well-known **SCAN** (also called **LOOK**) disk-scheduling algorithm, reused here for elevators. It avoids starving requests and avoids wasteful direction changes.
6. **What happens when a hall call comes in while all elevators are busy?** The request queues on the elevator judged "best" by the strategy; it is not dropped. Fairness under heavy load is a follow-up.
7. **Is this single-threaded or concurrent?** Concurrent. Multiple people press buttons on different floors and inside different cars at the same time, from different threads. The design must not corrupt state or lose requests.

## Design / Approach

```
                         ┌───────────────────────────────────────┐
                         │            ElevatorController           │
                         │  - List<Elevator> elevators             │
                         │  - SchedulingStrategy strategy  (Strategy) │
                         │  - dispatchLock (ReentrantLock)          │
                         │  + submitHallCall(floor, dir)            │
                         │  + submitCarCall(elevatorId, floor)      │
                         │  + tick()   <- called by scheduler thread │
                         └───────────────────┬───────────────────┘
                                              │ uses
                     ┌────────────────────────┼────────────────────────┐
                     ▼                                                  ▼
        ┌───────────────────────┐                         ┌───────────────────────────┐
        │   SchedulingStrategy    │  (interface)            │          Elevator            │
        │ + selectElevator(list,  │                         │ - id, currentFloor           │
        │     hallCall): Elevator │◄────implemented by──────│ - state: ElevatorState        │
        └───────────────────────┘                         │     (IDLE/MOVING_UP/MOVING_DOWN)│
                     ▲                                      │ - doorState: OPEN/CLOSED       │
                     │                                      │ - upStops: NavigableSet<Integer>│
        ┌───────────────────────┐                          │ - downStops: NavigableSet<Integer>│
        │ NearestElevatorStrategy │                          │ + addStop(floor)                │
        └───────────────────────┘                          │ + step()                         │
                                                             └───────────────────────────────┘

        Request (abstract)
          ├── HallCall(floor, direction)   -- needs dispatch by strategy
          └── CarCall(elevatorId, floor)   -- goes straight to one elevator
```

**Patterns used:**
- **Strategy** — `SchedulingStrategy` decouples "how we pick an elevator" from the controller. New rules (zone-based, destination-dispatch) plug in without touching `ElevatorController`.
- **State machine via enum** — `ElevatorState` (`IDLE`, `MOVING_UP`, `MOVING_DOWN`) and `DoorState` (`OPEN`, `CLOSED`) model the elevator's behavior as a small, explicit finite state machine. Transitions are guarded inside `Elevator`, so no other class can put an elevator into an invalid combination (for example, moving with doors open).
- **Command-like request objects** — `HallCall` and `CarCall` are small, immutable data objects. They carry everything needed to act on them and are safe to pass between threads without locking.
- **Facade** — `ElevatorController` is the single entry point for the outside world (button presses). Callers never touch `Elevator` internals directly.

**Sorted set of stops (the SCAN/LOOK algorithm).** Each elevator keeps two sorted sets of pending stops:
- `upStops` — floors to visit while moving up, in ascending order.
- `downStops` — floors to visit while moving down, in descending order.

When moving up, the elevator always goes to `upStops.first()` (the nearest stop above it), never jumping past a closer floor to serve a farther one. When `upStops` becomes empty, and `downStops` is not empty, the elevator reverses direction. This is the **SCAN/LOOK** rule: sweep in one direction serving everything in order, then turn around only when nothing is left ahead. It gives predictable, fair service, and it is why a sorted set — not a plain queue — is the right data structure: we always need "next stop in current direction," which a sorted set gives us in O(1) (`first()`/`last()`), keeping insertion O(log n).

**Dispatch (nearest suitable elevator).** For each hall call, `NearestElevatorStrategy` scores every elevator:
- **Idle elevator:** cost = distance to the caller's floor. Cheapest option when available.
- **Elevator already moving in the same direction as the request, and not yet past the caller's floor:** cost = distance to the caller's floor. It can simply add one more stop to its sweep.
- **Elevator moving away, or already past the floor:** cost = a penalty (it must finish its current sweep, reverse, then come back) plus distance. This is deliberately more expensive, so idle or well-aligned elevators win first.

The elevator with the lowest cost gets the hall call added to its stop set.

## Java Solution

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;

// ---------- Enums: the state machine ----------

enum Direction { UP, DOWN }

enum ElevatorState { IDLE, MOVING_UP, MOVING_DOWN }

enum DoorState { OPEN, CLOSED }

// ---------- Requests (immutable, thread-safe by construction) ----------

abstract class Request {
    final int floor;
    Request(int floor) { this.floor = floor; }
}

/** External request: someone on a floor wants to go up or down. Needs dispatch. */
final class HallCall extends Request {
    final Direction direction;
    HallCall(int floor, Direction direction) {
        super(floor);
        this.direction = direction;
    }
}

/** Internal request: someone inside a specific car pressed a floor button. */
final class CarCall extends Request {
    final int elevatorId;
    CarCall(int elevatorId, int floor) {
        super(floor);
        this.elevatorId = elevatorId;
    }
}

// ---------- Elevator: owns its own state and stop list ----------

class Elevator {
    private final int id;
    private final int minFloor;
    private final int maxFloor;

    // volatile: many threads read these for dispatch decisions while
    // a single scheduler thread updates them during tick().
    private volatile int currentFloor;
    private volatile ElevatorState state = ElevatorState.IDLE;
    private volatile DoorState doorState = DoorState.CLOSED;

    // ConcurrentSkipListSet: a thread-safe, always-sorted set.
    // upStops ascending (next stop above = first()); downStops descending
    // (next stop below = first(), using a reverse comparator).
    private final NavigableSet<Integer> upStops = new ConcurrentSkipListSet<>();
    private final NavigableSet<Integer> downStops =
            new ConcurrentSkipListSet<>(Comparator.reverseOrder());

    Elevator(int id, int minFloor, int maxFloor, int startFloor) {
        this.id = id;
        this.minFloor = minFloor;
        this.maxFloor = maxFloor;
        this.currentFloor = startFloor;
    }

    int getId() { return id; }
    int getCurrentFloor() { return currentFloor; }
    ElevatorState getState() { return state; }
    DoorState getDoorState() { return doorState; }

    /** Add a stop. Decides which sorted set it belongs to, based on current position. */
    void addStop(int floor) {
        if (floor > currentFloor) {
            upStops.add(floor);
            if (state == ElevatorState.IDLE) state = ElevatorState.MOVING_UP;
        } else if (floor < currentFloor) {
            downStops.add(floor);
            if (state == ElevatorState.IDLE) state = ElevatorState.MOVING_DOWN;
        } else {
            // Already at this floor: just open the doors.
            doorState = DoorState.OPEN;
        }
    }

    /** Rough cost estimate used by the scheduling strategy. Lower is better. */
    int costFor(HallCall call) {
        switch (state) {
            case IDLE:
                return Math.abs(currentFloor - call.floor);
            case MOVING_UP:
                if (call.direction == Direction.UP && call.floor >= currentFloor) {
                    return call.floor - currentFloor; // on the way, cheap
                }
                // Must finish going up to the highest stop, then come back down.
                int topStop = upStops.isEmpty() ? currentFloor : upStops.last();
                return (topStop - currentFloor) + Math.abs(topStop - call.floor) + 100;
            case MOVING_DOWN:
                if (call.direction == Direction.DOWN && call.floor <= currentFloor) {
                    return currentFloor - call.floor; // on the way, cheap
                }
                int bottomStop = downStops.isEmpty() ? currentFloor : downStops.last();
                return (currentFloor - bottomStop) + Math.abs(bottomStop - call.floor) + 100;
            default:
                return Integer.MAX_VALUE;
        }
    }

    /** One discrete time step: move one floor, or open/close doors, following SCAN. */
    void step() {
        if (doorState == DoorState.OPEN) {
            doorState = DoorState.CLOSED; // doors close after one tick
            return;
        }
        switch (state) {
            case MOVING_UP:
                if (upStops.isEmpty()) {
                    state = downStops.isEmpty() ? ElevatorState.IDLE : ElevatorState.MOVING_DOWN;
                    return;
                }
                currentFloor = Math.min(currentFloor + 1, maxFloor);
                if (upStops.contains(currentFloor)) {
                    upStops.remove(currentFloor);
                    doorState = DoorState.OPEN;
                }
                if (upStops.isEmpty() && !downStops.isEmpty()) {
                    state = ElevatorState.MOVING_DOWN; // SCAN reversal
                } else if (upStops.isEmpty()) {
                    state = ElevatorState.IDLE;
                }
                break;
            case MOVING_DOWN:
                if (downStops.isEmpty()) {
                    state = upStops.isEmpty() ? ElevatorState.IDLE : ElevatorState.MOVING_UP;
                    return;
                }
                currentFloor = Math.max(currentFloor - 1, minFloor);
                if (downStops.contains(currentFloor)) {
                    downStops.remove(currentFloor);
                    doorState = DoorState.OPEN;
                }
                if (downStops.isEmpty() && !upStops.isEmpty()) {
                    state = ElevatorState.MOVING_UP; // SCAN reversal
                } else if (downStops.isEmpty()) {
                    state = ElevatorState.IDLE;
                }
                break;
            case IDLE:
            default:
                break; // nothing to do
        }
    }
}

// ---------- Strategy: pluggable dispatch rule ----------

interface SchedulingStrategy {
    Optional<Elevator> selectElevator(List<Elevator> elevators, HallCall call);
}

class NearestElevatorStrategy implements SchedulingStrategy {
    @Override
    public Optional<Elevator> selectElevator(List<Elevator> elevators, HallCall call) {
        return elevators.stream()
                .min(Comparator.comparingInt(e -> e.costFor(call)));
    }
}

// ---------- Controller: single entry point, thread-safe dispatch ----------

class ElevatorController {
    private final List<Elevator> elevators;
    private final SchedulingStrategy strategy;
    private final ReentrantLock dispatchLock = new ReentrantLock();

    ElevatorController(int numElevators, int minFloor, int maxFloor, SchedulingStrategy strategy) {
        this.strategy = strategy;
        List<Elevator> list = new ArrayList<>();
        for (int i = 0; i < numElevators; i++) {
            list.add(new Elevator(i, minFloor, maxFloor, minFloor));
        }
        this.elevators = list;
    }

    /** Called by the wall panel. Many threads may call this at once. */
    void submitHallCall(int floor, Direction direction) {
        HallCall call = new HallCall(floor, direction);
        dispatchLock.lock(); // guard "read all states, then commit one stop" as one step
        try {
            Elevator chosen = strategy.selectElevator(elevators, call)
                    .orElseThrow(() -> new IllegalStateException("No elevator available"));
            chosen.addStop(floor);
        } finally {
            dispatchLock.unlock();
        }
    }

    /** Called by a car's inside panel. Goes straight to the known elevator, no dispatch needed. */
    void submitCarCall(int elevatorId, int floor) {
        elevators.get(elevatorId).addStop(floor);
    }

    /** Advance every elevator by one discrete time unit. */
    void tick() {
        for (Elevator e : elevators) {
            e.step();
        }
    }

    List<Elevator> getElevators() {
        return Collections.unmodifiableList(elevators);
    }
}

// ---------- Demo ----------

public class ElevatorSystemDemo {
    public static void main(String[] args) throws InterruptedException {
        ElevatorController controller =
                new ElevatorController(2, 1, 10, new NearestElevatorStrategy());

        ScheduledExecutorService ticker = Executors.newSingleThreadScheduledExecutor();
        ticker.scheduleAtFixedRate(controller::tick, 0, 200, TimeUnit.MILLISECONDS);

        // Simulate people pressing buttons from different threads.
        ExecutorService people = Executors.newFixedThreadPool(3);
        people.submit(() -> controller.submitHallCall(5, Direction.UP));
        people.submit(() -> controller.submitHallCall(3, Direction.DOWN));
        people.submit(() -> controller.submitCarCall(0, 8));
        people.shutdown();

        Thread.sleep(3000);
        for (Elevator e : controller.getElevators()) {
            System.out.printf("Elevator %d: floor=%d state=%s door=%s%n",
                    e.getId(), e.getCurrentFloor(), e.getState(), e.getDoorState());
        }
        ticker.shutdown();
    }
}
```

## How It Works

Walk through one scenario. Building has floors 1–10 and two elevators, both idle at floor 1.

1. A person on floor 5 presses "up." `submitHallCall(5, UP)` builds a `HallCall` and calls `NearestElevatorStrategy.selectElevator`. Both elevators are `IDLE`, so cost is just distance (`|1-5| = 4` for both). The strategy picks the first one it finds tied lowest — elevator 0. `addStop(5)` puts `5` into elevator 0's `upStops`, and its state flips from `IDLE` to `MOVING_UP`.
2. A person on floor 3 presses "down" at the same moment, from a different thread. This call also goes through `NearestElevatorStrategy`. Elevator 0 is now `MOVING_UP` and the request direction is `DOWN`, so its cost gets the "must finish sweep, then come back" penalty. Elevator 1 is still `IDLE`, cost `2`. Elevator 1 wins and gets `3` added to its `downStops` — but since `3 > currentFloor (1)`, it actually lands in `upStops` (the elevator must go up to floor 3 first, pick up the passenger, then that passenger's actual trip down is a separate car call once boarded). This detail matters: a hall call's direction is the *caller's intent*, but the *elevator's approach* to reach that floor still follows normal SCAN rules.
3. Someone inside elevator 0 presses floor 8. `submitCarCall(0, 8)` skips the strategy entirely — it already knows which elevator — and adds `8` to elevator 0's `upStops` directly.
4. The `ScheduledExecutorService` ticks every 200 ms, calling `controller.tick()`, which calls `step()` on every elevator. Elevator 0 moves 1 → 2 → 3 → 4 → 5, sees `5` is in `upStops`, removes it, and opens its doors for one tick. Then it continues 6 → 7 → 8, opens doors again, empties `upStops`, and goes `IDLE`.
5. The `dispatchLock` in `ElevatorController` ensures that when two hall calls arrive within microseconds of each other, one dispatch decision fully completes (read all elevator costs, then commit the stop) before the next one starts reading. Without this, two hall calls could both see the same "idle" elevator as cheapest and both assign it, overloading one car while another sits empty.

## How to Extend (Follow-ups)

Interviewers commonly push into these directions after the base design works:

1. **Capacity limits.** Add a `maxWeight` or `maxOccupants` field to `Elevator`. `addStop` (or a new `boardPassenger` call) checks current load before accepting a car call, and can reject or defer a hall call whose only "nearest" elevator is full — the `SchedulingStrategy` interface makes this a small change inside `costFor`, not a rewrite.
2. **Alternative dispatch strategies.** Because dispatch is a `SchedulingStrategy`, you can add `ZoneBasedStrategy` (each elevator "owns" a range of floors during off-peak hours) or `DestinationDispatchStrategy` (passengers key in their target floor at a lobby kiosk before boarding, common in modern high-rises) without touching `ElevatorController`.
3. **Emergency / fire mode.** Add an `EmergencyState` that overrides normal dispatch: all elevators ignore hall calls, finish any open doors, and travel straight to the ground floor. This is a good place to mention the **State pattern** more formally — if the state machine grows past three simple enum values with a few `switch` branches, promote `ElevatorState` from an enum to an interface with one class per state (`IdleState`, `MovingUpState`, `EmergencyState`), each implementing its own `step()` and `addStop()`.
4. **Fairness / starvation prevention.** Under heavy load, a hall call might keep losing to "better" candidates as new calls arrive. Add an aging factor to `costFor` — reduce the cost slightly the longer a request has waited — so old requests eventually win.
5. **Notifications.** Add an **Observer** pattern: `Elevator` publishes events (`arrived`, `doorsOpened`) to listeners such as floor indicator displays or a logging service, instead of the controller polling state.
6. **Testability.** Because movement is driven by explicit `tick()` calls rather than real time, unit tests can call `tick()` manually and assert exact floor/state after each step — no `Thread.sleep` needed. The `SchedulingStrategy` interface also lets you unit-test dispatch logic alone, with fake `Elevator` states, without running a real simulation.

## Complexity & Thread-Safety Notes

- **`addStop`**: O(log n) where n is the number of pending stops for that elevator (insert into a skip-list-backed sorted set).
- **`step`**: O(log n) — checking and removing the current floor from a `NavigableSet` is O(log n); reading `first()`/`last()` for reversal checks is O(1).
- **`selectElevator`**: O(E) where E is the number of elevators, since each elevator's `costFor` call is O(1) beyond an O(log n) peek at its extreme stop.
- **Thread-safety choices:**
    - `Request` subclasses (`HallCall`, `CarCall`) are immutable — safe to hand across threads with no locking.
    - `upStops` and `downStops` use `ConcurrentSkipListSet`, a thread-safe `NavigableSet`. This gives lock-free-in-practice, always-sorted access, which is exactly the "sorted set of stops" the design needs, without hand-written locking around a plain `TreeSet` (a `TreeSet` is not thread-safe and would throw `ConcurrentModificationException` under concurrent access).
    - `currentFloor`, `state`, and `doorState` are `volatile`, so a dispatch decision on one thread always sees the latest value written by the ticking thread — this matters for visibility, not atomicity, because only one thread (the scheduler) ever writes them.
    - The `dispatchLock` in `ElevatorController` protects the "read every elevator's cost, then pick one, then commit a stop" sequence as a single atomic step. Without it, two concurrent hall calls could both read stale, pre-assignment costs and both pick the same elevator.
    - Movement (`tick()` → `step()` per elevator) is driven by one single-threaded scheduler. This avoids any race on `currentFloor` updates — only one thread ever writes it — while dispatch threads only read it.

## Interview Tips & Common Mistakes

- **Do not merge hall calls and car calls into one type carelessly.** A hall call needs dispatch (which elevator?); a car call already knows its elevator. Modeling them as one `Request` type with an "elevatorId or -1" field works, but a small class hierarchy (as shown here) reads more clearly and avoids null/sentinel checks scattered through the code.
- **Do not hardcode the dispatch rule inside the controller.** A common mistake is writing `if` chains for "nearest elevator" directly in `ElevatorController`. This violates the Open/Closed Principle — any new rule means editing the controller. Extracting `SchedulingStrategy` is the single most important design decision interviewers look for.
- **Remember the SCAN reversal case.** A frequent bug: the elevator empties `upStops` but forgets to check `downStops` before going `IDLE`, so a waiting passenger below never gets served until a new call "wakes" the elevator. Always check the opposite set before deciding `IDLE`.
- **Bring up concurrency before being asked.** Since this problem screams "real-time system with many callers," proactively mention thread-safety (as above). Candidates who only discuss the single-threaded scheduling logic usually get pushed hard on this in the interview anyway — get ahead of it.
- **Do not use a plain `TreeSet` or `LinkedList` for stops under concurrent access.** It is a common slip to reach for `TreeSet` because "it's sorted," forgetting it needs external synchronization. `ConcurrentSkipListSet` (or manual locking around a `TreeSet`) is the correct choice here.
- **Be ready to explain the cost function in plain terms**, not just show code: idle elevators are cheapest, elevators already heading toward the caller in the right direction are next, and elevators moving away are penalized because they must finish their sweep and reverse before they can help. This shows you understand *why* SCAN/LOOK is used, not just that you memorized the term.
