# Chapter 6 — Indexing Pipeline

> **Where we are.** Chapter 5 drew the two halves of GlobalMart: the *read* path (Client → CDN/Edge → API Gateway → Search Service → Elasticsearch) and the *write* path (Catalog Service → CDC → Kafka → Indexing Service → ES). Chapters 1–4 fixed the requirements, the capacity math, the API contracts, and the document schema. This chapter zooms all the way into the write path and answers one deceptively hard question: **how does a price change made by a seller in a SQL transaction become a searchable fact in a 10-billion-document Elasticsearch cluster in about one second, correctly, at 100K writes/sec, without ever taking the index down?**

This is a flagship chapter. We will build the pipeline component by component, and at every stage the recurring tension is the same: **freshness vs. correctness vs. throughput**. You can have any two cheaply; getting all three is the engineering.

---

## 6.1 The Goal, Stated Precisely

Let's restate the target from the brief in operational terms, because the whole design falls out of these numbers.

| Requirement | Value | Source |
|---|---|---|
| Peak catalog write rate | **~100K writes/sec** | brief §2 |
| New listings / day | **~50M** | brief §2 |
| Price / inventory freshness | **searchable in ~1s** (p99 lag ≤ 1s) | FR8 |
| New/updated listing freshness | searchable in **seconds** | FR8 |
| Consistency model | **eventual consistency acceptable** on the index (AP over CP) | NFR6 |
| Source of truth | **Catalog Service store**; ES is derived & rebuildable | brief §4 |

Two facts shape everything:

1. **The write mix is dominated by mutations, not creations.** 50M new listings/day is only ~580 creates/sec average. The 100K/sec peak is overwhelmingly **price and inventory updates** on *existing* documents. That asymmetry is the single most important design driver in this chapter — it justifies a dedicated *fast path* for partial updates (§6.7) and makes update-amplification (§6.7) the thing that will actually page you at 3 a.m.

2. **ES is derived state.** The Catalog Service's transactional store (sharded SQL / a NoSQL document store, per brief §4) is authoritative. Elasticsearch is a rebuildable read-optimized projection. This gives us enormous freedom: we can drop the index, rebuild it, run two copies in parallel, and reconcile against the source — because the source can always regenerate the truth. Internalize this: **the indexing pipeline is a stream-processing projection, not a database of record.**

A quick sanity check on the freshness budget. "~1s searchable" is an *end-to-end* budget spanning: commit in Catalog → CDC capture → Kafka → Indexing Service transform → ES `_bulk` → ES `refresh` makes it visible. Elasticsearch's default `refresh_interval` is 1s, and refresh alone can eat most of the budget. So the ~1s target is aggressive and forces specific choices (small bulk flush intervals, a tuned refresh, sometimes an explicit refresh on the fast path). We'll carry this budget through the chapter.

---

## 6.2 Capturing Changes: Outbox + CDC, and Why Not Dual-Write

The pipeline begins the instant a seller changes something. The question is how the search system *learns* about that change. There are three candidate mechanisms; only two are acceptable, and we combine them.

### The naive option: dual-write (rejected)

The obvious thing is to have the Catalog Service write to its database **and** publish to Kafka in the same request handler:

```
// ANTI-PATTERN — do not do this
db.commit(listing);            // (1) succeeds
kafka.publish(listingChange);  // (2) ...then the process crashes
```

This is the classic **dual-write problem**. The two writes are not in one atomic unit. Any of these failure interleavings silently corrupts the index:

- (1) commits, process dies before (2) → **DB has the new price, ES never hears about it.** Lost update. The index is now permanently stale until something else touches that listing.
- (2) publishes, (1) rolls back → **ES gets a price that was never committed.** Phantom data.
- Both succeed but in different orders across concurrent requests → **ordering is not guaranteed**, so a stale price can land after a fresh one.

There is no retry policy that fixes this, because the failure is *between* two non-transactional systems. You cannot make a database commit and a Kafka publish atomic without a distributed transaction (2PC), which is operationally toxic at 100K/sec and couples the Catalog Service's availability to Kafka's. **Reject dual-write outright** — and be ready to explain *why* in an interview, because it's a favorite trap.

### The transactional outbox pattern

The fix is to make the "intent to publish" part of the **same local transaction** as the business write. The Catalog Service writes the row *and* an event row into an `outbox` table atomically:

```sql
BEGIN;
  UPDATE listings
     SET price = 749.00, updated_at = now(), version = version + 1
   WHERE listing_id = 'L-42';

  INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at)
  VALUES (gen_id(), 'L-42', 'ListingPriceChanged',
          '{"listing_id":"L-42","price":{"amount":749.00,"currency":"USD"},"version":88}',
          now());
COMMIT;
```

Because both statements share one transaction, they commit or roll back together. There is no interleaving where the DB and the outbox disagree. The outbox is now a durable, ordered, per-row log of "things that happened," living inside the source of truth.

### Change Data Capture (Debezium-style)

A separate process now needs to move outbox rows into Kafka. We do **not** poll the outbox table with `SELECT ... WHERE published = false` (that adds load and lag). Instead we use **Change Data Capture**: Debezium tails the database's write-ahead log (the Postgres WAL / MySQL binlog), turning committed row changes into a stream of change events. Point the CDC connector at the `outbox` table (Debezium ships an *Outbox Event Router* SMT for exactly this) and every committed outbox row becomes a Kafka record — routed to the right topic, keyed by `aggregate_id` (the `product_id`/`listing_id`), with the payload unwrapped.

```
┌──────────────────── Catalog Service (source of truth) ────────────────────┐
│                                                                            │
│   handler ──BEGIN──▶ UPDATE listings                                       │
│                └────▶ INSERT outbox   ──COMMIT──▶  WAL / binlog            │
│                                                       │                    │
└───────────────────────────────────────────────────────┼──────────────────┘
                                                          │  tail the log
                                                   ┌──────▼───────┐
                                                   │  CDC         │  (Debezium)
                                                   │  connector   │  outbox SMT → unwrap + route
                                                   └──────┬───────┘
                                                          │ produce
                                                   ┌──────▼───────┐
                                                   │    Kafka     │
                                                   └──────────────┘
```

**Why CDC beats table-polling:** it reads the log the DB already writes for durability, so it's low-overhead and low-latency (millisecond capture lag), it never misses a change, and it preserves **commit order**. Debezium also gives you a resumable offset (the log position it last read), so a connector restart replays from exactly where it stopped — no gaps, no duplicates beyond the at-least-once guarantee we handle later.

### Ordering guarantees at the source

CDC delivers change events **in commit order per database / per log**. Combined with the outbox, this means: for a *single listing*, the sequence of events (price→749, then price→759) reaches Kafka in the order they committed. This is the foundation of the ordering story, but it is not yet sufficient — Kafka can reorder across partitions, and consumers can process out of order. §6.3 pins ordering to `product_id`, and §6.6 adds a version-based backstop so that even if ordering is violated end-to-end, we never overwrite fresh data with stale.

**Do we even need the outbox if CDC can tail the `listings` table directly?** You can CDC the base table and skip the outbox. The outbox buys you two things: (1) a clean, *purpose-shaped* event (you emit exactly the fields search cares about, not raw column diffs), and (2) decoupling from schema churn in the base table. At GlobalMart scale we keep the outbox for the listings we control, and note that either is defensible in an interview — the non-negotiable is **no dual-write**.

---

## 6.3 Kafka: The Backbone

Kafka is the durable, replayable buffer between a bursty producer (the Catalog Service, spiking to 100K/sec) and a consumer (the Indexing Service + Elasticsearch) whose throughput we want to keep *smooth*. It is also the thing that lets us **replay history** to rebuild the index (§6.9) and **quarantine poison messages** (§6.10). Getting its topology right is most of the battle.

### Topic design

We use a small number of topics, split by *what changes and how urgently*, because the fast path (§6.7) wants different handling from full-document upserts:

| Topic | Contents | Partitions | Retention |
|---|---|---|---|
| `catalog.listing.upsert` | full-document creates/updates (new listing, description edit, attribute change) | 200 | 7 days |
| `catalog.listing.price-inventory` | price & stock mutations (the high-volume fast path) | 300 | 3 days |
| `catalog.listing.delete` | listing/seller removals (tombstones, §6.8) | 50 | 30 days |
| `catalog.signal.popularity` | derived ranking signals (popularity, seller_rating) from analytics | 100 | 3 days |

Splitting price/inventory from full upserts lets us tune consumers independently (the fast path uses ES partial updates and a tighter flush) and lets us shed or slow the heavy full-reindex traffic without delaying a price drop. Retention of ≥3 days is a deliberate safety margin: it's our replay window if the Indexing Service or ES has a bad day.

### Partitioning by `product_id` — the ordering keystone

**All topics are partitioned by `product_id`.** (Not `listing_id`. More on that choice below.) Kafka guarantees ordering *within a partition*, and it routes a record to a partition by `hash(key) % partition_count`. So if every event for product `P-77` carries key `P-77`, all its events land on the same partition and are consumed **in order** by exactly one consumer thread.

```
key = "P-77"  ──hash──▶  partition 12  ──▶  [price→749][price→759][stock→0]   (strict order)
key = "P-31"  ──hash──▶  partition 4   ──▶  [create][attr edit]
```

This is what actually delivers "updates to the same product are ordered" end-to-end. Without it, `price→749` and `price→759` could be processed concurrently by two workers and applied in the wrong order — the exact "old price overwrites new price" race we harden against in §6.6.

**Why key by `product_id` and not `listing_id`?** A single product has many listings (product × seller × locale ⇒ up to thousands of documents). Many mutations are *product-level* (a shared popularity signal, a brand rename) and fan out to all of that product's listings — the update-amplification problem in §6.7. Keying by `product_id` keeps all those correlated writes on one partition, so a single consumer owns the whole fan-out and can batch/dedupe it and reason about ordering across the sibling listings. The cost is **partition skew**: a mega-product (a viral phone listed by 50K sellers) becomes a hot partition. We accept it and mitigate with per-key batching; if a single product's write rate ever exceeds one consumer's throughput, we'd sub-key by `product_id + bucket(seller_id)` — but that's a last resort because it sacrifices cross-listing ordering.

### Sizing the partition count

Partition count sets the ceiling on consumer parallelism: **max useful consumers in a group = partition count.** Size it from throughput and headroom, not vibes.

- Peak write rate on the hot path ≈ 100K/sec. Suppose a single partition-consumer, doing transform + enrich + contributing to an ES bulk batch, sustains ~500–1,000 msgs/sec comfortably.
- 100,000 / 750 ≈ **~135 consumers needed at peak.** Round up and add headroom for skew and future growth ⇒ **300 partitions** on `catalog.listing.price-inventory`.

That headroom matters because you **cannot easily decrease** partition count, and *increasing* it later rehashes keys to different partitions, which **breaks per-key ordering across the change** (old key→partition mapping ≠ new one). So over-provision partitions up front. The trade-off against going absurdly high: each partition costs broker file handles, memory, and replication overhead, and more partitions means more, smaller batches (worse compression/throughput) and longer leader-election storms on failure. 300 is a deliberate middle.

### Consumer groups and delivery semantics

The Indexing Service runs as a **consumer group**: Kafka assigns each partition to exactly one consumer in the group, and rebalances automatically when a consumer joins/dies. Scale out by adding consumers (up to the partition count); Kafka handles the assignment.

We run **at-least-once delivery**: a consumer commits its offset *after* it has successfully written the batch to Elasticsearch, not before.

```
poll(records) → transform → enrich → bulk-index to ES → (200 OK) → commit offsets
```

If the consumer crashes after indexing but before committing, on restart it re-reads and re-indexes the same records → **duplicate delivery**. That's fine, and it's *why* the whole downstream must be **idempotent** (§6.6): re-applying the same versioned update is a no-op. We explicitly do *not* attempt exactly-once (Kafka transactions across the ES sink); it's expensive and unnecessary when the sink is idempotent by design. **At-least-once + idempotent writes = effectively-once**, which is the standard senior answer.

---

## 6.4 The Indexing Service: Transform, Enrich, Normalize, Dedupe

The Indexing Service is the stream processor that turns raw change events into Elasticsearch documents. It's the "business logic" of the write path. Four responsibilities, in order:

### 1. Transform (DB row → ES doc)

The event payload is shaped like the source row. ES wants the document from brief §4. Transformation maps and reshapes:

- Rename/restructure: source `stock_qty` → `inventory`; source `price_cents=74900` → `price: {amount: 749.00, currency: "USD"}`.
- Build derived structures: assemble `category_path: ["Electronics","Phones","Smartphones"]` from category IDs; flatten `attributes` into the nested/keyword shape.
- Type coercion & validation: dates to ISO-8601, enforce `size ≤ ~2KB` (brief §2 avg doc size), drop unknown fields.

```jsonc
// change event in                    →   ES doc out (subset)
{ "listing_id":"L-42",                    { "listing_id":"L-42",
  "product_id":"P-77",                      "product_id":"P-77",
  "price_cents":74900,                      "price":{"amount":749.00,"currency":"USD"},
  "stock_qty":42,                           "inventory":42, "in_stock":true,
  "cat_ids":[10,44,181],                    "category_path":["Electronics","Phones","Smartphones"],
  "version":88 }                            "_version":88 }
```

### 2. Enrich (join in signals the event doesn't carry)

A price-change event does **not** contain `seller_rating` or `popularity` — those live in other systems and arrive on their own topics (`catalog.signal.popularity`) or in a cache. Ranking quality (NFR5) depends on the ES doc carrying these blended signals (FR7), so the Indexing Service **joins them in** at index time:

- **`seller_rating`, `popularity`, `review_count`**: looked up from a low-latency store (Redis / a local RocksDB state store fed by the signal topics). We denormalize the signal *into* the document because ES has no join at query time — the search path (Ch7) must find these fields already on the doc to rank on them.
- This is a **stream-table join**: the high-volume change stream joined against slowly-changing signal tables held in local state. Keying both by `product_id` (§6.3) makes the join local — the consumer that owns `P-77`'s listings also holds `P-77`'s popularity in its state store, so no remote fan-out per event.

Enrichment is where **update-amplification** is born: when `popularity` for `P-77` changes, *every* listing of `P-77` must be rewritten to carry the new value. Held here so we can attack it head-on in §6.7.

### 3. Normalize

Make documents consistent and search-friendly, matching the analyzers/mappings from Ch4: normalize currencies, trim/case brand strings for the `brand.keyword` facet, canonicalize locale/market codes (`en_us` → `en-US`), clamp out-of-range signals (`popularity ∈ [0,1]`). Normalization keeps facets clean (FR3) — messy brand casing shows up as duplicate facet buckets in the UI.

### 4. Deduplicate

Because delivery is at-least-once and the Catalog Service can emit redundant events, we dedupe **within a micro-batch** before hitting ES:

- Collapse multiple events for the same `listing_id` in one poll into a **single write carrying the highest version**. If a batch has `price→749 (v87)` and `price→759 (v88)` for `L-42`, we send only `v88`. This shrinks bulk payloads and sidesteps intra-batch ordering entirely.
- The cross-batch / cross-consumer dedupe (the stale-overwrite race) is handled by ES external versioning in §6.6 — dedupe here is a throughput optimization; versioning is the correctness guarantee. Don't conflate them.

---

## 6.5 Writing to Elasticsearch: `_bulk`, Batching, Backpressure

Now we get the transformed, enriched documents into ES efficiently. Indexing one document per HTTP request at 100K/sec would melt the cluster — per-request overhead dominates. Everything here is about **amortizing that overhead** while respecting ES's limits.

### The `_bulk` API

We batch many operations into one `_bulk` request. Each op is two lines: an action/metadata line and (for index/update) a source line:

```
POST /products/_bulk
{ "index": { "_id": "L-42", "version": 88, "version_type": "external" } }
{ "listing_id":"L-42","product_id":"P-77","price":{"amount":749.00,"currency":"USD"},"inventory":42, ... }
{ "update": { "_id": "L-91" } }
{ "doc": { "price": {"amount": 19.99, "currency":"USD"}, "in_stock": true } }
{ "delete": { "_id": "L-13" } }
```

Note the mixed op types: full `index` (upsert whole doc), partial `update` (fast path, §6.7), and `delete` (§6.8) all ride one request. The response is **per-item**: a top-level `"errors": true` flag plus a status for each op, so partial failure is normal and must be parsed per-item — you cannot treat a `_bulk` response as all-or-nothing.

### Batch sizing

Two knobs: **document count** and **payload bytes**. Flush when *either* trips, plus a **time-based flush** so low-traffic partitions don't starve on freshness:

| Knob | Starting value | Reasoning |
|---|---|---|
| Max docs / bulk | ~1,000–5,000 | enough to amortize overhead; small enough to bound latency |
| Max bytes / bulk | **~5–15 MB** | ES guidance; avoids huge requests that pressure heap & the `http.max_content_length` limit |
| Max linger / flush interval | **~200–500 ms** | caps freshness lag on quiet partitions — a core part of the ~1s budget |

These interact with the freshness budget: a 500ms linger + 1s refresh already spends 1.5s, so on the price/inventory fast path we run a **tighter linger (~100–200ms)** and lean on a low `refresh_interval`. Bigger batches = better throughput but worse tail latency and freshness; **tune empirically** — the right number depends on doc size and cluster hardware, and "I'd measure and tune" is the honest interview answer.

### Refresh-interval tuning during bulk loads

A `refresh` is what makes newly-indexed docs *visible* to search; by default ES refreshes every 1s (brief §6). Refresh is not free — it rolls the in-memory buffer into a new Lucene segment (Ch8). Two regimes:

- **Steady state / hot index:** keep `refresh_interval: 1s` on the `products` alias — this is what buys us NRT for the search path.
- **Bulk backfill / full reindex (§6.9):** the target index serves *no live traffic*, so set `refresh_interval: -1` (disable) and `number_of_replicas: 0` during the load, then restore them and force one `_refresh` + `_forcemerge` at the end. Disabling refresh during a big load can improve indexing throughput by a large factor because you stop creating tiny segments you'll only have to merge away.

```
PUT /products-v9/_settings          # before backfill
{ "index": { "refresh_interval": "-1", "number_of_replicas": 0 } }

PUT /products-v9/_settings          # after backfill
{ "index": { "refresh_interval": "1s", "number_of_replicas": 1 } }
```

### Backpressure and rate control

Elasticsearch pushes back when it can't keep up: bulk operations queue on the `write` thread pool, and when that queue is full ES **rejects** the op with **HTTP 429 (`es_rejected_execution_exception`)**. 429 is not an error to log-and-drop — it is **flow control**, and the correct response is to *slow down*, not retry instantly.

Our control loop:

1. **Bound in-flight bulk requests** per consumer (e.g. ≤ N concurrent bulks) so we never fire an unbounded flood at ES.
2. **On 429, exponential backoff with jitter**, and retry *only the rejected items* (parsed from the per-item response), never the whole batch.
3. **Kafka is the pressure-relief valve.** If ES stays slow, the consumer simply polls less and offsets stop advancing — data safely backs up in Kafka (that's what the 3–7 day retention is for), and **consumer lag** (§6.10) rises as our early-warning signal. Nothing is lost; freshness degrades gracefully. This is the payoff of decoupling with a durable log: **backpressure becomes lag, not data loss.**

```
Indexing Service ──bulk──▶ ES  ── 429 ──▶ backoff+jitter, retry rejected items only
       ▲                                        │
       └── poll slower, offsets stall ──────────┘   (lag rises → alert; Kafka absorbs the burst)
```

### Retry taxonomy

Be precise about what's retryable, because retrying the wrong thing corrupts data or wastes cycles:

| ES response | Meaning | Action |
|---|---|---|
| `429` rejected | overloaded | **retry** rejected items with backoff |
| `409` version conflict | a newer version already applied | **do not retry** — drop; this is correct (§6.6) |
| `503` / timeout | node/shard unavailable | retry with backoff |
| `400` mapping/parse error | bad document | **do not retry** — route to **DLQ** (§6.10) |

---

## 6.6 Idempotency & Correctness: External Versioning

At-least-once delivery + partition rebalances + retries mean the *same* update can be applied twice, and — despite per-`product_id` ordering — a stale update can occasionally chase a fresh one (e.g. a retry of an old batch lands after a new batch). We need a guarantee that is independent of delivery order: **a document must never be overwritten by an equal-or-older version.**

Elasticsearch gives us exactly this with **external versioning**. Every ES doc has a `_version`. Normally ES manages it internally, but we set `version_type: external` and supply our *own* monotonic version — the `version`/`_version` from the Catalog source (brief §4: "index writes keyed by `listing_id` + version"). The Catalog Service bumps this on every mutation (recall the `version = version + 1` in the outbox transaction, §6.2), so it is a strictly increasing, source-of-truth-authoritative counter.

The rule ES enforces:

> With `version_type: external`, ES accepts the write **only if the supplied version is strictly greater** than the currently stored `_version`. Otherwise it rejects with **409 version conflict** and leaves the stored doc untouched.

```
PUT /products/_doc/L-42?version=88&version_type=external
{ "listing_id":"L-42", "price":{"amount":749.00,"currency":"USD"}, ... }
```

### The "old price overwrites new price" race, solved

Walk the canonical race:

1. Seller sets price → **749** (Catalog `version=88`). Event A published.
2. Seller sets price → **759** (Catalog `version=89`). Event B published.
3. Due to a retry / rebalance / consumer hiccup, **B is indexed first** (doc now at `_version=89`, price 759 — correct, this is the newest).
4. Then the delayed **A** arrives and tries to write price 749 with `version=88`.

Without versioning, step 4 clobbers the fresh price with the stale one — a customer sees 749, buys, and GlobalMart eats the €10. **With external versioning**, step 4 supplies `version=88 ≤ 89` → ES returns **409** → the stale write is **rejected** → the doc stays at 759. Correct, regardless of arrival order.

```
stored _version:  (none) ──B(v89)──▶ 89 (price 759) ──A(v88)──▶ 409 REJECTED, stays 89 ✓
```

Crucially, **409 is a success for us, not a failure.** The consumer treats version conflicts as "already superseded — drop and commit the offset," and must *not* retry them (per the table in §6.5). Idempotency falls out for free: re-delivering event B (`version=89`) when the doc is already at 89 also yields 409 → no-op. So **at-least-once + external versioning = effectively-once**, achieved without any distributed transaction.

**Fast-path subtlety (partial updates).** ES external versioning works cleanly on whole-document `index` ops. Partial `update` ops (§6.7) don't take an external version directly, so on the fast path we either (a) attach the version in the doc and guard with a small scripted update (`if (ctx._source._version >= params.v) ctx.op='noop'`), or (b) keep price/inventory writes as tiny full-doc upserts so external versioning applies natively. GlobalMart uses the scripted-guard variant so a stale price update is a `noop`, preserving the same correctness guarantee on the high-volume path.

---

## 6.7 NRT Mechanics: Refresh, the Fast Path, and Update Amplification

Now the freshness engine itself.

### The refresh cycle (brief §6, detailed in Ch8)

ES indexing is not immediately visible. A write lands in an in-memory buffer + the translog (durability). Only a **refresh** turns the buffer into a searchable Lucene segment. With `refresh_interval: 1s`, a doc indexed at *t* is searchable by roughly *t+1s*. This is the dominant term in our ~1s freshness budget. We do **not** call `?refresh=true` on every write — forcing a refresh per request creates a storm of tiny segments and destroys throughput (Ch8's segment-merge cost). We rely on the 1s interval for the bulk of traffic, and reserve explicit/`wait_for` refresh for rare cases that need read-your-write (e.g. a seller-preview flow), not the firehose.

### The fast path: partial updates for price/inventory

Recall §6.1: the 100K/sec peak is mostly price/inventory. Re-indexing the *entire* ~2KB document (re-analyzing title, description, attributes, rebuilding all inverted-index terms) just to change a number is enormously wasteful. So price/inventory take a **fast path** using partial updates:

```
POST /products/_update/L-42
{ "doc": { "price": {"amount": 749.00, "currency":"USD"}, "inventory": 41, "in_stock": true } }
```

Under the hood ES still does a get-and-reindex of the doc (Lucene segments are immutable — there's no in-place field edit; Ch8), but we avoid re-sending and re-analyzing the heavy text fields. `price`, `inventory`, `in_stock`, and numeric signals live in `doc_values` / are cheap to reindex, so the fast path is dramatically lighter than a full upsert and hits the ~1s budget comfortably.

**When to use which:**

| Change | Mechanism | Why |
|---|---|---|
| Price / stock / a signal on one listing | **partial `update`** (fast path) | tiny payload, no text re-analysis |
| New listing / title / description / attributes | full **`index`** upsert | text fields must be (re)analyzed anyway |
| A shared field across *many* listings | **`_update_by_query`** (see amplification) | one query rewrites the matching set |

### `_update_by_query` vs. full reindex

For a bounded set matching a predicate (e.g. "all listings of `P-77` get `popularity=0.91`"), `_update_by_query` rewrites them in place without streaming every doc through the pipeline:

```
POST /products/_update_by_query
{ "query": { "term": { "product_id": "P-77" } },
  "script": { "source": "ctx._source.popularity = params.p", "params": { "p": 0.91 } } }
```

It's a heavy operation (it scans + reindexes every match, respecting versioning), so it's for *correlated bulk* changes, not the per-listing firehose — and it runs async with throttling (`requests_per_second`) so it doesn't starve the live write path.

### The update-amplification problem

This is the sharp edge the brief flags, and the thing most candidates miss. A **shared signal** changes and fans out to many documents:

- `popularity` for product `P-77` is recomputed by analytics. `P-77` is listed by 50,000 sellers across 30 locales ⇒ **50,000 listing documents** must be rewritten to carry the new value.
- One logical change → 50,000 physical writes. At scale, a batch of popularity recomputations can **multiply write volume by 100–1000×**, blowing past the 100K/sec budget and starving the price/inventory fast path — an entirely *self-inflicted* overload.

Mitigations, in the order we apply them:

1. **Don't denormalize what you can join cheaply — but ES can't join, so instead: throttle and batch.** Route signal changes through the lower-priority `catalog.signal.popularity` topic and process them with `_update_by_query` under a rate limit, so they never crowd out price/inventory.
2. **Coalesce.** Signals change often but tolerate seconds-to-minutes of lag (unlike price). Debounce: recompute-and-apply `popularity` at most every N seconds per product, collapsing a flurry into one fan-out. Keying by `product_id` (§6.3) makes this coalescing local to one consumer.
3. **Separate volatile signals into a side index / rescore.** The heaviest mitigation (deferred to Ch7/Ch9): keep ultra-volatile ranking signals *out* of the main doc and blend them at query time via a rescore against a tiny per-product signal store, so a signal change is **one** write, not 50,000. Trade-off: query-time cost and complexity. We mention it as the escape hatch when amplification becomes untenable; GlobalMart keeps moderately-stable signals denormalized and pushes only the most volatile ones to rescore.

**The senior framing:** freshness requirements are *per-field*, not per-document. Price needs ~1s and is cheap (one doc). Popularity tolerates minutes but is expensive (fan-out). Design the pipeline around that asymmetry — one size does not fit all fields.

---

## 6.8 Deletes and Tombstones

Removal is trickier than it looks because ES deletes aren't instantaneous and "gone from the catalog" has several flavors.

### Soft vs. hard delete

- **Soft delete (preferred default):** the listing isn't physically removed; we set a status flag (`in_stock:false`, or a `status:"inactive"` / `visible:false` field) and let the **search path filter it out** (Ch7 adds a `filter` clause). Reversible (seller relists, restock), auditable, and — critically — it's just a normal versioned `update`, so it rides the fast path and obeys the same ordering/versioning guarantees. Out-of-stock is almost always a *soft* delete: the listing should reappear the instant inventory returns.
- **Hard delete:** physically remove the document (`DELETE /products/_doc/L-13` or a `delete` op in `_bulk`). Used for genuine removals: seller closes the listing, a policy takedown, GDPR erasure. Irreversible; the doc must be re-created from source to come back.

### Tombstones and ordering

A delete is subject to the *same* stale-overwrite race as any update: a delete for `L-13` must not be resurrected by a late in-flight update, and a late delete must not wipe a listing that was legitimately re-created. So deletes flow through the **same partition** (keyed by `product_id`) and carry a **version**:

```
{ "delete": { "_id": "L-13", "version": 91, "version_type": "external" } }
```

ES applies the delete only if `91 >` stored version, and — importantly — **retains a version tombstone** for a while (`index.gc_deletes`, default 60s) so that a *stale* update arriving *after* the delete (say `version=90`) is still correctly rejected as a version conflict rather than re-creating the doc. Without the tombstone window, a delete followed by a delayed lower-version update could resurrect a deleted listing. This is the delete-side of the §6.6 guarantee.

### Seller / listing removal fan-out

When a **seller** is removed, *all* their listings must go — a fan-out just like update-amplification. We emit a single `SellerRemoved` event and expand it to a `delete_by_query`:

```
POST /products/_delete_by_query
{ "query": { "term": { "seller_id": "S-9" } } }
```

Same caveats as `_update_by_query`: heavy, throttled, async, versioning-aware. For a huge seller this can be millions of docs, so it runs on the low-priority lane and we monitor its progress. Physically, deleted docs only free space at **segment merge** time (Ch8) — until then they're marked deleted but still on disk, which is why a mass-delete doesn't immediately shrink the index.

---

## 6.9 Full Reindexing Without Downtime

Sometimes we must rebuild the entire index: a **mapping/analyzer change** that isn't dynamically updatable (new tokenizer for a locale, changed field type — Ch4/Ch8), a **shard-count change**, a Lucene/ES major upgrade, or recovery from corruption. The search path serves 100K QPS at 99.99% availability (NFR2) — **downtime is not an option.** The technique is **versioned indices behind an alias** (brief §6), and it leans entirely on ES being *derived, rebuildable* state (§6.1).

### The alias-swap pattern

Search never talks to a concrete index; it talks to the `products` **alias** (Ch7's queries all hit `products`). The alias points at a versioned index `products-vN`. To reindex, we build `products-v(N+1)` alongside the live one and swap the alias atomically.

```
BEFORE:   products (alias) ───▶ products-v8   ◀── live search + writes

BUILD:    products-v9  (new mapping, refresh off, replicas 0)  ◀── backfill
          products-v8                                          ◀── still live

CUTOVER:  products (alias) ───▶ products-v9   (atomic swap)
          products-v8  ◀── kept as instant rollback, then retired
```

The swap is a single atomic alias action — no window where the alias points at nothing:

```
POST /_aliases
{ "actions": [
    { "remove": { "index": "products-v8", "alias": "products" } },
    { "add":    { "index": "products-v9", "alias": "products" } }
] }
```

### Backfill: two sources, in order of preference

**(a) ES-to-ES Reindex API** — fastest when the new index is a *reshaping* of the current one (new mapping, more shards) and the source data is already correct in `products-v8`:

```
POST /_reindex?wait_for_completion=false
{ "source": { "index": "products-v8", "size": 5000 },
  "dest":   { "index": "products-v9", "version_type": "external" } }
```

It returns a task id; monitor via `_tasks`. `version_type: external` preserves each doc's version so concurrent live writes (below) aren't clobbered by the backfill.

**(b) Backfill from the source of truth (Catalog Service)** — the authoritative rebuild, used when the transform logic itself changed or the old index is suspect. We **replay from Kafka / scan the Catalog store** through the *same Indexing Service pipeline* (§6.4) into `products-v9`. This is where the 3–7 day Kafka retention (§6.3) and "ES is rebuildable" pay off: we can regenerate 10B documents from truth. For a from-scratch rebuild we snapshot the Catalog store, bulk-load the snapshot, then replay the Kafka backlog from the snapshot's offset to catch up.

### Dual-writing during reindex — the correctness crux

The catalog does **not** freeze during a multi-hour reindex of 10B docs. New writes keep arriving. If we backfilled `v9` and *then* swapped, `v9` would be stale by hours. The fix: **during reindex, the Indexing Service writes to *both* `products-v8` (live) and `products-v9` (new) simultaneously.**

```
                     ┌─▶ products-v8  (live, serving search)
Indexing Service ────┤
                     └─▶ products-v9  (new, being backfilled)
```

Combined with external versioning (§6.6), this is race-free: the historical backfill and the live dual-write may touch the same doc in either order, but the **higher version always wins**, so `v9` converges to correct state no matter the interleaving. Sequence:

1. Create `products-v9` (new mapping; `refresh_interval:-1`, `replicas:0`).
2. Turn on **dual-write** to both v8 and v9 (all live changes now hit both).
3. Backfill v9 (Reindex API or source replay) — versioned, so it never overwrites a fresher live write.
4. When backfill catches up and lag ≈ 0, restore v9's `refresh_interval:1s` and `replicas:1`; let it warm.
5. **Validate** v9: doc counts vs. source, spot-check queries, compare relevance on a canary.
6. **Atomic alias swap** v8 → v9.
7. Keep dual-write briefly for **instant rollback** (just swap the alias back), then stop writing v8 and delete it.

Zero downtime, zero data loss, instant rollback. This procedure is the concrete payoff of the "ES is derived, rebuildable state" principle — you can only do this fearlessly because the Catalog Service can always re-supply the truth.

---

## 6.10 Observability: Lag, DLQ, and Replay

A pipeline you can't see is a pipeline that's already broken. Three pillars.

### Freshness / indexing-lag metrics

The SLO is "~1s to searchable" (FR8), so we measure the end-to-end lag directly. Stamp each event with `catalog_commit_ts` at the outbox (§6.2) and, when the doc becomes searchable, compute `now - catalog_commit_ts`. Break it down by stage so you know *which* stage blew the budget:

| Metric | What it catches |
|---|---|
| **End-to-end indexing lag** (commit → searchable), p50/p99 | the actual FR8 SLO; alert when p99 > 1s (fast path) |
| **CDC capture lag** (commit → Kafka) | Debezium falling behind / connector stall |
| **Consumer lag** (Kafka log-end offset − committed offset), per partition | the #1 early-warning signal — Indexing Service or ES can't keep up |
| **Bulk reject (429) rate & retry rate** | ES write-side saturation → backpressure kicking in |
| **Bulk latency & batch size** | tuning signal for §6.5 knobs |
| **Version-conflict (409) rate** | expected-nonzero; a *spike* means reordering/replay storms |

**Consumer lag is the master health signal.** Because Kafka absorbs backpressure as lag (§6.5), rising lag is the leading indicator of *every* downstream problem — ES slowness, a hot partition, a stuck consumer — usually well before freshness SLO breach. Alert on lag trend, not just absolute value.

### Dead-letter queue (DLQ) for poison messages

Some messages can **never** succeed: a malformed payload, a document that violates the mapping (400 parse error), a bug that throws on a specific record. If the consumer retries such a "poison" message forever, it **blocks its entire partition** — head-of-line blocking that stalls every well-behaved product hashed to that partition. Unacceptable.

The rule: **retry transient failures (429/503), quarantine permanent ones.** After a bounded number of attempts, publish the failing record — plus error context (exception, stack, ES response, original offset) — to a **`catalog.index.dlq`** topic, commit past it, and keep the partition flowing.

```
process(record)
  ├─ success ──────────────▶ commit offset
  ├─ transient (429/503) ──▶ backoff + retry (bounded)
  └─ permanent / max-retries ▶ produce to DLQ (with context) → commit offset → CONTINUE
```

The DLQ is a *staffed* queue: it alerts, and its contents are triaged. Most entries reveal a transform bug or a bad-data case in the Catalog; you fix the code/data and then **replay**. (Failure-handling patterns — DLQ, retries, circuit breakers across the whole system — are covered end-to-end in **Ch10**; here we only wire the indexing-side DLQ.)

### Replay

Because Kafka retains events (§6.3) and every write is **idempotent + versioned** (§6.6), replay is safe and boring — which is exactly what you want in an incident. Three replay modes:

- **DLQ replay:** after fixing the transform bug, re-publish DLQ records back to the source topic; versioning makes reprocessing a no-op where the doc is already current.
- **Offset reset:** if a bad deploy corrupted a window of documents, reset the consumer group to an earlier offset and reprocess — stale re-applications are rejected by version, fresh ones re-land.
- **Full rebuild:** the §6.9 reindex-from-source, the ultimate replay.

Replay being safe is the *cumulative* dividend of every decision in this chapter: no dual-write (§6.2), ordered per-`product_id` partitions (§6.3), at-least-once + idempotent-versioned writes (§6.5–6.6), and rebuildable derived state (§6.9). Get those right and operating the pipeline becomes routine.

---

## Interview Tips

- **Lead with "no dual-write," then say "outbox + CDC" in the same breath.** This is the single highest-signal move in the indexing question. If you can explain *why* dual-write is unsafe (two non-atomic writes, no retry fixes it) and that the outbox makes the event part of the DB transaction while CDC ships it from the log, you're already ahead of most candidates.
- **Name the partition key and defend it.** "Partition Kafka by `product_id` so all updates to one product are ordered on one partition." Then volunteer the trade-off (hot-partition skew for mega-products) — interviewers probe exactly there.
- **Have the "old price overwrites new price" race ready with the exact fix.** External versioning (`version_type: external`) with the source's monotonic version; a stale write gets 409 and that 409 is a *success*. This is the correctness money-shot.
- **Distinguish freshness per field.** Price = ~1s, cheap, fast path (partial update). Popularity = tolerates minutes, expensive (fan-out). Volunteering "freshness is per-field, not per-document" signals seniority.
- **Bring up update amplification unprompted.** "A shared signal change fans out to every listing of a product — 50K writes for one logical change." Then give the mitigations (throttle via a separate topic, coalesce, or push volatile signals to query-time rescore). Most candidates never see this; it's a strong differentiator.
- **Know the reindex dance cold:** versioned index + alias, dual-write during backfill, versioning prevents backfill/live races, validate, atomic alias swap, keep old index for rollback. Tie it to "ES is derived, rebuildable state."
- **Treat 429 as flow control, not failure.** Backpressure → Kafka absorbs → consumer lag rises → alert. Retry rejected items with jitter; never retry 409s or 400s.
- **Common traps to avoid:** calling `?refresh=true` per write (segment storm); retrying poison messages forever (head-of-line blocking — use a DLQ); claiming exactly-once (say "at-least-once + idempotent = effectively-once"); forgetting that deletes need versioning + a tombstone window too.

## Key Takeaways

- **Freshness vs. correctness vs. throughput** is the governing tension of the write path; every decision trades among the three. GlobalMart's write mix is *mutation-dominated*, so a per-field fast path matters more than raw create throughput.
- **Capture changes with the transactional outbox + CDC (Debezium).** Never dual-write. The outbox makes the event atomic with the business write; CDC ships it from the WAL/binlog in commit order.
- **Kafka is the durable, replayable backbone.** Partition by `product_id` for per-product ordering; size partitions (~300 on the hot topic) for consumer parallelism + headroom; run a consumer group with **at-least-once** delivery.
- **The Indexing Service transforms, enriches (denormalizes signals in), normalizes, and dedupes** — turning DB change events into ES documents, since ES has no query-time join.
- **Write via `_bulk`** with size/byte/time-based batching; tune `refresh_interval` (1s hot, `-1` during backfill); treat **429 as backpressure** (backoff + retry rejected items), letting Kafka absorb bursts as lag.
- **External versioning (`version_type: external`) is the correctness backbone:** the higher source version always wins, stale writes get 409, and at-least-once + versioning = **effectively-once**. It also makes the "old price overwrites new price" race impossible.
- **Exploit per-field freshness:** partial updates / `_update_by_query` for cheap price/inventory; watch for **update amplification** when a shared signal fans out across a product's listings, and mitigate with throttling, coalescing, or query-time rescore.
- **Deletes are versioned too:** prefer soft delete (filterable, reversible) for out-of-stock; hard delete + version tombstone for real removals; seller removal fans out via `_delete_by_query`.
- **Reindex with zero downtime** via versioned `products-vN` + atomic alias swap, dual-writing to old and new during backfill; versioning keeps the backfill/live race safe. This works only because **ES is derived, rebuildable state** over the Catalog source of truth.
- **Observe lag (end-to-end, CDC, consumer), reject-rates, and 409s; DLQ poison messages** to avoid head-of-line blocking; **replay** is safe precisely because writes are idempotent and versioned. (Full failure handling: Ch10.)
