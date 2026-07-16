# Chapter 11 — Trade-offs

Every chapter before this one made choices. This chapter is where we own them out loud.
A system design is not a pile of correct answers — it is a *coherent set of trade-offs* that
all point the same direction. GlobalMart's north star is fixed by the brief: search p99 ≤ 200 ms,
99.99% availability on the read path, 10B documents, ~100K peak QPS, NRT freshness, and an
explicit acceptance that **the index is eventually consistent** (§1, NFR6). Almost every
decision below falls out of taking that priority ordering seriously and refusing to hedge.

The senior skill on display here is not knowing that Elasticsearch exists. It is being able to
say "we chose X over Y, we paid cost Z for it, and here is the specific condition under which we
would choose differently." That last clause — the *reversal condition* — is what separates a
decision from a preference. We state one for every trade-off.

---

## 11.1 CAP positioning: AP on the search path, CP where money moves

The first and most consequential decision. Under a network partition, a distributed system can
guarantee **C**onsistency or **A**vailability, not both. Which do we sacrifice?

For the **search read path, GlobalMart chooses AP** — availability and partition tolerance,
with eventual consistency of the index. Concretely: if a replica is momentarily behind, or a
data node is cut off, we would rather serve *slightly stale* results (a product whose price is
2 seconds old, a listing that was edited but not yet reindexed) than serve an error or block the
request. A buyer typing "wireless earbuds" does not know or care that one shard is 800 ms behind;
they care that results appear. **Serving stale-but-fast beats fresh-but-down** on a
revenue-critical read path where every added 100 ms measurably drops conversion.

This is also just *what the datastore is*. Elasticsearch is an AP-leaning system: replicas are
refreshed asynchronously, and the cluster favors returning partial results (`_shards.failed > 0`
with `partial: true`) over failing the whole query. We are not fighting the tool; we are choosing
a tool whose consistency model matches the requirement.

| Path | Data | CAP stance | Why |
|---|---|---|---|
| Search / autocomplete | ES index (derived) | **AP**, eventually consistent | Availability + latency are revenue; staleness of seconds is invisible to buyers |
| Catalog writes (listing edits) | Transactional store | **CP** | Source of truth must not lose or corrupt seller data |
| **Price at checkout** | Pricing / order service | **CP**, read-through | Charging the wrong price is a financial + legal event |
| **Inventory decrement** | Order service, transactional | **CP** (strong / linearizable) | Overselling one unit to two buyers is unacceptable |

The nuance that earns senior points: **CAP is chosen per data flow, not per system.** The *search
index* is AP, but the *facts it caches* have different owners. We happily show an AP, possibly-stale
price and stock badge on the search results page — it is a hint, not a promise. The moment the buyer
adds to cart or checks out, we re-read price and inventory from their CP source of truth inside a
transaction. The search index says "probably $799, probably in stock"; the order path says "exactly
$799, exactly 41 units, decrement atomically." Staleness on the search page is a UX footnote;
staleness at checkout is a lawsuit.

> **Reversal condition:** if search were the system of record for a regulated attribute (say, a
> legally-binding displayed price with no checkout re-read), we would be forced toward CP and a
> synchronous write path — and we would then miss the 200 ms p99. We avoid that by design: keep
> the authoritative read on the CP path, keep search advisory.

---

## 11.2 Consistency: eventual consistency and read-your-writes

Accepting AP means accepting **eventual consistency**: an update to the Catalog service becomes
searchable only after it flows through CDC → Kafka → Indexing Service → ES bulk → refresh
(Chapter 6). Our target lag is NRT — seconds for content edits, ~1 s for price/inventory.

The classic failure this creates is the **read-your-writes** problem. A seller edits their
listing title, hits save, immediately searches for it, and… sees the old title (or their new
listing isn't there at all). The write succeeded in the source of truth, but the derived index
hasn't caught up. To the seller this looks like a lost edit — a trust-destroying bug even though
the system is behaving exactly as designed.

We do **not** solve this by making the whole pipeline synchronous — that would sacrifice the
throughput and availability we just fought for. Instead we mitigate at the edges:

| Mitigation | How | Cost |
|---|---|---|
| **Read-your-writes for the editor** | After a seller edit, the seller-facing UI reads the listing from the Catalog service (source of truth), not from ES | Extra read path; only for the author's own view |
| **Optimistic UI** | Show the edited listing immediately from the write response; don't round-trip search | None functionally; UI complexity |
| **Version echo** | Write returns `_version`; UI can poll/`refresh=wait_for` on that listing until the indexed version ≥ written version | Small latency on confirmation, not on search |
| **`?refresh=wait_for` on targeted writes** | For high-value single-doc edits, block the write ack until that doc is refresh-visible | Adds up to one refresh interval (~1s) to that write only — never bulk |

The key insight: **read-your-writes is a per-user, per-document guarantee, and we provide it only
where a human is watching** — the seller looking at their own edit. We do **not** promise it globally.
Buyer B does not need to see seller A's edit within one second; nobody's mental model is violated
by that. Scoping the guarantee narrowly is what keeps it affordable. Chasing global strong
consistency on a 10B-doc index to fix a UX papercut would be a textbook over-engineering mistake.

> **Reversal condition:** if a large fraction of searches were sellers immediately verifying their
> own edits, the "read from source of truth for the author" path would dominate and we'd invest in
> a proper CP seller-preview service.

---

## 11.3 CQRS: a search index separate from the transactional store

GlobalMart runs two stores for the same data: the **Catalog transactional store** (source of
truth) and the **Elasticsearch index** (derived, rebuildable). This is Command Query
Responsibility Segregation — writes go to one model, reads to another, with a pipeline between.

The alternative is to search directly against the transactional database (Postgres full-text,
or a search extension on the OLTP store). Tempting: one store, no pipeline, no consistency lag,
read-your-writes for free.

| | Search directly on OLTP store | Separate ES index (CQRS) — **chosen** |
|---|---|---|
| Consistency | Strong, read-your-writes free | Eventually consistent (seconds lag) |
| Read latency at scale | Poor — OLTP not built for inverted-index scans, faceting, relevance | Purpose-built; p99 ≤ 200 ms at 100K QPS |
| Relevance / typo / synonym | Crude to impossible | First-class (BM25, LTR, analyzers) |
| Scaling reads independently | Couples read load to write store | Scale query nodes without touching OLTP |
| Blast radius | Heavy search traffic can knock over the write store | Index outage never blocks catalog writes |
| Operational cost | One store | **Whole extra pipeline + second store + lag** |

At 100K QPS over 10B documents with faceting, typo tolerance, and business-signal ranking, the
OLTP option is not viable — full-text search on a transactional store falls over well before this
scale, and even if it held, it couldn't do relevance ranking. So CQRS isn't really optional here;
it's the price of admission. What matters is naming the cost honestly: **we now own an entire
indexing pipeline (Ch6), we accept eventual consistency (§11.2), and ES is never the source of
truth** — it is rebuildable from the Catalog store, which is exactly why we can treat it as
disposable during reindexing (alias `products` → `products-vN`, §6).

The decisive framing: CQRS **decouples the read and write models so each can be optimized and
scaled independently.** The transactional store optimizes for correctness and write integrity;
the index optimizes for retrieval and relevance at read scale. That decoupling is worth an extra
pipeline. It would not be worth it for a catalog of 100K products at 50 QPS — there, Postgres FTS
wins on simplicity (see §11.4).

---

## 11.4 Engine choice: why Elasticsearch

The brief fixes ES as the core engine, but a senior candidate must be able to justify it against
the field and know when each rival wins. Build-vs-buy is the meta-question underneath.

| Engine | Model | Strengths | Weaknesses | Wins when… |
|---|---|---|---|---|
| **Elasticsearch** ✅ | Self/managed, Lucene | Mature, huge ecosystem, faceting, aggregations, NRT, LTR plugin, hot/warm tiers, ops tooling | Operationally heavy; JVM/heap tuning; licensing (SSPL) | You need scale + facets + relevance control + a hiring pool. **GlobalMart.** |
| **OpenSearch** | Self/managed, Lucene fork | ES-compatible API, Apache-2.0, AWS-native | Slightly behind ES on features/vector maturity | You want ES capabilities with a permissive license / deep AWS integration |
| **Solr** | Self, Lucene | Battle-tested, strong faceting, mature | Smaller momentum; clunkier ops/scaling story | Existing Solr expertise; on-prem; heavy faceting workloads |
| **Vespa** | Self/managed | Best-in-class ranking + vector + structured at serving time; true real-time | Steep learning curve; smaller community | Ranking/ML-serving is *the* product (recsys, ad serving) at huge scale |
| **Algolia / Typesense** | Managed SaaS | Instant setup, superb latency, great DX | Cost at 10B docs; less control; data residency limits | Small–mid catalogs, tiny team, speed-to-market over control |
| **Postgres FTS** | Extension on OLTP | Zero extra infra, strongly consistent | No real relevance/facet scale; ties reads to write store | Catalogs up to ~1M docs, low QPS, no dedicated search team |
| **Raw Lucene** | Library | Maximum control, no overhead | You rebuild sharding, replication, cluster mgmt yourself | You are literally building the next ES; almost never the right call |
| **Vector DB** (Pinecone, Weaviate, Milvus) | ANN store | Semantic / embedding retrieval | Weak at exact filters, facets, keyword precision alone | Semantic search — as a *complement* to lexical, not a replacement |

**Why ES for GlobalMart.** At our scale we need four things simultaneously: (1) inverted-index
retrieval over 10B docs, (2) rich faceting/aggregations with counts (FR3), (3) tunable relevance
including LTR reranking (FR7), and (4) proven horizontal scaling with hot/warm tiering (Ch9). ES
is the option that does all four at 100K QPS with a large hiring pool and mature operational
tooling. OpenSearch is the credible substitute (and our fallback if SSPL licensing became a
blocker — that is the honest reversal condition). Vespa is arguably *technically* superior for
pure ranking, but it trades away ecosystem and staffing — a real risk at an org that must hire and
operate this for years. Algolia/Typesense are ruled out by cost and control at 10B docs, though
they'd be the right call for a startup. Postgres FTS is ruled out by scale (§11.3). Vector DBs are
additive, not a base — semantic retrieval augments lexical, so we'd layer ANN vectors *inside* ES
(dense_vector) or alongside it, never instead of it.

**Build-vs-buy:** we buy the engine (ES/OpenSearch) and build the *pipeline and relevance* around
it. Building a search engine from raw Lucene would be re-implementing a solved problem; buying a
fully-managed SaaS would surrender the relevance control and unit economics that are core to an
e-commerce business where a 1% NDCG gain is real revenue. The middle path — managed engine,
owned relevance — is the senior choice.

---

## 11.5 Freshness vs indexing cost

FR8 wants new/updated listings searchable in seconds and price/inventory within ~1 s. Freshness
is not free: in Lucene, documents become searchable only after a **refresh** creates a new
searchable segment, and frequent refreshes plus frequent merges burn CPU and I/O.

| Knob | Fresher | Cheaper |
|---|---|---|
| `refresh_interval` | 1s (hot index) | 30s+ (bulk backfill, warm tier) |
| Indexing mode | NRT per-event (streaming) | Periodic batch |
| Signal denormalization | Inline every signal update | Recompute signals offline |

Our decision: **tiered freshness.** The hot index serving live buyer traffic runs
`refresh_interval: 1s`. During bulk reindexing (`products-vN` builds) we set `refresh_interval: -1`
and a big bulk size, then restore 1s and warm up — reindexing a 10B-doc index with 1s refreshes on
would be ruinously slow. Warm-tier indices (older/less-popular locales, §6/Ch9) tolerate a higher
interval because nobody's watching them per-second.

The subtle cost is **update amplification from denormalized signals.** Our document embeds
`popularity`, `seller_rating`, `price`, `inventory` (§4) directly so we can rank and filter without
joins. But `popularity` is a normalized sales/CTR signal that drifts constantly, and price/inventory
mutate at ~100K writes/sec (§2). Every such change is a *full document reindex* in Lucene — segments
are immutable, so an "update" is delete + re-add, feeding the merge machinery. A signal that changes
on every product for every buyer view could, naively, multiply our write volume many-fold.

Mitigations we adopt:
- **Batch signal recomputation.** `popularity` is recomputed offline (minutes/hours) and pushed in
  bulk, not per-click. Ranking does not need second-fresh popularity.
- **Separate the volatile from the stable.** Price and inventory — the ~1 s freshness fields — are
  the mutation firehose. We still reindex them into the main doc for filter/sort, but we consider
  them the dominant write cost and provision the pipeline for *their* rate, not the (much lower)
  content-edit rate.
- **Coalesce updates.** The Indexing Service dedupes by `listing_id` + `_version` within a window,
  so ten price ticks in a second become one reindex (Ch6).

> **Reversal condition:** if price volatility grew until reindexing dominated cluster CPU, we'd move
> price/inventory *out* of the searchable doc into a fast key-value side store, joined at query time
> or decorated after retrieval — trading the query-time cost of §11.7 for lower write amplification.

---

## 11.6 Relevance vs latency

Better ranking costs milliseconds; our budget allocates just 30 ms to ranking/rerank inside the
200 ms p99 (§2). Three sub-decisions.

**How deep to rerank (top-K).** Retrieval returns a broad candidate set with cheap BM25 + filters;
the expensive learning-to-rank model reranks only the **top K** (e.g. K=200–500), not the whole
result set. Reranking 10,000 candidates with a heavy model would blow the 30 ms budget; reranking
the top few hundred captures nearly all the NDCG gain because the truly relevant docs almost always
survive first-pass retrieval. **Rerank shallow, retrieve wide-enough.**

| Rerank depth | Relevance (NDCG) | Latency |
|---|---|---|
| Top 100 | Good | Cheapest |
| **Top 200–500** ✅ | Near-optimal | Fits 30 ms budget |
| Top 5,000+ | Marginal gain | Blows budget |

**Synchronous LTR vs precomputed.** We rerank **synchronously per query** because personalization
and query context (the actual query terms, applied filters, locale) can't be precomputed for every
combination. But we keep the model cheap enough to fit the budget (gradient-boosted trees / a
compact model, feature values already in `doc_values`), and we **precompute the static signals**
(popularity, seller_rating) offline so runtime only does the query-dependent combination. Fully
precomputed ranking (per-query cached order) doesn't work for a long-tail, personalized query space;
fully synchronous heavy neural reranking doesn't fit 30 ms. The middle — synchronous light model
over precomputed features — is the fit.

**Aggregation cost vs facet richness.** Facet counts (FR3) are aggregations, and aggregations over
high-cardinality fields (e.g. seller_id across billions of docs) are expensive. We cap facet richness:
compute counts for the facets users actually filter on (category, brand, price bucket, rating,
availability), use `doc_values` and eager global ordinals for those keyword fields, and avoid
exact high-cardinality counts where an approximate count is acceptable. Richer facets = more
aggregation CPU per query; we spend it only where it drives filtering behavior.

---

## 11.7 Denormalization vs joins / parent-child / nested

Our document is **denormalized** (§4): seller_rating, price, category_path, and attributes live
inside the listing, not in separate related documents. Elasticsearch offers alternatives — `nested`
documents, and `parent-child` (`join`) relations — that avoid duplicating shared data.

| Approach | Query-time cost | Update cost (amplification) | Use for |
|---|---|---|---|
| **Denormalized (flat)** ✅ | Cheapest — single doc, no join | High — a shared field change rewrites every listing that copies it | Fields read on every query; our default |
| `nested` | Moderate — nested query overhead, hidden sub-docs | Whole parent reindexed on any nested change | Arrays of objects needing independent matching (variant-level attributes) |
| `parent-child` (`join`) | Expensive — join executed at query time, must be same shard | Low — update child without touching parent | Rarely-queried, frequently-updated relations |

The trade is **update amplification vs query-time cost**, and at 100K QPS the read path is
sacred — so we bias hard toward denormalization and pay the write cost. Example: if a seller's
rating changes, and that seller has 50,000 listings, denormalization means reindexing 50,000
documents. That is real amplification, but it happens on the *write* path (async, batched,
off the SLA) and buys us join-free reads on the *revenue* path. We would rather amplify writes
than slow reads.

We use `nested` sparingly — only where an array of objects genuinely needs correlated matching
(e.g. matching color=black AND storage=128GB *within the same variant*, not across variants). We
essentially never use `parent-child`: the same-shard constraint and query-time join cost are
poison at our QPS. The one place we'd reconsider is exactly the §11.5 reversal — if a
super-high-cardinality seller changed a shared field constantly, a `join` (or an external side
store) would beat reindexing millions of children.

---

## 11.8 Push vs pull indexing

How does data get from the Catalog store into ES? **Push** (change-data-capture streaming: the DB
emits changes → Kafka → Indexing Service, §5) or **pull** (a job periodically queries "what changed
since T?" and re-indexes it).

| | Pull (periodic batch) | Push (CDC/streaming) — **chosen** |
|---|---|---|
| Freshness | Bounded by poll interval (minutes) | Seconds — meets FR8 / ~1 s |
| Load pattern | Spiky — every poll hammers the source | Smooth — continuous, backpressured |
| Missed updates | Needs reliable `updated_at` watermark; deletes are hard | Captures every change incl. deletes |
| Source coupling | Repeated heavy scans of the OLTP store | One CDC reader; no query load |
| Complexity | Simpler to build | Needs CDC infra + Kafka + ordering/versioning |
| Replay / recovery | Re-run the job | Replay from Kafka offset / DLQ (Ch10) |

We **push via CDC → Kafka** (canonical architecture, §5). The decisive factor is FR8: minute-level
poll intervals cannot deliver ~1 s price/inventory freshness, and polling for changes at 100K
writes/sec would batter the transactional store with scan load. CDC also naturally captures deletes
(a pull job keying on `updated_at` silently misses hard deletes) and, via Kafka, gives us durable
replay, ordering per `listing_id`, and a dead-letter queue for poison records (Ch10). The cost is
real — we operate CDC connectors and a Kafka cluster — but it is the only architecture that hits
the freshness SLA without melting the source of truth.

We still keep a **pull-based bulk path** for one job: full reindexing (`products-vN` rebuilds) reads
the Catalog store in bulk. So it's not push-*or*-pull; it's **push for the steady state, pull for
the periodic full rebuild.** Naming that hybrid is the mature answer.

> **Reversal condition:** for a low-freshness, low-write-rate dataset (e.g. a nightly-updated
> reference catalog), CDC is over-engineering — a scheduled batch pull is simpler and sufficient.

---

## 11.9 Cost vs performance

We could hit every SLA by brute force — provision for peak everywhere, three replicas, all-hot,
all SSD. We don't, because the economics don't survive contact with a 48 TB, 100K-QPS cluster.

**Hot/warm tiering.** Not all 10B documents are equal. Popular locales and recent listings serve
the overwhelming majority of queries; older/less-popular locales are rarely touched. We put the hot
set on fast, RAM-rich, expensive **data-hot** nodes and the cold tail on cheaper, denser, slower
**data-warm** nodes (§6/Ch9). This is the single biggest cost lever: paying premium hardware only
for data that earns it. The trade is a latency cliff for the rare warm-tier query — acceptable,
because it's rare and the p99 SLA is dominated by the hot path.

**Replica count.** One replica (1,200 total shards, §6) is our baseline: it gives us HA (a lost
node doesn't lose data or reads) and doubles read throughput for hot shards. A second replica would
add read capacity and resilience but **also add ~24 TB and a third of our node bill.** We instead
add replicas *selectively* to the hottest indices rather than globally — buy read throughput where
QPS actually concentrates. Remember the cluster is **QPS-bound, not storage-bound** (§2): we
provision 60–100 data nodes for query throughput and RAM, not disk, so replicas (which multiply
serving capacity) are the relevant lever, not raw storage.

**Over-provisioning for peak.** Peak is ~3× average (100K vs 35K QPS, §2). Provisioning the whole
fleet for 100K QPS 24/7 wastes ~two-thirds of the capacity most of the day. We instead:
- size the baseline for comfortably above average with headroom,
- **autoscale coordinating/query nodes** and lean on the **Result Cache (Redis, §5)** to absorb
  spikes (popular queries repeat heavily during peaks — cache hit rates rise exactly when we need
  them),
- keep enough always-on hot capacity that a scale-up event never *starts* from cold.

| Lever | Saves cost | Costs performance |
|---|---|---|
| Hot/warm tiering | Big — premium HW only for hot data | Latency cliff on rare warm queries |
| 1 replica vs 2 | ~⅓ of node bill + 24 TB | Less read headroom / one-node margin |
| Autoscale + cache vs static peak | ~⅔ idle capacity reclaimed | Scale-up lag; cache-miss risk on novel spikes |

The philosophy: **provision the steady state, absorb the spikes with elastic + cached capacity, and
never pay hot-tier prices for cold-tier data.** Chapter 9 has the numbers; the trade-off is that we
accept a small, bounded risk (a scale-up lag, a warm-query latency cliff) in exchange for a cluster
that is economically sane.

---

## 11.10 Deep pagination

The brief bans `from`+`size` beyond page ~50 and mandates `search_after` cursors (§3). Here's why,
and why we also make it a *product* decision.

In a scatter-gather engine, `from: 100000, size: 10` forces **every shard to build and return
100,010 sorted hits** to the coordinating node, which merges and discards all but 10. Cost grows
with page depth × shard count — a memory and CPU bomb, and ES enforces `index.max_result_window`
(default 10,000) precisely to stop this.

| Method | How | Cost | Good for |
|---|---|---|---|
| **`from` + `size`** | Skip N, take M | O(from × shards) memory; capped at 10K | Shallow pages (1–~50) |
| **`search_after`** ✅ | Cursor = sort values of last hit; "give me what comes after" | Flat cost per page, any depth | Deep, stateless, live-data pagination |
| **`scroll`** | Server holds a frozen snapshot cursor | Holds segments open — resource-heavy, point-in-time | Batch export/reindex, **not** live user paging |

We use **`from`+`size` for the first ~50 pages** (simple, and nobody needs a cursor for page 3) and
**`search_after` for anything deeper** — its cost is flat regardless of depth and it's stateless
(the cursor token is self-contained), which matters at 100K QPS where server-held `scroll` contexts
would pin resources across millions of concurrent sessions. `scroll` is reserved for internal batch
jobs (export, reindex), never buyer traffic.

The senior move is recognizing this is **also a product decision, not only a technical one.** We
**cap pagination depth** (e.g. ~a few hundred results / page ~50 in the UI). Justification: no buyer
meaningfully browses to result 8,000 — relevance has long since collapsed, and search behavior data
shows engagement falling off a cliff after the first pages. Deep pagination is overwhelmingly *bots
and scrapers*, not customers. Capping depth simultaneously (a) removes an entire class of expensive
queries, (b) shrinks abuse surface, and (c) costs real users essentially nothing. When the honest
answer is "make the requirement go away," a senior engineer says so — the best-optimized query is
the one you never have to run.

> **Reversal condition:** a genuine deep-scan use case (seller analytics, "export all my listings")
> is served by a *separate* cursor/scroll API on a warm path with its own quotas — not by lifting
> the cap on the buyer search endpoint.

---

## Interview Tips

- **Trade-offs are the interview.** Junior candidates list features; senior candidates say
  "I chose X over Y because of Z, and here's when I'd flip." Always volunteer the alternative you
  *rejected* and the cost you *accepted* — never present a choice as free.
- **Lead with CAP framed per-data-flow.** Say "search is AP, checkout is CP" and you've shown you
  understand that consistency is a property of a *flow*, not a company. This one line signals
  seniority faster than anything else.
- **Name eventual consistency before the interviewer does,** then immediately address
  read-your-writes for the seller. Interviewers *wait* to see if you notice the "seller edits and
  can't find their listing" trap. Noticing it unprompted is a strong signal.
- **Justify ES, don't assume it.** Have the comparison table in your head: OpenSearch (licensing),
  Vespa (ranking-first, staffing risk), Algolia (great for small, dies on cost at 10B), Postgres FTS
  (fine under ~1M docs). Knowing *when each rival wins* proves you chose rather than defaulted.
- **Denormalization vs joins is a favorite.** Answer with "amplify writes to protect reads" and give
  the seller-rating-changes-50K-listings example — concrete numbers beat hand-waving.
- **Deep pagination has a trapdoor answer:** the strongest response is "cap it, because real users
  don't go there and it's mostly bots." Showing you'll push back on a requirement is a senior tell.
- **Always state a reversal condition.** "We'd move price out of the doc if write amplification
  dominated CPU" shows you understand the decision is contextual, not dogmatic. Certainty *with*
  a stated escape hatch reads as judgment; certainty without one reads as inexperience.
- **Don't hedge.** "It depends" is only acceptable if you then say *on what*, and *which way you'd
  go by default.* Decisiveness anchored in reasoning is the whole game.

## Key Takeaways

- **AP on the search path, CP where money moves.** Serve fast, slightly-stale search results;
  re-read authoritative price and inventory inside a transaction at checkout. CAP is chosen
  per data flow.
- **Eventual consistency is a feature, not a bug** — but scope read-your-writes narrowly to the
  seller viewing their own edit (read from the source of truth), rather than chasing global strong
  consistency.
- **CQRS is the price of admission at scale:** a separate, rebuildable ES index decouples reads from
  writes and lets each be optimized independently; the cost is an entire pipeline plus lag, and ES
  is never the source of truth.
- **Elasticsearch wins on the intersection** of scale, faceting, relevance control, and hiring pool;
  OpenSearch is the licensing fallback, Vespa the ranking-purist alternative, Algolia/Postgres the
  small-scale answers, vector DBs a complement not a base.
- **Freshness is bought with indexing cost:** tier refresh intervals, batch volatile signals, and
  fear denormalized-signal update amplification.
- **Protect the read path:** rerank shallow (top-K), keep LTR synchronous-but-light over precomputed
  features, denormalize to avoid joins, and cap facet richness — spending the 30 ms ranking budget
  only where it moves NDCG.
- **Push (CDC/Kafka) for steady-state freshness, pull for full rebuilds** — the hybrid hits the ~1 s
  SLA without hammering the source of truth.
- **Provision the steady state, absorb spikes with autoscale + cache, never pay hot prices for cold
  data.** The cluster is QPS-bound; replicas and caching are the levers, not raw disk.
- **`search_after` over `from`+`size`/`scroll` for depth — and cap pagination depth as a product
  decision:** the query you never run is the cheapest one.
- **Every decision carries a reversal condition.** Knowing when you'd choose differently is what
  turns a preference into engineering judgment.
