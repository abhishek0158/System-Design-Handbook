# Meeting Room Booking (LLD)

## Problem

Design a meeting room booking system for an office. The system stores rooms, users, and meetings. A meeting has a room, a start time, and an end time.

A user can book a room for a time range. The system must never let two meetings use the same room at overlapping times. This overlap check is the main design challenge. The system must also let a user search for free rooms in a time range, find a room that fits a given number of people, and cancel a meeting.

Many users can try to book rooms at the same time from different requests. The system must handle this safely, so two users never end up double-booking the same room.

## Requirements & Clarifying Questions

Functional requirements:

1. Model `Room` (with capacity and amenities), `User`, and `Meeting` (room, organizer, attendees, start time, end time).
2. Book a room for a given time range, for a given organizer and a list of attendees.
3. Detect and reject overlapping bookings for the same room. This is the core requirement.
4. Find all rooms that are free for a given time range.
5. Suggest a free room for N people at a given time (filter by capacity, then by availability).
6. Cancel a booking.
7. (Optional) Support recurring meetings (e.g., every Monday, 10 AM to 11 AM, for 8 weeks).
8. (Optional) Notify attendees when a meeting is booked, changed, or cancelled.

Non-functional requirements:

1. No two confirmed meetings can hold the same room for overlapping times, even under concurrent booking requests.
2. The booking check and the booking write must be atomic together (check-then-act must not race).
3. The design should be easy to extend (new notification channels, new search filters, recurring rules).

Clarifying questions to ask the interviewer:

- Do time ranges use a fixed time zone, or must we support multiple time zones? (I will assume a single time zone for simplicity, and use `LocalDateTime`. I will note how `ZonedDateTime` extends this.)
- Are back-to-back meetings allowed, for example one meeting ending at 10:00 and another starting at 10:00 in the same room? (I will assume yes — touching at a boundary point is not an overlap.)
- Is this a single JVM / single process design, or must it work across multiple servers? (I will build a thread-safe single-process design, and add a note on scaling to multiple servers with a database.)
- Should cancelled meetings still show up in history? (Yes, I will keep a status field instead of deleting records.)
- Do we need recurring meetings and notifications for the first version, or are they follow-ups? (I will treat them as optional extensions, shown briefly.)

## Design / Approach

### Key entities

- `User` — id and name. Can be an organizer or an attendee.
- `Room` — id, name, capacity, and a set of amenities (e.g., "projector", "video conference").
- `TimeSlot` — a value object holding `start` and `end` (both `LocalDateTime`). Owns the overlap check.
- `Meeting` — id, room, organizer, attendee list, `TimeSlot`, and a `MeetingStatus` (`CONFIRMED`, `CANCELLED`).
- `BookingService` — the main entry point. Exposes book, cancel, findAvailableRooms, and suggestRoom.
- `RoomScheduleStore` — holds, per room, the list of confirmed meetings, and protects it with a lock. This is where the atomic check-then-book happens.
- `NotificationService` (Observer pattern) — notifies attendees of booking events.
- `RecurrenceRule` (optional) — describes how a meeting repeats, and expands into multiple `Meeting` bookings.

### The interval overlap check (the key part)

Two time ranges `[start1, end1)` and `[start2, end2)` overlap if and only if:

```
start1 < end2   AND   start2 < end1
```

This single line replaces four separate cases (new meeting starts inside existing one, ends inside, fully contains it, or is fully contained by it). If this condition is false, the ranges do not touch, or they only touch at a boundary point (e.g., one ends exactly when the other starts), which we treat as "not overlapping" — back-to-back meetings are allowed.

Example: existing meeting is 10:00–11:00.

- New meeting 10:30–10:45 → `10:30 < 11:00` and `10:00 < 10:45` → both true → overlap. Reject.
- New meeting 09:00–10:00 → `09:00 < 11:00` true, but `10:00 < 10:00` false → no overlap. Allow (back-to-back).
- New meeting 11:00–12:00 → `11:00 < 11:00` false → no overlap. Allow.
- New meeting 09:00–12:00 → `09:00 < 11:00` true and `10:00 < 12:00` true → overlap. Reject (fully contains existing meeting).

A room is free for a new time slot if **no** confirmed meeting in that room overlaps it. So booking is: for a room, scan its confirmed meetings, run the overlap check against each; if none overlap, the slot is free.

### Class sketch

```
+----------------+        +------------------+        +----------------+
|     Room       |        |    TimeSlot      |        |     User       |
+----------------+        +------------------+        +----------------+
| id             |        | start            |        | id             |
| name           |        | end              |        | name           |
| capacity       |        | overlaps(other)  |        +----------------+
| amenities      |        +------------------+
+----------------+

+---------------------+          +------------------------+
|      Meeting        |          |  RoomScheduleStore     |
+---------------------+          +------------------------+
| id                   |          | rooms: Map<id, Room>   |
| room                 |          | locks: Map<id, Lock>   |
| organizer            |          | meetingsByRoom:        |
| attendees            |          |   Map<id, List<Meeting>>|
| slot: TimeSlot       |          +------------------------+
| status               |          | book(room, slot, ...)  |
+---------------------+          | cancel(meetingId)      |
                                  | isFree(room, slot)     |
                                  | findAvailable(slot)    |
                                  +------------------------+
                                            ^
                                            |
                                  +------------------------+
                                  |    BookingService       |
                                  +------------------------+
                                  | bookMeeting(...)         |
                                  | cancelMeeting(id)        |
                                  | findAvailableRooms(slot) |
                                  | suggestRoom(slot, size)  |
                                  +------------------------+
                                            |
                                            v
                                  +------------------------+
                                  |  NotificationService     |
                                  |  (Subject, Observer)     |
                                  +------------------------+
```

### Design patterns used

- **Strategy** — `RoomSelectionStrategy` picks which free room to suggest (e.g., smallest room that fits, or the one with a projector). Lets us change the suggestion rule without touching `BookingService`.
- **Observer** — `NotificationService` and `NotificationChannel` (email, Slack, etc.) observe booking events. `BookingService` does not need to know how attendees are notified.
- **Factory (small)** — `MeetingFactory` builds `Meeting` objects, including expanding a `RecurrenceRule` into multiple meetings.
- **Single Responsibility Principle** — `TimeSlot` only knows about overlap math. `RoomScheduleStore` only knows about storage and locking. `BookingService` only orchestrates. Each class has one reason to change.
- **Open/Closed Principle** — new notification channels or new room-selection rules are added as new classes implementing an interface, without editing existing code.

## Java Solution

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

// ---------- Value objects ----------

final class TimeSlot {
    private final LocalDateTime start;
    private final LocalDateTime end;

    public TimeSlot(LocalDateTime start, LocalDateTime end) {
        if (!start.isBefore(end)) {
            throw new IllegalArgumentException("start must be before end");
        }
        this.start = start;
        this.end = end;
    }

    public LocalDateTime getStart() { return start; }
    public LocalDateTime getEnd() { return end; }

    // Core overlap check: start1 < end2 AND start2 < end1
    public boolean overlaps(TimeSlot other) {
        return this.start.isBefore(other.end) && other.start.isBefore(this.end);
    }

    @Override
    public String toString() {
        return start + " - " + end;
    }
}

// ---------- Entities ----------

final class User {
    private final String id;
    private final String name;

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }

    public String getId() { return id; }
    public String getName() { return name; }
}

final class Room {
    private final String id;
    private final String name;
    private final int capacity;
    private final Set<String> amenities;

    public Room(String id, String name, int capacity, Set<String> amenities) {
        this.id = id;
        this.name = name;
        this.capacity = capacity;
        this.amenities = amenities;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public int getCapacity() { return capacity; }
    public Set<String> getAmenities() { return amenities; }
}

enum MeetingStatus { CONFIRMED, CANCELLED }

final class Meeting {
    private final String id;
    private final Room room;
    private final User organizer;
    private final List<User> attendees;
    private final TimeSlot slot;
    private volatile MeetingStatus status;

    public Meeting(String id, Room room, User organizer, List<User> attendees, TimeSlot slot) {
        this.id = id;
        this.room = room;
        this.organizer = organizer;
        this.attendees = attendees;
        this.slot = slot;
        this.status = MeetingStatus.CONFIRMED;
    }

    public String getId() { return id; }
    public Room getRoom() { return room; }
    public User getOrganizer() { return organizer; }
    public List<User> getAttendees() { return attendees; }
    public TimeSlot getSlot() { return slot; }
    public MeetingStatus getStatus() { return status; }
    public void cancel() { this.status = MeetingStatus.CANCELLED; }
}

// ---------- Observer pattern for notifications ----------

interface NotificationChannel {
    void notify(User user, String message);
}

class EmailChannel implements NotificationChannel {
    @Override
    public void notify(User user, String message) {
        System.out.println("EMAIL to " + user.getName() + ": " + message);
    }
}

class SlackChannel implements NotificationChannel {
    @Override
    public void notify(User user, String message) {
        System.out.println("SLACK to " + user.getName() + ": " + message);
    }
}

class NotificationService {
    private final List<NotificationChannel> channels;

    public NotificationService(List<NotificationChannel> channels) {
        this.channels = channels;
    }

    public void notifyMeetingBooked(Meeting meeting) {
        String msg = "Meeting booked in " + meeting.getRoom().getName() + " at " + meeting.getSlot();
        publish(meeting, msg);
    }

    public void notifyMeetingCancelled(Meeting meeting) {
        String msg = "Meeting cancelled in " + meeting.getRoom().getName() + " at " + meeting.getSlot();
        publish(meeting, msg);
    }

    private void publish(Meeting meeting, String msg) {
        List<User> people = new ArrayList<>(meeting.getAttendees());
        people.add(meeting.getOrganizer());
        for (User u : people) {
            for (NotificationChannel c : channels) {
                c.notify(u, msg);
            }
        }
    }
}

// ---------- Room selection strategy ----------

interface RoomSelectionStrategy {
    Optional<Room> select(List<Room> candidateRooms);
}

class SmallestFitStrategy implements RoomSelectionStrategy {
    @Override
    public Optional<Room> select(List<Room> candidateRooms) {
        return candidateRooms.stream()
                .min(Comparator.comparingInt(Room::getCapacity));
    }
}

// ---------- Custom exception ----------

class RoomNotAvailableException extends RuntimeException {
    public RoomNotAvailableException(String message) { super(message); }
}

// ---------- Core store: rooms + per-room lock + schedule ----------

class RoomScheduleStore {
    private final Map<String, Room> rooms = new ConcurrentHashMap<>();
    private final Map<String, List<Meeting>> meetingsByRoom = new ConcurrentHashMap<>();
    private final Map<String, Lock> roomLocks = new ConcurrentHashMap<>();

    public void addRoom(Room room) {
        rooms.put(room.getId(), room);
        meetingsByRoom.put(room.getId(), new ArrayList<>());
        roomLocks.put(room.getId(), new ReentrantLock());
    }

    public Collection<Room> allRooms() {
        return rooms.values();
    }

    public Room getRoom(String roomId) {
        Room r = rooms.get(roomId);
        if (r == null) throw new NoSuchElementException("No such room: " + roomId);
        return r;
    }

    // Atomic check-then-book for one room. Only one thread at a time
    // can run this block for a given room, because we hold that room's lock.
    public Meeting bookIfFree(String roomId, TimeSlot slot, User organizer,
                               List<User> attendees, String meetingId) {
        Lock lock = roomLocks.get(roomId);
        lock.lock();
        try {
            List<Meeting> existing = meetingsByRoom.get(roomId);
            for (Meeting m : existing) {
                if (m.getStatus() == MeetingStatus.CONFIRMED && m.getSlot().overlaps(slot)) {
                    throw new RoomNotAvailableException(
                        "Room " + roomId + " already booked for an overlapping time: " + m.getSlot());
                }
            }
            Meeting meeting = new Meeting(meetingId, getRoom(roomId), organizer, attendees, slot);
            existing.add(meeting);
            return meeting;
        } finally {
            lock.unlock();
        }
    }

    public boolean isFree(String roomId, TimeSlot slot) {
        Lock lock = roomLocks.get(roomId);
        lock.lock();
        try {
            return meetingsByRoom.get(roomId).stream()
                    .filter(m -> m.getStatus() == MeetingStatus.CONFIRMED)
                    .noneMatch(m -> m.getSlot().overlaps(slot));
        } finally {
            lock.unlock();
        }
    }

    public void cancel(String roomId, String meetingId) {
        Lock lock = roomLocks.get(roomId);
        lock.lock();
        try {
            for (Meeting m : meetingsByRoom.get(roomId)) {
                if (m.getId().equals(meetingId) && m.getStatus() == MeetingStatus.CONFIRMED) {
                    m.cancel();
                    return;
                }
            }
            throw new NoSuchElementException("No active meeting " + meetingId + " in room " + roomId);
        } finally {
            lock.unlock();
        }
    }
}

// ---------- Booking service (facade / orchestrator) ----------

class BookingService {
    private final RoomScheduleStore store;
    private final NotificationService notificationService;
    private final RoomSelectionStrategy selectionStrategy;
    private final Map<String, String> meetingIdToRoomId = new ConcurrentHashMap<>();
    private final java.util.concurrent.atomic.AtomicLong idCounter = new java.util.concurrent.atomic.AtomicLong(1);

    public BookingService(RoomScheduleStore store,
                           NotificationService notificationService,
                           RoomSelectionStrategy selectionStrategy) {
        this.store = store;
        this.notificationService = notificationService;
        this.selectionStrategy = selectionStrategy;
    }

    public Meeting bookMeeting(String roomId, TimeSlot slot, User organizer, List<User> attendees) {
        String meetingId = "M-" + idCounter.getAndIncrement();
        Meeting meeting = store.bookIfFree(roomId, slot, organizer, attendees, meetingId);
        meetingIdToRoomId.put(meetingId, roomId);
        notificationService.notifyMeetingBooked(meeting);
        return meeting;
    }

    public void cancelMeeting(String meetingId) {
        String roomId = meetingIdToRoomId.get(meetingId);
        if (roomId == null) throw new NoSuchElementException("Unknown meeting: " + meetingId);
        store.cancel(roomId, meetingId);
    }

    public List<Room> findAvailableRooms(TimeSlot slot) {
        List<Room> free = new ArrayList<>();
        for (Room room : store.allRooms()) {
            if (store.isFree(room.getId(), slot)) {
                free.add(room);
            }
        }
        return free;
    }

    public Optional<Room> suggestRoom(TimeSlot slot, int headCount) {
        List<Room> candidates = new ArrayList<>();
        for (Room room : findAvailableRooms(slot)) {
            if (room.getCapacity() >= headCount) {
                candidates.add(room);
            }
        }
        return selectionStrategy.select(candidates);
    }

    // Optional: expand a recurring meeting into individual bookings.
    // Skips slots that fail (already booked) and reports which ones failed.
    public List<Meeting> bookRecurring(String roomId, RecurrenceRule rule, User organizer, List<User> attendees) {
        List<Meeting> booked = new ArrayList<>();
        for (TimeSlot slot : rule.expand()) {
            try {
                booked.add(bookMeeting(roomId, slot, organizer, attendees));
            } catch (RoomNotAvailableException e) {
                System.out.println("Skipped occurrence " + slot + ": " + e.getMessage());
            }
        }
        return booked;
    }
}

// ---------- Optional: recurring meetings ----------

class RecurrenceRule {
    private final LocalDateTime firstStart;
    private final LocalDateTime firstEnd;
    private final int intervalDays;
    private final int occurrences;

    public RecurrenceRule(LocalDateTime firstStart, LocalDateTime firstEnd,
                           int intervalDays, int occurrences) {
        this.firstStart = firstStart;
        this.firstEnd = firstEnd;
        this.intervalDays = intervalDays;
        this.occurrences = occurrences;
    }

    public List<TimeSlot> expand() {
        List<TimeSlot> slots = new ArrayList<>();
        for (int i = 0; i < occurrences; i++) {
            LocalDateTime start = firstStart.plusDays((long) i * intervalDays);
            LocalDateTime end = firstEnd.plusDays((long) i * intervalDays);
            slots.add(new TimeSlot(start, end));
        }
        return slots;
    }
}

// ---------- Demo ----------

public class MeetingRoomBookingDemo {
    public static void main(String[] args) {
        RoomScheduleStore store = new RoomScheduleStore();
        store.addRoom(new Room("R1", "Falcon", 6, Set.of("projector")));
        store.addRoom(new Room("R2", "Eagle", 12, Set.of("projector", "video-conf")));

        NotificationService notifier = new NotificationService(
                List.of(new EmailChannel(), new SlackChannel()));
        BookingService bookingService = new BookingService(store, notifier, new SmallestFitStrategy());

        User alice = new User("U1", "Alice");
        User bob = new User("U2", "Bob");

        TimeSlot slot1 = new TimeSlot(
                LocalDateTime.of(2026, 8, 24, 10, 0),
                LocalDateTime.of(2026, 8, 24, 11, 0));

        Meeting m1 = bookingService.bookMeeting("R1", slot1, alice, List.of(bob));
        System.out.println("Booked: " + m1.getId() + " in " + m1.getRoom().getName());

        // Overlapping request on the same room -> rejected
        TimeSlot overlapping = new TimeSlot(
                LocalDateTime.of(2026, 8, 24, 10, 30),
                LocalDateTime.of(2026, 8, 24, 10, 45));
        try {
            bookingService.bookMeeting("R1", overlapping, bob, List.of(alice));
        } catch (RoomNotAvailableException e) {
            System.out.println("Rejected as expected: " + e.getMessage());
        }

        // Back-to-back booking, same room, no overlap -> allowed
        TimeSlot backToBack = new TimeSlot(
                LocalDateTime.of(2026, 8, 24, 11, 0),
                LocalDateTime.of(2026, 8, 24, 11, 30));
        bookingService.bookMeeting("R1", backToBack, bob, List.of(alice));
        System.out.println("Back-to-back booking allowed.");

        // Find rooms for 10 people at a free time
        TimeSlot afternoon = new TimeSlot(
                LocalDateTime.of(2026, 8, 24, 14, 0),
                LocalDateTime.of(2026, 8, 24, 15, 0));
        Optional<Room> suggestion = bookingService.suggestRoom(afternoon, 10);
        suggestion.ifPresent(r -> System.out.println("Suggested room: " + r.getName()));

        // Cancel
        bookingService.cancelMeeting(m1.getId());
        System.out.println("Cancelled " + m1.getId());
        System.out.println("R1 free at slot1 now? " + store.isFree("R1", slot1));
    }
}
```

## How It Works

`TimeSlot.overlaps()` holds the one rule the whole system depends on: `start1 < end2 AND start2 < end1`. Every availability check and every booking check calls this method, so the overlap logic exists in exactly one place. If the rule ever changes (for example, to disallow back-to-back meetings), we edit one method.

`RoomScheduleStore` owns the list of confirmed meetings per room and a `ReentrantLock` per room. `bookIfFree()` takes the lock, scans existing meetings for an overlap, and only if none is found, adds the new meeting — all inside the same locked block. This makes "check" and "add" one atomic step. Two threads racing to book room `R1` for overlapping times cannot both pass the check, because the second thread only enters the lock after the first thread has already added its meeting and released the lock.

`BookingService` is the facade the client (UI, REST controller, etc.) talks to. It does not know about locks or storage details — it delegates to `RoomScheduleStore` and then tells `NotificationService` about the result. This keeps orchestration separate from concurrency control (Single Responsibility Principle).

`NotificationService` implements the Observer pattern in a simple form: it holds a list of `NotificationChannel` implementations (email, Slack) and calls each one whenever a meeting is booked or cancelled. Adding a new channel, like SMS, means writing one new class — no existing class changes (Open/Closed Principle).

`RoomSelectionStrategy` lets `suggestRoom()` change its picking rule (smallest room that fits today, most amenities tomorrow) without changing `BookingService`. This is the Strategy pattern.

`RecurrenceRule.expand()` turns one recurring rule into a list of `TimeSlot`s. `BookingService.bookRecurring()` tries to book each one independently and reports which occurrences failed, instead of failing the whole series if one date has a conflict.

## How to Extend (Follow-ups)

- **Multiple time zones**: replace `LocalDateTime` with `ZonedDateTime`, and convert to a common zone (e.g., UTC) before calling `overlaps()`. The overlap formula itself does not change.
- **Persistent storage**: replace the in-memory `Map`s in `RoomScheduleStore` with a database table `meetings(room_id, start_time, end_time, status)`. Use a database-level lock (`SELECT ... FOR UPDATE` on the room row, or a unique constraint plus retry) instead of `ReentrantLock`, so the atomicity holds across multiple application servers.
- **Room amenities filter**: extend `suggestRoom()` to take a `Set<String> requiredAmenities` and filter candidates by `room.getAmenities().containsAll(requiredAmenities)`.
- **Waitlist**: if no room is free, add the request to a waitlist queue for that time slot. When a booking is cancelled, check the waitlist and auto-book the next request (Observer pattern again — the waitlist observes cancellations).
- **Editing a meeting (change time or room)**: implement as cancel-then-rebook inside one lock scope (lock the old room and the new room, in a fixed order such as sorted by room id, to avoid deadlock) so the change is atomic and does not leave a gap where the room looks free to someone else.
- **Buffer time between meetings**: add a `bufferMinutes` field, and treat a proposed slot as `[start - buffer, end + buffer)` only for the overlap check, not for the stored meeting time.
- **Fairness / booking limits**: add a `BookingPolicy` interface (Strategy again) that `BookingService` consults before booking, for example, "one user cannot hold more than 3 active meetings."

## Complexity & Thread-Safety Notes

- **Booking a room**: O(k) where k is the number of existing confirmed meetings in that room, because we scan the room's meeting list for an overlap. This can be improved to O(log k) with a sorted interval tree or a `TreeMap` keyed by start time, but for typical room schedules (tens of meetings per day), a linear scan is fast enough and much simpler to write in an interview.
- **findAvailableRooms**: O(n × k) where n is the number of rooms and k is meetings per room — we check each room's schedule once.
- **suggestRoom**: same as `findAvailableRooms`, plus O(n log n) for picking the smallest fit.
- **Thread-safety**: the important part is that "check for overlap" and "add the meeting" happen under the same lock, for the same room. Locking is per-room, not global, so bookings for different rooms proceed fully in parallel — only requests for the *same* room ever wait on each other.
- **Why not just `synchronized` on the whole service?** That would work correctly, but it forces every booking request, even for different rooms, to run one at a time. Per-room locks give the same correctness with much better throughput.
- **Alternative to explicit locks**: use `ConcurrentHashMap<roomId, List<Meeting>>.compute()` so the read-check-write happens inside the map's own per-key atomic operation. This is a valid alternative to a `ReentrantLock` map and is worth mentioning if the interviewer asks for another approach.
- **Deadlock care**: any operation that must lock two rooms at once (like the "move meeting to a different room" follow-up) must always acquire locks in a fixed, consistent order (e.g., by room id) to avoid two threads deadlocking by locking in opposite order.

## Interview Tips & Common Mistakes

- State the overlap formula out loud and explain it with a picture or example before writing code. Interviewers often just want to hear you say `start1 < end2 AND start2 < end1` and explain why it covers all four overlap cases in one line.
- A very common mistake: checking `start1 <= end2` (using `<=` instead of `<`). This wrongly rejects back-to-back meetings that touch at a single point. Confirm with the interviewer whether back-to-back is allowed, then pick `<` or `<=` on purpose.
- Do not do the overlap check and the "add meeting" write as two separate steps without a lock between them. That is a classic check-then-act race condition — two threads can both pass the check before either writes, and both bookings succeed, double-booking the room.
- Locking the entire system (one global lock for all rooms) is a common shortcut that "works" but kills concurrency. Mention per-room locking to show you understand fine-grained locking.
- Keep `TimeSlot` as its own class with its own `overlaps()` method, instead of passing raw `start`/`end` fields around and repeating the overlap check in multiple places. Interviewers watch for this kind of encapsulation.
- Do not delete a cancelled meeting from the list. Keep it with `status = CANCELLED` so history and audits work, and make sure the overlap check skips cancelled meetings.
- Be ready to explain how the design changes for multiple servers: the per-JVM lock only works within one process. In a distributed system, you need the database itself to guarantee atomicity, for example a unique constraint on `(room_id, time range)` with an exclusion constraint, or a distributed lock.
- Mention the Observer pattern by name when discussing notifications, and the Strategy pattern by name when discussing room suggestion. Naming the pattern signals you know the vocabulary, not just the code.
