# Chapter 10 — Failure Handling

> "Everything fails, all the time." At GlobalMart's scale — ~10 B listings, ~100 K peak search
> QPS, 60–100 data nodes, three regions — the question is never *whether* a node, network link,
> or dependency will fail, but *what the buyer sees when it does*. This chapter is about making
> the answer to that question boring.

The single most important framing for this chapter comes straight from the requirements
(§1): **the search/read path is revenue-critical and carries a 99.99% availability target.**
A hard failure on the search path is lost revenue and lost trust. So the guiding principle
running through everything below is:

> **On the read path, degrade gracefully — never hard-fail.** A slightly stale, slightly
> less-relevant, or facet-stripped result page is a *good day* compared to an error page.
> The index is a *derived, rebuildable* artifact (§4), so we can always trade freshness and
> richness for availability.

The indexing (write) path has the opposite bias: it is allowed to slow down, buffer, and
catch up later, because eventual consistency is acceptable for the index (§1, NFR6). We never
sacrifice *correctness* of the index to keep it fast, but we happily sacrifice *latency*.

Keep those two biases in mind — read path favors availability (AP), write path favors
correctness-eventually — and most of the design decisions here fall out naturally.

---

## 10.1 A Failure Taxonomy for GlobalMart Search

Before mitigating anything, enumerate what can break. A good interview answer starts by
*classifying* failures rather than listing random incidents. Here is the taxonomy we use,
mapped onto the canonical architecture (§5).

| # | Failure class | Concrete example in GlobalMart | Primary blast radius | Read-path bias |
|---|---|---|---|---|
| 1 | **Node / hardware** | An ES `data-hot` node loses a disk or is OOM-killed | Shards on that node | Degrade: replicas serve |
| 2 | **Network partition** | A region or AZ link drops; masters can't see data nodes | Cluster coordination | Degrade: quorum protects |
| 3 | **Dependency failure** | Redis (Result Cache), Catalog Service, or Kafka unavailable | One component | Degrade / fallback |
| 4 | **Data corruption** | A corrupt Lucene segment, bad translog, silent bit-rot | One or more shards | Restore from snapshot |
| 5 | **Overload** | Peak QPS > provisioned; a "query of death"; a viral SKU | Whole tier | Shed load, bulkhead |
| 6 | **Poison data** | A malformed CDC event the Indexing Service cannot map | One Kafka partition / consumer | DLQ + skip |

Two cross-cutting observations:

- **Failures cascade.** An overloaded node (class 5) times out, its shards get marked
  failed (looks like class 1), replicas take over and *they* now carry more load (class 5
  again). Most real incidents are a chain, not a single event. The mitigations below are
  designed to *break* cascades — bulkheads, circuit breakers, backpressure.
- **The read and write paths fail independently and are isolated on purpose.** If Kafka or
  the Indexing Service is down (class 3/6), search keeps serving; the index just goes stale.
  If ES data nodes are struggling (class 1/5), we throttle indexing to give queries headroom.
  This isolation is a *design goal*, not an accident.

```
        ┌─────────────── READ PATH (99.99%, AP-biased) ───────────────┐
Client→ CDN/Edge → API Gateway → Search Service → ES coord → ES data
                                     │  Query Understanding
                                     │  Result Cache (Redis)   ← degrade here first
                                     │  Ranking / LTR reranker ← degrade here second
                                     └ Catalog Service (exact-lookup fallback)

        ┌────────── WRITE PATH (eventually-consistent, buffer-and-catch-up) ──────────┐
Catalog → CDC → Kafka → Indexing Service → ES _bulk
                  │        └ DLQ (poison messages)
                  └ durable buffer absorbs downstream outages
```

Everything that follows walks these two paths and, for each failure class, gives the
**detection → mitigation** pair the house style asks for.

---

## 10.2 ES Data Node Failure — Replicas Exist for HA, Not Just Throughput

Recall the topology (§6): ~600 primary shards, 1 replica each ⇒ ~1,200 total shards spread
across 60–100 data nodes. When a `data-hot` node dies (disk failure, kernel panic, OOM,
someone reboots the wrong box), here is the sequence Elasticsearch runs:

1. **Detection.** The elected master pings data nodes on a fixed interval
   (`cluster.fault_detection.follower_check`). After a small number of missed checks
   (~30 s by default; we tune tighter for hot nodes), the node is declared *left*. The master
   removes it from the cluster state.
2. **Replica promotion (fast).** For every primary shard that lived on the dead node, a
   replica elsewhere is **instantly promoted to primary**. This is a metadata flip in the
   cluster state — no data movement — so it happens in seconds. Queries and writes for those
   shards continue against the promoted primary. *This is the moment HA pays for itself.*
3. **Shard reallocation (slow, expensive).** The cluster is now under-replicated: every
   shard that lost its replica has only one copy. The master schedules new replicas onto
   surviving nodes. Each new replica is a **full shard copy** (~40 GB) over the network,
   throttled by `indices.recovery.max_bytes_per_sec` so recovery doesn't starve live queries.
   Rebuilding replicas for a dead node holding ~15–20 shards (~600–800 GB) can take **tens of
   minutes to hours**, bounded by that throttle and network capacity.

**The recovery cost is the real lesson.** Promotion is cheap; *re-replication* is what hurts.
During that window you are running with reduced redundancy and elevated load on survivors, so
node failures are riskiest in bursts. Peer-recovery optimizations help — ES reuses identical
segments already on the target and only ships the delta via the translog — but a cold target
still pays the full copy.

**Why replicas exist for HA, not just throughput.** It is tempting (and a common junior
mistake) to reason "we added a replica to double read capacity." True, replicas *do* serve
reads. But their load-bearing job is **survivability**: with `number_of_replicas: 1`, we can
lose any single node — or, with allocation awareness, a whole AZ — without losing data or
availability. Set replicas to 0 to save disk and you have converted every node failure into
**data loss on the read path**. For a 99.99% revenue path, one replica is a floor, not a
tuning knob. We use **shard allocation awareness** (`cluster.routing.allocation.awareness.attributes: zone`)
so a primary and its replica never share an AZ — otherwise a single-AZ outage could take both
copies of a shard.

---

## 10.3 Master Node Failure & Split-Brain

The cluster state — which shards live where, index mappings, the routing table — is owned by
a single **elected master**. If the master dies or is partitioned away, someone must take
over, and the cardinal sin is **split-brain**: two masters both accepting cluster-state
changes, then reconciling into an inconsistent, corrupt cluster.

**Dedicated masters (×3).** Per §6 we run **three dedicated master-eligible nodes** that do
*no* data or coordinating work. Keeping them dedicated means a query storm or a data-node OOM
can never knock the brain of the cluster over. Three is the minimum for a fault-tolerant
quorum.

**How ES avoids split-brain (quorum / voting).** Modern Elasticsearch (7.x+) replaced the
old, footgun-prone `minimum_master_nodes` setting with a built-in **Raft-like voting
configuration**. The rules that make split-brain impossible:

- Electing a master and committing any cluster-state change both require a **majority
  (quorum) of the voting configuration**. With 3 master-eligible nodes, quorum = **2**.
- On a partition, **at most one side can hold 2 of 3 votes**. The minority side (1 node)
  *cannot* elect a master and *cannot* commit state changes — it steps down and serves
  nothing authoritative. No second brain is ever born.
- The voting configuration is managed automatically; ES will refuse to shrink it below a
  safe size, and voting-config exclusions let you remove masters cleanly.

```
   Healthy: [m1] [m2] [m3]   quorum = 2, elected master = m1

   Partition:  { m1 } | { m2, m3 }
                 1 vote        2 votes  → elects new master, cluster proceeds
                 (minority,    (majority, authoritative)
                  no master)
```

**Cluster-state safety.** Each cluster-state update is versioned and must be persisted by a
quorum before it is acknowledged, so a committed state survives the loss of a minority of
masters. When the partition heals, the minority master-eligible node rejoins and syncs the
newer state; it never overwrites the majority's decisions. The practical operator rules:
**always run an odd number of masters (3)**, **never 2** (quorum = 2 means losing one makes
the cluster read-only for state changes — worst of both worlds), and put the three masters in
**three different AZs** so no single-AZ loss removes quorum.

---

## 10.4 Indexing-Path Failures — Kafka/CDC, Lag, Backpressure, DLQ

The write path (§5) is `Catalog → CDC → Kafka → Indexing Service → ES _bulk`. Its failure
philosophy: **buffer and catch up; never lose a change; never let a bad message wedge the
pipeline.**

### Kafka / CDC outage

- **CDC stops** (the connector tailing the Catalog binlog dies): no new events flow. Because
  CDC reads from the database's log with a committed **offset/LSN**, on restart it resumes
  exactly where it left off. Nothing is lost — the index simply lags, and freshness (NFR4)
  degrades gracefully until CDC recovers. Search keeps serving the last-known state.
- **Kafka broker/partition unavailable:** Kafka is our **durable shock absorber**. With
  replicated partitions (`replication.factor ≥ 3`, `min.insync.replicas = 2`), a broker loss
  is transparent. If Kafka is fully unreachable, the Indexing Service's consumers block and
  CDC buffers upstream in the DB log — retention (we keep days of log) is what buys us time.

### Consumer lag spikes & backpressure

The 100 K writes/sec peak (§2) is bursty — a price war or flash sale can spike mutations.
Detection is **consumer lag** (Kafka `records-lag-max`): how far behind the latest offset the
Indexing Service is. When lag climbs:

1. **Scale consumers** up to the partition count (lag is the autoscaling signal).
2. **Apply backpressure the right way:** slow the *consumer*, not by dropping messages, but
   by letting lag grow in Kafka's durable log. Kafka decouples producer rate from consumer
   rate — that is the entire point. We size retention so we can absorb a multi-hour outage of
   the Indexing Service or ES without data loss.
3. **Shed indexing to protect search.** If ES itself is the bottleneck, we *deliberately*
   throttle bulk indexing (smaller/less frequent bulk requests, raise the hot-tier refresh
   interval from 1 s). Freshness degrades; query latency is protected. This is the write-path
   embodiment of the read-path-first principle.

Bulk requests are the coupling point: ES returns **per-item statuses** in a `_bulk` response.
A `429 TOO_MANY_REQUESTS` (write queue full / circuit breaker) is a **retryable backpressure
signal** — retry *only the rejected items* with backoff (§10.8), never the whole batch.

### At-least-once + idempotency (from Ch6) makes replay safe

Kafka gives us **at-least-once** delivery: on consumer restart or rebalance, some messages are
redelivered. That is fine because — per Ch6 — every index write is **idempotent**, keyed by
`listing_id` with **external versioning** (`_version` from the Catalog, §3/§4):

```jsonc
// Indexing Service → ES: version comes from the Catalog mutation, not ES
PUT /products/_doc/L-8842?version=1234&version_type=external
{ "listing_id": "L-8842", "price": { "amount": 799.00, "currency": "USD" }, ... }
// ES rejects the write (409) if it already holds _version ≥ 1234 → stale replays are no-ops
```

So a replay of a whole Kafka segment cannot corrupt the index: newer versions win, older or
duplicate versions are silently dropped. **This is why "replay from Kafka" is always safe** —
lean on it in interviews.

### Dead Letter Queue for poison messages

A **poison message** is one the Indexing Service can never process — a malformed CDC event, an
unmappable field, a schema it doesn't understand. If we let it block the partition (retry
forever) we wedge the whole pipeline behind one bad record. Instead:

1. Try to process; on a **non-retryable** error (parse/mapping failure, not a transient 429/503),
   after a bounded number of attempts, **publish the record + error context + offset to a
   `indexing.dlq` topic** and **commit past it** so the partition keeps flowing.
2. The DLQ is monitored and alerted on (any non-zero rate is a bug signal).
3. **Replay:** once the transform bug or mapping is fixed, a tool re-publishes DLQ records back
   onto the main topic (or directly to `_bulk`). Because writes are idempotent and
   version-keyed, replay is safe and ordering-tolerant.

Distinguish **transient** failures (retry with backoff — ES 503, 429, network blip) from
**permanent** ones (DLQ — bad data). Retrying poison data forever is a classic outage cause.

---

## 10.5 Query-Path Degradation — Timeouts, Partial Results, Breakers, Bulkheads

Now the read path, where the 99.99% target lives. The scatter-gather query (a coordinating
node fans out to ~600 shards, gathers, merges) has a fragile property: **it is as slow as its
slowest shard.** One GC-paused or overloaded data node can drag a whole query past the 120 ms
ES budget (§2). We defend with layered controls.

**Timeouts everywhere.** Every hop has a budget derived from the §2 latency budget (edge 5 ms,
API+auth 10 ms, query understanding 15 ms, ES 120 ms, rerank 30 ms). Each hop's client sets a
timeout *tighter* than its caller's deadline so failures surface fast and locally, never as a
hung request holding a connection. No unbounded waits, anywhere.

**Partial results (`allow_partial_search_results`).** Elasticsearch defaults this to `true`,
and we keep it that way on the search path. If some shards fail or time out, ES returns results
from the shards that *did* answer, flagged with `_shards.failed > 0` and `timed_out`. For a
buyer, results from 595 of 600 shards is an excellent page; a 503 is not. The Search Service
surfaces a subtle "some results may be missing" only if failures are significant, and always
logs/metrics the partiality.

```jsonc
GET /products/_search
{
  "timeout": "100ms",                    // per-shard soft timeout → return what you have
  "allow_partial_search_results": true,  // one bad shard ≠ failed query
  "query": { ... }
}
// Response: "timed_out": false, "_shards": { "total": 600, "successful": 598, "failed": 2 }
```

**Per-shard timeout** (`timeout` in the body) bounds each shard's work; a runaway shard is
cut off instead of holding the whole gather.

**Circuit breakers — two layers:**

- **ES memory circuit breakers** (built in): the *request*, *fielddata*, and *parent*
  breakers track real heap usage and trip a query with `CircuitBreakingException` (a 429)
  *before* it OOMs the node. This converts "one huge aggregation kills the JVM" (class 5) into
  "one query is rejected." Preferable in every way — reject one request, not the node.
- **Service-level circuit breakers** (in the Search Service): if ES (or Redis, or the Ranking
  Service) starts failing/timing out beyond a threshold, the breaker **opens** and we stop
  hammering the sick dependency, fast-failing to the degradation path (§10.6) instead. It
  half-opens periodically to probe recovery. This prevents the Search Service from turning a
  slow dependency into a thread-pool exhaustion that takes *itself* down.

**Bulkheads.** Isolate resource pools so one workload can't drown another:

- Separate ES **coordinating nodes** and **thread pools** (`search` vs `write`) mean indexing
  bulk work cannot consume the search thread pool.
- The **autocomplete path is physically separated** (§2 note: ~500 K QPS on a lighter path) —
  often its own smaller index/cluster — so a typeahead storm cannot exhaust the main search
  cluster.
- In the Search Service, separate connection pools / thread pools per downstream (ES, Redis,
  Catalog, Ranking) so a stall in one is contained. This is the software analogue of a ship's
  watertight compartments — a breach floods one compartment, not the hull.

---

## 10.6 Graceful Degradation Playbook

This is the heart of the chapter and the strongest thing to say in an interview: a **ranked,
explicit ladder of what we shed, in what order, as pressure rises.** Each rung trades some
quality or freshness for availability. We never step off the bottom into a hard error.

| Rung | Trigger | Action | Buyer impact |
|---|---|---|---|
| 0 | Normal | Full: fresh, faceted, LTR-ranked | None |
| 1 | ES latency ↑ / breaker warm | **Serve slightly stale from Result Cache (Redis)** | Results seconds-to-minutes old |
| 2 | Load ↑ further | **Drop expensive facets & aggregations** | No/again fewer facet counts |
| 3 | Ranking Service slow/down | **Fall back to simpler ranking** (ES BM25 / cached model) | Slightly worse ordering |
| 4 | ES search failing broadly | **Fall back to Catalog Service for exact lookups** | Exact SKU/keyword hits only |
| 5 | System-wide pressure | **Shed autocomplete first**, then throttle low-value traffic | Typeahead disabled |

Walking the rungs:

1. **Serve stale from the Result Cache.** The Redis Result Cache (§5) normally holds hot
   queries with a short TTL. Under stress we **extend TTLs and serve stale-on-error**: if ES
   times out but we have a cached page, return the cached page. For head queries (a small set
   of queries drives a huge share of traffic), this alone absorbs most incidents. Stale
   results beat no results on a revenue path.

2. **Drop expensive facets/aggregations.** Facet counts over 10 B docs are among the priciest
   parts of a query (heap-hungry aggregations). Under load we serve the **result list without
   facet counts**, or with cached/approximate counts. The buyer still finds products; they
   just lose some filter refinement. This is a huge, cheap latency win exactly when it's
   needed.

3. **Fall back to a simpler ranking.** The LTR reranker (§5, Ranking Service) adds 30 ms and a
   network dependency. If it is slow or down, we **skip reranking and return ES's native
   BM25 + static business-signal ordering** (popularity/rating baked into the index). Ordering
   is slightly worse; the page still lands within budget. Never block the page on the
   reranker.

4. **Fall back to the Catalog Service for exact lookups.** In the rare case ES search is broadly
   unavailable, many buyer queries are actually *navigational* — a specific SKU, a known brand,
   a product ID. The Catalog Service (the source of truth, §4) can answer **exact-match lookups**
   directly. This is a narrow, last-resort path: no relevance, no facets, but "search for the
   exact thing" still works while ES recovers.

5. **Shed autocomplete first.** When the whole system is redlining, autocomplete is the *most
   sheddable* feature — it's a nicety, it's the highest-volume path (500 K QPS), and buyers can
   still type a full query and hit search. We disable/thin it first (return empty suggestions
   fast), reclaiming capacity for the core search path. Load-shedding is priority-aware:
   protect the checkout-adjacent search traffic, drop the optional stuff.

The playbook is driven by **feature flags and automatic triggers** (breaker state, latency
SLOs) so on-call can force a rung manually *and* the system self-degrades without a human. The
golden rule: **each rung is reversible and independently toggleable**, and we always prefer
degrade over deny.

---

## 10.7 Data Loss & Disaster Recovery

The reassuring truth, repeated because it drives every DR decision: **Elasticsearch is a
derived index, never the source of truth (§4).** The Catalog Service owns the authoritative
data. That gives us two independent recovery mechanisms with very different RTO/RPO.

**Layer 1 — Snapshots to object storage (fast restore).** We take **incremental ES snapshots**
via the Snapshot & Restore API to durable object storage (S3/GCS), using the repository plugin.
Snapshots are incremental at the Lucene-segment level — after the first, each only ships new
segments — so we can snapshot the hot tier frequently (e.g. every 30–60 min) cheaply. Restore
copies segments back and replays. This handles **data corruption (class 4)** and accidental
index deletion.

- Corruption detection: Lucene checksums every segment; ES surfaces `corrupt` shards and
  refuses to serve bad data. Recovery = restore that index (or shard) from the latest good
  snapshot, then catch up the delta by **replaying Kafka** from a safe offset (idempotent, §10.4).

**Layer 2 — Rebuild from the Catalog source of truth (ultimate backstop).** If snapshots are
also lost, or a bad mapping/transform silently corrupted the index at the *application* level
(snapshots would faithfully preserve the corruption), we **rebuild the entire index from the
Catalog.** This is the same machinery as a zero-downtime reindex (§6): build a fresh
`products-vN+1` index from a Catalog scan + Kafka tail, then atomically flip the `products`
alias. Slow (hours for 10 B docs), but it **cannot lose data that the Catalog still has.** This
is the property that lets us sleep at night, and the answer that impresses interviewers.

**RTO / RPO, stated explicitly:**

| Scenario | Mechanism | RPO (data loss) | RTO (time to recover) |
|---|---|---|---|
| Single node loss | Replica promotion + re-replicate | 0 | seconds (serving); minutes–hours (redundancy) |
| Single AZ loss | Allocation awareness; survivors serve | 0 | seconds |
| Index corruption/deletion | Restore snapshot + Kafka catch-up | ≤ snapshot interval + replayable log | tens of minutes |
| Whole-cluster / region loss | Failover to another region (§6 multi-region) | ~0 (async replicated + Kafka) | minutes (DNS/traffic shift) |
| Snapshots + cluster lost | **Rebuild from Catalog** | 0 (Catalog is truth) | hours |

Because the read path is multi-region (§6, cross-cluster), a regional outage is handled by
**shifting traffic to a healthy region** — RTO in minutes, no data loss — long before we ever
reach the rebuild backstop. The rebuild exists so that *no combination of failures* is
unrecoverable, not because we expect to use it often.

---

## 10.8 Retries Done Right

Retries are a double-edged sword: they mask transient blips *and* they can turn a small stumble
into a full outage via **retry storms** (every client retrying in lockstep hammers a recovering
service back down). The rules:

- **Exponential backoff + jitter.** Never retry immediately or on a fixed interval. Delay grows
  exponentially (e.g. 50 ms, 100 ms, 200 ms…) and each client adds **random jitter** so
  retries *de-synchronize* instead of arriving as a thundering herd. Jitter is the part people
  forget and the part that actually prevents storms.

  ```
  delay = min(cap, base * 2^attempt) * random(0.5, 1.5)   // full/decorrelated jitter
  ```

- **Bounded attempts + budgets.** Cap retries (2–3), and enforce a **retry budget** (e.g.
  retries ≤ 10% of requests) so a broad outage doesn't multiply load. When the budget is
  exhausted, fail fast to the degradation path (§10.6) instead of retrying.

- **Retry only idempotent, retryable failures.** Read queries are idempotent — safe to retry.
  Index writes are idempotent *because of version keys* (§10.4), so retrying a `_bulk` item is
  safe — but **only retry the items ES actually rejected** (429/503), never 4xx bad-data
  (that's DLQ territory).

- **Idempotency keys.** The `listing_id` + `_version` pair is our idempotency key on writes;
  duplicates and reorderings collapse to no-ops. Search requests carry a request ID for
  dedupe/tracing, not for state.

- **Timeouts everywhere, retries paired with circuit breakers.** A retry only makes sense
  inside a deadline; once the caller's budget is blown, stop. And when the service-level
  breaker (§10.5) is open, **don't retry at all** — retrying a known-sick dependency is exactly
  how you keep it down. Retries + breaker + backoff + jitter are one system, not four options.

---

## 10.9 Observability & Operations

You cannot degrade gracefully around a failure you cannot see. Detection latency *is* recovery
latency. We instrument the golden signals per component and alert on **symptoms the buyer feels**
(latency, errors, freshness), not just causes.

**Key metrics & alerts:**

| Metric | Where | Alert threshold (example) | Why it matters |
|---|---|---|---|
| **Search p99 latency** | Search Service / API GW | > 200 ms sustained (§1 SLA) | Direct SLA + revenue signal |
| **Autocomplete p99** | Autocomplete path | > 100 ms | Its own SLA (§1) |
| **Query error / partial rate** | Search Service | 5xx > 0.01%; `_shards.failed` ↑ | 99.99% budget; shard health proxy |
| **Indexing lag** | Indexing Service (Kafka consumer lag) | freshness > ~5 s / lag climbing | Freshness NFR4; early-warning |
| **DLQ rate** | `indexing.dlq` | any non-zero | Poison data / transform bug |
| **Shard health** | ES `_cluster/health` | status `yellow`/`red`; unassigned shards | Redundancy / data availability |
| **JVM heap / GC** | ES data nodes | heap > 75%, long GC pauses | #1 cause of node instability & timeouts |
| **Circuit-breaker trips** | ES + Search Service | trip rate ↑ | Overload / bad query detection |
| **Result Cache hit ratio** | Redis | drop from baseline | Cache miss storm → ES load spike |

Cluster health deserves a note: **green** = all shards assigned; **yellow** = replicas missing
(HA at risk, still serving); **red** = a primary is unassigned (data unavailable). Yellow is a
warning; red is a page.

**Runbooks.** Each alert links to a runbook with the *decision*, not a novel: e.g. "Indexing
lag climbing → check ES bulk rejections (429) → if ES-bound, throttle indexing (protect search)
and scale data nodes; if consumer-bound, scale Indexing Service to partition count; if DLQ
spiking, a transform is broken — page the owning team." Runbooks encode the §10.6 playbook so a
tired on-call makes the same call at 3 a.m. that we'd make in daylight.

**Chaos testing.** Unexercised failover *is* broken failover. We run **game days / chaos
engineering** in staging and (carefully) production: kill a data node and verify replica
promotion + re-replication; partition a master and verify quorum holds with no split-brain;
block Redis and verify the degradation ladder engages; inject a poison message and verify it
lands in the DLQ without wedging the partition; fail a region and time the traffic shift against
RTO. The point isn't to break things — it's to convert "we think it degrades gracefully" into
"we watched it degrade gracefully last Tuesday."

---

## Interview Tips

- **Lead with the principle, not the incident list.** State up front: "the read path is
  revenue-critical at 99.99%, so my bias is *degrade gracefully, never hard-fail*; the write
  path is eventually consistent, so I let it buffer and catch up." That framing organizes the
  whole answer and signals seniority.
- **The two strongest signals in this chapter are (1) graceful degradation and (2)
  rebuild-from-source.** Interviewers love hearing a *ranked degradation ladder* — serve stale
  from cache → drop facets → simpler ranking → Catalog exact-lookup → shed autocomplete — and
  they love "ES is a derived index; worst case I rebuild it from the Catalog source of truth."
  Say both explicitly.
- **Show you understand replicas are for HA, not just throughput.** "Replica → 0 turns every
  node loss into data loss" is a one-line way to prove it.
- **Nail split-brain:** 3 dedicated masters, quorum = 2, minority side can't elect or commit.
  Mention "odd number, never 2, spread across AZs."
- **Distinguish transient vs. permanent failures:** transient → retry with backoff+jitter;
  permanent (poison data) → DLQ. Add "retry only the rejected `_bulk` items, and only when the
  circuit breaker is closed" to show you've been burned by retry storms.
- **Common traps:** retrying without jitter (retry storm); retrying poison messages forever
  (wedged pipeline); `allow_partial_search_results: false` on a revenue path; running 2 master
  nodes; treating ES as the source of truth.
- **Always pair a failure with detection + mitigation.** "It fails" is junior; "here's the
  metric that catches it and here's what auto-happens" is senior.

## Key Takeaways

- **Classify before you mitigate:** node/hardware, network partition, dependency, corruption,
  overload, poison data — most real incidents are *cascades* across these, so the job is to
  break the cascade (bulkheads, breakers, backpressure).
- **Read path = availability (AP), write path = eventual correctness.** Isolate them so one's
  failure can't take out the other; when in doubt, protect search and let indexing lag.
- **Replicas are for survivability first.** Replica promotion is instant; re-replication is the
  expensive part. Use allocation awareness so a primary and replica never share an AZ.
- **Split-brain is solved by quorum:** 3 dedicated masters, majority (2) required to elect or
  commit cluster state; the minority side goes silent.
- **Idempotent, version-keyed writes make Kafka replay and DLQ recovery safe** — the foundation
  of the whole write-path resilience story.
- **Graceful degradation is a ranked, reversible ladder,** driven by breakers/flags: stale cache
  → drop facets → simpler ranking → Catalog exact-lookup → shed autocomplete.
- **Two-layer DR:** fast restore from snapshots; ultimate backstop = rebuild the derived index
  from the Catalog source of truth. State RTO/RPO per scenario.
- **Retries are dangerous without discipline:** backoff + jitter + bounded budget + idempotency +
  breaker awareness + timeouts everywhere.
- **You degrade around what you can see:** alert on buyer-felt symptoms (p99, error/partial rate,
  freshness lag), keep runbooks that encode the playbook, and prove it all with chaos testing.
