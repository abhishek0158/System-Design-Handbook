# Java Coding & LLD Problems — Solutions (3–4 Years)

Worked Java solutions for the most common **machine-coding, low-level design (LLD), and
concurrency** interview problems. Made for a backend Java developer with **3–4 years** of
experience. Language is kept **simple and easy to read**.

Each solution has the same format:
1. **Problem** — what is asked and why.
2. **Requirements & Clarifying Questions** — what to ask the interviewer.
3. **Design / Approach** — classes, patterns, or concurrency strategy (with a small sketch).
4. **Java Solution** — clean, correct code.
5. **How It Works** — plain-English walkthrough.
6. **How to Extend (Follow-ups)** — the next questions interviewers ask.
7. **Complexity & Thread-Safety Notes.**
8. **Interview Tips & Common Mistakes.**

> **Note:** These are the **LLD / code** versions of the problems. Some names (URL Shortener,
> Notification System, Rate Limiter, Task Scheduler, Parking Lot) also appear in the System
> Design handbooks — there they are **HLD** (distributed architecture). Both skills matter.

## Two tracks

### Track A — Concurrency & Java coding (`concurrency/`)
| Tier | Problems |
|------|----------|
| **1 (now)** | LRU Cache · Producer–Consumer · Thread-safe Counter · Blocking Queue · Print in Order (Odd/Even + N threads) · Thread-safe Singleton · Rate Limiter · In-Memory Cache (TTL + eviction) · Core Java warm-ups (equals/hashCode, Immutable, Comparator) |
| 2 | Custom Thread Pool · CompletableFuture parallel tasks · Deadlock (create/detect/fix) · ReadWriteLock cache · Custom HashMap · Dining Philosophers · Connection/Object Pool · Semaphore resource pool |
| 3 | LFU Cache · Job Queue with workers · Circuit Breaker · Retry with backoff · Scheduled Task Executor (delay queue) |

### Track B — LLD / machine-coding design (`lld/`)
| Tier | Problems |
|------|----------|
| **1 (now)** | Parking Lot · Notification System · Task Scheduler · Logging Framework · Splitwise (Expense Splitter) · In-Memory Key-Value Store |
| 2 | URL Shortener (LLD) · Elevator System · Movie Ticket Booking · Inventory Management · Pub/Sub System · Vending Machine · Tic-Tac-Toe / Snake & Ladder |
| 3 | File System (in-memory) · ATM · Coffee Machine · Meeting Room Booking · Library Management · Cab Booking · Digital Wallet |

## Status
**COMPLETE — all 41 solutions written** (21 in `concurrency/`, 20 in `lld/`), across all three
tiers. Each solution follows the full format: Problem → Requirements/Clarifying Questions →
Design (with class sketch + patterns) → Java Solution → How It Works → How to Extend →
Complexity & Thread-Safety → Interview Tips & Common Mistakes.
