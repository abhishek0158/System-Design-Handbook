# Library Management System (LLD)

## Problem

Design a library management system for a single branch. A library has book titles (like "Clean Code" by Robert Martin) and physical copies of each title on the shelf. The system must:

- Store book metadata: title, author, ISBN (a unique code for a book title).
- Store many physical copies for one title. Each copy is a separate item that can be on the shelf or out with a member.
- Let a user search books by title, author, or ISBN.
- Let a member borrow (issue) one copy of a title, with a due date and a limit on how many books one member can hold at a time.
- Let a member return a copy.
- Let a member reserve a title when every copy is currently borrowed, and hold their place in a queue.
- Charge a late fine when a copy is returned after its due date. The fine rule must be easy to swap (flat rate today, a different rule tomorrow).
- Track each copy's status: `AVAILABLE`, `ISSUED`, or `RESERVED`.
- Support a librarian, who manages the catalog (add books, add copies) and can also issue and return copies on a member's behalf at the front desk.

## Requirements & Clarifying Questions

1. **One branch or many?** Assume one branch, one in-memory catalog, behind repository interfaces. Multi-branch support is a Follow-up.
2. **What is the real difference between a Book and a BookCopy?** A `Book` is the title — the ISBN, the author, the description. It exists once. A `BookCopy` is one physical item on the shelf with a barcode. A popular title can have five `BookCopy` objects, all pointing to the same `Book`. Borrowing, returning, and status all happen at the `BookCopy` level, never at the `Book` level. This split is the core modeling decision of this sheet.
3. **How many books can one member hold at once?** A configurable limit per member, e.g. 5. The limit must never be exceeded, even if the member issues two requests at nearly the same time.
4. **What is the loan period?** Fixed at 14 days for this sheet, stored as a constant. A follow-up covers member-tier-based periods (e.g. staff vs. student).
5. **When can a member reserve a title?** Only when no copy of that title is `AVAILABLE` right now. If a copy is free, the member should borrow it directly.
6. **What happens when a reserved copy comes back?** The next member in the queue, first-in-first-out (FIFO), gets first claim on the returned copy. The copy status becomes `RESERVED` (held for that one member), not `AVAILABLE`, so nobody else can take it during the pickup window.
7. **How is the fine computed?** `(days late) x (fine per day)`, using a pluggable strategy, so the rate — or the whole rule — can change without touching the rest of the system.
8. **Is money exact?** Fines use `long` minor currency units (e.g. cents), never `double`, to avoid rounding errors.

Assumptions: single JVM, in-memory storage behind repository interfaces, one branch, whole-day due dates, FIFO reservation queue, no renewals (listed as a Follow-up).

## Design / Approach

```
+-------+  1..n copies of  +-----------+          +--------------------+
| Book  | <--------------- | BookCopy  |          |   LibraryService    |
+-------+                  | (status)  |          |      (Facade)       |
                           +-----+-----+          +----------+-----------+
                                 ^                            |
                                 | findCopiesByIsbn            | issueCopy / returnCopy /
                                 |                              | reserveBook / search
                    +------------+-------------+               |
                    |      BookRepository       |<--------------+
                    |       (interface)         |
                    +------------+-------------+                |
                                 |                                v
                        InMemoryBookRepository            +---------------+
                                                           |    Member     |
                                                           | (borrow slot) |
                                                           +---------------+
                                                                   ^
                                                                   | MemberRepository
                                                                   |
                    On return, if a hold queue exists for the ISBN:
                    +---------------------+   Strategy    +------------------+
                    |   FineStrategy       |<------------- | LibraryService   |
                    | (fine calculation)   |               +---------+--------+
                    +----------------------+                         |
                                                                       v
                    +----------------------+   Observer    +------------------+
                    | NotificationService  |<------------- |  Hold Queue      |
                    | (tell next member)   |               | (per ISBN, FIFO) |
                    +----------------------+               +------------------+
```

Patterns and principles used:

- **Repository (`BookRepository`, `MemberRepository`)** — `LibraryService` talks to an interface, not to a `HashMap` directly. Swapping in a database-backed store later needs no change to the service (**Dependency Inversion Principle**).
- **Facade (`LibraryService`)** — one entry point for search, issue, return, and reserve. Callers never touch `BookCopy` or the hold queue directly.
- **Strategy (`FineStrategy`)** — the fine rule is an interface. Adding a new rule (e.g. a cap on the maximum fine) means adding one new class, with no change to `LibraryService` (**Open/Closed Principle**).
- **Observer (`NotificationService`)** — when a reserved copy is ready, `LibraryService` does not know or care how the member is told (console message today, email or push notice tomorrow). It only calls `notifyReservationReady`.
- **Single Responsibility** — `BookCopy` only owns one physical item's status and its own lock. `Member` only owns its own borrow count. `LibraryService` only coordinates the steps; it holds no state of its own beyond the hold queues.

**Why `Book` and `BookCopy` are two classes, not one.** A common mistake is to put a `boolean available` flag directly on `Book`. That breaks the moment a title has more than one copy — a status flag on the title cannot say "3 copies are out, 2 are on the shelf." Each physical copy needs its own status, its own due date, and its own borrower. `Book` stays a pure, almost-unchanging value (title, author, ISBN). `BookCopy` is the unit that moves through `AVAILABLE -> ISSUED -> AVAILABLE` (or `-> RESERVED -> ISSUED`) over its life.

**Concurrency design in one line.** Two members must never both walk away with the same physical copy. `BookCopy` guards its own status, borrower, and due date behind one `synchronized` block per copy, so a race only ever serializes threads fighting over the *same* copy — copies of other titles, or other copies of the same title, are untouched and run in parallel. This is explained in full in **How It Works**.

## Java Solution

### Book, CopyStatus, and BookCopy

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/** A title's metadata. One Book can back many physical BookCopy objects. */
public final class Book {
    private final String isbn;
    private final String title;
    private final String author;

    public Book(String isbn, String title, String author) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
    }
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
}

public enum CopyStatus { AVAILABLE, ISSUED, RESERVED, LOST }

/** Small, immutable receipt of what a copy looked like right before it was returned. */
public final class IssueRecord {
    private final String memberId;
    private final LocalDate dueDate;

    public IssueRecord(String memberId, LocalDate dueDate) {
        this.memberId = memberId;
        this.dueDate = dueDate;
    }
    public String getMemberId() { return memberId; }
    public LocalDate getDueDate() { return dueDate; }
}

/**
 * One physical item on the shelf. All mutable state (status, borrower, due date,
 * who a hold is reserved for) lives here and is guarded by this object's own lock,
 * so only threads racing for THIS copy ever wait on each other.
 */
public class BookCopy {
    private final String copyId;
    private final String isbn;
    private CopyStatus status;
    private String issuedToMemberId;
    private LocalDate dueDate;
    private String heldForMemberId;

    public BookCopy(String copyId, String isbn) {
        this.copyId = copyId;
        this.isbn = isbn;
        this.status = CopyStatus.AVAILABLE;
    }

    /**
     * Tries to hand this copy to memberId. Succeeds if the copy is free, or if it
     * is held (RESERVED) for exactly this member. Synchronized so two threads
     * racing for the same copy cannot both succeed.
     */
    public synchronized boolean tryIssue(String memberId, LocalDate dueDate) {
        boolean freeForThisMember = status == CopyStatus.AVAILABLE
                || (status == CopyStatus.RESERVED && memberId.equals(heldForMemberId));
        if (!freeForThisMember) return false;
        status = CopyStatus.ISSUED;
        issuedToMemberId = memberId;
        this.dueDate = dueDate;
        heldForMemberId = null;
        return true;
    }

    /** Marks the copy returned and clears its issue data. */
    public synchronized IssueRecord markReturned() {
        IssueRecord record = new IssueRecord(issuedToMemberId, dueDate);
        status = CopyStatus.AVAILABLE;
        issuedToMemberId = null;
        dueDate = null;
        return record;
    }

    /** Earmarks this copy for one waiting member. Called right after a return. */
    public synchronized void holdFor(String memberId) {
        status = CopyStatus.RESERVED;
        heldForMemberId = memberId;
    }

    public synchronized CopyStatus getStatus() { return status; }
    public synchronized LocalDate getDueDate() { return dueDate; }
    public synchronized String getIssuedToMemberId() { return issuedToMemberId; }
    public String getCopyId() { return copyId; }
    public String getIsbn() { return isbn; }
}
```

### Member and Librarian

```java
/** A library user who can borrow copies, up to a fixed limit at any one time. */
public class Member {
    private final String memberId;
    private final String name;
    private final int maxBooksAllowed;
    private final AtomicInteger borrowedCount = new AtomicInteger(0);

    public Member(String memberId, String name, int maxBooksAllowed) {
        this.memberId = memberId;
        this.name = name;
        this.maxBooksAllowed = maxBooksAllowed;
    }

    /**
     * Atomically claims one borrow slot, but only if the member is still under
     * the limit. Uses a compare-and-swap (CAS) loop: read the count, check the
     * limit, then try to write count+1 back only if nobody changed it meanwhile.
     */
    public boolean tryReserveBorrowSlot() {
        while (true) {
            int current = borrowedCount.get();
            if (current >= maxBooksAllowed) return false;
            if (borrowedCount.compareAndSet(current, current + 1)) return true;
            // another thread changed the count first; retry with a fresh read
        }
    }

    /** Gives a slot back, e.g. on return, or if issuing failed after the slot was claimed. */
    public void releaseBorrowSlot() {
        borrowedCount.updateAndGet(c -> Math.max(0, c - 1));
    }

    public String getMemberId() { return memberId; }
    public String getName() { return name; }
    public int getBorrowedCount() { return borrowedCount.get(); }
}

/** Front-desk staff. Uses the same LibraryService as a member's self-checkout would. */
public final class Librarian {
    private final String employeeId;
    private final String name;

    public Librarian(String employeeId, String name) {
        this.employeeId = employeeId;
        this.name = name;
    }
    public String getEmployeeId() { return employeeId; }
    public String getName() { return name; }
}
```

### Repositories (Repository Pattern)

```java
public interface BookRepository {
    void addBook(Book book);
    Optional<Book> findByIsbn(String isbn);
    List<Book> findByTitle(String titleQuery);
    List<Book> findByAuthor(String authorQuery);
    void addCopy(BookCopy copy);
    List<BookCopy> findCopiesByIsbn(String isbn);
    Optional<BookCopy> findCopyById(String copyId);
}

public class InMemoryBookRepository implements BookRepository {
    private final Map<String, Book> booksByIsbn = new ConcurrentHashMap<>();
    private final Map<String, List<BookCopy>> copiesByIsbn = new ConcurrentHashMap<>();
    private final Map<String, BookCopy> copiesById = new ConcurrentHashMap<>();

    @Override
    public void addBook(Book book) {
        booksByIsbn.put(book.getIsbn(), book);
        copiesByIsbn.putIfAbsent(book.getIsbn(), new CopyOnWriteArrayList<>());
    }
    @Override
    public Optional<Book> findByIsbn(String isbn) { return Optional.ofNullable(booksByIsbn.get(isbn)); }
    @Override
    public List<Book> findByTitle(String titleQuery) {
        String needle = titleQuery.toLowerCase();
        List<Book> result = new ArrayList<>();
        for (Book b : booksByIsbn.values()) {
            if (b.getTitle().toLowerCase().contains(needle)) result.add(b);
        }
        return result;
    }
    @Override
    public List<Book> findByAuthor(String authorQuery) {
        String needle = authorQuery.toLowerCase();
        List<Book> result = new ArrayList<>();
        for (Book b : booksByIsbn.values()) {
            if (b.getAuthor().toLowerCase().contains(needle)) result.add(b);
        }
        return result;
    }
    @Override
    public void addCopy(BookCopy copy) {
        copiesById.put(copy.getCopyId(), copy);
        copiesByIsbn.computeIfAbsent(copy.getIsbn(), k -> new CopyOnWriteArrayList<>()).add(copy);
    }
    @Override
    public List<BookCopy> findCopiesByIsbn(String isbn) {
        return copiesByIsbn.getOrDefault(isbn, Collections.emptyList());
    }
    @Override
    public Optional<BookCopy> findCopyById(String copyId) { return Optional.ofNullable(copiesById.get(copyId)); }
}

public interface MemberRepository {
    void addMember(Member member);
    Optional<Member> findById(String memberId);
}

public class InMemoryMemberRepository implements MemberRepository {
    private final Map<String, Member> members = new ConcurrentHashMap<>();

    @Override
    public void addMember(Member member) { members.put(member.getMemberId(), member); }
    @Override
    public Optional<Member> findById(String memberId) { return Optional.ofNullable(members.get(memberId)); }
}
```

### Reservation and Hold Queue

```java
public enum ReservationStatus { WAITING, READY_FOR_PICKUP, FULFILLED, CANCELLED }

/** One member's place in line for a title that has no free copy right now. */
public class Reservation {
    private final String reservationId;
    private final String isbn;
    private final String memberId;
    private final LocalDateTime createdAt;
    private volatile ReservationStatus status;

    public Reservation(String reservationId, String isbn, String memberId) {
        this.reservationId = reservationId;
        this.isbn = isbn;
        this.memberId = memberId;
        this.createdAt = LocalDateTime.now();
        this.status = ReservationStatus.WAITING;
    }
    public String getReservationId() { return reservationId; }
    public String getIsbn() { return isbn; }
    public String getMemberId() { return memberId; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public ReservationStatus getStatus() { return status; }
    public void setStatus(ReservationStatus status) { this.status = status; }
}
```

The hold queue itself is one `ConcurrentLinkedQueue<Reservation>` per ISBN, kept inside `LibraryService`. `ConcurrentLinkedQueue` is a lock-free, first-in-first-out (FIFO) queue — the natural fit for "first member to reserve gets served first."

### Fine Strategy (Strategy Pattern)

```java
public interface FineStrategy {
    /** Returns the fine in minor currency units (e.g. cents). Zero if not overdue. */
    long calculateFine(LocalDate dueDate, LocalDate returnDate);
}

public class FixedDailyFineStrategy implements FineStrategy {
    private final long finePerDayMinorUnits;

    public FixedDailyFineStrategy(long finePerDayMinorUnits) {
        this.finePerDayMinorUnits = finePerDayMinorUnits;
    }
    @Override
    public long calculateFine(LocalDate dueDate, LocalDate returnDate) {
        if (!returnDate.isAfter(dueDate)) return 0L;
        long overdueDays = ChronoUnit.DAYS.between(dueDate, returnDate);
        return overdueDays * finePerDayMinorUnits;
    }
}
```

### Notification (Observer Pattern)

```java
public interface NotificationService {
    void notifyReservationReady(String memberId, String isbn, String copyId, LocalDate pickupDeadline);
}

public class ConsoleNotificationService implements NotificationService {
    @Override
    public void notifyReservationReady(String memberId, String isbn, String copyId, LocalDate pickupDeadline) {
        System.out.printf("Member %s: your reserved title (ISBN %s) is ready. " +
                "Pick up copy %s by %s.%n", memberId, isbn, copyId, pickupDeadline);
    }
}
```

### Exceptions

```java
public class MaxBooksExceededException extends RuntimeException {
    public MaxBooksExceededException(String memberId) {
        super("Member " + memberId + " has reached the maximum number of borrowed books");
    }
}
public class NoCopyAvailableException extends RuntimeException {
    public NoCopyAvailableException(String isbn) {
        super("No copy available right now for ISBN " + isbn);
    }
}
```

### Library Service (Facade)

```java
public class LibraryService {
    private static final int LOAN_PERIOD_DAYS = 14;
    private static final int PICKUP_WINDOW_DAYS = 3;

    private final BookRepository bookRepository;
    private final MemberRepository memberRepository;
    private final FineStrategy fineStrategy;
    private final NotificationService notificationService;
    private final Map<String, Queue<Reservation>> holdQueues = new ConcurrentHashMap<>();

    public LibraryService(BookRepository bookRepository, MemberRepository memberRepository,
                           FineStrategy fineStrategy, NotificationService notificationService) {
        this.bookRepository = bookRepository;
        this.memberRepository = memberRepository;
        this.fineStrategy = fineStrategy;
        this.notificationService = notificationService;
    }

    public void addBookTitle(Book book) { bookRepository.addBook(book); }
    public void addCopy(BookCopy copy) { bookRepository.addCopy(copy); }
    public void registerMember(Member member) { memberRepository.addMember(member); }

    public List<Book> searchByTitle(String query) { return bookRepository.findByTitle(query); }
    public List<Book> searchByAuthor(String query) { return bookRepository.findByAuthor(query); }
    public Optional<Book> searchByIsbn(String isbn) { return bookRepository.findByIsbn(isbn); }

    /** Issues (borrows) one copy of the given title to the given member. Returns the copy id. */
    public String issueCopy(String isbn, String memberId) {
        Member member = requireMember(memberId);
        if (!member.tryReserveBorrowSlot()) {
            throw new MaxBooksExceededException(memberId);
        }
        LocalDate dueDate = LocalDate.now().plusDays(LOAN_PERIOD_DAYS);
        for (BookCopy copy : bookRepository.findCopiesByIsbn(isbn)) {
            if (copy.tryIssue(memberId, dueDate)) {
                return copy.getCopyId();
            }
        }
        member.releaseBorrowSlot(); // nothing was won; give the slot back
        throw new NoCopyAvailableException(isbn);
    }

    /** Returns a copy. Computes a fine if overdue, then serves the next reservation, if any. */
    public long returnCopy(String copyId) {
        BookCopy copy = bookRepository.findCopyById(copyId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown copy: " + copyId));
        IssueRecord record = copy.markReturned();
        if (record.getMemberId() != null) {
            memberRepository.findById(record.getMemberId()).ifPresent(Member::releaseBorrowSlot);
        }
        long fine = (record.getDueDate() == null)
                ? 0L
                : fineStrategy.calculateFine(record.getDueDate(), LocalDate.now());

        Queue<Reservation> queue = holdQueues.get(copy.getIsbn());
        if (queue != null) {
            Reservation next = queue.poll();
            if (next != null) {
                copy.holdFor(next.getMemberId());
                next.setStatus(ReservationStatus.READY_FOR_PICKUP);
                LocalDate deadline = LocalDate.now().plusDays(PICKUP_WINDOW_DAYS);
                notificationService.notifyReservationReady(
                        next.getMemberId(), copy.getIsbn(), copy.getCopyId(), deadline);
            }
        }
        return fine;
    }

    /** Reserves a title that has no free copy right now. Adds the member to a FIFO queue. */
    public Reservation reserveBook(String isbn, String memberId) {
        boolean anyAvailable = bookRepository.findCopiesByIsbn(isbn).stream()
                .anyMatch(c -> c.getStatus() == CopyStatus.AVAILABLE);
        if (anyAvailable) {
            throw new IllegalStateException("A copy is available; borrow it directly instead of reserving.");
        }
        Reservation reservation = new Reservation(UUID.randomUUID().toString(), isbn, memberId);
        holdQueues.computeIfAbsent(isbn, k -> new ConcurrentLinkedQueue<>()).add(reservation);
        return reservation;
    }

    private Member requireMember(String memberId) {
        return memberRepository.findById(memberId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown member: " + memberId));
    }
}
```

## How It Works

**Adding a title with several copies.** A librarian calls `addBookTitle` once for "Clean Code" (one `Book`, one ISBN). Then `addCopy` is called three times, once per physical copy, each with a new `copyId` but the same ISBN. The catalog now holds one title and three independent `BookCopy` objects, each starting as `AVAILABLE`.

**Normal borrow.** A member calls `issueCopy(isbn, memberId)`. First, `member.tryReserveBorrowSlot()` runs its CAS loop: it reads the current borrowed count, checks it against the limit, and only writes the incremented value back if nothing changed in between. If the member is already at the limit, the call returns `false` right away and `issueCopy` throws `MaxBooksExceededException` before touching any copy. If the slot is granted, `LibraryService` loops over the title's copies and calls `tryIssue` on each until one accepts — that copy flips from `AVAILABLE` to `ISSUED`, records the borrower and due date, and its `copyId` is returned.

**Normal return.** A member calls `returnCopy(copyId)`. `BookCopy.markReturned()` clears the copy's issue data and reports who had it and what the due date was. `LibraryService` gives the member's borrow slot back, and asks `FineStrategy` to compute a fine from the due date and today. If the return is on time, `calculateFine` returns `0`. If it is late, it returns `(days late) x (rate)`.

**Reservation flow.** Say all three copies of "Clean Code" are `ISSUED`. A fourth member calls `reserveBook(isbn, memberId)`. The availability check finds nothing free, so the member is added to the ISBN's `ConcurrentLinkedQueue`. Later, one copy comes back through `returnCopy`. Before returning the fine, `LibraryService` checks the hold queue for that ISBN, polls the oldest reservation, and calls `copy.holdFor(memberId)` — the copy becomes `RESERVED`, earmarked for that one member, not `AVAILABLE` to everyone. `NotificationService` tells the member their book is ready. When that member later calls `issueCopy`, `tryIssue` sees the copy is `RESERVED` and `heldForMemberId` matches, so it succeeds and the copy becomes `ISSUED`. If a different member had tried `issueCopy` in the meantime, `tryIssue` on that copy would return `false`, since the hold does not match their id.

**The last-copy race — why it is safe.** Picture the very last `AVAILABLE` copy of a title, with two members calling `issueCopy` for the same ISBN at the same instant. Both threads pass their own `tryReserveBorrowSlot()` check (that check only limits how many books *one* member holds, so it does not block either of them). Both threads then loop over the same list of copies and both reach the same last `BookCopy`. `tryIssue` is `synchronized` on that one `BookCopy` instance: only one thread can be inside it at a time. The first thread to enter sees `status == AVAILABLE`, flips it to `ISSUED`, and returns `true`. The second thread then enters the same synchronized block, now sees `status == ISSUED`, and returns `false`. Its outer loop finds no other copy, so `issueCopy` calls `member.releaseBorrowSlot()` and throws `NoCopyAvailableException` — the caller can catch this and call `reserveBook` instead. Exactly one member gets the copy; nobody silently loses stock, and nobody is double-booked.

## How to Extend (Follow-ups)

- **Multiple branches.** Add a `Branch` id to `BookCopy` and let `issueCopy` prefer copies at the member's home branch, with an inter-branch transfer request as a separate flow.
- **Renewals.** Add `LibraryService.renewCopy(copyId)` that extends `dueDate` by the loan period, but only if no one is waiting in that title's hold queue — otherwise the waiting member is starved.
- **Reservation expiry.** A reserved copy should not sit forever. Add a background job that scans reservations in `READY_FOR_PICKUP` past their pickup deadline, moves the copy back to `AVAILABLE` (or serves the next person in queue), and cancels the stale reservation.
- **Tiered or capped fines.** Add `TieredFineStrategy` (a higher rate after 30 days) or wrap any `FineStrategy` in a `CappedFineStrategy` decorator that limits the maximum fine — no change needed to `LibraryService`, since it only depends on the `FineStrategy` interface.
- **Lost or damaged copies.** Add a `LibraryService.markLost(copyId)` that transitions a copy to `LOST` and charges a replacement cost, removing it from circulation.
- **Persistence.** Swap `InMemoryBookRepository` for a database-backed one. `BookCopy` status changes should use an optimistic-lock `version` column (`UPDATE ... WHERE version = ?`) to keep the same check-then-write safety across multiple servers, since `synchronized` only works within one JVM.
- **Role-based access.** Give `Librarian` a set of permissions (add book, waive fine) separate from what a `Member` can do through self-checkout, enforced by a small authorization check in front of `LibraryService`.

## Complexity & Thread-Safety Notes

| Operation | Time | Notes |
|---|---|---|
| `searchByTitle` / `searchByAuthor` | O(n) | n = total titles; a real system would add a search index |
| `findByIsbn` | O(1) | `ConcurrentHashMap` lookup |
| `issueCopy` | O(k) | k = copies of that title; each `tryIssue` call is O(1) |
| `returnCopy` | O(1) | `markReturned` is O(1); `queue.poll()` on `ConcurrentLinkedQueue` is O(1) |
| `reserveBook` | O(k) | k = copies of that title, to check availability, plus O(1) to enqueue |

**Thread-safety:**

- `BookCopy` bundles `status`, `issuedToMemberId`, `dueDate`, and `heldForMemberId` behind one `synchronized` block per method, because these fields must change together as one unit. Reading `status` alone and writing it separately from the borrower field would let one thread see a half-updated copy.
- Locking is per copy, not per title and not library-wide. Two members borrowing two *different* copies of the same title never block each other; only a genuine race for the *same* copy serializes.
- `Member.borrowedCount` uses a lock-free compare-and-swap (CAS) loop instead of a lock, since it is a single number with a simple rule ("increment only if under the limit"). This is cheaper than a lock when many members borrow at once and contention on any one member's counter is rare (one member rarely issues two requests at the same instant).
- The hold queue uses `ConcurrentLinkedQueue`, a lock-free FIFO structure, so `reserveBook` (enqueue) and `returnCopy` (poll) on the same title's queue never corrupt each other, and ordering (first reserved, first served) is preserved.
- `reserveBook`'s availability check and its enqueue are two separate steps, not one atomic step. In the rare case where a copy is returned in between, the member ends up in the queue for a brief moment even though a copy just became free. This is harmless: `returnCopy` always checks the queue first, so that member (or whoever is now at the front) is served on the very next return, and no request is lost.

## Interview Tips & Common Mistakes

- **State the Book vs. BookCopy split early, unprompted.** It is the single modeling decision interviewers look for in this problem. Say out loud: "status, due date, and borrower belong to the copy, not the title."
- **Never model status as a single boolean.** `isAvailable` cannot represent "reserved for one specific member" or "lost." Use an enum (`CopyStatus`) from the start.
- **Do not check-then-act without a lock.** `if (copy.status == AVAILABLE) { copy.status = ISSUED; }` written as two separate statements is a race: two threads can both pass the check before either writes. `synchronized` (or a CAS loop) must cover the check and the write as one step, exactly as `tryIssue` does here.
- **Match lock granularity to the data.** A single lock over the whole library would make every borrow and return wait on every other one, even for unrelated titles. Lock at the level of the thing that actually needs protecting — one `BookCopy`.
- **Always give the borrow slot back on failure.** If `issueCopy` claims a slot and then finds no copy free, it must call `releaseBorrowSlot()` before throwing. Forgetting this slowly locks members out even though they hold fewer books than their limit.
- **Keep the fine rule outside the core flow.** Hard-coding `daysLate * rate` inside `returnCopy` works today, but the moment the rule changes (a grace period, a cap, a different rate per member tier), that logic needs to move without touching `LibraryService`. That is exactly what `FineStrategy` is for.
- **Reservations must be FIFO, not "whoever calls issueCopy first after a return wins."** Without a queue, an unrelated member could grab a copy meant for someone who has been waiting for days. The `RESERVED` status plus `heldForMemberId` is what enforces fairness.
