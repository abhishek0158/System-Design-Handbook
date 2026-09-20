# Chapter 15 — CAP, Consistency & Quorum

CAP theorem is the most quoted, and most misunderstood, idea in distributed databases. Interviewers ask about it to check if you actually understand the trade-off, or if you just memorized the word "CAP." This chapter also covers PACELC, consistency models, and quorum reads/writes — the practical tools that come from CAP thinking.

## Key Concepts

### What is a distributed database?

Chapter 13 and Chapter 14 covered this, but here is the short version. A **distributed database** stores data on more than one machine (node). It does this for two reasons: to hold more data than one machine can (sharding), and to survive a machine failing (replication). Once data lives on more than one node, the nodes must talk to each other over a network. That network can fail. CAP is about what happens when it does.

### The CAP theorem, stated correctly

CAP theorem says: when a **network partition** happens, a distributed database must choose between **Consistency** and **Availability**. You cannot have both at that moment.

First, define the three letters:

- **C — Consistency.** Every read gets the most recent write, or an error. All nodes show the same data at the same time. (Note: this is a different meaning from the "C" in ACID, which is about constraints. Say this out loud in an interview — it shows you know the overlap in terms.)
- **A — Availability.** Every request gets a response (not an error), even if it is not the latest data. The system never refuses to answer.
- **P — Partition tolerance.** The system keeps working even when the network between nodes drops messages or splits nodes into groups that cannot talk to each other.

A **network partition** is a break in communication between nodes. Node A and Node B are both up and running, but the network link between them fails. Each node is alive, but they cannot talk to each other.

Now the actual theorem: **in a network with more than one node, partitions will happen sooner or later.** This is a fact of real networks (cables get cut, switches fail, packets get dropped). So partition tolerance (P) is not really optional — you must handle it. The real choice CAP forces on you is between C and A, **only during a partition**:

- Choose **C (Consistency)**: when a partition happens, the database refuses some requests (returns an error or times out) rather than risk returning stale or conflicting data. It gives up availability to protect correctness.
- Choose **A (Availability)**: when a partition happens, the database keeps answering every request, even on nodes that are cut off from each other. Different nodes may now disagree about the latest data.

**Important nuance for interviews:** CAP is not "pick 2 of 3 all the time." It only applies **during a partition**. When there is no partition, a well-designed system can give you both C and A. The trade-off is a partition-time trade-off, not an always-on trade-off.

### Why "CA" is not a real choice

You will see systems described as "CA" (Consistent and Available, no partition tolerance). This label is misleading for any system that runs on more than one node.

Here is why. If a system claims to be "CA," it means: during a partition, it stays both fully consistent and fully available. But that is impossible. If nodes cannot talk to each other, and a client writes to Node A, then Node B either:
- refuses to serve reads/writes until it can confirm the latest state (giving up availability), or
- serves reads/writes using its own local, possibly stale, data (giving up consistency).

There is no third option. So "CA" only makes sense for a **single-node** database (like one PostgreSQL instance with no replicas) — it never faces a network partition between nodes, because there is only one node. The moment you add a second node for replication, you are in CP-vs-AP territory, whether you like it or not. So in interviews, if someone says a distributed system is "CA," push back — say partition tolerance is not optional once you have more than one node.

### CP and AP: the real choices

| Choice | Behavior during a partition | Example systems |
|---|---|---|
| **CP** (Consistent + Partition-tolerant) | Give up availability. Some nodes return errors or time out rather than serve stale data. | ZooKeeper, etcd, HBase, MongoDB (default config), traditional RDBMS with synchronous replication |
| **AP** (Available + Partition-tolerant) | Give up strict consistency. Every node keeps answering, using whatever data it has. Data may be briefly stale or conflicting. | Cassandra, DynamoDB, Riak, CouchDB |

A useful way to reason in an interview: ask **"what happens to a write, or a read, on the minority side of a partition?"**
- If the answer is "it fails / is rejected," the system leans **CP**.
- If the answer is "it still succeeds, using local data," the system leans **AP**.

Also note: many real systems are **tunable**. Cassandra can be configured to behave more like CP or more like AP, depending on settings (see the quorum section below). So "is Cassandra CP or AP" has an "it depends on configuration" answer — say this out loud, it is the right answer.

### PACELC — the fuller picture

CAP only describes behavior **during a partition**. But partitions are rare. Most of the time, the network is fine. So what trade-off exists when there is **no partition**? CAP does not say. **PACELC** extends CAP to answer this.

PACELC reads as: **if Partition, choose between Availability and Consistency; Else (no partition), choose between Latency and Consistency.**

- **P / A vs C** — this is plain CAP: during a Partition, choose A or C.
- **E / L vs C** — Else (normal operation, no partition), choose between Latency and Consistency.

Why does a latency-vs-consistency trade-off exist even without a partition? Because making data strongly consistent across nodes takes coordination — nodes must confirm with each other before answering. That coordination adds waiting time (latency). If you want a fast (low-latency) response, you may need to skip that coordination and accept a system that could serve slightly stale data.

Examples:
- **PC/EC** — always favors consistency, both during a partition and normally. Example: a system using synchronous replication to every replica before confirming a write (Chapter 13). Slower, but always consistent.
- **PA/EL** — favors availability during a partition, and favors low latency normally. Example: DynamoDB and Cassandra in their default tunable settings. Fast, but can serve stale reads.
- **PC/EL** — consistent during a partition (refuses requests), but optimizes for low latency normally (e.g., reads a nearby replica that might lag slightly under normal conditions, depending on configuration).

**Interview tip:** if you can say "CAP is not the whole story, PACELC also matters because trade-offs exist even without a partition," you are already ahead of most candidates.

### Consistency models, in plain words

"Consistency" is not just one thing — there are different strengths. From strongest (safest, slowest) to weakest (fastest, riskiest):

**1. Strong consistency.** After a write finishes, every later read (from any node) sees that write. It feels like there is only one copy of the data, even though there are many. This needs coordination between nodes on every write or read, which adds latency. Example: reading your bank balance right after a transfer — you expect the new balance immediately, from any ATM.

**2. Eventual consistency.** After a write, the system does not update all replicas immediately. If you stop writing, all replicas will **eventually** converge to the same value — but there is no guarantee of *when*. A read right after a write might return old data. Example: a "like" count on a social media post. It is fine if it takes a second or two to update everywhere.

**3. Read-your-writes consistency.** A weaker promise than strong consistency, but stronger than plain eventual consistency. It guarantees: **the user who made a write will always see their own write** in their later reads, even if other users might briefly see stale data. Example: you update your profile picture, and you see the new picture right away — but a friend viewing your profile from a different replica might see the old picture for a moment.

**4. Causal consistency.** If write B happens **because of** (is caused by, or depends on) write A, then every node must show A before B. Writes that are not causally related can appear in any order on different nodes. Example: you post a comment ("Great trip!") replying to a photo. Causal consistency guarantees no one sees your comment before they see the photo it replies to. But two unrelated comments from two different users can appear in a different order on different nodes — and that is fine.

A simple way to rank them for an interview: **strong > causal > read-your-writes > eventual**, in terms of how strict the guarantee is. Stronger guarantees generally cost more latency and reduce availability during partitions; weaker guarantees are cheaper and more available.

### Quorum reads and writes (Dynamo-style tunable consistency)

Many AP-leaning distributed databases (Cassandra, DynamoDB, Riak) do not pick one fixed consistency level. Instead, they let you tune it per request, using a **quorum** system. This idea comes from Amazon's Dynamo paper.

Define three numbers:

- **N** — the number of replicas that store a copy of the data.
- **W** — the number of replicas that must confirm a write before the write is considered successful.
- **R** — the number of replicas that must respond to a read before the read returns a result.

The key rule: **if W + R > N, every read is guaranteed to see the latest write.**

Why does this work? Because if a write went to `W` replicas, and a read checks `R` replicas, and `W + R > N`, then the read set and the write set must **overlap by at least one replica** (there just are not enough spare replicas to avoid it). That overlapping replica holds the latest write, so the read is guaranteed to see it (the read compares versions/timestamps across the replicas it contacted, and returns the newest one).

**Worked example.** Say `N = 3` (data is copied to 3 replicas: R1, R2, R3).

- Set `W = 2, R = 2`. Check: `W + R = 4 > N = 3`. ✅ Strong consistency (also called a "quorum" read/write). A write must reach 2 of the 3 replicas before it succeeds. A read must check 2 of the 3 replicas. Since both the write set and read set contain 2 out of 3 replicas, they must share at least one replica — so the read always sees the latest write.
- Set `W = 1, R = 1`. Check: `W + R = 2`, not `> N = 3`. ❌ Not guaranteed. A write to R1 only, and a read from R3 only, never overlap — the read can return stale data. This favors speed and availability over consistency (leans AP).
- Set `W = 3, R = 1` (write to all replicas, read from just one). Check: `W + R = 4 > 3`. ✅ Strong consistency, but writes are slower (must wait for all 3 replicas) while reads are fast.
- Set `W = 1, R = 3` (write to one replica, read from all). Check: `W + R = 4 > 3`. ✅ Also strongly consistent, but now writes are fast and reads are slower.

This is why it is called **tunable consistency** — you pick `W` and `R` per operation, trading off latency against consistency, without changing the database software. A write-heavy workload might use low `W` and high `R`. A read-heavy workload might do the opposite.

**Extra note on partitions and quorum.** If a partition splits `N = 3` replicas into a group of 2 and a group of 1, a quorum write (`W = 2`) can still succeed on the side with 2 replicas. The side with only 1 replica cannot get a quorum, so writes there fail or are rejected. This is exactly the CP-style trade-off in action, applied per-request through quorum settings.

## The Questions They Ask

**Q1: Explain CAP theorem in simple terms.**
A: In a distributed database, when the network between nodes breaks (a partition), you must choose between consistency (always return the latest, correct data) and availability (always answer, even with possibly stale data). You cannot fully have both at that moment. Partitions are rare but they will happen, so every real distributed system quietly makes this choice, whether the team realizes it or not.
*Follow-up: "does this mean the system is inconsistent all the time?"* — No. This trade-off only applies during a partition. Most of the time there is no partition, and the system can be both consistent and available. PACELC explains the trade-off for normal (non-partition) times.

**Q2: Is [database X] CP or AP?**
A: Answer using the "what happens on the minority side of a partition" test.
- PostgreSQL with synchronous replication: **CP**. If the replica cannot confirm, the primary can block or reject the write rather than risk data loss.
- MongoDB (default): mostly **CP** — a write needs acknowledgement from a majority of replica set members by default; if the primary is isolated from the majority, it steps down and stops accepting writes.
- Cassandra, DynamoDB: **AP** by default, but tunable toward CP using quorum settings (`W + R > N`).
- ZooKeeper, etcd: **CP** — they are built specifically to give a single, agreed answer (useful for leader election, config storage), and will refuse to answer rather than risk two different answers.
  *Follow-up: "can a database be both, depending on settings?"* — Yes, many modern databases let you tune this per request or per table (see quorum section). Say "it depends on configuration" — this is the correct senior-level answer.

**Q3: What is eventual consistency, and when is it OK to use it?**
A: Eventual consistency means replicas may briefly disagree after a write, but they converge to the same value if no new writes happen. It is OK when:
- Being slightly stale for a short time causes no real harm (like counts, view counts, product recommendations, activity feeds).
- Availability and low latency matter more than perfect freshness (a global app should not fail for a user just because one data center is unreachable).
  It is **not OK** for cases needing correctness right now: bank balances, inventory counts used for "in stock" decisions during checkout, or anything where a stale read causes a real-world mistake (like selling the same seat twice).

**Q4: Explain quorum, and what does R + W > N mean?**
A: `N` is how many replicas hold a copy of the data. `W` is how many replicas must confirm a write to call it successful. `R` is how many replicas a read must check. If `W + R > N`, the set of replicas touched by any write and the set touched by any later read must overlap by at least one node — so a read is guaranteed to see the latest write. This lets you tune the consistency-vs-latency trade-off per operation: high `W` and `R` means strong consistency but slower; low `W` and `R` means faster but reads can be stale.
*Follow-up: "what if two writes happen to different replicas at the same time?"* — This is a write-write conflict. The system needs a way to resolve it: last-write-wins (using timestamps), vector clocks (track causality, sometimes require the application to merge conflicting versions), or CRDTs (data structures designed to merge automatically without conflict). This is a good chance to mention Chapter 13's discussion of replication conflicts.

**Q5: Why can't a distributed system just be "CA"?**
A: "CA" would mean: during a network partition, the system is still both fully consistent and fully available. That is impossible once there is more than one node — if nodes cannot communicate, you must either block requests (lose availability) or answer with possibly-stale local data (lose consistency). "CA" only makes sense for a single-node system, which never has an inter-node partition in the first place. This shows you understand CAP is about a forced choice, not a menu of three equal options.

## Rapid-Fire

- **CAP theorem** → during a network partition, choose Consistency or Availability; cannot have both.
- **Partition** → a break in network communication between nodes that are both still running.
- **Why not "CA"?** → a multi-node system must give up one of C or A during a partition; "CA" only fits a single node.
- **PACELC** → if Partition, choose Availability or Consistency; Else, choose Latency or Consistency.
- **Strong consistency** → every read sees the latest write, from any node, immediately.
- **Eventual consistency** → replicas converge to the same value over time, but a read right after a write can be stale.
- **Read-your-writes** → you always see your own latest write; others might briefly see stale data.
- **Causal consistency** → related (cause-and-effect) writes are seen in order everywhere; unrelated writes can appear in any order.
- **N, W, R** → N = replica count, W = replicas needed to confirm a write, R = replicas checked on a read.
- **Quorum rule** → `W + R > N` guarantees a read always sees the latest write.
- **CP example** → ZooKeeper, etcd, synchronous-replication RDBMS.
- **AP example** → Cassandra, DynamoDB (default settings).
- **Is CAP always active?** → No, the C-vs-A trade-off only applies during a partition, not all the time.

## Common Traps & Mistakes

- **Saying "pick any 2 of C, A, P" like a menu.** This is the most common wrong summary. P is not optional in a real multi-node system — partitions happen whether you "pick" them or not. The real choice is C vs A, and only during a partition.
- **Calling a distributed system "CA."** As explained above, this is not a valid category once you have more than one node. If an interviewer hears "CA" used this way, it signals a shallow understanding.
- **Forgetting PACELC and only talking about partitions.** Partitions are rare. The latency-vs-consistency trade-off during **normal** operation (no partition) matters more day to day, and PACELC is the part of the theory that covers it.
- **Confusing CAP's "C" with ACID's "C."** ACID consistency means the database follows its defined rules and constraints (Chapter 9). CAP consistency means all nodes agree on the latest value. They are different ideas that happen to share a letter. Say this explicitly if asked — interviewers like when candidates flag this overlap.
- **Assuming eventual consistency means "wrong" or "buggy."** It is a deliberate design choice for availability and speed, used correctly in many large systems (Cassandra, DNS, DynamoDB). It is a trade-off, not a defect.
- **Getting the quorum formula backwards.** The rule is `W + R > N`, not `W + R >= N` or `W + R < N`. Also remember: if `W + R > N` is violated, the system can still work — it is simply not guaranteed to return the latest write on every read. It may return stale data sometimes, not always.
- **Thinking quorum with W + R > N gives you ACID-level strong consistency in every sense.** It guarantees you read the latest **acknowledged** write. It does not, by itself, give you transactions across multiple keys, or protect against all conflict scenarios (like two concurrent writes to the same key on different replicas before either is fully propagated) — those need extra mechanisms like vector clocks or last-write-wins rules.
- **Treating N, W, R as fixed per database.** In tunable systems (Cassandra, DynamoDB), these can often be set per query or per table, not just once for the whole database. Say "it depends on the settings chosen for this operation," not "this database is CP" or "this database is AP" as an absolute fact.
