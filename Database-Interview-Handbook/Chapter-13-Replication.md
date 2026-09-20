# Chapter 13 — Replication

**Replication** means keeping copies of the same data on more than one server. Interviewers
ask about it because almost every real system needs it: to survive server crashes, and to
handle more read traffic than one server can serve.

## Key Concepts

### 1. Why we replicate data

One database server is a **single point of failure**. If it goes down, the whole app goes
down. It also has a limit on how many reads it can serve per second.

Replication solves three problems:

- **High availability** — if one server dies, another copy can take over. The app stays up.
- **Read scaling** — reads can be spread across many copies, so total read throughput grows.
- **Backups / geo-placement** — a copy can sit in another region, closer to some users, or be
  used to run backups without slowing down the main server.

Replication does **not** help with **write scaling**. Every copy still needs to receive every
write. Writing to more copies makes writes slower, not faster. Write scaling needs a different
tool: **partitioning / sharding** (Chapter 14).

### 2. Leader–follower replication (a.k.a. primary–replica)

This is the most common setup. One server is the **leader** (also called the **primary** or
**master**). All the other copies are **followers** (also called **replicas** or **slaves**,
though "leader/follower" is now the preferred term).

Rule: **all writes go to the leader**. The leader applies the write, then sends it to every
follower, usually as a stream of changes taken from its **write-ahead log (WAL)** — the same
log used for crash recovery (see Chapter 12). Each follower replays this log to stay in sync.

**Reads can go to the leader or to any follower.** This is where the read scaling comes from:
put a load balancer in front of the followers and spread read traffic across them.

```
                 writes
   client  ───────────────────►  LEADER  ────┐
                                              │  replication stream (WAL)
                 reads (any)                 ▼
   client  ◄───────────  FOLLOWER 1    FOLLOWER 2    FOLLOWER 3
```

This is sometimes called **CQRS-ish read/write splitting**, though CQRS is really an
application pattern — here it happens at the database level with almost no app changes:
your app's write path uses one connection string (the leader), your read path uses another
(the follower pool).

**How followers actually stay in sync.** The leader does not resend whole tables. It streams
the same log it already writes for crash recovery — the write-ahead log (WAL) in PostgreSQL,
the binlog in MySQL. Each entry describes a change ("row X in table Y changed to this value").
A follower reads this stream and replays each entry in the same order the leader applied it.
This is why replication is naturally ordered: a follower cannot apply entry 51 before entry 50.

**Two ways followers can be added:** PostgreSQL supports **physical replication** (byte-level
copy of data pages — fast, but the follower must be the same Postgres version and cannot have
extra indexes) and **logical replication** (row-level changes, decoded to SQL-like operations —
more flexible, can replicate to a different schema or version, slightly slower). MySQL has a
similar split: statement-based, row-based, and mixed binlog formats.

### 3. Synchronous vs. asynchronous replication

This is the core trade-off in this chapter. It decides **when the leader tells the client
"write done."**

**Synchronous replication**: the leader waits for at least one follower to confirm it received
and applied the write, before telling the client "success."

- Good: if the leader dies right after, the follower already has the write. **No data loss.**
- Bad: every write is now as slow as the slowest synchronous follower, plus network time. Also,
  if that follower is down, writes can stall or fail (a new availability risk).

**Asynchronous replication**: the leader tells the client "success" immediately, and sends the
write to followers in the background, whenever it gets to it.

- Good: writes are fast. The leader does not wait on anyone.
- Bad: if the leader dies before the write reaches a follower, and that follower is promoted
  to the new leader, **that last write is lost.** The client was told "success," but the data
  is gone.

```
Sync:   client → leader → wait for follower ack → "OK" to client   (safe, slower)
Async:  client → leader → "OK" to client → (later) send to follower (fast, riskier)
```

**Semi-synchronous** is a middle ground: the leader waits for **one** follower to confirm
(not all of them), and replicates to the rest asynchronously. This limits the slow-down to one
extra hop, while still guaranteeing the write exists in at least two places before it is
confirmed.

**How to answer "sync or async?" in an interview:** say "it depends" and give the reason.

- Financial transactions, order placement, anything where losing a write is unacceptable →
  lean synchronous (or semi-sync).
- Social media likes, view counters, analytics events → async is fine. Losing a few is okay,
  speed matters more.

PostgreSQL supports both. Set `synchronous_standby_names` and
`synchronous_commit = on` to make a follower synchronous. By default, PostgreSQL replication
is asynchronous.

### 4. Replication lag and the read-your-own-writes problem

**Replication lag** is the delay between a write landing on the leader and that same write
showing up on a follower. With async replication, this lag is normal — it can be milliseconds,
or seconds under heavy load.

This causes a very common bug, called **read-your-own-writes** (or "read-after-write")
**inconsistency**:

1. A user updates their profile picture. The write goes to the leader.
2. The app immediately reloads the page. The read goes to a follower (for read scaling).
3. The follower has not caught up yet. The user sees their **old** picture.

The user knows they just saved a change. Seeing the old data looks like a bug, even though the
system is working as designed.

**How to fix it (say more than one option in an interview):**

- **Route that user's read to the leader** for a short time after they write (e.g., for 5–10
  seconds, or for any read of a record the user just wrote).
- **Read from a follower, but only one you know is caught up.** Track the WAL position (an LSN,
  log sequence number) the write produced, and only route to a follower whose applied LSN is
  at least that far. Postgres exposes this as `pg_last_wal_replay_lsn()`.
- **Sticky sessions**: pin a user to the same follower for their whole session, combined with a
  short "wait for catch-up" step right after a write.
- **Client-side wait**: the client remembers the write's timestamp/version and refuses to trust
  a read that is older than that version ("read-your-writes" via version tokens).
- **Just use the leader for reads that must be fresh**, and followers only for reads that can
  tolerate being stale (e.g., a public feed, a search index, a dashboard).

There is a related problem: **monotonic reads**. Without care, a user could read from a
fast-catching-up follower, see new data, then on the next request land on a slower follower and
see *older* data than before — data appears to go backward in time. Fix: stick each user to one
follower, so their own view of time never goes backward.

### 5. Failover: what happens when the leader dies

**Failover** means promoting a follower to become the new leader after the old leader fails.
Steps, at a high level:

1. **Detect** the leader is really down (not just slow). This usually needs a timeout plus
   agreement from more than one observer, to avoid false alarms.
2. **Choose** which follower becomes the new leader — usually the one with the most up-to-date
   data (least lag).
3. **Promote** that follower: it stops following and starts accepting writes.
4. **Reconfigure**: other followers now replicate from the new leader. Clients and the app's
   connection routing must be pointed at the new leader (via DNS change, a proxy, or a
   coordination service like a `etcd`/ZooKeeper-based tool, or a managed cloud failover).

**Two failure modes to mention:**

- **Lost writes**: with async replication, any write that had not yet reached the promoted
  follower is gone. The old leader, if it comes back, may still have that write locally — this
  data now conflicts with the new leader and is usually just discarded.
- **Split-brain**: the old leader comes back online (network partition healed) and does **not**
  know it was replaced. Now two servers both think they are the leader, and both accept
  writes. This causes silent data conflicts and is one of the more dangerous failure modes in
  distributed systems.
    - Fix: **fencing** — when a new leader is promoted, the old one must be forcibly stopped or
      blocked from accepting writes (e.g., cut its access, use a "fencing token" that storage
      checks, or have a majority-vote system that only lets the true leader hold a lease).

Failover can be **manual** (an on-call engineer runs a promote command) or **automatic** (a
tool like Patroni, repmgr, or a managed service like RDS/Aurora does it). Automatic failover is
faster but riskier — a flaky network can trigger an unwanted failover.

**Failover time matters.** The gap between "leader dies" and "new leader accepts writes" is
called **failover time** or **downtime window**. It has a few parts: time to detect the failure
(usually a few seconds of missed heartbeats, to avoid reacting to one slow response), time to
pick and promote the best follower, and time to redirect clients (DNS changes can take minutes
to propagate; a proxy or a virtual IP is faster). Managed cloud databases often quote failover
times of 30–120 seconds. A well-tuned self-hosted setup with a proxy like PgBouncer or HAProxy
in front, and a coordination tool like Patroni, can fail over in single-digit seconds.

**Why not always pick the follower with the least lag automatically, without a check?** Because
"least lag" is only known at the moment of failure, and that measurement itself may be stale if
the network is partitioned. Some followers may not be reachable from the coordinator at all
during the outage, so "best follower we can currently reach" is really what gets promoted, not
necessarily the true best one.

### 6. Multi-leader and leaderless replication (short note)

Leader–follower is the default answer, but interviewers may ask "what else exists?"

**Multi-leader replication**: more than one node accepts writes (e.g., one leader per data
center). Each leader replicates to the others. Good for multi-region write availability, but
now two leaders can accept **conflicting writes** to the same row at the same time, and the
system must resolve conflicts (last-write-wins, merge functions, or CRDTs — conflict-free
replicated data types). More complex, used when write latency across regions matters more than
simplicity.

**Leaderless replication (Dynamo-style)**: there is no leader at all. A client write goes to
several nodes directly (say, 3 nodes), and a write is called successful once enough of them
(a **quorum**, e.g., 2 of 3) confirm it. Reads also go to several nodes and the freshest value
wins. Cassandra, DynamoDB, and Riak use this style. It trades strict consistency for very high
write availability — there is no single leader to become a bottleneck or a single point of
failure. Chapter 15 covers quorums (`W + R > N`) in detail.

| Style | Who can write | Main strength | Main risk |
|---|---|---|---|
| Leader–follower | Only the leader | Simple, strong consistency on the leader | Leader is a bottleneck / failover needed |
| Multi-leader | More than one node | Fast local writes in each region | Write conflicts to resolve |
| Leaderless | Any node, via quorum | No single point of failure, high write availability | Weaker consistency guarantees |

## The Questions They Ask

**Q1: The app has too many reads for one database server. How do you scale reads?**
Add read replicas (followers) using leader–follower replication. Writes still go to the
leader. Reads get spread across the followers, usually behind a load balancer or a
read-connection pool. Mention the trade-off: replicas lag behind the leader, so this only works
for reads that can tolerate being slightly stale. If the follow-up is "what if reads still
don't scale," mention caching (a cache in front of the DB) and, if writes are also the
bottleneck, sharding (Chapter 14).

**Q2: Explain synchronous vs. asynchronous replication. Which would you use for a payments
table?**
Sync replication waits for a follower to confirm before telling the client the write
succeeded — safe, slower, and can even block writes if the follower is unreachable. Async
replication confirms immediately and replicates in the background — fast, but can lose the
most recent writes if the leader crashes before they replicate. For a payments table, prefer
synchronous or semi-synchronous replication: losing a confirmed payment is unacceptable, and it
is worth the extra latency.

**Q3: What is replication lag? How do you handle a user seeing stale data right after they
write?**
Replication lag is the delay before a follower has the same data as the leader. The classic
symptom is the read-your-own-writes problem: a user writes, then immediately reads from a
lagging follower and sees old data. Fixes: route that user's next read to the leader for a
short window, check the follower's replication position before trusting it for that read, or
stick the user to one follower and accept slightly stale (but never "going backward")
reads (monotonic reads).

**Q4: What happens if the leader dies? Walk me through failover.**
A monitor (or the DBA) detects the leader is down. A follower with the least lag is chosen and
promoted to be the new leader. Other followers are pointed at the new leader. The app's
connection routing is updated to send writes to the new leader. Call out two risks: data loss
on the writes that had not replicated yet (if replication was async), and split-brain if the
old leader comes back and does not know it was replaced — fixed by fencing the old leader off.

**Q5 (follow-up): Can you avoid data loss on failover completely?**
Only if you replicate synchronously to at least the node that gets promoted. Many systems use
"semi-sync": wait for one follower to confirm before acknowledging the write, so failover to
that follower never loses the confirmed write. Fully async replication cannot give this
guarantee — it always has a small at-risk window.

**Q6: What is multi-leader or leaderless replication, and when would you use it?**
Multi-leader lets more than one node accept writes (e.g., one per region), trading write
availability for conflict resolution complexity. Leaderless (Dynamo-style) has no leader —
writes and reads use quorums across many nodes. Use these when write availability across
regions matters more than strict consistency, e.g., a shopping cart service that must accept
writes even during a network partition. For most CRUD apps with one primary region, plain
leader–follower is simpler and is the right default.

**Q7: Your app writes to the leader and reads from a replica. Sometimes a `SELECT` right after
an `INSERT` returns zero rows for the row you just inserted. Why, and how do you fix it?**
This is read-your-own-writes lag: the `INSERT` committed on the leader, but the replica used
for the `SELECT` has not replayed it yet. Fix options: read that specific row from the leader
right after writing it (simplest), pass along the WAL position from the write and have the read
path wait until the chosen replica reaches at least that position, or, if the read is inside
the same request/transaction as the write, just keep using the leader connection for that
request instead of switching to the replica pool.

**Q8: How would you test that your replication setup actually protects you, before it is
needed in production?**
Say this out loud, it is a strong interview answer: simulate the failure you are protecting
against. Kill the leader process (or block its network) in a staging environment and time how
long the app is unavailable. Check whether any writes that were reported as successful are
missing on the new leader (this tells you if async replication is losing data in practice).
Confirm the old leader, once it rejoins, does not come back as a second writer (tests your
fencing). Doing this once, before an incident, is far better than discovering the failover
script has a bug during a real outage.

## Rapid-Fire

- **What is replication?** — Keeping copies of the same data on more than one server.
- **Leader vs. follower?** — Leader takes all writes; followers replicate from the leader and
  can serve reads.
- **Does replication help write throughput?** — No. Every copy still needs every write. Use
  sharding for write scaling.
- **Sync replication trade-off?** — No data loss on failover, but slower writes and a new
  availability risk if the sync follower is down.
- **Async replication trade-off?** — Fast writes, but can lose the last few writes on a leader
  crash.
- **Semi-sync?** — Wait for one follower to confirm, replicate to the rest async. A middle
  ground.
- **Replication lag?** — The delay before a follower catches up to the leader's latest write.
- **Read-your-own-writes problem?** — User writes, then reads a stale follower and sees old
  data.
- **Fix for read-your-own-writes?** — Read from the leader right after a write, or check the
  follower is caught up first.
- **Monotonic reads?** — A guarantee that a user's reads never go backward in time; fixed by
  sticking a user to one follower.
- **Failover?** — Promoting a follower to leader after the old leader fails.
- **Split-brain?** — Two nodes both think they are the leader and both accept writes; fixed by
  fencing the old leader.
- **Multi-leader replication?** — More than one node accepts writes; needs conflict resolution.
- **Leaderless replication?** — No leader; writes/reads use quorums across nodes (e.g.,
  Cassandra, DynamoDB).

## Common Traps & Mistakes

- **Assuming replication scales writes.** It does not. Every follower must apply every write,
  so more followers can only slow writes down (more replicas to send to), never speed them up.
- **Treating async replication as "no data loss, just a little delay."** It is real data loss
  risk, not just delay — if the leader dies before a write replicates, that write is gone for
  good, even though the client was told "success."
- **Forgetting the read-your-own-writes problem exists** when adding read replicas for
  scaling. This is one of the most common bugs reported as "stale UI" or "changes don't save"
  by confused users, and it is almost always a replica-read issue, not a real save failure.
- **Assuming automatic failover is always safe.** A flaky network can cause a false failover,
  promoting a follower while the old leader is still alive and reachable by some clients —
  this is exactly how split-brain happens. Fencing the old leader is not optional.
- **Confusing "replica" with "backup."** A replica has close-to-live data and mirrors mistakes
  too — if someone runs a bad `DELETE` on the leader, it replicates to every follower almost
  immediately. Replicas are for availability and read scaling, not for undoing mistakes. You
  still need real backups / point-in-time recovery.
- **Not stating an assumption before answering "sync or async."** Interviewers want to hear
  "it depends on whether losing the last write is acceptable for this data," not a flat answer.
- **Mixing up leader-based and leaderless quorum systems.** In leader–follower, only the leader
  accepts writes. In leaderless (Dynamo-style) systems, any node can accept a write, and a
  quorum decides success. These are different models with different failure behavior — do not
  describe leaderless replication as "leader–follower with more leaders."
