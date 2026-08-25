# Movie Ticket Booking (LLD)

## Problem

Design a movie ticket booking system, like BookMyShow. The system must let a user search for movies playing in a city, see the shows for a movie, check seat availability for a show, and book seats.

The hard part is concurrency. Many users can try to book the same seat at the same time. The system must make sure only one user ends up with that seat. We also need a payment step and a clear booking status.

## Requirements & Clarifying Questions

Functional requirements:

1. Model movies, cities, theatres, screens, shows, and seats.
2. Search shows by movie and city.
3. Show seat availability for a given show.
4. Let a user select seats and hold them for a short time while they pay.
5. Confirm booking only after payment succeeds.
6. Release held seats automatically if payment does not happen in time.
7. Track booking status: `CREATED`, `PAYMENT_PENDING`, `CONFIRMED`, `CANCELLED`, `EXPIRED`.

Non-functional requirements:

1. No two users can book the same seat for the same show. This is the top priority.
2. The system should work correctly with many threads (many users) at the same time.
3. The design should be easy to extend later (e.g., add discounts, multiple payment methods, seat types).

Clarifying questions to ask the interviewer:

- Do we need to support seat-level pricing (e.g., gold, silver, premium)? (I will assume yes, with a `SeatType`.)
- Is this a single-process system (one JVM) for the interview, or should I also mention how it scales to many servers? (I will build a single-process, thread-safe design, then add a note on distributed scaling.)
- How long should a seat hold last before it expires? (I will assume 5 minutes, configurable.)
- Can a user hold more than one seat at a time? (Yes, a booking can have multiple seats, and all seats are held together.)
- What happens if payment fails? (Seats go back to `AVAILABLE`, booking becomes `CANCELLED`.)

## Design / Approach

### Key entities

- `City` — a city where theatres exist.
- `Movie` — movie details (title, language, duration).
- `Theatre` — belongs to a city, has one or more `Screen`s.
- `Screen` — a physical hall inside a theatre, has a fixed layout of `Seat`s.
- `Seat` — belongs to a screen, has a `seatNumber` and `SeatType`.
- `Show` — a movie playing on a screen at a specific start time. A show has its own seat status map, because the same physical seat can be `AVAILABLE` for a 3 PM show and `BOOKED` for a 6 PM show.
- `ShowSeat` — the booking-relevant state of one `Seat` for one `Show`: its current `SeatStatus` and lock/hold metadata.
- `Booking` — a user's attempt to reserve one or more `ShowSeat`s for a `Show`.
- `Payment` — the payment step tied to a `Booking`.

### Seat state machine (the key part)

Each `ShowSeat` moves through these states:

```
AVAILABLE  --(user selects seat)-->  LOCKED (HELD)
LOCKED     --(payment succeeds)-->   BOOKED
LOCKED     --(hold expires / payment fails / user cancels)--> AVAILABLE
```

- `AVAILABLE` — free to select.
- `LOCKED` (a temporary hold) — one specific booking has reserved this seat for a short time (e.g., 5 minutes) so it can complete payment. No other booking can lock it while it is `LOCKED` and not expired.
- `BOOKED` — payment succeeded. The seat is permanently taken for this show.

A `LOCKED` seat that is not confirmed within the timeout must go back to `AVAILABLE` automatically. This is done with a hold-expiry timestamp checked on every access, plus a background scheduler that sweeps expired holds.

### How we stop two users from booking the same seat

This is the core interview question. Two approaches, I use approach 1 for the code, and explain approach 2 as an alternative:

**Approach 1: Per-seat locking with atomic compare-and-set state transition.**

Each `ShowSeat` has its own `ReentrantLock` (or we use a `ConcurrentHashMap<seatId, ShowSeat>` and do the state check-and-update as one atomic, synchronized operation per seat). When a user tries to hold a seat:

1. Acquire the lock for that exact seat only (not the whole show). This keeps other seats free to be booked by other users at the same time — fine-grained locking, not one big lock for the whole show.
2. Inside the lock, check: is the seat `AVAILABLE`, or `LOCKED` but expired? If yes, set it to `LOCKED`, record `lockedBy = bookingId` and `lockExpiryTime = now + holdDuration`. If no (seat is `LOCKED` by someone else and not expired, or already `BOOKED`), reject with "seat not available."
3. Release the lock.

Because the check-and-set happens inside one lock, it is atomic. Two threads racing for the same seat cannot both see `AVAILABLE` and both succeed — only one gets in at a time, and the second one sees `LOCKED` and fails. This is the same idea as `synchronized` on the seat, or a single row-level lock on a database row (`SELECT ... FOR UPDATE` in SQL). We use `ConcurrentHashMap.compute()` in the code below because `compute()` runs its function atomically per key.

**Approach 2 (alternative, worth mentioning): Optimistic locking with a version number.**

Instead of locking, give each `ShowSeat` a `version` field. To lock a seat: read current state and version, then try to update with `WHERE version = <read version>`. If the update affects zero rows (because someone else updated it first), the version changed, so retry or fail. This works well when seat contention is low and avoids holding locks. Databases implement this naturally with an `UPDATE ... WHERE id = ? AND version = ?` statement. Per-seat pessimistic locking (Approach 1) is simpler to reason about for an interview and is what I code below.

### Patterns used

- **State pattern** (via enum `SeatStatus` and `BookingStatus`) — models the seat and booking lifecycle cleanly, avoids messy `if/else` chains.
- **Strategy pattern** — `PaymentStrategy` interface, so we can plug in `CardPayment`, `UpiPayment`, `WalletPayment` without changing `Payment` or `Booking`.
- **Factory pattern** — `SeatLockManager` centralizes how locks are created and expiry is checked, hiding the concurrency detail from `BookingService`.
- **Observer pattern** (mentioned in extensions) — notify user on booking confirmation / hold expiry.
- **Single Responsibility Principle** — `BookingService` handles booking flow, `PaymentService` handles payment, `SeatLockManager` handles locking, `ShowRepository` handles data lookup. Each class has one reason to change.
- **Open/Closed Principle** — new payment methods or seat types can be added without modifying existing classes.

### ASCII class sketch

```
City ----< Theatre >---- Screen >---- Seat
                                        |
Movie ----< Show >------- Screen       | (per show)
              |                        v
              +----------------< ShowSeat (status, lockExpiry, lockedBy)
              |
              +----< Booking >---- Payment
                        |
                   BookingStatus (enum)

SeatLockManager  --uses-->  ConcurrentHashMap<showSeatId, ShowSeat>
BookingService   --uses-->  SeatLockManager, PaymentService, ShowRepository
PaymentService   --uses-->  PaymentStrategy (interface: CardPayment, UpiPayment)
```

## Java Solution

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicReference;

// ===================== Enums =====================

enum SeatType { REGULAR, PREMIUM, RECLINER }

enum SeatStatus { AVAILABLE, LOCKED, BOOKED }

enum BookingStatus { CREATED, PAYMENT_PENDING, CONFIRMED, CANCELLED, EXPIRED }

enum PaymentStatus { PENDING, SUCCESS, FAILED }

// ===================== Core entities =====================

class City {
    final String id;
    final String name;
    City(String id, String name) { this.id = id; this.name = name; }
}

class Movie {
    final String id;
    final String title;
    final int durationMinutes;
    Movie(String id, String title, int durationMinutes) {
        this.id = id; this.title = title; this.durationMinutes = durationMinutes;
    }
}

class Seat {
    final String seatId;
    final String seatNumber; // e.g. "A1"
    final SeatType type;
    Seat(String seatId, String seatNumber, SeatType type) {
        this.seatId = seatId; this.seatNumber = seatNumber; this.type = type;
    }
}

class Screen {
    final String screenId;
    final String name;
    final List<Seat> seats;
    Screen(String screenId, String name, List<Seat> seats) {
        this.screenId = screenId; this.name = name; this.seats = seats;
    }
}

class Theatre {
    final String theatreId;
    final String name;
    final City city;
    final List<Screen> screens;
    Theatre(String theatreId, String name, City city, List<Screen> screens) {
        this.theatreId = theatreId; this.name = name; this.city = city; this.screens = screens;
    }
}

// One seat's booking state for one specific show
class ShowSeat {
    final String showSeatId;   // showId + seatId
    final Seat seat;
    final double price;
    private volatile SeatStatus status = SeatStatus.AVAILABLE;
    private volatile String lockedByBookingId = null;
    private volatile long lockExpiryEpochMs = 0;

    ShowSeat(String showSeatId, Seat seat, double price) {
        this.showSeatId = showSeatId; this.seat = seat; this.price = price;
    }

    // package-private, only SeatLockManager mutates these under its own lock
    SeatStatus getStatus() { return status; }
    void setStatus(SeatStatus s) { this.status = s; }
    String getLockedByBookingId() { return lockedByBookingId; }
    void setLockedByBookingId(String id) { this.lockedByBookingId = id; }
    long getLockExpiryEpochMs() { return lockExpiryEpochMs; }
    void setLockExpiryEpochMs(long t) { this.lockExpiryEpochMs = t; }

    boolean isHoldExpired() {
        return status == SeatStatus.LOCKED && System.currentTimeMillis() > lockExpiryEpochMs;
    }
}

class Show {
    final String showId;
    final Movie movie;
    final Screen screen;
    final Theatre theatre;
    final LocalDateTime startTime;
    final Map<String, ShowSeat> showSeats = new ConcurrentHashMap<>();

    Show(String showId, Movie movie, Screen screen, Theatre theatre, LocalDateTime startTime) {
        this.showId = showId; this.movie = movie; this.screen = screen;
        this.theatre = theatre; this.startTime = startTime;
        for (Seat seat : screen.seats) {
            double price = switch (seat.type) {
                case REGULAR -> 150.0;
                case PREMIUM -> 250.0;
                case RECLINER -> 400.0;
            };
            String showSeatId = showId + "_" + seat.seatId;
            showSeats.put(showSeatId, new ShowSeat(showSeatId, seat, price));
        }
    }

    List<ShowSeat> getAvailableSeats() {
        List<ShowSeat> result = new ArrayList<>();
        for (ShowSeat ss : showSeats.values()) {
            if (ss.getStatus() == SeatStatus.AVAILABLE || ss.isHoldExpired()) {
                result.add(ss);
            }
        }
        return result;
    }
}

// ===================== Concurrency: the key part =====================

/**
 * Owns all locking decisions for seats. Uses one lock object per seat so that
 * booking seat A does not block a concurrent attempt to book seat B on the
 * same show. The check-and-set of seat status happens inside that per-seat
 * lock, so it is atomic: two threads can never both succeed in locking the
 * same seat.
 */
class SeatLockManager {
    private final long holdDurationMs;
    // one lock per showSeatId, created lazily, reused across calls
    private final ConcurrentHashMap<String, Object> seatLocks = new ConcurrentHashMap<>();

    SeatLockManager(long holdDurationMs) {
        this.holdDurationMs = holdDurationMs;
    }

    /**
     * Try to hold a seat for a booking. Returns true if the hold succeeded.
     * Thread-safe: only one caller can win the race for a given seat.
     */
    boolean tryLockSeat(ShowSeat showSeat, String bookingId) {
        Object lock = seatLocks.computeIfAbsent(showSeat.showSeatId, k -> new Object());
        synchronized (lock) {
            boolean free = showSeat.getStatus() == SeatStatus.AVAILABLE || showSeat.isHoldExpired();
            if (!free) {
                return false; // someone else holds it, and hold has not expired
            }
            showSeat.setStatus(SeatStatus.LOCKED);
            showSeat.setLockedByBookingId(bookingId);
            showSeat.setLockExpiryEpochMs(System.currentTimeMillis() + holdDurationMs);
            return true;
        }
    }

    /** Called when payment succeeds: LOCKED (by this booking) -> BOOKED. */
    boolean confirmSeat(ShowSeat showSeat, String bookingId) {
        Object lock = seatLocks.computeIfAbsent(showSeat.showSeatId, k -> new Object());
        synchronized (lock) {
            if (showSeat.getStatus() == SeatStatus.LOCKED
                    && bookingId.equals(showSeat.getLockedByBookingId())
                    && !showSeat.isHoldExpired()) {
                showSeat.setStatus(SeatStatus.BOOKED);
                return true;
            }
            return false; // hold expired or seat taken by mistake — reject
        }
    }

    /** Called on payment failure, user cancel, or hold expiry sweep. */
    void releaseSeat(ShowSeat showSeat, String bookingId) {
        Object lock = seatLocks.computeIfAbsent(showSeat.showSeatId, k -> new Object());
        synchronized (lock) {
            if (showSeat.getStatus() == SeatStatus.LOCKED
                    && bookingId.equals(showSeat.getLockedByBookingId())) {
                showSeat.setStatus(SeatStatus.AVAILABLE);
                showSeat.setLockedByBookingId(null);
                showSeat.setLockExpiryEpochMs(0);
            }
        }
    }

    /** Background sweep: release any seat whose hold has expired. Runs on a timer. */
    void releaseExpiredHolds(Show show) {
        for (ShowSeat ss : show.showSeats.values()) {
            if (ss.isHoldExpired()) {
                releaseSeat(ss, ss.getLockedByBookingId());
            }
        }
    }
}

// ===================== Payment (Strategy pattern) =====================

interface PaymentStrategy {
    boolean pay(double amount);
}

class CardPayment implements PaymentStrategy {
    public boolean pay(double amount) {
        System.out.println("Charging card for amount: " + amount);
        return true; // assume success for demo
    }
}

class UpiPayment implements PaymentStrategy {
    public boolean pay(double amount) {
        System.out.println("Charging UPI for amount: " + amount);
        return true;
    }
}

class Payment {
    final String paymentId;
    final double amount;
    final PaymentStrategy strategy;
    private volatile PaymentStatus status = PaymentStatus.PENDING;

    Payment(String paymentId, double amount, PaymentStrategy strategy) {
        this.paymentId = paymentId; this.amount = amount; this.strategy = strategy;
    }

    boolean process() {
        boolean success = strategy.pay(amount);
        status = success ? PaymentStatus.SUCCESS : PaymentStatus.FAILED;
        return success;
    }

    PaymentStatus getStatus() { return status; }
}

// ===================== Booking =====================

class Booking {
    final String bookingId;
    final String userId;
    final Show show;
    final List<ShowSeat> seats;
    private volatile BookingStatus status = BookingStatus.CREATED;
    Payment payment;

    Booking(String bookingId, String userId, Show show, List<ShowSeat> seats) {
        this.bookingId = bookingId; this.userId = userId; this.show = show; this.seats = seats;
    }

    BookingStatus getStatus() { return status; }
    void setStatus(BookingStatus s) { this.status = s; }

    double totalAmount() {
        return seats.stream().mapToDouble(s -> s.price).sum();
    }
}

// ===================== Booking service (orchestration) =====================

class BookingService {
    private final SeatLockManager lockManager;
    private final Map<String, Booking> bookings = new ConcurrentHashMap<>();
    private int bookingCounter = 0;

    BookingService(SeatLockManager lockManager) {
        this.lockManager = lockManager;
    }

    private synchronized String nextBookingId() {
        return "BK" + (++bookingCounter);
    }

    /** Step 1: user selects seats. Returns null if any seat could not be held. */
    Booking createBookingAndHoldSeats(String userId, Show show, List<String> showSeatIds) {
        String bookingId = nextBookingId();
        List<ShowSeat> lockedSoFar = new ArrayList<>();

        for (String showSeatId : showSeatIds) {
            ShowSeat showSeat = show.showSeats.get(showSeatId);
            if (showSeat == null || !lockManager.tryLockSeat(showSeat, bookingId)) {
                // rollback whatever we already locked in this attempt
                for (ShowSeat s : lockedSoFar) {
                    lockManager.releaseSeat(s, bookingId);
                }
                return null; // caller should tell user: "some seats are no longer available"
            }
            lockedSoFar.add(showSeat);
        }

        Booking booking = new Booking(bookingId, userId, show, lockedSoFar);
        booking.setStatus(BookingStatus.PAYMENT_PENDING);
        bookings.put(bookingId, booking);
        return booking;
    }

    /** Step 2: user pays. On success, seats move LOCKED -> BOOKED. */
    boolean completePayment(Booking booking, PaymentStrategy strategy) {
        if (booking.getStatus() != BookingStatus.PAYMENT_PENDING) {
            return false;
        }
        Payment payment = new Payment("PAY_" + booking.bookingId, booking.totalAmount(), strategy);
        booking.payment = payment;

        boolean paid = payment.process();
        if (!paid) {
            cancelBooking(booking);
            return false;
        }

        // confirm each seat; if any fails (hold expired mid-payment), roll back the whole booking
        List<ShowSeat> confirmed = new ArrayList<>();
        for (ShowSeat seat : booking.seats) {
            if (lockManager.confirmSeat(seat, booking.bookingId)) {
                confirmed.add(seat);
            } else {
                for (ShowSeat c : confirmed) {
                    // best-effort: seat already BOOKED cannot be un-booked here in real system
                    // this branch means a hold expired during payment — treat as failure
                }
                booking.setStatus(BookingStatus.EXPIRED);
                return false;
            }
        }

        booking.setStatus(BookingStatus.CONFIRMED);
        return true;
    }

    void cancelBooking(Booking booking) {
        for (ShowSeat seat : booking.seats) {
            lockManager.releaseSeat(seat, booking.bookingId);
        }
        booking.setStatus(BookingStatus.CANCELLED);
    }
}

// ===================== Search =====================

class ShowRepository {
    private final List<Show> shows = new ArrayList<>();

    void addShow(Show show) { shows.add(show); }

    List<Show> searchByMovieAndCity(String movieTitle, String cityName) {
        List<Show> result = new ArrayList<>();
        for (Show s : shows) {
            if (s.movie.title.equalsIgnoreCase(movieTitle)
                    && s.theatre.city.name.equalsIgnoreCase(cityName)) {
                result.add(s);
            }
        }
        return result;
    }
}

// ===================== Demo =====================

public class MovieTicketBookingDemo {
    public static void main(String[] args) throws InterruptedException {
        City mumbai = new City("C1", "Mumbai");
        Movie movie = new Movie("M1", "Inception", 148);

        List<Seat> seatList = List.of(
                new Seat("S1", "A1", SeatType.REGULAR),
                new Seat("S2", "A2", SeatType.REGULAR),
                new Seat("S3", "B1", SeatType.PREMIUM)
        );
        Screen screen = new Screen("SC1", "Screen 1", seatList);
        Theatre theatre = new Theatre("T1", "PVR Phoenix", mumbai, List.of(screen));
        Show show = new Show("SH1", movie, screen, theatre, LocalDateTime.now().plusHours(2));

        ShowRepository repo = new ShowRepository();
        repo.addShow(show);

        SeatLockManager lockManager = new SeatLockManager(5 * 60 * 1000L); // 5-minute hold
        BookingService bookingService = new BookingService(lockManager);

        String showSeatId = "SH1_S1"; // seat A1 for this show

        // Simulate two users racing for the same seat at the same time.
        ExecutorService pool = Executors.newFixedThreadPool(2);
        AtomicReference<Booking> winner = new AtomicReference<>();

        Runnable attempt = () -> {
            Booking b = bookingService.createBookingAndHoldSeats("user-" + Thread.currentThread().getId(),
                    show, List.of(showSeatId));
            if (b != null) {
                winner.compareAndSet(null, b);
                System.out.println(Thread.currentThread().getName() + " got the hold: " + b.bookingId);
            } else {
                System.out.println(Thread.currentThread().getName() + " failed to get seat A1");
            }
        };

        pool.submit(attempt);
        pool.submit(attempt);
        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);

        // Only one booking should have won the hold. Now complete payment for it.
        Booking booking = winner.get();
        if (booking != null) {
            boolean ok = bookingService.completePayment(booking, new CardPayment());
            System.out.println("Payment result: " + ok + ", booking status: " + booking.getStatus());
        }
    }
}
```

## How It Works

1. **Catalog browse.** `ShowRepository.searchByMovieAndCity` filters shows by movie title and city name. In a real system this would be a database query with indexes on `(movie_id, city_id, show_date)`.

2. **Seat availability.** `Show.getAvailableSeats()` scans `showSeats` and returns seats that are `AVAILABLE`, or `LOCKED` but expired (treated as available). Each `Show` keeps its own `ShowSeat` map, so the same physical `Seat` can be free for one show time and booked for another.

3. **Holding a seat.** When a user picks seats, `BookingService.createBookingAndHoldSeats` calls `SeatLockManager.tryLockSeat` for each seat, one at a time. Inside `tryLockSeat`, the check ("is it free?") and the update ("mark it LOCKED") happen inside one `synchronized` block keyed on that seat's own lock object. This is the atomic compare-and-set. If two threads call `tryLockSeat` on the same `ShowSeat` at the same instant, the JVM's monitor lock forces one thread to wait. The first thread to enter sees `AVAILABLE`, locks it, and leaves. The second thread then enters, sees `LOCKED` (and not expired), and returns `false`. This is how we guarantee no double-booking.

4. **Fine-grained locking.** We lock per seat, not per show. Booking seat A1 does not block another user from booking seat B1 on the same show, because they use different lock objects. This keeps the system responsive under load — a busy show with 200 seats can process up to 200 seat-holds concurrently in theory.

5. **Rollback on partial failure.** If a booking wants 3 seats and the 3rd one is already taken, `createBookingAndHoldSeats` releases the first 2 seats it already locked, so it does not leave orphaned holds. This is important: a booking must succeed for all requested seats, or none.

6. **Payment step.** `completePayment` runs the `PaymentStrategy` (Strategy pattern lets us swap `CardPayment` for `UpiPayment` without touching `Payment` or `BookingService`). On success, it calls `confirmSeat` for each seat, moving `LOCKED -> BOOKED`. `confirmSeat` re-checks that the hold has not expired and still belongs to this booking, in case payment took too long.

7. **Expiry.** `ShowSeat.isHoldExpired()` compares the current time to `lockExpiryEpochMs`. A background job (in production, a `ScheduledExecutorService` or a message queue delayed job) calls `releaseExpiredHolds` periodically to reclaim seats abandoned mid-checkout, so they do not stay locked forever if a user closes the browser tab.

8. **Booking status.** `BookingStatus` enum tracks the full lifecycle: `CREATED` -> `PAYMENT_PENDING` -> `CONFIRMED` (happy path), or `CANCELLED` / `EXPIRED` (failure paths). This makes the booking's history auditable.

## How to Extend (Follow-ups)

- **Add seat-level pricing tiers and discounts.** Add a `PricingStrategy` interface (Strategy pattern), applied when computing `Booking.totalAmount()`. Keeps `Booking` unaware of discount rules.
- **Notify users.** Add an `Observer` pattern: `BookingService` publishes events (`SeatHeld`, `BookingConfirmed`, `HoldExpired`) to listeners like `EmailNotifier` or `SmsNotifier`, without coupling booking logic to notification code.
- **Waitlist for sold-out shows.** When a show has no available seats, let users join a queue; if a hold expires or a booking is cancelled, offer the seat to the next person in the queue.
- **Multiple payment attempts / retries.** Wrap `PaymentStrategy.pay` with a retry policy (Decorator pattern) for transient payment gateway failures, with a max retry count before giving up and releasing the hold.
- **Admin cancels a show.** Add a `cancelShow` operation that walks all `BOOKED` seats, triggers refunds via `PaymentStrategy`, and notifies affected users.
- **Different seat layouts per screen.** Introduce a `SeatLayout` builder so screens can have curved rows, gaps for aisles, etc., without changing `Seat` or `Show`.
- **Rate limiting.** Add a per-user limit on how many active holds they can have at once, to stop one user from locking all seats and never paying.

## Complexity & Thread-Safety Notes

- **Time complexity.** Searching shows: O(n) over shows in this demo; O(log n) with a database index in production. Checking seat availability: O(k) where k is the number of seats in the show. Locking a seat: O(1) average, since `ConcurrentHashMap.computeIfAbsent` and a `synchronized` block are both constant time.
- **Space complexity.** O(k) per show for `ShowSeat` objects, plus O(k) for lock objects in `SeatLockManager`. For a system with many shows, seat-lock objects for old, past shows should be cleaned up (garbage collected once the `Show` object itself is no longer referenced, since the lock map key is `showSeatId`).
- **Thread-safety of `ShowSeat`.** Fields are `volatile` so reads always see the latest write across threads. But `volatile` alone does not make check-and-set atomic — that is why the actual state transition happens inside a `synchronized` block in `SeatLockManager`, not directly on `ShowSeat`.
- **Why one lock per seat, not one lock for the whole show.** A single lock on the whole `Show` object would work correctly, but it forces every seat-hold request for that show to wait in line, even for unrelated seats. Per-seat locks let unrelated bookings proceed in parallel, which matters a lot for a popular show with hundreds of seats and thousands of concurrent users.
- **Avoiding deadlock.** Because a booking may lock multiple seats, always lock them in one order to avoid two bookings deadlocking each other. Or simpler: allow only one seat lock attempt to be "in flight" per `showSeatId`, and if a multi-seat booking fails partway, always roll back in reverse order (as done in `createBookingAndHoldSeats`). Since we use `synchronized` blocks that are each acquired and released quickly (no nested waiting on another seat's lock), true deadlock is avoided here, but it is worth mentioning to the interviewer.
- **What happens under real concurrency load.** With thousands of users hitting the same popular seat, most `synchronized` calls resolve in microseconds because the critical section is tiny (just a status check and field writes). This design scales well vertically. For horizontal scale, see the note below.

## Interview Tips & Common Mistakes

- **Do not synchronize on the whole `Show` or a global lock.** This is the most common mistake. It works, but interviewers will ask, "what if 500 people try to book different seats on the same hot show at once?" A single lock turns that into a bottleneck. Always argue for per-seat (or per-resource) locking.
- **Explain why `volatile` alone is not enough.** Many candidates say "I'll just use `volatile` on the status field" and think that solves the race condition. It does not — two threads can both read `AVAILABLE` before either writes `LOCKED`. You need an atomic check-and-set, done via `synchronized`, a `ConcurrentHashMap` atomic method (`compute`, `computeIfAbsent`), or a `CAS` (compare-and-swap) primitive like `AtomicReference.compareAndSet`.
- **Always mention the hold/expiry mechanism.** A booking flow without a timeout on the `LOCKED` state is broken: a user who abandons checkout would lock a seat forever. Interviewers specifically probe for this.
- **Say the word "idempotency" for payment.** If a payment webhook fires twice (network retry), calling `confirmSeat` twice should not cause errors — this is why `confirmSeat` checks `bookingId` ownership and current status before acting, so a duplicate call is a safe no-op.
- **Do not forget rollback for multi-seat bookings.** If a booking needs 3 seats and only 2 are available, do not leave the 2 held — release them. This is easy to forget under time pressure and interviewers do check it.
- **Know the two main approaches:** pessimistic locking (lock the seat, as coded here) versus optimistic locking (version check, retry on conflict). Mention both — it shows depth.
- **Distributed scale note.** This whole design lives in one JVM's memory (`ConcurrentHashMap`). At real BookMyShow scale, bookings happen across many servers, so an in-memory `synchronized` block on one server cannot protect a seat that another server's thread might also try to lock. The fix is to move the seat-lock decision to a shared, single source of truth: either (a) a relational database row with `SELECT ... FOR UPDATE` or an `UPDATE ... WHERE status = 'AVAILABLE'` conditional update (the database's own row lock replaces our Java `synchronized` block), or (b) a distributed lock using Redis (`SETNX` with a TTL matching the hold duration) or Redlock, or (c) routing all requests for a given show to one partition/shard so seat state for that show always lives on one node. The state machine (`AVAILABLE -> LOCKED -> BOOKED`) and the hold-with-expiry idea stay exactly the same — only the mechanism enforcing atomicity changes, from a JVM lock to a database or distributed lock. This is a good way to close the interview: show you know the single-JVM answer cold, then name the distributed-systems upgrade path in two or three sentences.
