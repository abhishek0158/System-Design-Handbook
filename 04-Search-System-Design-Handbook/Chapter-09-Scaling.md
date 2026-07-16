# Chapter 9 — Scaling

> "Anyone can build a search box that works for one user. Scaling is what happens
> when three billion queries a day, a hundred thousand at the peak second, all
> want that box to feel instant." — the framing for this chapter.

Chapters 5 through 8 gave us a working system: a read path (CDN → API Gateway →
Search Service → Elasticsearch coordinating and data nodes) and a write path
(Catalog → CDC → Kafka → Indexing Service → ES bulk API). Chapter 8 explained the
engine internals — segments, shards, refresh, merge, the request cache. This
chapter answers a different question: **how do we make all of that hold up at
GlobalMart scale, at peak, without setting fire to the budget?**

Scaling is not one technique. It is a portfolio of levers, and the art is knowing
which lever to pull for which constraint. Pull the wrong one — add data nodes when
you're actually CPU-bound on ranking, or add replicas when your problem is a cold
cache — and you spend money without moving the metric. So we start by naming the
constraint precisely.

---

## 1. Framing: what "scaling" means for GlobalMart

Let's restate the numbers from Chapter 2, because every decision in this chapter is
justified against them:

| Dimension | Value | Source |
|---|---|---|
| Registered users | 2 B | Ch2 §2 |
| Daily active users | 500 M | Ch2 §2 |
| Search queries/day | ~3 B | 500M × 6 |
| **Avg search QPS** | **~35 K** | 3B / 86,400 |
| **Peak search QPS** | **~100 K** | ~3× average |
| **Autocomplete peak QPS** | **~500 K** | 5–8 keystrokes/search, lighter path |
| **Catalog write rate** | **~100 K writes/sec peak** | mostly price/inventory |
| Total listings (documents) | ~10 B | product × seller × locale |
| Raw corpus | ~20 TB | 10B × 2 KB |
| ES primaries (with doc values) | ~24 TB | 1.2× raw |
| With 1 replica | ~48 TB | on disk |
| Primary shards | ~600 | ~40 GB each |
| Total shards | ~1,200 | 600 × 2 |

Now the single most important insight, established in Chapter 2 and carried through
the whole design:

> **GlobalMart search is QPS-bound, not storage-bound.**

Here is why that phrase decides everything. If we only cared about storing 48 TB, a
handful of fat machines with big disks would do it — 48 TB is roughly 12 nodes at
4 TB each. But that cluster would collapse the instant real traffic arrived,
because **each search is a scatter-gather across ~600 shards**, and every shard that
holds data has to do CPU-and-RAM work (term lookups, scoring, aggregations for
facets) for every query that touches it. The limiting resource is not bytes on
disk; it is *query concurrency* — CPU cores, JVM heap, filesystem-cache RAM, and the
coordinating-node fan-out budget.

So when we size the cluster, we do not ask "how many nodes fit 48 TB?" We ask "how
many nodes serve 100K QPS at p99 ≤ 200 ms?" The brief's answer is **~60–100 data
nodes**, and the reason we land there is throughput, not capacity. We keep hot data
below ~2 TB per node *specifically so that there is enough spare RAM and CPU per byte
of data to sustain the query rate.* State this explicitly in an interview; it is the
sentence that signals you understand the workload rather than reciting a formula.

**A useful mental model — three independent scaling axes.** They do not move
together, and conflating them is the classic mistake:

```
   Axis            Scaled by                      Bound by
   ─────────────────────────────────────────────────────────────
   Read QPS    →   replicas + coordinating tier + caches   →  CPU/RAM/fan-out
   Write TPS   →   partitions, bulk, refresh control        →  indexing threads, I/O
   Corpus size →   primary shards, hot/warm/cold tiers       →  disk + heap
```

- **Read throughput** scales with *replica count* and *caching* (add copies of data,
  each copy can serve queries independently) plus the stateless coordinating and
  Search Service tiers.
- **Write throughput** scales with *partition parallelism* and *bulk batching*, and
  is protected by isolating it from reads.
- **Corpus size** scales with *primary shard count* and *tiering*, decided once and
  expensive to change.

Because we are QPS-bound, this chapter spends most of its ink on the read axis and on
caching — but a hyperscale system has to be competent on all three, so we cover each.

---

## 2. Scaling reads: replicas, coordinating tier, stateless services

### 2.1 Replicas are the read-throughput lever

In Elasticsearch, a query against an index is routed to **one copy of each shard** —
either the primary or one of its replicas. Both primaries and replicas can serve
searches. So if a shard has 1 primary + 1 replica, that shard's data can be searched
by *two* nodes in parallel; add a second replica and it's three. **Replica count is
therefore a direct multiplier on read capacity.**

Worked example. Suppose one data node, fully warmed, sustains **~1,500 shard-queries/
sec** at acceptable latency (a defensible number for ~40 GB shards on hot NVMe with
enough heap; you'd confirm it by load test). A search touching all 600 shards means
one query = 600 shard-queries of work spread across the cluster.

```
Peak load in shard-queries/sec = 100,000 QPS × 600 shards = 60,000,000 shard-q/s
Per-node capacity              = 1,500 shard-q/s
Nodes needed (raw)             = 60,000,000 / 1,500 = 40,000  ← absurd?
```

That number looks insane, and it exposes *why raw scatter-gather over 600 shards per
query is the thing we must attack.* Two forces rescue us, and they are the heart of
read scaling:

1. **Caching removes most queries before they reach ES** (Section 3). If 85% of
   searches are served from cache, only 15,000 QPS hit the cluster.
2. **Routing and filtering shrink the fan-out** (Section 6.4). A locale-routed or
   category-constrained query may touch 20 shards, not 600.

Redo the math with an 85% cache hit ratio and an effective average fan-out of ~120
shards (many queries are locale-routed):

```
ES-bound QPS       = 100,000 × (1 − 0.85) = 15,000 QPS
Shard-queries/sec  = 15,000 × 120 = 1,800,000 shard-q/s
Nodes (raw)        = 1,800,000 / 1,500 = 1,200 shard-slots
With 2 copies/shard actually it's about data-node CPU:
≈ 60–100 data nodes once you fold in headroom and replica parallelism.
```

That reconciles with the brief's 60–100 data nodes. The lesson: **you do not brute-
force 100K QPS into ES; you shrink the number and the fan-out first, then size the
cluster for the remainder.** Replicas serve the remainder in parallel.

Replicas buy us three things at once — read throughput, availability (a lost primary
is replaced by a replica; Chapter 10), and — modestly — latency, because the
coordinating node can pick the least-loaded copy. Our baseline is **1 replica**
(1,200 total shards). We can raise replica count on the *hot* tier during Black
Friday (Section 10) to add read capacity, at the cost of more disk and more indexing
work.

### 2.2 The coordinating-node tier

A search is a two-phase scatter-gather (query-then-fetch, Chapter 8): the
**coordinating node** receives the request, fans it out to one copy of each relevant
shard, collects the top-K from each, merges and re-sorts, then fetches the actual
documents for the final page. That merge/sort/aggregate work is real CPU, and doing
it on data nodes steals cycles from the shard queries themselves.

So at scale we run **dedicated coordinating nodes** (data:false, master:false,
ingest:false) — a stateless routing/merge tier that:

- absorbs the fan-out and gather-merge CPU, protecting data-node heap;
- terminates client connections and does result aggregation and facet merging;
- can be scaled **horizontally and independently** of data nodes — add coordinators
  when merge cost (deep pagination, heavy facets, large `size`) is the bottleneck,
  not when shard search is.

```
                         ┌─ data-hot-1 (shards …)
Search Service ─▶ Coordinating tier ─┼─ data-hot-2 (shards …)
   (N stateless)     (M stateless)   ├─ ...
                         └─ data-warm-k (shards …)
        fan-out ─────────────────────▶  gather + merge ◀──── back up
```

Rule of thumb: budget a handful of coordinating nodes per few dozen data nodes, and
watch their heap/GC — a coordinator OOMs on a giant aggregation long before a data
node does. Facet-heavy queries and deep `search_after` pages are the usual culprits.

### 2.3 Horizontal, stateless Search Service

The Search Service (Chapter 5/7) — query understanding, cache lookup, ES call,
ranking/LTR rerank, response shaping — is **stateless**. That is a deliberate design
property that makes it trivially scalable: put N identical instances behind the API
Gateway, load-balance across them, and scale N to whatever the request rate demands.
No sticky sessions, no local state that can't be lost (the Result Cache lives in
Redis, not in-process; see 3.2). If an instance dies, the Gateway routes around it.

This is the cheapest tier to scale — commodity stateless app servers autoscaling on
CPU/RPS (Section 10) — so we push as much work here as possible (caching, cheap
rejection of bad queries, coalescing) to keep the expensive stateful tier (ES) doing
less. The order of scaling difficulty, cheapest to hardest:

```
Search Service (stateless)  <  Coordinating (stateless)  <  Data nodes (stateful)  <  Primary shard count (fixed)
       add pods                     add pods                    rebalance data          reindex — expensive
```

Always scale left-to-right: exhaust the cheap levers (and caching) before you touch
the expensive, stateful ones.

---

## 3. Caching tiers as a scaling lever

Caching is the highest-leverage tool we have, because **a cache hit is a query that
never touches the 600-shard scatter-gather.** In a QPS-bound system, cache hit ratio
is almost linearly a node-count reducer. We run a **layered cache**, each layer
catching what the one above missed:

```
Client
  │
  ▼  (1) CDN / edge cache          — popular queries, ~seconds TTL, per-region
API Gateway
  │
  ▼  (2) Result Cache (Redis)      — normalized full responses, short TTL
Search Service
  │
  ▼  (3) ES shard request cache    — per-shard aggregation/hits cache
Elasticsearch
```

### 3.1 CDN / edge cache

The cheapest hit is the one served closest to the user. Search *responses* for
popular, non-personalized queries (`q=iphone&locale=en-US&page=1`, no
personalization token) are cacheable at the CDN/edge with a very short TTL (a few
seconds). This is extraordinarily effective for **head queries**: on a launch day,
"iphone" may be 2–3% of all traffic in a locale. Even a 3-second edge TTL collapses
thousands of identical requests per second into one origin fetch.

Cacheability rules:
- **Only anonymous / non-personalized** responses. If the response varies by user
  (personalized ranking, `Authorization`-dependent), it is `private` and skips the
  edge — or we split personalization out of the cache key (cache the base result set,
  personalize on the way out).
- Vary on the fields that change the result: `q`, `locale`, `market`, `filters`,
  `sort`, `page`, `size`. These become the cache key (see 3.2).
- Short TTL to bound staleness against NRT (see the tension in 3.3).

### 3.2 Result Cache (Redis)

The **Result Cache (Redis)** — a named component in the brief's architecture — sits
in the Search Service. It caches the *assembled* search response (post-ranking) so a
repeat query skips query understanding, ES, and reranking entirely.

**Cache key.** A canonical, normalized key so that logically-identical queries
collide (a hit):

```
key = sha1( lower(trim(q))            # normalized query text
          | sorted(filters)          # order-independent
          | sort | page | size
          | locale | market
          | ranking_model_version )  # bust cache on model change
```

Normalization matters: `"iPhone 15 "`, `"iphone 15"`, and `"IPHONE 15"` should be one
key. Sorting the filter list makes `brand:apple,color:black` equal
`color:black,brand:apple`. Including `ranking_model_version` means shipping a new LTR
model invalidates stale rankings automatically (no manual flush).

**What we store.** The final result page (listing IDs + rendered fields + facet
counts + query metadata). We often store **IDs + a hydrate step** rather than full
docs, so price/inventory can be refreshed from a fast store on the way out — this
lets us use a *slightly longer* TTL for the expensive relevance computation while
keeping volatile fields fresh (see 3.3).

**TTL.** Short — on the order of **5–30 seconds** for head queries. This is the knob
that trades freshness for hit ratio.

### 3.3 The TTL ↔ NRT tension

Here is the fundamental conflict. Chapter 6 promised **NRT indexing**: new listings
searchable within seconds, price/inventory within ~1 s. A Result Cache with a 30 s
TTL means a user can see a **30-second-stale** result — a product that just went out
of stock, or a price that just changed. Caching and freshness pull in opposite
directions.

We resolve it by **caching by volatility class**, not one TTL for everything:

| Data class | Volatility | Cache TTL | Strategy |
|---|---|---|---|
| Result set / ranking (which docs, what order) | low | 10–30 s | cache the ID list + order |
| Facet counts | low–med | 10–30 s | cache with the result |
| Price / inventory / in_stock | high (~1 s SLA) | **do not cache in result** | hydrate at read time from Catalog fast store |
| Autocomplete suggestions | very low | minutes | separate cache, long TTL |

So the cache stores "for query X the ranked listing IDs are [L1, L2, …]" (stable for
tens of seconds), and the Search Service **hydrates volatile fields per request**
from a fast key-value store fed by the same CDC stream (≤1 s fresh). We get the
cache's throughput win on the expensive part (retrieval + ranking) *and* honor the
1-second price/inventory SLA. This split is the senior answer to "how can you cache
when you also promised freshness?"

### 3.4 ES shard request cache

The last layer is inside ES. The **shard request cache** caches the results of the
query phase *per shard* — especially valuable for the **facet aggregations**, which
are expensive and identical across many users issuing the same filtered query. It is
keyed by the shard-level request body and **automatically invalidated on refresh**
(so it self-heals on the refresh interval — no stale facets beyond one refresh). It
only caches requests with `size:0` or where hits are cacheable, so it's a big win for
"give me the facet counts for category=Phones" style sub-requests. We keep it on for
the hot indices.

### 3.5 Cache hit-ratio → node-count sensitivity

This is the table to draw on the whiteboard. It shows how brutally the required data-
node count depends on aggregate cache hit ratio (edge + Redis combined), at 100K peak
QPS, holding fan-out and per-node capacity fixed:

| Aggregate cache hit ratio | QPS reaching ES | Relative ES load | Approx data nodes* |
|:---:|:---:|:---:|:---:|
| 0% | 100,000 | 1.00× | ~400 |
| 50% | 50,000 | 0.50× | ~200 |
| 70% | 30,000 | 0.30× | ~120 |
| **85% (target)** | **15,000** | **0.15×** | **~60–100** |
| 95% | 5,000 | 0.05× | ~30 |

*Illustrative, linear in ES-bound QPS above a fixed floor for redundancy/HA; the
point is the *shape*, not the exact integers.

The takeaways an interviewer wants to hear:

1. **Cache hit ratio is a first-class capacity parameter.** Moving from 70% → 85%
   nearly halves the cluster. That is millions of dollars of hardware bought by a
   Redis tier that costs a rounding error by comparison.
2. **Returns diminish and risk grows** past ~85–90%: you're now depending on the
   cache so heavily that a cache outage becomes a cluster-killer (Section 4 & Ch10).
   So we design the ES cluster to survive a **partial** cache loss — we don't size it
   assuming 95% forever.
3. Head-heavy query distributions (Zipfian — a few queries are enormously popular)
   make high hit ratios *achievable* with a small cache: caching the top ~10K queries
   can cover a large share of traffic.

---

## 4. Thundering herd / cache stampede on "celebrity" queries

The dark side of relying on cache: **the cache entry for a celebrity query expiring
under load.**

Scenario. It's launch day. "iphone" is running at, say, 3,000 QPS in `en-US` alone,
almost all served from one Redis entry with a 20 s TTL. At second 20 the entry
expires. Now, in the window before it's repopulated, **all 3,000 QPS simultaneously
miss**, all decide they must recompute, and all stampede the ES cluster with the same
expensive query at once. The cluster, sized for 15K aggregate QPS, suddenly eats a
3,000-QPS spike of identical work; latency spikes, which makes requests pile up,
which makes it worse. This is the **thundering herd** (a.k.a. **cache stampede**), and
on a popular query it can take down a healthy cluster in seconds.

Mitigations, layered (use several — they defend at different points):

**1. Request coalescing (single-flight).** In the Search Service, when a request
misses and goes to recompute, register an in-flight marker (a per-key lock/promise in
the local process and/or a short-lived Redis lock). Concurrent requests for the *same
key* do **not** each recompute — they **wait on the first one's result** and share it.
So 3,000 concurrent misses become **one** ES query whose result is fanned back out to
all 3,000 waiters. This alone neutralizes most of the herd.

```
req1 (miss) ─┐
req2 (miss) ─┼─▶ [single-flight lock on key] ─▶ 1 ES query ─▶ fill cache ─┐
req3 (miss) ─┘                                                            │
   └────────────── all three get the same result ◀──────────────────────┘
```

**2. Soft / stale TTL (serve-stale-while-revalidate).** Store two timestamps: a
**soft TTL** (e.g., 15 s) and a **hard TTL** (e.g., 60 s). Between soft and hard, the
entry is *usable but stale*: the request that first notices it's past soft-TTL
triggers an **asynchronous refresh** and everyone keeps getting the slightly-stale
cached value until the refresh lands. The entry is never a hard "gone → everyone
recompute" cliff. Combined with single-flight, the refresh is done by exactly one
worker.

**3. TTL jitter.** Never expire many entries at the same instant. Add randomness:
`ttl = base ± rand(0, jitter)`. This prevents *synchronized* expiry of a batch of
popular queries (e.g., everything cached during a traffic spike expiring together 20 s
later). Cheap, and it smears the stampede risk across time.

**4. Negative caching.** Queries that return **zero results** (typos that even spell-
correction can't save, junk, probes) are otherwise *uncached* and hit ES every time —
an easy accidental DoS. Cache the empty result too, with its own short TTL, so a
flood of "asdfghjkl" costs one ES query, not thousands.

**5. Cache warming / pre-computation.** For *known* celebrity events (a scheduled
product launch, a Black Friday deal page), pre-populate the cache for the expected
hot queries *before* the traffic arrives, and refresh those keys proactively on a
timer so they never expire under load. The head of the distribution is predictable;
pin it.

The senior framing: **request coalescing + soft-TTL is the belt; jitter + negative
caching + warming is the suspenders.** Never rely on a single mechanism, because a
cache miss on a celebrity key is a self-amplifying failure.

---

## 5. Scaling writes

Reads are the SLA-critical path, but writes are relentless: **~100K writes/sec at
peak**, dominated by price/inventory mutations, plus ~50M new listings/day. The
guiding principle (from Chapter 6) is **isolate indexing from querying** so a write
surge never eats the read SLA.

### 5.1 Bulk indexing, not per-doc

Never index one document per request at this scale — per-request overhead
(HTTP, routing, refresh pressure) dominates. The Indexing Service batches CDC events
into **`_bulk`** requests (Chapter 6): typically a few thousand docs or ~5–15 MB per
bulk call, tuned by load test. Bulk amortizes coordination cost and lets Lucene write
larger, more efficient segments. Worked figure:

```
100,000 writes/sec ÷ 5,000 docs/bulk = 20 bulk requests/sec  ← trivial request rate
```

The write problem becomes *segment and merge pressure*, not request rate. We size
indexing threads and I/O for the merge load, and we watch the bulk **rejection**
queue — rejections are the signal to add indexing capacity or throttle upstream.

### 5.2 Partition parallelism via Kafka

Kafka is the shock absorber and the parallelism unit. The CDC stream is **partitioned
by `listing_id`** (or `product_id` for locality), giving us:

- **Ordering per key** — all mutations for a listing land in one partition, so version
  conflicts are avoided and the latest `_version` wins (idempotent writes, brief §3).
- **Horizontal write scale** — add partitions and Indexing Service consumers to raise
  throughput; consumers scale independently of ES.
- **Buffering** — a write spike (flash sale re-pricing 50M items) is absorbed by
  Kafka retention; the Indexing Service drains at ES's sustainable rate instead of
  overwhelming it. Lag is visible and alertable.

```
Catalog ─▶ CDC ─▶ Kafka (partitioned by listing_id) ─▶ Indexing Service (N consumers) ─▶ ES _bulk
                     │                                        │
             buffers surges                          batches + versions + throttles
```

### 5.3 Throttle refresh during heavy load

The refresh interval (Chapter 8) controls how often new segments become searchable —
and refresh is *expensive*. During normal operation the hot index runs a **1 s
refresh** to honor NRT. But during a **heavy bulk load** (an initial reindex, a
massive re-price), we deliberately **raise the refresh interval** (e.g., 30 s, or
`-1` = off) so ES spends its I/O on ingesting rather than on constant tiny refreshes
and the resulting merge storm. Throughput can improve severalfold. We restore the 1 s
refresh (and force a refresh) when the bulk load finishes. This is a load-time knob,
applied per index, not a permanent setting.

Same spirit: set `number_of_replicas: 0` during a from-scratch reindex (no replica-
side indexing work) and bump it back to 1 afterward — replicas rebuild from the
primary. Reserve this for offline reindexes, not the live index (it sacrifices
availability while replicas are absent).

### 5.4 Separating indexing load from query load

The strongest isolation would be to index and query on different hardware. In ES,
primaries and replicas *both* do indexing work and *both* can serve queries, so we
can't fully split them within one index. What we do instead:

- **Route heavy background jobs to off-peak windows** and to the warm tier where
  possible (reindexing old locales).
- **Coordinating-node tier** already shields data-node query CPU from merge/gather
  work (Section 2.2).
- **Cap indexing throughput** with a governor in the Indexing Service so a write
  surge can't starve query threads; the excess parks safely in Kafka.
- For the most extreme isolation, some shops maintain a **separate indexing-optimized
  cluster** and ship segments to the query cluster via CCR (Section 8) — powerful but
  operationally heavy; call it out as an option, not the default.

---

## 6. Sharding strategy at scale

### 6.1 The ~600 primary shard decision

From the brief: **~24 TB of primaries ÷ ~40 GB target shard = ~600 primary shards**
(1,200 total with 1 replica). Why 40 GB?

- **Too big** (say 200 GB shards): a single shard's queries get slow (more segments,
  more terms to scan per shard), recovery/rebalance moves huge chunks, and a hot shard
  can't be split off. Latency and operability suffer.
- **Too small** (say 2 GB shards → 12,000 shards): **over-sharding.** Every query
  fans out to thousands of shards; the coordinating node's gather-merge and the per-
  shard fixed overhead dominate; cluster state (managed by masters) bloats; and each
  shard carries fixed Lucene/heap overhead. This is the more common and more
  dangerous mistake at scale.

40 GB is the sweet spot ES operators converge on: large enough to amortize per-shard
overhead, small enough to search quickly, recover fast, and rebalance smoothly. It
keeps fan-out at ~600 (manageable with a coordinating tier and routing) rather than
thousands.

### 6.2 Over-sharding is the classic trap

```
Shards per query   Per-shard fixed cost   Coordinator merge cost   Verdict
────────────────────────────────────────────────────────────────────────
   12,000 (small)        crushing               crushing            over-sharded ✗
      600 (target)       amortized              manageable          right ✓
       30 (huge)         fine                   trivial             hot-shard risk ✗
```

The heuristic: **aim for ~20–40 GB shards and keep total shards per node in the low
hundreds.** With 60–100 data nodes and 1,200 shards, that's ~12–20 shards/node —
comfortable.

### 6.3 You cannot change primary count without a reindex

Critical constraint, and a favorite interview probe. **The number of primary shards
is fixed at index creation** because ES routes a document by `hash(routing) %
number_of_primary_shards`. Change the divisor and every document would route to a
different shard — the index would be corrupt. So to change primary count you must
**reindex** into a new index with the new shard count.

This is *exactly why* the brief mandates the **alias pattern**: the `products` alias
points at a versioned index `products-vN`. To re-shard (or change mappings/analyzers),
you:

1. Create `products-v(N+1)` with the new primary count.
2. Reindex `products-vN` → `products-v(N+1)` (via `_reindex` or by replaying from the
   Catalog / Kafka — ES is a rebuildable derived index, brief §4).
3. Atomically **swap the alias** to the new index (zero downtime).
4. Drop the old index once verified.

Because re-sharding is this expensive, **choose 600 with headroom for growth up
front.** A common tactic: slightly over-provision primaries now (e.g., pick a count
that leaves room for the corpus to grow ~2× before shards exceed ~50 GB), so you're
not reindexing 24 TB every quarter. But don't over-do it — that's just over-sharding
with extra steps.

### 6.4 Routing to reduce fan-out

Default routing spreads a document across shards by `hash(listing_id)`, so a query
must hit **all 600 shards**. But GlobalMart queries are almost always **scoped to a
locale/market** (a buyer in `en-US` doesn't want `ja-JP` listings). If we **route by
locale/market** (`routing = market`), all documents for a market live on a known
subset of shards, and a market-scoped query fans out only to *those* shards:

```
Query "iphone" (market=US)  →  route to US shards only  →  fan-out ~20–60, not 600
```

This is a huge lever — it directly shrinks the shard-queries/sec term in Section 2.1's
math. Trade-off: **routing can create imbalance** — the US and EU markets are far
larger than small locales, so their shards get hot (Section 9). We mitigate with a
routing scheme that spreads large markets across multiple shards
(`routing = market + bucket`) while keeping small markets compact, and we co-locate
hot markets on the hot tier (Section 7). The senior point: **routing trades even
distribution for reduced fan-out — worth it when queries are naturally partitioned,
but you must actively manage the resulting skew.**

---

## 7. Hot / warm / cold architecture & ILM

Not all 10B listings are equal. A small fraction of locales and products serve the
overwhelming majority of queries; the long tail (obscure locales, discontinued
listings, old catalog) is rarely searched but must remain available. Putting all 48 TB
on identical premium hardware is wasteful. Enter **tiered storage**.

| Tier | Hardware | Holds | Replicas | Refresh | Purpose |
|---|---|---|---|---|---|
| **Hot** | fast CPU, NVMe SSD, lots of RAM | popular locales/markets, recent + active listings | 1–2 | 1 s (NRT) | serve the bulk of QPS at low latency |
| **Warm** | fewer cores, cheaper/denser disk, less RAM | older/less-popular locales, long-tail listings | 1 | higher / read-mostly | serve occasional queries cheaply |
| **Cold / frozen** | object storage / searchable snapshots | archival, rarely-queried history | 0–snapshot | n/a | keep searchable at minimal cost |

**Index Lifecycle Management (ILM)** automates the transitions. We manage time- and
popularity-based indices (e.g., per-market, rolling) through ILM phases:

```
hot  ──(age > 30d OR low query rate)──▶  warm  ──(age > 6m)──▶  cold  ──▶ delete/snapshot
      • allocate to data-hot nodes            • move to data-warm       • searchable snapshot
      • force-merge to fewer segments         • reduce replicas
```

`_rollover` creates a fresh write index when the current one hits a size/age/doc
threshold; ILM then ages indices through the phases and reallocates shards to the
matching node tier via shard-allocation awareness (`index.routing.allocation.
require.data_tier: warm`).

**Why this is a scaling lever, not just cost hygiene:** by keeping only the hot
working set on premium nodes, we get the *QPS-serving power* concentrated where the
queries actually are. We might hold ~2–4 TB of genuinely hot data on fast nodes with
enormous filesystem-cache RAM, and 40+ TB of tail on cheap warm nodes that barely see
traffic. The result: **better p99 on the queries that matter, at a fraction of the
all-hot cost.** In an interview, tie it back: "we're QPS-bound, so we spend our RAM/
CPU budget on the hot set and let the cold tail live cheaply."

Cost sketch (illustrative):

```
All-hot:   48 TB on premium nodes           = $$$$$
Tiered:    4 TB hot (premium)               = $$$
         + 44 TB warm/cold (cheap/dense)    = $
           ───────────────────────────────────
           ~50–70% lower spend, same or better hot-path latency
```

---

## 8. Multi-region

GlobalMart is global; a single region cannot serve 500M DAU with low latency or the
required 99.99% availability. We go multi-region for **latency locality, availability,
and data residency.**

### 8.1 Latency locality

A buyer in Tokyo hitting a US-east cluster pays ~150 ms of round-trip *before* any
work happens — blowing the 200 ms p99 budget. So we run **regional clusters** and
route each buyer to the **nearest** one (GeoDNS / anycast at the CDN/Gateway). The
buyer's locale/market usually matches their region, so the data they want is local.

```
   EU buyers ─▶ eu-cluster   ┐
   US buyers ─▶ us-cluster   ┼─ each serves its region's hot locales locally
   APAC buyers ─▶ ap-cluster ┘   (low latency, data residency honored)
```

### 8.2 CCR vs CCS

Two Elasticsearch primitives, different jobs — know the difference cold:

- **Cross-Cluster Replication (CCR):** asynchronously **replicates indices** from a
  leader cluster to follower clusters in other regions. Use it to **place a read-only
  copy of data near buyers** and for **DR** (a follower can be promoted if the leader
  region dies). Follower is near-real-time but eventually consistent.
- **Cross-Cluster Search (CCS):** lets one cluster **query across** remote clusters at
  request time without copying data. Use it for **occasional cross-region queries**
  (an admin searching globally, or a rare "search all markets" request) — but it pays
  the cross-region latency, so it's *not* the hot path.

Our default: **CCR to keep each region's hot data local** (fast, resilient reads),
and **CCS reserved for rare global queries.** Most buyer traffic never crosses a
region.

### 8.3 Data residency

Regulations (GDPR, and various national data-localization laws) may require that data
for a market **stay within** a jurisdiction. Because we already partition by
locale/market and run regional clusters, residency falls out naturally: EU listings
live (only) in the EU cluster, and CCR replication targets are chosen to respect
residency rules (don't replicate EU personal data to a non-compliant region). This is
a design constraint to *state explicitly* — it can forbid otherwise-attractive
active-active setups.

### 8.4 Active-active vs active-passive

| Model | Description | Pro | Con | Use when |
|---|---|---|---|---|
| **Active-passive** | one region takes writes; others are read replicas / DR standby | simple, no write-conflict problem | failover has RTO; standby capacity idles | catalog is naturally single-writer |
| **Active-active** | multiple regions accept traffic (and possibly writes) | best latency + full HA | write-conflict resolution, residency, cost | truly global, latency-critical |

For **the search index**, active-active *reads* are the easy win — every region serves
searches from its local (CCR-followed) copy. **Writes** are the subtlety: the Catalog
(source of truth) is authoritative, and CDC → Kafka feeds the indexing pipeline.
Cleanest design: **writes flow from the Catalog's home region and CCR fans the index
out** (active-active reads, effectively single-writer indexing) — this sidesteps
index write-conflicts entirely, since ES is a derived, rebuildable index (brief §4).
If the write region fails, promote a follower (active-passive failover for the write
path). This hybrid — **active-active reads, active-passive writes** — is the pragmatic
GlobalMart answer.

### 8.5 Routing users to the nearest cluster

- **GeoDNS / anycast** resolves the buyer to the nearest edge and regional Gateway.
- The Gateway routes to the **local** regional cluster by default.
- **Health-aware failover:** if the local cluster is degraded, the Gateway fails over
  to the next-nearest healthy region (accepting higher latency over an outage) — this
  is the availability half of multi-region (Chapter 10 covers the failure handling).

---

## 9. Hotspot mitigation

Even a well-sized cluster dies if load is **skewed** onto a few shards or nodes.
Uniform average utilization hides lethal hotspots.

**Sources of skew at GlobalMart:**

1. **Skewed shards from routing** (Section 6.4): routing by market puts the giant US/
   EU markets' data on a few shards while tiny locales' shards idle. Those big-market
   shards get both more data *and* more queries.
2. **Popular sellers / brands / categories:** a mega-seller or a hot category
   ("Electronics" on launch day) concentrates both documents and query load.
3. **Celebrity queries** (Section 4): a single term hammering the shards that hold its
   postings.

**Mitigations:**

- **Bucketed routing.** Instead of `routing = market`, use `routing = market + bucket`
  where `bucket = hash(listing_id) % K`, spreading a large market across *K* shards
  while a small market uses one. Sizes *K* per market to its volume — big markets get
  more buckets. This keeps fan-out low *and* spreads the big markets. (Query then fans
  to the market's *K* shards, still far fewer than 600.)
- **Shard allocation awareness & rebalancing.** ES rebalances shards across nodes by
  count/disk, but not by *query heat*. We monitor per-shard/per-node query load and
  **manually move or split** hot shards (`_split` into more primaries — via reindex —
  or relocate to less-loaded nodes). Allocation *awareness* also spreads
  primary+replica across racks/AZs so a shard's copies aren't co-located.
- **More replicas on hot shards.** Adding replicas to the hot tier multiplies read
  capacity for the hottest data specifically (Section 2.1) — the targeted fix for a
  read-hot shard.
- **Caching in front of the hottest keys** (Sections 3–4): the celebrity-query defense
  is also a hotspot defense — a coalesced, cached "iphone" never reaches its shards at
  full volume.
- **Balance data at ingest.** For pathological mega-sellers, consider spreading their
  listings across buckets so no single shard holds a disproportionate slice.

The principle: **watch the p99 and the per-shard heat, not the averages.** A cluster
at 40% average CPU with three shards pinned at 100% is an outage waiting to happen.

---

## 10. Autoscaling, capacity headroom, and load testing for peak

### 10.1 What autoscales, and what doesn't

| Tier | Autoscale? | Signal | Speed |
|---|---|---|---|
| Search Service (stateless) | **Yes, aggressively** | CPU / RPS / latency | seconds–minutes |
| Coordinating nodes (stateless) | **Yes** | CPU / heap / queue | minutes |
| Redis cache | scale out / bigger | hit ratio, memory, evictions | minutes |
| Data nodes (stateful) | **Cautiously** | CPU, search queue, p99 | **minutes–hours** (shards must rebalance) |
| Primary shard count | **No** — reindex only | n/a | offline |

The asymmetry is the whole point: **stateless tiers scale in seconds; the stateful ES
data tier does not.** Adding a data node triggers shard relocation — gigabytes moving
across the network, warming caches — which takes minutes to hours and *itself*
consumes cluster resources. So **you cannot autoscale data nodes reactively fast
enough for a Black Friday spike.** You must **pre-provision.**

### 10.2 Provisioning for peak — the surge factor

We size for **peak, not average**, plus a **surge factor** for the truly exceptional
day:

```
Average QPS               = 35,000
Normal peak (3× avg)      = 100,000                    ← brief's peak
Black Friday surge factor = ~2× normal peak            ← planned event
Provisioned target        = 100,000 × 2 = 200,000 QPS peak capacity
Headroom margin           = +30% above provisioned target
─────────────────────────────────────────────────────────────
Design point              ≈ 260,000 QPS "must not fall over" ceiling
```

For a *known* event (Black Friday, Prime-style sale), we **scale up the ES cluster
ahead of time** — add hot-tier nodes and raise replica counts days before, let shards
rebalance and caches warm, then scale back down after. The stateless tiers and cache
follow demand automatically on top of that pre-warmed base. Combine with **cache
warming** (Section 4.5) for the known hot queries and **graceful degradation levers**
(Chapter 10 — shed load by trimming facets, reducing `size`, disabling expensive
rerank) as the safety valve if we still exceed the ceiling.

### 10.3 Load testing

You cannot claim a capacity number you haven't measured. The discipline:

- **Realistic query mix.** Replay *production query logs* (or a Zipfian synthetic that
  matches the head/tail shape), including the celebrity-query concentration — a uniform-
  random load test lies, because it defeats caching and hides the stampede risk.
- **Realistic cache state.** Test both **warm cache** (steady state) and **cold cache**
  (post-deploy, post-Redis-failover) — the cold-cache number is your true worst case
  and the one that sizes the ES floor (Section 3.5, point 2).
- **Find the knee.** Ramp QPS until p99 breaks 200 ms; that inflection is your per-
  configuration ceiling. Divide desired peak by it to get node count, then add the
  headroom margin.
- **Test failure under load** (Chapter 10): kill a node / a Redis shard *during* a
  peak test and confirm you degrade gracefully rather than cascade.
- **Continuously.** Re-run before every big event and after significant changes
  (new LTR model, mapping change) — capacity drifts.

```
p99 latency
   │                              ┌── knee (~here = ceiling)
200ms ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─
   │                         ____/
   │      ______________----
   └───────────────────────────────────────▶ offered QPS
   provision the fleet a comfortable margin left of the knee
```

---

## Interview Tips

- **Lead with the constraint.** Say out loud: *"This system is QPS-bound, not storage-
  bound — 100K peak QPS scatter-gathering across ~600 shards is the pressure, so I
  provision for query throughput, not disk."* This one sentence reframes the entire
  scaling discussion and signals seniority immediately.
- **Separate the three axes** (read QPS / write TPS / corpus size) and say which lever
  scales which. Interviewers probe when candidates conflate "add nodes for storage"
  with "add nodes for QPS."
- **Make caching quantitative.** Draw the hit-ratio → node-count table. Saying "85%
  cache hit roughly halves the cluster versus 70%" is far stronger than "we'd add a
  cache." Then immediately note the risk: *don't size ES assuming the cache never
  fails.*
- **Volunteer the thundering-herd problem before you're asked.** Naming cache
  stampede on a celebrity query, then giving *layered* mitigations (single-flight
  coalescing + soft TTL + jitter + negative caching + warming), is a strong senior
  signal.
- **Know the primary-shard immutability cold.** "You can't change primary shard count
  without a reindex, which is why we use the alias → versioned-index pattern" is a
  frequent gotcha. Have the zero-downtime reindex+alias-swap steps ready.
- **Explain routing's trade-off honestly:** it slashes fan-out but creates skew, and
  you must actively manage the resulting hotspots (bucketed routing). Don't present it
  as free.
- **CCR vs CCS:** replicate data near users (CCR) vs query across regions at request
  time (CCS). Mixing these up is a classic slip.
- **Pre-provision for stateful tiers.** State plainly that you *cannot* reactively
  autoscale ES data nodes fast enough for Black Friday — you scale up ahead of the
  event and lean on graceful degradation as the safety valve.
- **Common traps to avoid:** over-sharding (thousands of tiny shards); assuming
  replicas help write throughput (they don't — they add write work); caching
  personalized or volatile (price/inventory) data with a long TTL; sizing only for
  average QPS; quoting a capacity number you never load-tested.

## Key Takeaways

- GlobalMart search is **QPS-bound**: the ~60–100 data nodes exist to serve 100K peak
  QPS across ~600 shards, not to store 48 TB. Provision for query throughput.
- Scaling splits into **three independent axes** — read QPS (replicas + coordinating
  tier + caches), write TPS (Kafka partitions + bulk + refresh control), corpus size
  (primary shards + hot/warm/cold). Scale the cheap stateless tiers first.
- **Replicas multiply read capacity** (and give HA); the **coordinating tier** absorbs
  fan-out/merge CPU and scales independently; the **Search Service is stateless** and
  the cheapest thing to scale.
- **Caching is the top scaling lever.** A layered cache (CDN → Redis Result Cache → ES
  shard request cache) at ~85% hit ratio roughly halves the cluster vs. 70%. Resolve
  the **TTL ↔ NRT tension** by caching the stable ranked-ID list and **hydrating
  volatile price/inventory** (≤1 s) on read.
- **Thundering herd** on celebrity queries is defended in layers: **request coalescing
  (single-flight) + soft TTL + jitter + negative caching + pre-warming.**
- **Writes** scale via **bulk indexing** (100K writes/sec = ~20 bulk req/s), **Kafka
  partition parallelism** (ordered per `listing_id`, buffers surges), and **refresh
  throttling** during heavy loads — always isolating indexing from the read SLA.
- **~600 primary shards @ ~40 GB** is the sweet spot; **over-sharding** is the common
  trap; **primary count is immutable** (reindex + alias swap to change it); **routing
  by locale/market** cuts fan-out but must be balanced against skew.
- **Hot/warm/cold + ILM** concentrates premium hardware on the hot working set,
  cutting cost ~50–70% while improving hot-path latency.
- **Multi-region** gives latency locality, HA, and data residency: **CCR** places data
  near buyers, **CCS** for rare global queries; the pragmatic model is **active-active
  reads, active-passive writes**, with GeoDNS routing to the nearest healthy cluster.
- **Hotspots** (skewed shards, popular sellers/categories, celebrity queries) are
  fought with bucketed routing, targeted replicas, hot-shard relocation, and front-
  line caching. Watch p99 and per-shard heat, not averages.
- **Pre-provision for peak** with a surge factor (≈2× normal peak + headroom); you
  **cannot autoscale stateful data nodes fast enough** for Black Friday. **Load-test**
  with realistic Zipfian traffic and cold-cache scenarios to find the p99 knee.
