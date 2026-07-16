# Chapter 2 — Capacity Estimation

In Chapter 1 we scoped **GlobalMart Search**: a hyperscale product-search system over
~10B listings, serving buyers who expect an instant, typo-tolerant search box. We fixed the
functional and non-functional requirements — search p99 ≤ 200 ms, autocomplete p99 ≤ 100 ms,
99.99% read availability, near-real-time indexing. This chapter turns those requirements into
*numbers you can provision against*: how many queries per second, how much disk, how many
shards, how many nodes, how much RAM, how much bandwidth, and how much write throughput.

Every number here derives from the canonical scale table in the design brief (§2). We do not
invent contradicting figures — we start from the fixed inputs (500M DAU, ~6 searches/user/day,
~10B docs, ~2 KB/doc, ~100K peak QPS, ~100K writes/sec) and show the arithmetic that turns
them into a provisioned cluster.

---

## 2.1 Why estimation matters (and how to present it)

Capacity estimation is one of the highest-signal segments of a system-design interview, and
it is routinely done badly. Candidates either skip it ("we'll autoscale") or drown in it,
computing storage to three significant figures while the interviewer waits for the *insight*.
The point of this exercise is not precision — it is to demonstrate three things:

1. **You can reason from a handful of top-line numbers to a physical footprint.** Given DAU
   and a per-user activity rate, you can land on QPS, disk, and node count in a couple of
   minutes.
2. **You know which resource is the binding constraint.** For GlobalMart the punchline is that
   the cluster is **QPS-bound, not storage-bound** — we buy nodes for query throughput and
   RAM, and the disk comes along for free. Surfacing that insight early is a senior signal.
3. **You state your assumptions out loud** so the interviewer can redirect you. "I'll assume a
   3× peak-to-average factor and a 2 KB average document" is worth more than a silent
   calculation, because the *method* is what is being graded.

### Presentation discipline

A few habits keep the estimation clean and fast:

- **Round aggressively.** 86,400 seconds/day becomes "~10⁵ s" or just "~86K". 3B / 86,400 =
  34,722 becomes "~35K". Nobody provisions for 34,722; they provision for 35K with headroom.
  Carrying extra digits costs time and buys nothing.
- **Work in powers of ten where you can.** A day is ~10⁵ seconds (86,400, close enough for a
  first pass). A year is ~3 × 10⁷ seconds. Memorize these two and most traffic math collapses
  to shifting exponents.
- **State the assumption, then the number.** "Assume 3× peak factor → ~100K peak QPS." The
  brief has already fixed 3× and 100K, so we reuse them verbatim; in a live interview you'd
  name the factor before applying it.
- **Separate average from peak.** Averages size your steady-state cost; peaks size the
  cluster you must survive. Confusing them under-provisions you into an outage.
- **Compute both the read path and the write path**, and note that they scale independently —
  reads fan out and are SLA-critical; writes are a firehose but tolerant of buffering.

With the method fixed, let's run the numbers.

---

## 2.2 Traffic estimation (QPS)

### From DAU to searches per day

The canonical inputs are 500M daily active users, each performing ~6 searches per day:

```
Searches/day = DAU × searches per user per day
             = 500,000,000 × 6
             = 3,000,000,000
             ≈ 3 B searches/day
```

This matches the brief's canonical figure of ~3B queries/day. Note we are counting *search
requests* here (the `GET /v1/search` calls), not autocomplete keystrokes — those are a
separate, much larger stream we handle below.

### From searches per day to average QPS

Spread 3B searches across the 86,400 seconds in a day:

```
Avg QPS = 3,000,000,000 / 86,400
        = 34,722
        ≈ 35 K QPS
```

So the steady-state, all-hours-averaged load is **~35K search QPS**. This is the number you'd
use to reason about total daily compute and cost — but never the number you provision peak
capacity against, because traffic is not flat.

### From average to peak QPS

Real e-commerce traffic is spiky: it follows daytime/evening curves per region, spikes on
paydays, weekends, flash sales, and catastrophically during events like Black Friday or
Singles' Day. The brief fixes a **3× peak-to-average factor** for the ordinary daily/weekly
peak:

```
Peak QPS = Avg QPS × 3
         = 35,000 × 3
         = 105,000
         ≈ 100 K QPS
```

**~100K search QPS at peak** is the headline number the read path must survive. (Mega-events
like Black Friday can push well beyond 3×; we treat those as a burst-scaling scenario in
Chapter 9 rather than baking a 10× factor into the always-on cluster, because provisioning
for the once-a-year peak 24/7 is enormously wasteful.)

### The autocomplete keystroke multiplier

Search QPS is only half the read story. Every character a user types into the search box can
fire an autocomplete request. The brief fixes **~5–8 keystrokes per completed search**. Taking
the upper end and applying it to the peak search stream:

```
Autocomplete peak QPS ≈ Peak search QPS × keystrokes per search
                      ≈ 100,000 × 5
                      ≈ 500 K QPS
```

So the **autocomplete path peaks at ~500K QPS — roughly 5× the search path.** This is exactly
why the brief calls autocomplete "a lighter path." We do not send half a million QPS into the
full scatter-gather Elasticsearch query engine. Autocomplete is served by a purpose-built,
ultra-low-latency path (a dedicated completion-suggester / edge-ngram index, or an in-memory
prefix structure like a ternary/FST-backed cache), with a tight p99 ≤ 100 ms budget, tiny
responses (≤10 suggestions), and aggressive caching of popular prefixes. Chapter 7 details
that design; here the capacity takeaway is that **autocomplete, not search, is the highest-QPS
surface in the system, and it must be architected as a separate tier.**

| Traffic dimension | Average | Peak | Multiplier applied |
|---|---:|---:|---|
| Search requests | ~35K QPS | ~100K QPS | 3× peak factor |
| Autocomplete requests | ~165K QPS | ~500K QPS | 5× keystrokes on top of search |
| **Combined read peak** | — | **~600K QPS** | search + autocomplete |

Two independent read tiers, sized separately, is the correct mental model.

---

## 2.3 Storage estimation

### Raw corpus

The corpus is ~10B listing documents at ~2 KB average each:

```
Raw corpus = docs × avg doc size
           = 10,000,000,000 × 2 KB
           = 20,000,000,000 KB
           = 20,000,000 MB
           = 20,000 GB
           = 20 TB
```

**~20 TB of raw JSON** — matching the brief. That 2 KB average covers the analyzed text
fields (title, description), the keyword fields (brand, category path, attributes), numeric
signals (price, rating, popularity, inventory), and metadata (seller, locale, timestamps,
`_version`) from the canonical data model in §4.

### Index overhead: from raw to primaries

Elasticsearch does not store raw JSON one-for-one. It builds several structures on top of the
source, and their combined size is *larger* than the raw text. The brief fixes the multiplier
at **~1.2× raw** for the primaries:

```
Primary index size = raw × 1.2
                    = 20 TB × 1.2
                    = 24 TB
```

**~24 TB of primary index.** Where does that 1.2× come from? Qualitatively, an ES index for a
document like ours is the sum of several components:

- **Inverted index** (the postings lists): for every analyzed field, a term dictionary plus
  postings (doc IDs, term frequencies, positions). This is the core of full-text search and a
  major contributor. Text-heavy fields (title, description) dominate here.
- **`doc_values`** (columnar, on-disk): required for sorting, aggregations, and faceting —
  and GlobalMart facets heavily (category, brand, price, rating, seller). Every numeric,
  keyword, and date field we sort or facet on gets a `doc_values` column. This is a large,
  often underestimated contributor.
- **Stored fields / `_source`**: the original JSON, kept so we can return and re-index
  documents. This is roughly the raw size itself, compressed (LZ4/DEFLATE), so it lands
  *below* 1× per field but is still real.
- **Norms, term vectors, and completion structures**: field-length norms for scoring, plus
  the autocomplete completion/edge-ngram structures. Smaller, but non-zero.

The reason the *net* multiplier is only ~1.2× rather than 2–3× is that `_source` compresses
well and we are disciplined about what we make searchable, sortable, and stored (e.g.
`image_url` is stored but **not** indexed for search, per §4). If we naively enabled term
vectors everywhere and indexed every attribute in multiple analyzers, this number would
balloon — a real tuning lever discussed in Chapters 4 and 8.

### Replication: from primaries to total disk

The brief specifies **1 replica** (one copy of every shard). Replicas exist for high
availability (survive a node loss) and for read throughput (queries can hit replicas), both of
which we need for a 99.99% revenue-critical read path:

```
Total on-disk = primaries × (1 + replicas)
              = 24 TB × (1 + 1)
              = 48 TB
```

**~48 TB total on disk across the cluster.** This is the number that sizes raw storage
purchasing — and, as we'll see, it is *not* the number that determines node count.

| Storage layer | Size | Derivation |
|---|---:|---|
| Raw JSON corpus | ~20 TB | 10B × 2 KB |
| Primary index | ~24 TB | 20 TB × 1.2 overhead |
| Total (1 replica) | ~48 TB | 24 TB × 2 |

---

## 2.4 Shard sizing math

A shard is the unit of scale, distribution, and recovery in Elasticsearch — each shard is a
self-contained Lucene index. We must choose how many primary shards to split those ~24 TB
into. The governing heuristic, which the brief fixes as the target, is **~40 GB per shard**:

```
Primary shards = primary index size / target shard size
               = 24 TB / 40 GB
               = 24,576 GB / 40 GB
               = 614
               ≈ 600 primary shards
```

With 1 replica, total shard count doubles:

```
Total shards = primaries × (1 + replicas)
             = 600 × 2
             = 1,200 shards
```

**~600 primary shards, ~1,200 total** — exactly the brief's topology (§6).

### Why 20–50 GB is the sweet spot

The 40 GB target sits inside the widely used **20–50 GB per shard** band. That band exists
because both extremes are costly:

**Too many small shards** (say, thousands of 5 GB shards):

- Each shard carries fixed overhead — open file handles, Lucene segment memory, and an entry
  in the cluster state that the master node must track and gossip. Thousands of shards bloat
  the cluster state and slow master operations.
- A search request fans out to *every* shard of the target index (scatter), then the
  coordinating node merges (gather). More shards means more fan-out messages, more thread-pool
  contention, and a slower gather — directly hurting our 120 ms ES query budget.
- The old rule of thumb — keep shard count roughly proportional to heap, targeting **~20
  shards per GB of heap** per node — is blown quickly if shards are tiny.

**Too few huge shards** (say, 200 GB each):

- A shard cannot be split cheaply once created, and it is the unit of rebalancing and
  recovery. A 200 GB shard takes far longer to relocate or rebuild when a node dies,
  lengthening the window of degraded redundancy.
- Huge shards concentrate load unevenly and cap parallelism — a single query against one giant
  shard can't be spread across cores the way many mid-sized shards can.

40 GB balances these: fan-out stays manageable (~600 primaries), recovery of any one shard is
minutes not hours, and per-shard overhead stays proportionate. It also leaves room for shards
to grow with the catalog before we must reindex into a higher shard count.

---

## 2.5 Node count — the QPS-bound insight

Here is the crux of the whole chapter. There are two ways to size the number of data nodes,
and they give very different answers.

### The naive (storage-bound) calculation

If disk were the constraint, we'd divide total data by per-node hot-data capacity. The brief
budgets **~2 TB of hot data per node**:

```
Nodes (storage-bound) = total on-disk / per-node hot capacity
                      = 48 TB / 2 TB
                      = 24 nodes
```

So *if all we cared about was fitting the bytes*, ~24 data nodes would do it. But that cluster
would fall over instantly at 100K search QPS — because disk capacity says nothing about how
many concurrent queries a node can serve.

### The real (QPS-bound) calculation

What actually limits a data node under our load is the combination of:

- **RAM for the filesystem cache.** Elasticsearch queries hit Lucene segments on disk; the OS
  filesystem cache is what keeps hot postings and `doc_values` in memory. Query latency lives
  and dies by cache hit rate. A node can only cache a fraction of its 2 TB, so we want the
  *working set* resident, and we want more nodes so each holds a smaller, hotter slice.
- **Query concurrency and CPU.** Each search fans out to hundreds of shards; each shard search
  consumes a search-thread-pool slot and CPU for scoring, aggregation (facet counts), and
  merge. At 100K QPS with heavy faceting, CPU and thread-pool depth — not gigabytes — are what
  saturate first.

Working backward from throughput: suppose a well-tuned data node sustains on the order of
**~1–2K search QPS** at our latency target (a reasonable planning figure for facet-heavy
e-commerce queries; the exact number is validated by load testing, Chapter 9). Then:

```
Nodes (QPS-bound) ≈ peak search QPS / per-node sustainable QPS
                  ≈ 100,000 / ~1.25K
                  ≈ 80 data nodes
```

That lands squarely in the brief's **~60–100 data node** range. Add replica-driven read
parallelism and headroom for node-failure absorption, and 60–100 is the provisioning band.

**The one-liner to say in the interview: "GlobalMart is QPS-bound, not storage-bound — 48 TB
would fit on ~24 nodes for disk, but sustaining 100K QPS within our latency budget needs
~60–100 nodes for RAM/filesystem-cache and query concurrency. So we provision for query
throughput, and the disk comes along for free."** That single sentence demonstrates you
understand what actually breaks at scale.

A useful consequence: because nodes are added for QPS, each node ends up holding well under
its 2 TB disk ceiling (48 TB / 80 nodes ≈ 0.6 TB/node), which is *good* — it means more of
each node's data set fits in filesystem cache, reinforcing latency. Over-provisioning disk
per node to "save money on servers" would be a false economy here.

---

## 2.6 Memory / RAM

Node memory splits into two consumers: the **JVM heap** and the **OS filesystem cache**.

### The 31 GB heap ceiling (compressed oops)

Elasticsearch/Lucene run on the JVM. There is a hard rule: **keep the heap at or below ~31 GB
(sometimes quoted as ~32 GB / ~30.5 GB depending on JVM).** Below this threshold the JVM uses
**compressed ordinary object pointers (compressed oops)** — 32-bit object references that
still address a ~32 GB heap. Cross the threshold and pointers balloon to 64 bits, so you
actually get *less usable heap* per byte and worse cache behavior — a heap of 40 GB can hold
fewer objects than a heap of 31 GB. So:

```
JVM heap per data node ≤ ~31 GB   (and no more, ever)
```

### Leave the rest for the filesystem cache

The second, larger rule: **give the JVM no more than ~50% of a node's RAM, and leave the rest
to the OS filesystem cache.** Lucene is designed to lean on the OS page cache for segment
data; that cache is what makes searches fast. A sensible data-node shape:

```
Node RAM        = 64 GB
JVM heap        = 31 GB   (capped for compressed oops)
Filesystem cache≈ 33 GB   (remaining RAM, for hot Lucene segments)
```

64 GB nodes with a 31 GB heap is the canonical hot-tier data-node profile.

### Sizing the cache working set (the 80/20 rule)

We can't cache all 48 TB in RAM — nor do we need to. E-commerce search follows a sharp
**power law**: a small fraction of listings (popular products, trending categories, hot
locales) absorb the overwhelming majority of queries. Applying the **80/20 rule** as a
planning approximation — ~20% of the corpus serves ~80% of the traffic — the hot working set is:

```
Hot working set ≈ 20% × primary index
                ≈ 0.20 × 24 TB
                ≈ 4.8 TB
```

Now compare that to aggregate cluster filesystem cache:

```
Aggregate FS cache ≈ nodes × per-node cache
                   ≈ 80 × 33 GB
                   ≈ 2,640 GB
                   ≈ 2.6 TB
```

So we can hold roughly half the hot working set resident in RAM across the cluster, and the
rest is served from a still-fast SSD tier plus the Result Cache (Redis) in front of ES for
whole-query hits. This is why the brief separates **data-hot** and **data-warm** tiers (§6):
hot nodes (more RAM per byte of data) carry popular locales and recent listings; warm nodes
carry the long tail at higher disk-to-RAM ratios and looser latency. That tiering is what
makes the RAM budget affordable — we don't pay for hot-tier RAM on cold data. Details in
Chapters 8 and 9; here the takeaway is that **RAM, sized against the hot working set, is a
first-class capacity dimension — arguably the one that drives node count.**

---

## 2.7 Bandwidth

Bandwidth follows directly from QPS × payload size. We assume modest per-request sizes
consistent with the API in §3 (`size` ≤ 100 results, facets included).

**Assumptions:** average search request ~1 KB (query string, filters, headers); average
search response ~20 KB (a page of results with titles, prices, thumbnails-as-URLs, plus facet
counts). Autocomplete request ~0.5 KB; autocomplete response ~2 KB (≤10 short suggestions).

### Search path at peak (100K QPS)

```
Ingress = 100,000 QPS × 1 KB   = 100,000 KB/s ≈ 100 MB/s ≈ 0.8 Gbps
Egress  = 100,000 QPS × 20 KB  = 2,000,000 KB/s ≈ 2 GB/s ≈ 16 Gbps
```

### Autocomplete path at peak (500K QPS)

```
Ingress = 500,000 QPS × 0.5 KB = 250,000 KB/s ≈ 250 MB/s ≈ 2 Gbps
Egress  = 500,000 QPS × 2 KB   = 1,000,000 KB/s ≈ 1 GB/s ≈ 8 Gbps
```

| Path | Ingress (peak) | Egress (peak) |
|---|---:|---:|
| Search (100K QPS) | ~0.8 Gbps | ~16 Gbps |
| Autocomplete (500K QPS) | ~2 Gbps | ~8 Gbps |
| **Combined** | **~3 Gbps** | **~24 Gbps** |

Egress dominates (responses are far larger than requests) and search egress is the single
biggest line at ~16 Gbps. ~24 Gbps aggregate egress is well within a datacenter/cloud LB
budget, but it explains three architectural choices: (1) **CDN/edge caching** of common
query+facet responses offloads a large slice of that egress before it reaches origin; (2)
**response compression** (gzip/br) cuts egress several-fold on JSON; (3) keeping result
payloads lean (return `listing_id` + display fields, not the whole `_source`) matters at this
scale. Internal scatter-gather bandwidth *inside* the ES cluster is a separate, larger figure
(each query fans to ~600 shards) and is why coordinating nodes and data nodes sit on
high-bandwidth intra-cluster networking — covered in Chapter 9.

---

## 2.8 Write / indexing throughput

Reads are the SLA-critical path, but writes are the higher-*volume* op stream — the brief
notes a read:write ratio of ~1:3 by operations. Two canonical write figures drive sizing.

### Peak mutation rate

```
Peak writes = 100,000 writes/sec
```

The brief notes these are **mostly price/inventory mutations** — sellers repricing, stock
counts changing — not full new documents. Each is a small partial update keyed by `listing_id`
+ `_version` for idempotency (§3). At 100K writes/sec, synchronous per-document indexing would
melt the cluster, so the write path is fundamentally different from the read path:

- Writes arrive via **CDC → Kafka → Indexing Service → ES `_bulk`** (§5), never as synchronous
  calls. Kafka absorbs bursts and decouples producer spikes from ES ingest capacity.
- The Indexing Service **batches** mutations into bulk requests (e.g. thousands of docs per
  `_bulk` call) — bulk indexing is dramatically more efficient than single-doc writes because
  it amortizes network round-trips, translog fsyncs, and segment work.

### Daily new-listing volume

Distinct from mutations, brand-new listings are created at:

```
New listings/day = 50,000,000
New listings/sec (avg) = 50,000,000 / 86,400 ≈ 580/sec
```

So the *creation* stream averages only ~580 docs/sec — trivial compared to the 100K/sec
mutation firehose. The capacity implication: the write path is dominated by **high-frequency
small updates**, which is what makes NRT (near-real-time) freshness both necessary and
achievable. It also motivates tuning the **refresh interval** (§6): 1s on the hot index for
freshness, but batched/higher during bulk backfills to avoid segment-churn overhead.

### Bulk capacity needed

To sustain 100K writes/sec with, say, 5,000 docs per bulk request:

```
Bulk requests/sec = 100,000 / 5,000 = 20 bulk calls/sec
```

Twenty fat bulk calls per second is very comfortable for an 80-node cluster — indexing load is
spread across primary shards (each of ~600 primaries takes ~1/600th of the write stream, ~170
writes/sec/shard). This confirms writes are not the binding constraint; the read path is.
Write amplification from replication (each write also applied to its replica) doubles internal
write work but is absorbed by the same node fleet. Chapter 6 details the pipeline, versioning,
and back-pressure handling.

---

## 2.9 Growth projection and headroom

Capacity is a moving target. Sizing only for today guarantees a painful re-architecture next
year. We project over a **2–3 year horizon**, assuming e-commerce-typical growth of roughly
**~1.5–2× every ~18 months** in both corpus and traffic (a planning assumption, stated
explicitly — actual growth is validated against real trend data).

Taking ~2× over ~2 years as the planning multiplier:

| Dimension | Today | ~2-year projection (×2) |
|---|---:|---:|
| Documents | 10B | ~20B |
| Raw corpus | ~20 TB | ~40 TB |
| Primary index | ~24 TB | ~48 TB |
| Total on-disk (1 replica) | ~48 TB | ~96 TB |
| Peak search QPS | ~100K | ~200K |
| Peak autocomplete QPS | ~500K | ~1M |
| Primary shards (@40 GB) | ~600 | ~1,200 |
| Data nodes (QPS-bound) | ~60–100 | ~120–200 |

Practical headroom rules baked into the provisioning:

- **Run the cluster at ~50–70% of capacity, not 100%.** Headroom absorbs traffic spikes,
  node failures (a lost node's shards must be re-hosted by survivors), and rolling restarts
  during upgrades. Provisioning to the peak leaves nothing for the *unexpected* peak.
- **Choose the shard count with growth in mind.** Because primaries can't be increased without
  a reindex, ~600 shards at 40 GB today have room to grow toward ~50 GB before we reindex into
  the `products-vN+1` index behind the `products` alias (§6) — a planned, zero-downtime event,
  not an emergency.
- **Scale nodes horizontally.** Doubling QPS roughly doubles data nodes; the QPS-bound design
  means capacity planning stays a linear, predictable exercise.

### Summary: the provisioned cluster

| Resource | Provisioned value | Basis |
|---|---:|---|
| Peak search QPS | ~100K | 35K avg × 3 |
| Peak autocomplete QPS | ~500K | 100K × ~5 keystrokes (separate tier) |
| Raw corpus | ~20 TB | 10B × 2 KB |
| Primary index | ~24 TB | ×1.2 overhead |
| Total on-disk | ~48 TB | ×2 for 1 replica |
| Primary shards | ~600 | 24 TB / 40 GB |
| Total shards | ~1,200 | ×2 replica |
| Data nodes | ~60–100 | **QPS-bound**, ~80 typical |
| Node profile | 64 GB RAM / 31 GB heap | compressed oops + FS cache |
| Dedicated masters | 3 | quorum, no split-brain |
| Peak egress | ~24 Gbps | search + autocomplete responses |
| Peak write rate | ~100K/sec | via Kafka + `_bulk`, ~20 bulk calls/sec |

---

## Interview Tips

- **Lead with the method, not the digits.** Say "day ≈ 10⁵ seconds, so 3B/day ≈ 35K QPS,
  times 3 for peak ≈ 100K" — fast, rounded, correct. Don't compute 34,722 and stall.
- **State every assumption before you use it** (peak factor, doc size, keystrokes/search,
  per-node QPS). The interviewer grades the reasoning; a stated assumption invites them to
  redirect you rather than mark you wrong.
- **The single strongest signal in this chapter is "QPS-bound, not storage-bound."** Show the
  storage math giving ~24 nodes, then the QPS math giving ~80, and explain why the larger
  number wins. Most candidates size on disk and stop — surfacing that RAM/filesystem-cache and
  query concurrency dominate marks you as senior.
- **Don't forget autocomplete.** Many candidates size only the search path and miss that
  keystroke traffic is ~5× larger — and that it demands a *separate, lighter* tier. Naming
  that trade-off is a differentiator.
- **Separate the read and write stories.** Reads are latency/SLA-critical and fan out; writes
  are a high-volume but bufferable firehose absorbed by Kafka + bulk indexing. Sizing them
  together is a classic mistake.
- **Always add headroom.** If asked "how many nodes," answer with a range and mention running
  at 50–70% utilization for failures and spikes — provisioning to 100% of peak is a red flag.
- **Know the two magic constants:** ~31 GB heap ceiling (compressed oops) and 20–50 GB/shard.
  They come up in almost every ES capacity discussion.

## Key Takeaways

- **Traffic:** 500M DAU × 6 searches = 3B/day → ~35K avg QPS → **~100K peak** (3×). Autocomplete
  keystrokes push a **separate ~500K QPS** low-latency tier.
- **Storage:** 10B × 2 KB = **~20 TB raw** → ×1.2 = **~24 TB primaries** → ×2 replica =
  **~48 TB total**. The 1.2× is inverted index + `doc_values` + compressed `_source`.
- **Shards:** target ~40 GB → **~600 primaries, ~1,200 total**. 20–50 GB is the sweet spot;
  too many bloats cluster state and fan-out, too few slows recovery and caps parallelism.
- **Nodes:** **~60–100 data nodes, QPS-bound.** Disk alone fits on ~24; RAM/filesystem-cache
  and query concurrency at 100K QPS drive the count up to ~80. Provision for throughput; disk
  comes free.
- **RAM:** 64 GB nodes, **≤31 GB heap** (compressed oops), rest to OS filesystem cache; size
  cache against the **~20% hot working set** (~4.8 TB) via hot/warm tiering + Redis result cache.
- **Bandwidth:** ~24 Gbps peak egress (response-dominated) — motivates CDN caching,
  compression, and lean payloads.
- **Writes:** ~100K/sec peak (mostly price/inventory), 50M new listings/day (~580/sec) — via
  CDC → Kafka → `_bulk`, ~20 bulk calls/sec, comfortably absorbed. Reads, not writes, bind.
- **Growth:** plan ~2× over 2 years, run at 50–70% utilization, and pick shard counts that
  reindex gracefully behind the `products` alias.
