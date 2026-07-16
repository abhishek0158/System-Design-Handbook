# Chapter 8 — Elasticsearch Internals

Every previous chapter treated Elasticsearch as a well-behaved black box: we threw documents at the bulk API (Chapter 6), fired queries at coordinating nodes (Chapter 7), and trusted that ~600 primary shards would hold our ~24 TB of primaries (Chapter 2). This chapter opens the box. If you cannot explain *why* a document is searchable ~1 second after indexing, *why* deep pagination melts the cluster, or *why* your primary shard count is frozen the moment you create the index, you do not really understand the system you are operating — and an interviewer will find the seam in about ninety seconds.

The thesis of this chapter is simple: **Elasticsearch is a distributed coordination and orchestration layer wrapped around many independent copies of Apache Lucene.** Almost everything that makes ES fast, near-real-time, and occasionally surprising is a Lucene property. Almost everything that makes it scalable, highly available, and eventually consistent is an Elasticsearch property. Keep that dividing line in your head and the whole system decomposes cleanly.

We will build up from the smallest unit (a Lucene segment) to the largest (a multi-region cluster), and at every layer we will tie the internal back to a concrete GlobalMart decision.

---

## 8.1 The Mental Model: Shard = One Lucene Index

Start with the single most important sentence in this chapter:

> **A shard is a complete, self-contained Lucene index.**

Not a piece of an index. Not a partition that needs its siblings to answer a query. A shard is a fully functional inverted index that can, on its own, tokenize text, score documents with BM25, run aggregations, and return ranked results. Elasticsearch's distributed layer exists to (a) spread documents across many such Lucene indices, (b) route each document to exactly one of them, and (c) fan a query out to all of them and merge the answers.

```
┌─────────────────────────────────────────────────────────────┐
│ Elasticsearch cluster  (distributed layer: routing,          │
│  cluster state, scatter-gather, replication, election)       │
│                                                               │
│   Index "products-v7"  (a logical name / alias target)        │
│   ├── shard 0  ──▶ Lucene index  ├─ segment ─ segment ─ ...   │
│   ├── shard 1  ──▶ Lucene index  ├─ segment ─ segment ─ ...   │
│   ├── ...                                                     │
│   └── shard 599 ─▶ Lucene index  ├─ segment ─ segment ─ ...   │
│                                                               │
│   Each Lucene index (shard) = inverted index + doc_values     │
│   + stored fields + translog, living on ONE data node.        │
└─────────────────────────────────────────────────────────────┘
```

So GlobalMart's `products` alias points at `products-v7`, which is configured with 600 primary shards. That means **600 independent Lucene indices**, each holding roughly 1/600th of our 10B listings (~16–17M docs, ~40 GB), plus a replica copy of each living on a different node. When a buyer searches for "wireless earbuds", the query touches one copy of all 600 shards. When the Indexing Service writes a listing, it lands in exactly one shard, determined by routing.

Everything below is either "what happens inside one of those Lucene indices" (§8.2–8.5, §8.8, §8.11) or "how ES coordinates across them" (§8.6, §8.7, §8.10).

### What Lucene actually is (and isn't)

Apache Lucene is a single-node, embeddable Java **library** — not a server, not a database, not a cluster. It knows how to build an inverted index, store documents, score with BM25, and run aggregations over the data on *one machine's* disk. It has no concept of nodes, sharding, replication, HTTP, JSON, cluster state, or high availability. It has been battle-hardened for over two decades and is the retrieval core of Elasticsearch, OpenSearch, and Solr alike.

Elasticsearch's value-add is precisely everything Lucene *lacks*: a REST/JSON API, a mapping and query DSL layer, and — most importantly — the **distributed systems machinery** that turns hundreds of independent Lucene indices into one logical, horizontally scalable, highly available search service. When you understand this split, a lot of "why does ES do X?" questions answer themselves: if X is about text, tokens, scoring, or on-disk structure, it's Lucene; if X is about routing, consistency, failover, or fan-out, it's Elasticsearch. This chapter deliberately walks that line, bottom-up.

---

## 8.2 The Inverted Index

The inverted index is the data structure that makes full-text search fast, and it is the reason we use Lucene at all rather than a `LIKE '%earbuds%'` scan over 10B rows in the Catalog's SQL store.

A **forward index** maps documents to their terms (`doc 42 → {wireless, bluetooth, earbuds}`). An **inverted index** flips it: it maps each term to the list of documents containing it (`earbuds → {doc 42, doc 91, doc 5501, ...}`). Search is fundamentally the question "which documents contain these terms?", so the inverted layout answers it directly.

### Anatomy

For a single analyzed field (say `title`), Lucene stores:

1. **Terms dictionary** — the sorted set of all unique terms in that field, across the whole shard. For GlobalMart's `title` field in one shard this might be tens of millions of distinct terms (brand names, model numbers, misspellings sellers typed in).
2. **Postings list** — for each term, the sorted list of document IDs (within the shard) that contain it. This is the core of the inverted index.
3. **Term frequency (TF)** — for each (term, doc) pair, how many times the term appears in that doc's field. Needed for scoring.
4. **Positions** — the token offsets at which the term occurs (`"wireless"` at position 0, `"earbuds"` at position 1). Needed for phrase and proximity queries (`"wireless earbuds"` as an exact phrase).
5. **Offsets / payloads** (optional) — character offsets for highlighting, and arbitrary per-position byte payloads.

Conceptually:

```
term "earbuds":
   docFreq = 3            (appears in 3 docs in this shard)
   postings:
     doc 42  | tf=1 | positions=[1]
     doc 91  | tf=2 | positions=[0, 7]
     doc 5501| tf=1 | positions=[3]
```

### How a term lookup works — the FST

The terms dictionary is not a hash map. It is stored on disk as a **Finite State Transducer (FST)** — a compact, immutable automaton that maps term strings to their metadata (the file offset of their postings). The FST is the single most elegant data structure in Lucene, and it is worth being able to describe it:

- It exploits **shared prefixes and suffixes**. The terms `iphone`, `iphone13`, `iphone14`, `iphones` share the prefix `iphone`; the FST stores that prefix once as a chain of states and branches only where the terms diverge. This makes the terms index tiny enough to hold **in memory (heap/off-heap)** even for tens of millions of terms.
- Lookup is O(length of the term), following one transition per character. It emits an output (an offset) as it walks, so by the time you reach the accepting state you have the location of that term's postings on disk.
- Because it is sorted, it also supports **range** and **prefix** enumeration cheaply — which is exactly what powers our `edge_ngram`/prefix autocomplete path and range facets.

So a lookup for `earbuds` is: walk the FST in the terms index (in memory) → get the on-disk offset of the postings → seek and decode the postings block (delta-encoded, compressed doc IDs). That is why term lookups are microseconds, not milliseconds, even at 16M docs per shard.

**Postings compression.** Doc IDs in a postings list are monotonically increasing, so Lucene delta-encodes them and packs them with **Frame of Reference / PForDelta** block encoding, plus skip lists so a query can jump ahead when intersecting two postings lists (e.g. `brand:apple AND category:phones`). This is why boolean filter intersection is so cheap and why we push structured constraints into `filter` clauses in Chapter 7.

**Worked example — how a boolean query executes.** Take `brand:apple AND category:phones`. Lucene resolves each term to its postings list via the FST, then intersects the two sorted doc-ID lists using a **leapfrog / zig-zag** walk: advance the cursor on the *rarer* term (fewer postings — say `brand:apple` has 40K docs) and use its skip list to jump the more common term's cursor (`category:phones`, 400K docs) forward to the next candidate, skipping entire compressed blocks that can't match. The cost is roughly proportional to the size of the *smaller* postings list, not their product — which is why leading with the most selective clause matters, and why filter ordering can affect latency even when the result is identical. A phrase query like `"wireless earbuds"` adds a second step: after intersecting the two docs' postings, it checks the **positions** to confirm `earbuds` occurs at `position(wireless)+1` in the same doc.

**Terms index vs. terms dictionary.** A precise nuance: the full terms *dictionary* (every term + its stats) lives on disk in blocks; only a sparse **terms index** — the FST — is held in memory, mapping term prefixes to the on-disk block that contains them. So a lookup walks the in-memory FST to find the right on-disk block, then does one seek and a short scan within that block. This two-level design is what lets a shard with tens of millions of terms keep only a few MB of terms index resident while still resolving any term in one disk seek (usually a page-cache hit).

**GlobalMart tie-in.** The FST is also why our `title.raw` keyword field and `brand` keyword field are cheap to facet on and why prefix-based typeahead is viable: prefix enumeration over a sorted, in-memory automaton is inherently fast.

---

## 8.3 Segments

Here is where Lucene's core design decision lives, and it explains half the behaviors in this chapter.

> **A Lucene index (= a shard) is not one monolithic file. It is a set of immutable segments, each of which is itself a miniature, self-contained inverted index.**

### Why segments exist

Building an inverted index incrementally is hard. If the index were a single mutable structure, every new document would require inserting doc IDs into the middle of sorted postings lists on disk — random writes, lock contention, fragmentation. Lucene sidesteps all of this with an append-only strategy:

- New documents accumulate in an in-memory buffer.
- Periodically, that buffer is serialized to disk as a brand-new **segment** — a small but complete inverted index (terms dict + postings + doc_values + stored fields).
- **A written segment is never modified.** It is immutable.

Immutability is the gift that keeps giving:

- **No locking on read.** Because a segment never changes after it is written, any number of searches can read it concurrently without locks. This is central to how one shard serves high QPS.
- **Filesystem cache friendliness.** Immutable files can be memory-mapped (`mmap`) and cached by the OS page cache aggressively; there is never a dirty-page write-back to worry about for the segment data itself.
- **Cheap, safe replication and backup.** A file that never changes can be copied byte-for-byte.

The cost of immutability: you accumulate *many* segments over time, a search must consult *all* of them, and deletes/updates cannot edit in place. Both costs are managed by merging (§8.4) and soft-deletes (§8.5).

### What a segment physically contains

Within one segment, for our product listings:

| Structure | Purpose | GlobalMart fields |
|---|---|---|
| **Inverted index** (terms/postings) | Full-text matching & scoring | `title`, `description`, `brand`, analyzed text |
| **Stored fields** (`_source`) | The original JSON, returned on fetch | the whole document, row-oriented, compressed (LZ4) |
| **doc_values** | Columnar, per-field values for sort/agg/script | `price`, `rating`, `popularity`, `seller_rating`, `category_path`, all keywords |
| **norms** | Per-field length normalization factor for scoring | `title`, `description` |
| **Live docs bitset** (`.del`) | Which docs are still alive | maintained per segment (§8.5) |

Two of these deserve emphasis because interviewers love them:

**Stored fields vs. doc_values — row vs. column.** `_source` (stored fields) is **row-oriented**: it stores each document's original JSON blob together, optimized for "give me the full document." That is perfect for the fetch phase (§8.7) where we return the top 10 listings. But it is terrible for "sort 16M docs by price" — you would have to decompress 16M blobs. So Lucene *also* stores **doc_values**: a **columnar**, on-disk structure that keeps all values of a single field contiguous (`price` for every doc, packed together). Sorting by price, computing the "Price" facet, or scripting on `popularity` reads the doc_values column — sequential, mmap-friendly, and off-heap. Chapter 4's insistence that sortable/facetable fields keep `doc_values: true` (the default) is a direct consequence.

**norms** encode field-length normalization: a match in a 3-word title is worth more than a match in a 300-word description. Norms are lossily compressed to a single byte per field per doc. If a field is used only for filtering and never scored, we can disable norms (`"norms": false`) to save space — relevant for GlobalMart's many keyword attribute fields.

### The files on disk

For the curious operator, a segment is a small family of files sharing a generation prefix (e.g. `_a1`), written by the current Lucene **codec**:

| Extension | Contents |
|---|---|
| `.tim` / `.tip` | Terms dictionary (blocks) / terms index (the FST) |
| `.doc` / `.pos` / `.pay` | Postings: doc IDs+freqs / positions / offsets+payloads |
| `.fdt` / `.fdx` | Stored fields data / index (`_source`) |
| `.dvd` / `.dvm` | doc_values data / metadata |
| `.nvd` / `.nvm` | Norms data / metadata |
| `.liv` | Live-docs bitset (soft-deletes) — the only per-segment file that changes |
| `segments_N` | The **commit point**: the manifest of which segments are live |

You do not manage these directly, but recognizing them turns "the shard has 4,000 segments and search is slow" from a mystery into a diagnosis (too-frequent refresh, insufficient merging), and it makes the immutability claim concrete: every file above is write-once except `.liv`, which is why deletes are the one thing that can touch an existing segment (and even then, a new `.liv` generation is written, not the segment body).

---

## 8.4 Write Internals: Buffer → Refresh → Translog → Flush → Merge

This is the most examined mechanism in all of Elasticsearch. Get the vocabulary exactly right, because "refresh", "flush", and "commit" are constantly confused and precision here is a strong signal.

Here is the full lifecycle of a write on a single shard:

```
                        ┌───────────────────────── in-memory ─────────────────────────┐
 index request ──▶ ①  Lucene in-memory buffer          + ②  translog (append, on disk) │
                        └───────────────┬───────────────────────────┬──────────────────┘
                                        │ refresh (default 1s)       │ (fsync on request
                                        ▼                            │  or every 5s)
                     ③ new IMMUTABLE segment in filesystem cache     │
                        → now SEARCHABLE (this is NRT ≈ 1s)          │
                                        │                            │
                                        │ flush (Lucene commit)      │
                                        ▼                            ▼
                     ④ segments fsync'd to disk, commit point written, translog truncated
                                        │
                                        ▼
                     ⑤ background merge: many small segments → fewer big segments
```

### ① In-memory buffer + ② translog

When an index request lands on the primary shard, two things happen:

1. The document is added to Lucene's **in-memory indexing buffer**. It is **not yet searchable** — it is not in any segment.
2. The operation is appended to the **translog** (transaction log), a per-shard write-ahead log on disk.

Why both? The buffer gives us fast batched indexing; the translog gives us **durability**. Segments are only made durable periodically (on flush), so if the node crashed between segment creations, everything in the buffer would be lost. The translog is replayed on recovery to reconstruct those un-flushed operations.

By default the translog is **fsync'd on every request** (`index.translog.durability: request`) before ES acknowledges the write — this is what lets ES claim a write is durable once acknowledged. You can relax it to `async` (fsync every 5s) to trade a tiny durability window for throughput. **GlobalMart decision:** ES is a *derived, rebuildable* index (brief §4) and the true source of truth is the Catalog + Kafka log (Chapter 6). We can therefore safely run `index.translog.durability: async` on bulk-load indices, because a lost second of writes is replayable from Kafka. On the live `products-v7` index we keep `request` durability for the hot write path but lean on Kafka as the ultimate backstop.

### ③ Refresh — the birth of a searchable segment (and NRT)

A **refresh** takes whatever is in the in-memory buffer and writes it as a **new segment into the filesystem cache** (crucially, *not* necessarily fsync'd to disk yet — the OS page cache is enough to make it readable). The moment that segment exists, its documents are **searchable**.

This is the entire secret of **Near-Real-Time (NRT) search**: documents are not searchable the instant they are indexed; they become searchable at the next refresh. The default refresh interval is **1 second**, which is exactly the source of the "~1s freshness" figure quoted throughout the brief (FR8, NFR4).

```jsonc
PUT /products-v7/_settings
{ "index.refresh_interval": "1s" }   // hot path: NRT freshness
```

Two knobs matter enormously for GlobalMart:

- **Refresh is not free.** Each refresh creates a segment. Refreshing too often creates a swarm of tiny segments that must later be merged (§8.4 merge), burning CPU and I/O. This is the classic tension.
- **During bulk reindex** (building `products-v8` for a zero-downtime swap, brief §6), we do **not** need 1s freshness — nobody is searching the new index yet. So we set `refresh_interval: -1` (disable) during the bulk load and restore it at the end. This alone can double or triple bulk indexing throughput because it eliminates the tiny-segment churn.

```jsonc
// during a full reindex into products-v8:
PUT /products-v8/_settings
{ "index.refresh_interval": "-1", "index.number_of_replicas": 0 }
// ... bulk load ~10B docs ...
{ "index.refresh_interval": "1s", "index.number_of_replicas": 1 }  // then flip alias
```

> **Refresh ≠ durability.** A refreshed segment lives in the filesystem cache and *may* be lost on a hard crash — but the operations that built it are still in the translog, so nothing is actually lost. Refresh is about **visibility**, not durability.

### ④ Flush — Lucene commit + fsync

A **flush** performs a **Lucene commit**: it fsyncs the in-memory buffer and all filesystem-cached segments to physical disk, writes a new **commit point** (the list of segments that constitute a durable, consistent view of the index), and then **truncates the translog** (those operations are now safely in segments, so the log can be discarded).

Flush happens automatically when the translog gets large (default `index.translog.flush_threshold_size: 512mb`) or periodically. You rarely trigger it manually.

> **The precise distinction, memorize this:**
> - **Refresh** = make recent writes *searchable* by creating a new segment in the filesystem cache. Frequent (1s). Cheap-ish. About **visibility**.
> - **Flush** = fsync segments to disk + write a Lucene commit point + truncate translog. Infrequent. About **durability**.
> - **Commit** = the Lucene term for the durable operation a flush performs; a "commit point" is the on-disk manifest of which segments are live. So *flush* is the ES verb; *commit* is the Lucene noun for what it accomplishes.

### ⑤ Segment merging

Because every refresh spawns a segment and segments are immutable, a busy shard would accumulate thousands of segments — and since a search must visit *every* segment (walk every segment's FST, merge results), too many segments directly degrades query latency. Lucene runs a background **merge** process that reads several smaller segments and writes their contents into one larger new segment, then drops the originals.

Merging does three jobs at once:

1. **Reduces segment count** → fewer FSTs to consult per query → faster search.
2. **Physically removes deleted documents** (§8.5) → reclaims disk and improves scoring accuracy.
3. **Improves compression** by re-encoding larger contiguous postings.

The default **tiered merge policy** groups segments into size tiers and merges within a tier, keeping merges roughly logarithmic. Two knobs govern it: `index.merge.policy.segments_per_tier` (how many segments accumulate in a tier before a merge fires, default 10) and `max_merged_segment` (default 5 GB — segments above this are not merged further, so a huge shard settles into a handful of ~5 GB segments plus smaller live ones). Because each merge produces a segment ~N× the size of its inputs, a document is rewritten roughly `log_N(shard_size / segment_size)` times over its lifetime — a real, if modest, write-amplification tax on top of the delete+reindex tax from §8.5. But merging is **I/O- and CPU-expensive**: it rewrites gigabytes. On GlobalMart's hot data nodes, a merge storm competing with 100K QPS of reads is a real latency risk, so we:

- Cap merge throughput with `indices.store.throttle` / `index.merge.scheduler.max_thread_count` tuned to SSD parallelism.
- On the **warm tier** (older/less-popular locales, brief §6), optionally run a **force-merge to 1 segment** on read-only indices. A static, force-merged, single-segment index is the fastest possible thing to search and the cheapest to cache — perfect for warm data that never changes. *Never* force-merge an index still being written to.

```jsonc
// warm-tier, read-only monthly index → collapse to 1 segment for peak read speed:
POST /products-2026-05/_forcemerge?max_num_segments=1
```

---

## 8.5 Deletes and Updates: Soft-Deletes, the .del Bitset, and Why Updates Are Expensive

Segments are immutable — so how do you delete or update a document? You **don't touch the segment**. Instead:

**Delete.** Lucene keeps a per-segment **live-docs bitset** (historically the `.del` file). Deleting document 42 does not remove it from the segment's postings; it flips bit 42 to 0 in the live-docs bitset. The document is now a **soft-deleted / "tombstoned"** doc: it still occupies space in the segment, still appears in postings lists, but is **filtered out of every search result** by consulting the bitset. Its space is only truly reclaimed when a **merge** eventually rewrites that segment and simply omits the dead docs.

**Update.** There is no in-place update in Lucene. An update is mechanically a **delete + re-index**:

1. The old version of the document is soft-deleted (bit flipped in its segment's bitset).
2. The new version is indexed as a fresh document into the current in-memory buffer → next refresh → new segment.

This is why **updates are more expensive than they look**, and why it matters hugely for GlobalMart:

> Our dominant write workload is **price and inventory mutations** — ~100K writes/sec peak (brief §2), *mostly* small field changes on existing listings. But Elasticsearch cannot patch one field in a segment. Every price tick re-indexes the *entire* ~2 KB document as a new doc and tombstones the old one.

Consequences and mitigations:

- **Write amplification.** A 4-byte price change costs a full-document re-index plus a future merge to reclaim the tombstone. This is a core reason (Chapter 6) we debounce/coalesce rapid price updates in the Indexing Service and batch them via the bulk API rather than firing one ES update per tick.
- **Versioning to avoid stale writes.** Out-of-order updates (a common hazard with CDC + Kafka retries) are rejected using external versioning: ES keeps `_version` per doc and drops any write whose version is older. This is the `_version` / external-version idempotency the brief mandates (brief §3, §4).

```jsonc
// idempotent NRT price update keyed by listing_id + external version:
PUT /products-v7/_doc/L-99271?version=1699999999&version_type=external
{ "price": { "amount": 749.00, "currency": "USD" }, "_version": 1699999999 }
```

- **Deleted-doc bloat inflates scoring.** Until a merge runs, tombstoned docs still count toward term/document frequencies used by BM25 (§8.8), subtly skewing relevance. Aggressive-enough merging on the hot tier keeps this bounded.
- **`_update` vs full re-index.** ES's `_update` API retrieves `_source`, applies the partial change, and re-indexes the whole doc — it is *convenience*, not a cheaper physical operation. Understanding that it is still delete+reindex under the hood is the insight interviewers probe for.

---

## 8.6 Sharding and Routing

We have 600 primary shards. How does ES decide which one a given listing lives in, and why can't we change that number later?

### The routing formula

```
shard = murmur3_hash(_routing) % number_of_primary_shards
```

By default `_routing` is the document's `_id` (our `listing_id`). ES hashes it with **murmur3** and takes it modulo the **primary** shard count to pick the shard. This is deterministic: the same `listing_id` always maps to the same shard, so a GET-by-id or an update goes straight to the right shard without a broadcast.

### Why the primary shard count is immutable

Look at the `% number_of_primary_shards` in that formula. If we created the index with 600 primaries and later changed it to 700, then `hash(id) % 600` and `hash(id) % 700` would point almost every document at a *different* shard — the entire routing scheme would break and nearly every document would be "in the wrong place." So Elasticsearch **fixes the primary shard count at index-creation time**. You cannot add primaries to a live index; you must **reindex into a new index** with a new shard count.

This is precisely why the brief prescribes the **alias + versioned index** pattern (`products` → `products-vN`). When GlobalMart's corpus grows and 40 GB shards become 80 GB shards, we don't resize in place — we create `products-v8` with more primaries and reindex. (`_split` and `_shrink` APIs exist for multiplying/dividing shard counts by integer factors on a read-only index, but they still produce a new index.)

**GlobalMart sizing recap (brief §2):** ~24 TB primaries ÷ ~40 GB target shard ≈ **600 primary shards**. We deliberately size shards around 40 GB because shards much larger than ~50 GB recover and rebalance slowly, while thousands of tiny shards waste heap on cluster-state and per-shard overhead. 600 is the balance point.

### Primary vs. replica

Each of the 600 primaries has **1 replica** (brief §6) → 1,200 total shards. The two roles:

- **Primary shard** — the authoritative copy. All writes go to the primary first; it assigns the sequence number and then forwards the operation to its replicas.
- **Replica shard** — a full, searchable copy on a *different node*. Replicas provide (a) **HA** — if the primary's node dies, a replica is promoted to primary; and (b) **read throughput** — searches are load-balanced across primary *and* replica copies. Because GlobalMart is **QPS-bound, not storage-bound** (brief §2), replicas are as much about serving 100K QPS as about durability. If we needed more read headroom for a traffic spike, adding replicas is the lever (Chapter 9).

**Custom routing** is a power feature worth mentioning: if we routed by `market` (e.g. `_routing=US`), all US listings would co-locate on a subset of shards, so a market-scoped query would hit fewer shards. The trade-off is potential hotspots and skew (some markets are far larger), so GlobalMart keeps default `_id` routing for even distribution and relies on the `market` filter instead.

### Primary-replica replication and sequence numbers

How does a replica stay a faithful copy of its primary, and how does ES recover a replica without recopying 40 GB after a brief disconnect? Two per-operation counters, assigned by the primary, make this work:

- **`_seq_no`** — a monotonically increasing sequence number the primary stamps on every write it accepts. It defines a total order of operations on that shard.
- **`_primary_term`** — incremented each time a *new* primary is elected for the shard. It lets replicas detect and reject operations from a stale primary (one that was demoted during a partition), preventing conflicting histories.

The write flow on a shard is: client write → coordinator routes to **primary** → primary validates, assigns `_seq_no`/`_primary_term`, writes locally (buffer + translog) → forwards **in parallel** to all in-sync replicas → each replica applies it and acks → once enough replicas ack, the primary acks the client. This is why a write's latency includes a replica round trip, and why the number of replicas affects write cost as well as read capacity.

Two derived concepts you should be able to name:

- **Local checkpoint** — the highest `_seq_no` below which a shard copy has *no gaps* (everything up to it is applied).
- **Global checkpoint** — the highest `_seq_no` that *all* in-sync copies have reached. Operations at or below it are safely replicated everywhere.

When a replica reconnects after a short outage, ES doesn't recopy the whole shard. It runs a **sequence-number-based recovery**: the primary retains recent operations (the `retention lease` / soft-deletes retention window) and replays only the ops the replica missed — from the replica's local checkpoint forward. Only if the gap is too large to replay does ES fall back to a full **file-based recovery** (copying segment files). For GlobalMart, where node restarts and brief network blips are routine at 60–100 data nodes, seq-no recovery is the difference between a shard rejoining in seconds versus re-streaming 40 GB. We revisit failure and recovery behavior in Chapter 10.

---

## 8.7 Distributed Search: Scatter-Gather, Query-then-Fetch, and Deep Pagination

Now the distributed read path — the mechanism behind the "ES query (scatter-gather) 120 ms" line in the brief's latency budget (§2).

A search does not know in advance which shards hold the best results, so it must ask **all** of them. This is **scatter-gather**, orchestrated by a **coordinating node** (our dedicated coordinating/query-router nodes, brief §6). It runs in **two phases**.

### Phase 1 — QUERY

```
                       coordinating node
                            │  (parse, rewrite, pick 1 copy of each shard)
        ┌──────────────┬────┴────┬──────────────┐  scatter
        ▼              ▼         ▼              ▼
     shard 0        shard 1   shard 2   ...  shard 599
   (top-K ids     (top-K)   (top-K)          (top-K)
    + scores)        │         │                │
        └──────────────┴────┬────┴──────────────┘  gather
                            ▼
              coordinating node merges 600 × top-K
              → global top-K, sorted by score
```

The coordinating node picks **one copy** (primary *or* replica) of each of the 600 shards and sends the query to each. Every shard **independently** executes the query against its own Lucene index, scores matching docs with BM25, and returns just the **top `K` document IDs + their scores** (where `K = from + size`) — *not* the document contents. Each shard returns a tiny payload: K ids and K floats.

The coordinator merges 600 sorted lists into one global sorted list and keeps the final top `size` results. At this point it knows *which* documents to return but not their contents.

### Phase 2 — FETCH

The coordinator now issues a **fetch** for just the final top-`size` documents (e.g. 10) to the specific shards that own them, retrieving `_source` (stored fields, §8.3) and any highlights. Only ~10 full documents cross the wire, not 6,000.

This **query-then-fetch** split is a deliberate optimization: expensive scoring is distributed and returns only cheap ids/scores; expensive `_source` retrieval happens only for the handful of docs that survive the merge.

### Why deep pagination is expensive — and search_after

Now the trap. Suppose a buyer asks for page 1,000 with `size=20`, i.e. `from=20000`. Each shard must return its **top `from + size` = 20,020** results, because the coordinator cannot know which shard holds global results 20,000–20,020 without seeing each shard's top 20,020. So:

```
requested: 20 results (page 1000)
each of 600 shards returns: 20,020 hits
coordinator sorts: 600 × 20,020 ≈ 12,000,000 hits
... to discard all but 20.
```

Memory and CPU on the coordinator grow with `from`, and it grows **× number of shards**. This is why ES caps `from + size` at `index.max_result_window` (default **10,000**) and why the brief mandates `search_after` for anything beyond page ~50 (brief §3).

**`search_after` internals.** Instead of an offset, you pass the **sort values of the last document on the previous page** as a cursor. The next query becomes "find docs whose sort tuple is *after* this tuple." Each shard can then use its sorted structures to seek directly to that point and return only the next `size` results — **no growing offset, constant per-shard cost regardless of page depth.**

```jsonc
// page N+1: hand back the sort tuple of the last hit from page N
GET /products-v7/_search
{
  "size": 20,
  "query": { ... },
  "sort": [ { "_score": "desc" }, { "listing_id": "asc" } ],  // tie-breaker = unique field
  "search_after": [ 12.734, "L-99271" ]                        // cursor from last hit
}
```

The tie-breaker on a **unique** field (`listing_id`) is mandatory — without a total ordering, "after this point" is ambiguous and you get duplicates or gaps across pages. This is exactly the cursor token the brief's API surface exposes (§3).

### Aggregations in scatter-gather (facets)

Facet counts (brief FR3) ride the *same* scatter-gather, but they aggregate rather than rank. Each shard computes a **partial aggregation** over its own docs — for a `terms` facet on `brand`, that's a per-shard count per brand read from the `brand` **doc_values** column (§8.3). The coordinator then performs a **reduce**, summing partial counts across all 600 shards into the final facet.

There is a subtle correctness gotcha worth mentioning in an interview: for high-cardinality `terms` aggregations each shard returns only its *top* `shard_size` buckets (not every brand), so a brand that ranks #12 on many shards but #1 nowhere can be undercounted or missed — the `doc_count_error_upper_bound` in the response quantifies this. GlobalMart raises `shard_size` for the brand/category facets where exactness matters and accepts approximate counts on long-tail attribute facets. Cardinality aggregations go further and use the **HyperLogLog++** sketch — approximate by design, merged across shards cheaply.

### A worked latency walkthrough

Tie it to the brief's budget ("ES query (scatter-gather) 120 ms"). A typical GlobalMart search — `q=wireless earbuds`, filters `market:US`, `in_stock:true`, `category:Electronics/Audio`, sorted by relevance, `size=24` with brand/price facets — flows as:

```
t=0    coordinating node parses & rewrites the query
t≈2ms  scatter to one copy of all 600 shards (fan-out, parallel)
       each shard, concurrently:
         - resolve terms via FST, intersect postings (leapfrog)
         - apply cached filter bitsets (market/in_stock/category)  ← node query cache hit
         - BM25-score survivors, keep local top-24
         - compute partial brand/price facets from doc_values
t≈90ms slowest shards' QUERY responses arrive (tail-latency bound)
       coordinator merges 600×top-24 → global top-24; reduces facets
t≈95ms FETCH: pull _source for the 24 winners from their shards
t≈118ms coordinator assembles hits + facets → returns
```

The wall-clock is dominated by the **slowest shard**, not the average — a scatter-gather is only as fast as its tail. This is why GlobalMart cares about shard *balance* (even doc distribution), guards against a single overloaded data node, and can set `allow_partial_search_results` so one slow/failed shard degrades the result set rather than failing the whole query (a graceful-degradation lever revisited in Chapter 10). It is also why 600 shards is a ceiling, not a floor: doubling shard count doubles the fan-out and the tail-latency exposure without helping a query that was never CPU-bound per shard.

### dfs_query_then_fetch

A scoring wrinkle covered next.

---

## 8.8 Scoring: TF-IDF → BM25

Relevance (NFR5: NDCG/conversion) begins with the per-shard textual score. ES's default is **BM25** (Best Matching 25), the successor to classic **TF-IDF**.

### From TF-IDF to BM25

TF-IDF scores a term in a document as `tf × idf`: more occurrences (term frequency) and rarer terms (inverse document frequency) score higher. Its weaknesses: term frequency grows without bound (a title that says "phone phone phone phone" shouldn't score 4× a natural one), and it doesn't cleanly account for document length. **BM25** fixes both with **saturation** and **length normalization**.

The BM25 score of document *D* for query *Q*:

```
                     n
score(D, Q)  =  Σ   IDF(qᵢ) · ────────────── f(qᵢ,D) · (k₁ + 1) ──────────────
                    i=1                   f(qᵢ,D) + k₁ · (1 − b + b · |D| / avgdl)

where
  f(qᵢ, D) = term frequency of query term qᵢ in D
  |D|      = length of D (in terms);   avgdl = average doc length in the shard
  IDF(qᵢ)  = ln( 1 + (N − n(qᵢ) + 0.5) / (n(qᵢ) + 0.5) )
  N        = number of docs in the shard;  n(qᵢ) = docs containing qᵢ
  k₁       = term-frequency saturation   (default 1.2)
  b        = length normalization        (default 0.75)
```

Read the two parameters physically:

- **k₁ (saturation, default 1.2).** Controls how quickly extra occurrences stop helping. As `f → ∞`, the TF term asymptotes to `k₁ + 1`. So the 1st occurrence of "earbuds" matters a lot, the 10th barely moves the score. Higher k₁ = slower saturation (TF matters more); lower k₁ = faster saturation.
- **b (length normalization, default 0.75).** Controls how much a document's length penalizes it. `b=1` fully normalizes by length (a match in a long description is heavily discounted vs. a short title); `b=0` ignores length entirely. `|D| / avgdl` is precisely what the **norms** (§8.3) encode.

BM25 is the default because it is robust, requires no per-corpus training, and its saturation behavior matches human relevance intuition far better than raw TF-IDF.

### A worked BM25 number

Take the query `earbuds` against a shard where `N = 16,000,000` docs and `n(earbuds) = 320,000` docs contain the term (so it is moderately common). First the IDF:

```
IDF = ln(1 + (16,000,000 − 320,000 + 0.5) / (320,000 + 0.5))
    = ln(1 + 15,680,000 / 320,000) ≈ ln(1 + 49) = ln(50) ≈ 3.91
```

Now two candidate listings, with `avgdl = 12` terms, defaults `k₁=1.2`, `b=0.75`:

- **Doc A** — a tight title "Wireless Earbuds": `f=1`, `|D|=2`.
  `norm = 1 − 0.75 + 0.75·(2/12) = 0.375`.
  `tf-part = 1·(1.2+1) / (1 + 1.2·0.375) = 2.2 / 1.45 = 1.517`.
  `score = 3.91 × 1.517 ≈ 5.93`.
- **Doc B** — a stuffed title repeating the term, "Earbuds earbuds earbuds bluetooth wireless sport gym running case ... (30 terms)": `f=3`, `|D|=30`.
  `norm = 1 − 0.75 + 0.75·(30/12) = 2.125`.
  `tf-part = 3·2.2 / (3 + 1.2·2.125) = 6.6 / 5.55 = 1.189`.
  `score = 3.91 × 1.189 ≈ 4.65`.

Even though Doc B mentions "earbuds" three times, BM25 scores the concise Doc A *higher* — saturation (`k₁`) blunts the repetition and length-norm (`b`) penalizes the padding. This is exactly the anti-keyword-stuffing behavior GlobalMart wants, and lowering `b` toward 0.4 on `title` (below) softens the length penalty for legitimately descriptive titles without rewarding spam.

### GlobalMart BM25 tuning

Product search is **not** long-form document search. Titles are short and keyword-dense; a listing padded with repeated keywords ("case cover case protector case") is usually **spam**, not relevance. So GlobalMart tunes BM25 per field:

- On `title`, we often **lower b** (e.g. 0.3–0.5): titles are naturally short and we don't want to over-penalize a slightly longer, more descriptive title.
- On `title`, we may **lower k₁** to saturate term repetition faster, blunting keyword-stuffing spam.

```jsonc
PUT /products-v8
{ "mappings": { "properties": {
    "title": {
      "type": "text",
      "similarity": "title_bm25"
    } } },
  "settings": { "index": { "similarity": {
    "title_bm25": { "type": "BM25", "k1": 0.9, "b": 0.4 }
  } } } }
```

Crucially, BM25 is only the **textual base score**. GlobalMart's final ranking blends it with business signals — `popularity`, `rating`, `seller_rating`, price competitiveness, availability — via `function_score` and the LTR reranker (Chapter 7). BM25 gets you the right *candidates*; the reranker orders them for conversion.

### Per-shard IDF and dfs_query_then_fetch

Here is a subtlety that separates seniors from juniors. Notice `N` and `n(qᵢ)` in the IDF formula are **per shard**. Each shard computes IDF from *its own* term statistics. If a term is unevenly distributed across shards (say "Xiaomi" is common in one shard and rare in another), the *same* document could score differently depending on which shard it landed in. With uniform random routing over 600 large shards this skew is usually negligible — the law of large numbers smooths it out.

When it does matter (small indices, skewed routing, relevance A/B tests), use **`dfs_query_then_fetch`**: a preliminary **DFS (Distributed Frequency Search)** round gathers *global* term statistics across all shards first, then scores every shard with consistent global IDF. It costs an extra round trip, so GlobalMart uses default `query_then_fetch` in production (600 big shards → negligible skew) and reserves `dfs_query_then_fetch` for offline relevance evaluation where exact scores matter.

### The _explain API

When someone asks "why is this cheap knock-off ranking above the brand-name product?", `_explain` decomposes the score into every contributing factor — IDF, TF, norms, boosts — for one document against one query. It is the debugging tool for relevance work and the honest answer to "how would you debug a ranking bug?" in an interview.

```jsonc
GET /products-v7/_explain/L-99271
{ "query": { "match": { "title": "wireless earbuds" } } }
// → tree: BM25 contribution of "wireless" + "earbuds", each with idf, tf, and norm terms
```

---

## 8.9 Caches

Three distinct caches sit on the read path; conflating them is a common mistake. Each caches a different thing at a different granularity.

| Cache | Scope | Caches | Keyed by | Invalidated by |
|---|---|---|---|---|
| **Node query cache** | Per node | **Filter bitsets** (which docs match a `filter` clause) | the filter query | segment change |
| **Shard request cache** | Per shard | Whole-request results: **aggregations, hits count, suggestions** | full request body (for `size:0` reqs) | refresh |
| **Fielddata cache** | Per node (heap) | In-heap uninverted values for **`text` field** sort/agg | field | segment change / eviction |

### Node query cache — filter bitsets

When you run a `filter` clause (`in_stock: true`, `category_path: Electronics/Phones`), Lucene computes a **bitset** marking which documents in each segment match, then caches that bitset in the node query cache. The next query with the same filter reuses the bitset — no re-evaluation. Because filters don't affect scoring, their results are perfectly cacheable and reusable across queries. This is the concrete reason Chapter 7 insists on putting structured constraints in `filter` (cacheable, no scoring) rather than `must` (scored, not cached). For GlobalMart's facet-heavy traffic — nearly every query carries `in_stock`, `market`, `category` filters — the query cache hit rate is high and it is one of our biggest latency wins.

Note bitsets are **per segment**; a new segment (from refresh) means its bitset is computed fresh, but existing segments' cached bitsets remain valid because segments are immutable (§8.3). Immutability makes caching correct-by-construction.

### Shard request cache — aggregations and counts

The shard request cache stores the **result of an entire search request** on a per-shard basis, but only for requests with `size: 0` (i.e. you want aggregations/facet counts or a total hit count, not the hits themselves). GlobalMart's **facet counts** (brief FR3 — brand/category/price-range counts shown next to results) are the prime beneficiary: identical facet queries return instantly. It is keyed on the serialized request and invalidated whenever the shard **refreshes** (new data could change counts) — so on a 1s-refresh hot shard the cache lifetime is short, but at 100K QPS even 1 second of reuse for popular facet queries is enormous.

### Fielddata cache — and why doc_values replaced it

Historically, to sort or aggregate on a field you needed the values in a column-like form, but the inverted index is term→docs, not doc→value. Early Lucene **"uninverted"** the index at query time and held the result in the JVM heap — the **fielddata cache**. It worked but was an OOM factory: aggregating on a high-cardinality field could blow the heap.

**doc_values** (§8.3) solved this by writing the columnar representation to disk **at index time**, off-heap and mmap-backed. Since ES 2.x, doc_values are the default for all non-analyzed fields, and fielddata is **only** needed for the rare case of sorting/aggregating on an **analyzed `text`** field (which has no doc_values). GlobalMart never does that — we sort/facet on keyword and numeric fields (`price`, `rating`, `brand`, `category_path`), all doc_values-backed. So **fielddata is effectively disabled** for us; text fields have `fielddata: false` (the default), which prevents an accidental heavy aggregation on `description` from OOMing a data node. This is a deliberate safety decision, not just a default we inherited.

Backing all of this are the **circuit breakers** — heap accounting guards (the `fielddata`, `request`, and parent breakers) that reject a query *before* it can exhaust the heap, turning a would-be node crash into a single failed request. Because our sort/agg workload lives in off-heap doc_values and disk-backed caches, GlobalMart's heap pressure comes mostly from coordinating-node result merging and aggregation reduce, not from fielddata — another reason we run **dedicated coordinating nodes** (§8.10) so a giant facet reduce trips a breaker there rather than jeopardizing a data node serving writes.

### Cache locality and the filesystem cache

The fourth, unnamed "cache" is the OS **page cache**, and it may be the most important of all. Because segments are immutable, mmap-ed files, the operating system caches hot segment pages in RAM transparently. This is why GlobalMart sizes data nodes with substantial RAM *beyond* the JVM heap (heap capped at ~31 GB for compressed oops; the rest left to the filesystem cache) and why we describe the cluster as **QPS-bound** (brief §2): serving 100K QPS at p99 200 ms depends on the working set of hot segments living in page cache, not on disk IOPS. The hot/warm tier split (§8.10) is fundamentally a page-cache economics decision — keep the popular locales' segments resident on high-RAM hot nodes; let cold locales fall back to disk on cheaper warm nodes.

---

## 8.10 Cluster Coordination

Everything so far lives inside data nodes. But *someone* must track which shards exist, where each copy lives, what the mappings are, and who is in charge. That is the **cluster coordination** layer.

### Node roles

GlobalMart uses **dedicated node roles** (brief §6) rather than letting every node do everything — role separation is what keeps a heavy query or a GC pause on one tier from destabilizing the rest.

| Role | Count | Job | Why dedicated |
|---|---|---|---|
| **Dedicated master** | **3** | Own the cluster state; elect a leader; publish metadata changes. Hold **no data**, serve **no queries**. | Isolating the master from data/query load prevents a busy data node from stalling cluster management — the classic cause of cascading failures. |
| **Coordinating** | several | Receive client requests, scatter-gather (§8.7), merge results. Hold no data. | Offloads the CPU/RAM of result merging from data nodes; a natural place to sit behind the load balancer. |
| **Data — hot** | ~60–100 | Recent/popular listings on fast SSDs; serve the bulk of the 100K QPS + all writes. | QPS-bound sizing (brief §2). |
| **Data — warm** | fewer | Older/less-popular locales on cheaper/denser storage; read-mostly, often force-merged. | Cost tiering; warm data is rarely queried and never written. |

### Cluster state

The **cluster state** is a single versioned metadata document the master maintains and replicates to every node. It contains: the list of indices and their mappings/settings, the **routing table** (which shard copy is on which node, and its status), node membership, and index templates/aliases (like `products → products-v7`). It is *not* your data — it is the map of where the data is and how it's shaped.

The master is the **only** node allowed to *mutate* cluster state. When a mapping changes, a shard relocates, or a node joins/leaves, the master computes the new state, increments its version, and **publishes** it in a two-phase commit: it sends the new state (or a diff — only the changed portion, to save bandwidth) to every node, waits for a quorum of master-eligible nodes to acknowledge, then sends a *commit* message that makes the new state active everywhere. Keeping cluster state small matters: this is why thousands of tiny shards are harmful — each shard is entries in the routing table that the master must track, diff, and republish on every change. At GlobalMart's 1,200 shards this is comfortable; at 100,000 shards the master would spend its life shuffling cluster state and publication would become a cluster-wide bottleneck. This is a concrete argument for the ~40 GB shard sizing (fewer, bigger shards) beyond just recovery speed.

### Master election and quorum (voting configuration)

Only one master may be active; two would be **split-brain** (divergent metadata → data loss). Modern Elasticsearch (7.0+, the Zen2 / `coordination` layer) prevents this with a **voting configuration** and strict quorum:

- The master-eligible nodes maintain a **voting configuration** (the set of nodes whose votes count). A cluster-state change or a master election requires a **strict majority (quorum)** of that voting config to agree.
- With **3 dedicated masters**, quorum = **2**. The cluster tolerates the loss of **1** master-eligible node and keeps functioning; lose 2 and the remaining node **refuses** to act as master (it cannot achieve quorum) — it would rather stop than risk split-brain.

**Why exactly 3, and why odd?** A quorum needs a strict majority. With 2 nodes, quorum is also 2 — so losing either one halts the cluster (no fault tolerance, worse than a single node). With 3, you tolerate 1 failure. Even counts waste a node without improving fault tolerance (4 nodes still only tolerate 1 loss before quorum risk). Hence the canonical **3 dedicated masters** — the smallest configuration that tolerates a single master failure. During a network partition, only the side holding ≥2 master-eligible nodes can elect a master and accept writes; the minority side goes read-only or unavailable. This is Elasticsearch **explicitly choosing consistency of metadata (CP) even though the data/search path is AP** (brief NFR6) — an important nuance: ES is AP for search results but CP for cluster metadata. We revisit split-brain and partition behavior in Chapter 10.

---

## 8.11 Analysis Internals: Index-Time vs. Search-Time

The last internal ties everything to text. An **analyzer** turns a string into the terms that actually go into (or query against) the inverted index. It is a pipeline:

```
raw text → [char filters] → [tokenizer] → [token filters] → terms
"Wireless Earbuds!" → strip punct → split on whitespace → lowercase, stem →
                                                      ["wireless", "earbud"]
```

- **Character filters** — pre-tokenization cleanup (strip HTML, map `&`→`and`).
- **Tokenizer** — split the stream into tokens (standard/whitespace/`edge_ngram`).
- **Token filters** — transform tokens: `lowercase`, `stop` (drop "the/a"), `stemmer` (`earbuds→earbud`), `synonym` (`sneakers↔trainers`, brief FR6), `asciifolding` (`café→cafe`).

### The cardinal rule: index-time and search-time analysis must be compatible

Analysis runs **twice**:

1. **Index time** — when a listing is indexed, `title` is analyzed and the resulting *terms* are what gets stored in the postings.
2. **Search time** — when a query hits `title`, the query string is analyzed the **same way**, and the resulting terms are matched against the postings.

If the two disagree, matching silently fails. If index-time lowercases but search-time doesn't, a query for `"iPhone"` produces the term `iPhone` which will never match the indexed term `iphone`. If index-time stems `earbuds→earbud` but the search analyzer doesn't, `"earbuds"` won't match. **The terms produced on both sides must line up.**

This is why ES lets you set them independently but usually you shouldn't:

```jsonc
"title": {
  "type": "text",
  "analyzer": "gm_english",          // index time
  "search_analyzer": "gm_english_search"   // search time (often identical)
}
```

### The one place they *deliberately* differ — synonyms

The classic asymmetry: **expand synonyms at search time, not index time.** If you bake `sneakers→[sneakers,trainers]` into the index, you (a) bloat the index and (b) can never change your synonym list without a full reindex. Applying synonyms only in the `search_analyzer` means synonym updates take effect immediately without touching stored data — exactly what GlobalMart wants for a synonym list that merchandising teams tweak weekly (Chapter 7's query-understanding stage).

Similarly, **autocomplete** uses `edge_ngram` at *index* time (`"iphone"` → `i, ip, iph, ipho, ...`) but a plain analyzer at *search* time (you don't want to n-gram the user's query too, or `"ip"` would match wildly). Mismatched-on-purpose, and correctly so. This asymmetry is a favorite interview question precisely because it looks like a bug but is the intended design.

### Per-locale analysis at GlobalMart scale

GlobalMart spans many markets and locales (brief: `locale`, `market` fields), and analysis is inherently **language-specific**: stemming, stop words, and tokenization all differ. German compounds (`Bluetoothkopfhörer`) need decompounding; Japanese and Chinese have no whitespace and need a dictionary/morphological tokenizer (kuromoji, ICU); English stemming rules are useless for French. There is no single analyzer that serves all locales well.

The practical pattern is a **per-locale analyzed sub-field** using a language analyzer chosen at mapping time, so the German title is stemmed with German rules and queried with the same. A `multi-fields` mapping keeps the language variant alongside the keyword and generic forms:

```jsonc
"title": {
  "type": "text",
  "analyzer": "gm_generic",                 // language-neutral fallback
  "fields": {
    "raw":  { "type": "keyword" },           // exact, for sort/facet/collapse
    "de":   { "type": "text", "analyzer": "german" },
    "ja":   { "type": "text", "analyzer": "kuromoji" }
  }
}
```

At search time the query-understanding stage (Chapter 7) picks the field matching the buyer's `locale` (`title.de` for a German buyer), guaranteeing index-time and search-time analysis agree *within* that locale. This is a direct, concrete instance of the cardinal rule operating at scale — and a reminder that "just use the standard analyzer" is a red flag answer for a genuinely global catalog.

---

## 8.12 Bringing It Together: Every Internal Maps to a GlobalMart Decision

| Internal (this chapter) | GlobalMart decision it drives |
|---|---|
| Shard = one Lucene index | 600 primaries = 600 Lucene indices; alias `products→products-vN` |
| FST terms dict, sorted & in-memory | Fast prefix enumeration → viable `edge_ngram` typeahead |
| Segment immutability | Lock-free reads at 100K QPS; correct-by-construction caches |
| doc_values (columnar) | Sort/facet on `price`/`rating`/`brand` without heap blowups |
| Refresh = visibility (1s) | NRT freshness (FR8); `refresh_interval:-1` during bulk reindex |
| Flush/commit + translog | Durability; but `translog.async` OK since ES is rebuildable from Kafka |
| Merge cost | Force-merge warm read-only indices to 1 segment; throttle on hot tier |
| Update = delete + reindex | Coalesce/batch the 100K/s price-inventory mutations (Chapter 6) |
| `murmur3(_id) % primaries` | Primary count frozen → reindex-to-grow via versioned indices |
| Query-then-fetch, `from` blowup | `search_after` cursors beyond page ~50 (API §3) |
| BM25 k₁/b | Lower b/k₁ on `title` to fight keyword-stuffing spam |
| Per-shard IDF | Default `query_then_fetch` (600 big shards → negligible skew) |
| Filter bitset cache | Push `in_stock`/`market`/`category` into `filter` clauses |
| Fielddata → doc_values | `text` fields `fielddata:false` → can't OOM on `description` agg |
| 3 dedicated masters + quorum | CP metadata layer; tolerate 1 master loss; no split-brain |
| Index-time = search-time analysis | Synonyms & edge_ngram deliberately search-time / index-time only |

---

## Interview Tips

- **Lead with the one-liner.** "A shard is a complete Lucene index; ES is a distributed layer over many of them." Establishing that boundary early makes every later answer coherent and signals you know where Lucene ends and ES begins.
- **Nail refresh vs. flush vs. commit — verbatim.** This is the single most common ES-internals question. Say: *refresh* creates a searchable segment in the filesystem cache (visibility, ~1s, the source of NRT); *flush* fsyncs segments and writes a Lucene *commit* point and truncates the translog (durability). Getting this precise instantly reads as senior.
- **Explain NRT from first principles**, not as a magic number: docs are searchable only after the next refresh creates a segment; default 1s ⇒ ~1s freshness. Then connect it to the tuning knob (`refresh_interval:-1` during bulk loads) to show operational judgment.
- **"Updates are cheap" is a trap.** Volunteer that an update is a delete + reindex of the *whole* document, that deletes are soft (tombstone in the live-docs bitset), and that space is reclaimed only on merge. Then tie it to GlobalMart's 100K/s price mutations and why we batch them.
- **Derive why deep pagination is expensive** with the arithmetic (each shard returns `from+size`; 600 shards × 20,020 ≈ 12M hits to return 20). Then present `search_after` with a unique tie-breaker as the fix. Interviewers love the number.
- **Know the routing formula and its consequence.** `murmur3(_routing) % num_primaries` ⇒ primary count is immutable ⇒ reindex to grow. That causal chain is a compact demonstration of deep understanding.
- **Write the BM25 formula and interpret k₁ and b physically** (saturation and length normalization), then say why product titles want lower b/k₁. Bonus: mention per-shard IDF and `dfs_query_then_fetch`.
- **Distinguish the three caches** — filter bitsets (node query cache) vs. request/agg results (shard request cache) vs. fielddata — and explain doc_values replacing fielddata. Conflating them is a common tell.
- **On coordination, say "3 dedicated masters, quorum = 2, tolerate 1 loss, no split-brain,"** and note ES is **CP for metadata but AP for the search path.** That nuance is memorable.
- **Common traps to avoid:** claiming ES updates are in-place; saying replicas are only for HA (they also serve read QPS); forgetting the `search_after` tie-breaker must be unique; expanding synonyms at index time; force-merging a live index.

## Key Takeaways

- **Shard = one Lucene index.** ES distributes, routes, replicates, and scatter-gathers; Lucene does the actual indexing, storing, and scoring. Keep the boundary crisp.
- The **inverted index** (terms dict as an in-memory **FST** + delta-compressed postings with positions and TF) is why term lookup is microseconds; **doc_values** give a columnar path for sort/facet without touching the heap.
- **Segments are immutable.** That single fact yields lock-free reads, cache correctness, and cheap replication — at the cost of many segments (→ merging) and no in-place edits (→ soft-deletes).
- **Refresh (visibility, ~1s → NRT) ≠ flush (durability, fsync + Lucene commit + translog truncate).** The translog provides durability between flushes.
- **Update = delete (tombstone in live-docs bitset) + reindex whole doc; merge reclaims space.** This makes GlobalMart's price/inventory churn expensive and drives batching + external versioning.
- **`murmur3(_id) % num_primary_shards`** fixes the primary count at creation, forcing the **alias + versioned-index reindex** pattern to grow. Replicas serve both HA and read QPS.
- **Query-then-fetch** distributes scoring cheaply; **`from` deep pagination scales with shard count** and is replaced by **`search_after`** cursors with a unique tie-breaker.
- **BM25** (with `k₁` saturation and `b` length-norm, per-shard IDF) is the tunable textual base score; GlobalMart lowers `b`/`k₁` on titles and layers business-signal reranking on top.
- **Three caches, three jobs:** node query cache (filter bitsets), shard request cache (aggs/counts), fielddata (legacy text sort/agg, superseded by doc_values).
- **3 dedicated masters + quorum voting** keep cluster metadata consistent (CP) and split-brain-free, even though the search path itself is AP.
- **Index-time and search-time analysis must produce compatible terms** — with synonyms (search-time) and `edge_ngram` (index-time) as the deliberate, correct exceptions.
