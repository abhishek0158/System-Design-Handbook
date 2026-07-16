# Chapter 12 — Mock Interview

> *"The design is only half the score. The other half is watching you think out loud,
> negotiate scope, recover from a wrong turn, and know which trade-off you'd actually ship."*

This chapter is the payoff for Chapters 1–11. Everything you've read — the 10B-document
corpus, the ~100K peak QPS, the CDC → Kafka → Indexing Service write path, the
Elasticsearch scatter-gather read path, the LTR reranker, the hot/warm topology — now has to
come out of your mouth in **45 minutes**, in natural spoken English, while someone probes for
weak spots.

Below is a full transcript of a realistic senior-level interview for the prompt **"Design the
search system for a global e-commerce marketplace."** The candidate ("C") is strong but not
superhuman — they make a couple of mistakes and recover. The interviewer ("I") throws the
curveballs you'd actually get. Interleaved **💡 Commentary** blocks explain *why* an answer
earns signal and what a weaker candidate would have said. At the end there's a scorecard and a
practice plan.

---

## 12.1 The Interview Format

Most senior system-design loops at large companies run **45–50 minutes** of design plus ~5
minutes for your questions. A single interviewer plays both the "user" (who wants a product)
and the "adversary" (who wants to find the load-bearing wall you forgot). You are scored on a
rubric, not on whether you "got the answer" — there is no single answer.

**The rubric dimensions** (the ones this scorecard uses):

| Dimension | What it measures |
|---|---|
| Problem-solving | Do you decompose an open problem and drive it yourself? |
| Requirements | Do you scope before you build? Functional + non-functional + explicit out-of-scope. |
| Estimation | Can you derive load/storage/shard numbers and let them shape the design? |
| High-level design | Is there a coherent end-to-end architecture with clean component boundaries? |
| Deep dive | Can you go two or three levels down on demand (ES internals, ranking, CDC)? |
| Scaling | Do you handle 10B docs / 100K QPS / surges with concrete mechanisms? |
| Trade-offs | Do you name alternatives and justify choices, not just assert them? |
| Communication | Structure, whiteboard hygiene, signposting, handling pushback. |

**A workable 45-minute timeline** (the candidate follows this and *says so* up front):

| Time | Phase |
|---|---|
| 0–5 min | Requirements & scope |
| 5–10 min | Capacity estimation |
| 10–18 min | High-level design (both paths) |
| 18–33 min | Deep dives (indexing, search, ES internals) |
| 33–40 min | Scaling & failure handling |
| 40–45 min | Trade-offs, wrap-up, your questions |

> 💡 **Commentary:** Stating the timeline out loud in the first minute is a cheap, high-value
> signal. It tells the interviewer "I've done this before, I'll manage the clock, you won't have
> to drag me." Weak candidates dive straight into drawing boxes and are still in "high-level
> design" at minute 40 with no estimation and no deep dive. The clock is part of the test.

---

## 12.2 The Transcript

### Phase 1 — Requirements (0–5 min)

**Interviewer:** Let's design the search system for a global e-commerce marketplace. Think
Amazon or AliExpress scale. Where do you want to start?

**Candidate:** Great — before I draw anything I want to spend two or three minutes nailing
down scope, because "search" can mean five different products. Let me ask a few clarifying
questions and state assumptions as I go, and please correct me.

First, what are we searching? I'll assume the core object is a **product listing** — a
specific product offered by a specific seller in a specific locale. That distinction matters:
the same iPhone listed by 500 sellers across 30 markets is not one document, it's up to
15,000 documents. Is that the right unit?

**Interviewer:** Yes, listing-level is right. Why does that distinction matter to you already?

**Candidate:** Because it's the difference between an index of maybe a billion distinct
products and an index of **10-billion-plus listings**. It drives storage, shard count, node
count — basically every number downstream. I'd rather be wrong about it now than at minute 30.

Second batch of questions: what capabilities are in scope? I'll assume we need full-text
search over title/brand/description/attributes; autocomplete; filtering and faceting with
counts — category, brand, price, rating, availability; sorting by relevance, price, rating,
recency; pagination; spell-correction and synonyms — "iphn" should find iPhone, "trainers"
and "sneakers" should be interchangeable; and ranking that blends text relevance with
business signals like popularity and conversion. Am I missing anything you care about?

**Interviewer:** That's the set. What are you explicitly *not* doing?

**Candidate:** Out of scope, and I want to say this explicitly so we don't wander: checkout,
cart, payments, the recommendations home feed, fraud, and the seller onboarding UI. I'll
mention image/visual search as a future extension but not build it. Good?

**Interviewer:** Good. Non-functional requirements?

**Candidate:** Let me prioritize them, because they conflict and the priority order *is* the
design.

1. **Latency.** Search p99 ≤ 200 ms, autocomplete p99 ≤ 100 ms. Search is on the critical
   revenue path; slow search directly kills conversion.
2. **Availability.** 99.99% for the read path. Buyers not being able to search is lost money;
   this is stricter than the write path.
3. **Scalability.** 10B documents, ~100K peak QPS, must scale horizontally.
4. **Freshness.** New/updated listings searchable within seconds; price and inventory changes
   within about a second — you can't show "in stock $799" for something that's out of stock or
   now $899.
5. **Relevance quality**, measured by NDCG and downstream conversion, not just latency.

And a key stance: **the search index is allowed to be eventually consistent.** On the search
path I'll choose availability over strict consistency — AP over CP. Elasticsearch is a
*derived* view; the transactional catalog is the source of truth. Everyone okay with that
framing before I estimate?

**Interviewer:** Yes. Go estimate.

> 💡 **Commentary:** This is a textbook strong open. Three things earn signal. (1) The
> candidate identified the *document = listing* decision unprompted and connected it to
> downstream numbers — that's the single most important scoping insight in this problem, and
> weak candidates blow past it, silently assuming "product = document" and under-sizing the
> whole system by 10×. (2) They **prioritized** the NFRs instead of listing them flat. Saying
> "these conflict and the order is the design" is senior-level. (3) They stated AP-over-CP and
> source-of-truth explicitly and got buy-in. A weak candidate says "we'll use Elasticsearch"
> in minute one with no requirements — that reads as pattern-matching, not engineering.

---

### Phase 2 — Estimation (5–10 min)

**Candidate:** I'll derive load, storage, and cluster size. I'll round aggressively — the goal
is the right order of magnitude, not four significant figures.

**Traffic.** 2B registered users, ~500M daily active. Say each active user does ~6 searches a
day. That's **~3B searches/day**. Divide by 86,400 seconds ≈ **~35K average QPS**. Peak is
spiky — Black Friday, flash sales, time-zone bunching — so I'll use ~3× average ≈ **~100K peak
QPS** for search.

Autocomplete is heavier per search: every keystroke can fire a request, call it 5–8 per
search. So typeahead peaks around **~500K QPS**. I'm flagging that now because it means
autocomplete needs its *own* lighter-weight path — I won't run 500K QPS through the full
relevance pipeline.

**Interviewer:** Hold on — 500K QPS on autocomplete. Does that scare you?

**Candidate:** It changes the design, it doesn't scare me. Autocomplete is a prefix problem
over a bounded suggestion dictionary — millions of entries, not 10 billion. It's tiny, cacheable,
and has a huge hit rate because prefixes repeat massively across users. I'd serve it from a
dedicated completion structure — an ES `completion` suggester or a purpose-built prefix service
backed by edge-n-grams — fronted by aggressive caching. It does not touch the main product
shards. So the 500K number is real but cheap.

**Storage.** ~10B listings × ~2KB average document = **~20TB raw corpus**. In Elasticsearch
the on-disk footprint is bigger because we store the inverted index plus doc-values for
sorting and faceting — call it ~1.2× raw ≈ **~24TB of primaries**. With **one replica** for
HA and read throughput that's **~48TB total on disk**.

**Sharding.** Target shard size ~40GB — big enough to amortize overhead, small enough to
recover and relocate quickly. 24TB of primaries / 40GB ≈ **~600 primary shards**, and with one
replica, **~1,200 total shards**.

**Nodes.** Here's the part people get wrong. If I only cared about disk — 48TB at ~2TB of hot
data per node — I'd need ~25 nodes. But we're not disk-bound, we're **QPS-bound**: 100K
queries fanning out across 600 shards, each doing scoring work, needs a lot of CPU and page
cache/RAM. So I provision for query throughput, not disk. That lands around **~60–100 data
nodes** in the hot tier. I'll say it explicitly: *this cluster is sized for query throughput,
not storage.*

**Write path.** ~50M new listings/day, but the dominant write volume is price and inventory
mutations — those churn constantly. Peak catalog write rate is around **~100K writes/sec**.
Note the ratio: writes actually out-number searches roughly 3:1 by raw ops, but reads are the
SLA-critical, fan-out-heavy path. That asymmetry is why the indexing pipeline is asynchronous
and decoupled — I'll get there.

**Latency budget.** For the 200ms p99, I mentally allocate: edge/LB ~5ms, API + auth ~10ms,
query understanding ~15ms, the ES scatter-gather ~120ms, ranking/rerank ~30ms, serialization
~10ms, and ~10ms buffer. The ES query is the fat part of the budget, which tells me that's
where optimization effort and caching pay off most.

**Interviewer:** You said writes outnumber reads 3:1 but reads are more critical. Reconcile
that.

**Candidate:** Right — they're critical along different axes. Writes are high-*volume* but
individually cheap, latency-tolerant (a price update being searchable in 800ms vs 1500ms
doesn't break anyone), and can be batched, buffered, and retried. Reads are lower-volume but
each one is latency-*bound* at 200ms, fans out across all 600 shards, and directly gates
revenue. So I optimize the two paths for completely different things: throughput and
buffering on writes, tail-latency and availability on reads. That's the core reason they're
separate pipelines.

> 💡 **Commentary:** The estimation is doing real work — it's not decoration. Notice the
> candidate lets numbers *change the design*: 500K autocomplete QPS → separate path; QPS-bound
> not disk-bound → node count driven by CPU/RAM not TB. That "QPS-bound" observation is the
> exact insight the interviewer is fishing for; a weak candidate computes 48TB / 2TB = 24 nodes
> and stops, badly under-provisioning. Also strong: the candidate rounds and moves fast, and
> when challenged on the 3:1 write:read paradox they give a crisp two-axis answer instead of
> getting flustered. Deriving the canonical numbers from first principles (not reciting them)
> is what earns the estimation score.

---

### Phase 3 — High-Level Design (10–18 min)

**Candidate:** Let me draw the two paths. The single most important architectural idea here is
**separation of the write path from the read path**, connected only through the search index.
Let me sketch.

```
              ┌───────────────────  SEARCH (READ) PATH  ───────────────────┐
              │                                                             │
 Client ─▶ CDN/Edge ─▶ API Gateway ─▶ Search Service ──────────────────────┼─▶ ES
                          (auth,        │  ├─ Query Understanding            │  coordinating
                           rate-limit)  │  │   (spell, synonyms, tokenize)   │  nodes
                                        │  ├─ Result Cache (Redis)           │     │
                                        │  └─ Ranking / LTR reranker         │     ▼
                                        └──── results + facets ◀─────────────┼─ data nodes
                                                                             │  (hot / warm)
              └─────────────────────────────────────────────────────────────┘

              ┌───────────────────  INDEXING (WRITE) PATH  ─────────────────┐
              │                                                             │
 Catalog Service ─▶ CDC ─▶ Kafka ─▶ Indexing Service ─▶ ES _bulk API ─▶ ES  │
 (source of truth)          (log)   (transform, enrich,                     │
  price/inv/product                  dedupe, version)                       │
              └─────────────────────────────────────────────────────────────┘
```

Walking the **read path** left to right: a client request hits **CDN/Edge** — mostly useful
for static assets and some cacheable autocomplete, less so for personalized search. Then the
**API Gateway** does auth (short-lived JWT for buyers), rate-limiting, and routing. Then the
**Search Service**, which is the orchestrator. Inside it: **Query Understanding** normalizes
and enriches the query — tokenization, spell-correction, synonym expansion, locale handling.
Then we check the **Result Cache** in Redis. On a miss, the Search Service builds an ES query
and sends it to the **coordinating nodes**, which scatter to the **data nodes** and gather.
Results come back, the **Ranking / LTR reranker** reorders the top-K by a machine-learned
model blending text score with business signals, we assemble facets, serialize, and return.

The **write path** is fully asynchronous. The **Catalog Service** owns the transactional
truth — a sharded SQL or NoSQL store. Every mutation emits a change via **CDC** (change data
capture) into **Kafka**. The **Indexing Service** consumes Kafka, transforms the raw catalog
row into our search document — enrich with computed signals, denormalize seller info, dedupe,
attach a version — and writes to ES via the **_bulk** API. ES is never written to
synchronously by a user request, and ES is a **derived, rebuildable index — never the source
of truth.** If the cluster burned down we'd rebuild it by replaying from the catalog.

**Interviewer:** Why Kafka in the middle? Why not have the Catalog Service just call the
Indexing Service directly, or write to ES itself?

**Candidate:** Three reasons, in order of importance. **Decoupling and buffering** — Kafka
absorbs the 100K writes/sec spikes so ES gets a smooth, batched feed instead of being
hammered synchronously; if ES slows during a merge or a hot spike, the queue backs up
harmlessly instead of failing user-facing catalog writes. **Replayability** — the log lets me
rebuild the entire index or a single `products-vN` index by replaying, and it's how I recover
from a bad transform or a poisoned document. **Fan-out** — the same change stream feeds the
search index, analytics, cache invalidation, and any future consumer without the Catalog
Service knowing about any of them. If the Catalog Service wrote ES directly, a search-cluster
hiccup would take down the ability to update prices — coupling a revenue-critical write to a
derived system. That's exactly backwards.

**Interviewer:** Fair. Keep going.

> 💡 **Commentary:** The candidate led with the *organizing principle* — "separate read and
> write paths connected only through the index" — before drawing boxes. That framing is worth
> more than the boxes themselves. The whiteboard sketch is legible and labeled with the
> canonical component names, which signals someone who's built this, not read a blog. The Kafka
> justification is ranked (buffering > replay > fan-out) rather than a flat list, and the
> "coupling revenue writes to a derived system is backwards" line shows systems judgment. A
> weak candidate says "Kafka because it's scalable" — true but empty.

---

### Phase 4 — Deep Dive: Indexing Pipeline (18–24 min)

**Interviewer:** Let's go deep on freshness. You promised price changes searchable in ~1
second. How do you actually keep price fresh at 100K writes/sec without melting the cluster?

**Candidate:** This is the tension between freshness and indexing efficiency, and it's worth
being concrete. A few mechanisms stacked together.

**First, the pipeline is already streaming, not batch.** CDC emits the price change within
milliseconds of the DB commit, Kafka delivers it, and the Indexing Service micro-batches — it
buffers for, say, a few hundred milliseconds or N documents, whichever comes first, then fires
one `_bulk` request. Micro-batching is the key: 100K individual index calls would kill ES, but
100K writes collapsed into bulk requests of a few thousand docs each is very comfortable.

**Second, ES refresh.** A document isn't searchable until a *refresh* turns the in-memory
buffer into a searchable Lucene segment. Default is 1 second; I'd keep the hot index at ~1s so
new data is visible within that window. That 1s refresh is precisely the knob that delivers
"searchable within seconds."

**Third — and this is the important optimization for price specifically — I'd avoid
reindexing the whole document for a price tick.** Re-analyzing the title and description every
time inventory decrements is wasteful. Options: partial updates (`_update`) so only changed
fields move; and for the highest-churn numeric signals like price and stock, I'd consider
storing them so they can be updated cheaply, or even splitting extremely hot, low-relevance
signals out. In practice `_update` with `doc_values` fields that don't require re-analysis
keeps price/inventory churn cheap.

**Fourth, versioning for correctness.** Price updates can arrive out of order — Kafka gives
per-partition ordering, but retries and multiple producers can reorder. So every document
carries `_version`, and I use **external versioning** on the write: ES rejects an update whose
version is older than what's indexed. That makes updates **idempotent and monotonic** — a
stale "$799" can never overwrite a fresh "$899". I'd also partition Kafka by `listing_id` so
all mutations for one listing land on one partition and stay ordered in the common case.

**Interviewer:** What if the transform step produces a bad document — say a schema change
ships a bug and 2% of documents fail to index?

**Candidate:** Two layers. Failed documents go to a **dead-letter queue** rather than blocking
the stream — I never let one poison pill stall the pipeline. I alert on DLQ depth, inspect,
fix the transform, and **replay** from Kafka (retention gives me a window to do this). For a
bad *deploy* — the whole new transform is wrong — this is why I use versioned indices behind
an alias. I build `products-v(N+1)` from a full replay, validate it offline against golden
queries, and only then atomically flip the `products` alias. If it's wrong, flip back. No
in-place corruption of the live index.

**Interviewer:** How long does a full reindex of 10B docs take, roughly, and is that
acceptable?

**Candidate:** It's hours, not minutes — you're bounded by bulk throughput across the cluster.
It's acceptable precisely *because* it runs on a parallel `v(N+1)` index while `vN` keeps
serving live traffic; nobody's search is degraded during the rebuild, and the cutover is an
atomic alias swap. Full reindex is a planned maintenance operation, not an outage. What I
optimize for is *not needing* it often — that's what the DLQ, partial updates, and versioning
buy me: routine correctness without full rebuilds.

> 💡 **Commentary:** This deep dive hits every ES-specific lever a senior is expected to know:
> micro-batching into `_bulk`, the 1-second refresh as the freshness knob, partial updates to
> avoid re-analysis, external versioning for idempotent/monotonic out-of-order writes, and
> Kafka partitioning by key for ordering. The DLQ + versioned-index-behind-alias answer to the
> "bad deploy" curveball is exactly right and shows the candidate has operated a system like
> this. Weak candidates answer "keep price fresh" with "just reindex faster" and have no answer
> for out-of-order updates — which is the subtle correctness bug that actually bites in
> production.

**Interviewer:** You mentioned enrichment in the transform step. What does the Indexing Service
actually *do* to a raw catalog row before it lands in ES, and why not just mirror the row?

**Candidate:** The catalog row is normalized transactional data; the search document is a
*denormalized, query-optimized* projection. Three kinds of work happen in the transform. First,
**denormalization** — the catalog stores `seller_id` as a foreign key, but for search I want
`seller_rating` and seller trust signals right on the document so ranking doesn't do joins at
query time; search engines don't join, so I fold related data in at write time. Second,
**enrichment with computed signals** — `popularity` (a normalized sales/CTR score),
conversion rate, maybe a price-competitiveness score relative to other sellers of the same
`product_id`. Those are computed by upstream jobs and merged in. Third, **shaping for the
analyzer** — deciding which fields are `text` (analyzed) versus `keyword` (exact), building the
`category_path` hierarchy for faceting, flattening attributes. So the write path is where I pay
the denormalization cost *once* so every one of 100K QPS reads is cheap. That's the classic
read-optimize-by-precomputing-on-write trade.

**Interviewer:** Those computed signals — popularity, conversion — change over time
independently of the seller editing the listing. How do they get refreshed?

**Candidate:** Good — they're a *different* update cadence than catalog edits, and I shouldn't
conflate them. Catalog edits (title, price, stock) come through CDC in near-real-time.
Signal refreshes (popularity, conversion) are computed by batch or streaming analytics jobs —
hourly or daily is usually fine, these don't need sub-second freshness — and they emit their
own update events into the same Kafka stream, which the Indexing Service applies as partial
`_update`s to the affected documents. So I have two producers feeding one indexing pipeline at
two cadences, and external versioning keeps them from clobbering each other. This is another
reason the pipeline is a stream of *events* rather than the Indexing Service pulling the whole
catalog row each time.

---

### Phase 5 — Deep Dive: Search Pipeline & Relevance (24–30 min)

**Interviewer:** Someone types "iphone". How do you make the *right* result rank first — not a
$5 phone case that happens to say "for iPhone" in its title?

**Candidate:** This is the heart of the problem, so let me separate **retrieval** (find the
candidate set) from **ranking** (order it), because they use different scoring and conflating
them is a classic mistake.

**Query understanding first.** "iphone" gets tokenized, lowercased, spell-checked (here it's
fine), and synonym-expanded if applicable. We detect it's likely a *brand/product* intent, not
an accessory intent.

**Retrieval.** I query ES over the analyzed fields, but with **field boosting** — a match in
`title` is worth far more than a match in `description`. The phone-case problem is largely a
field-and-phrase problem: the case has "iPhone" in a long description or as a secondary token;
the actual iPhone listing has it as the dominant term in a short, high-signal title. So
`title` boosting plus considering term frequency relative to field length (BM25 already
normalizes for field length, which naturally penalizes the case's keyword-stuffed title) gets
us a long way. I'd also use `match_phrase` boosting and possibly require the term in title for
head queries.

But — pure text relevance (BM25) is *not enough* for e-commerce, and I want to be clear about
that. BM25 doesn't know that the real iPhone converts 100× better than the case.

**Ranking / LTR.** So retrieval returns a candidate set of, say, a few hundred, and then a
**Learning-to-Rank reranker** reorders the top-K using a model trained on real engagement —
clicks, add-to-carts, conversions. Its features blend the text score with **business
signals**: `popularity` (normalized sales/CTR), `conversion rate`, `rating` and
`review_count`, `seller_rating`, price competitiveness, and availability. The iPhone wins not
because of text — text is a near-tie — but because its popularity and conversion features
dwarf the case's. That's the blend the requirements asked for: relevance × business value.

**Interviewer:** Why not run the fancy LTR model over all matching documents?

**Candidate:** Cost and latency. LTR feature extraction and model inference are expensive
per-document; running them over potentially millions of matches would blow the 30ms rerank
budget by orders of magnitude. So it's the standard **two-phase** approach: cheap, fast BM25 +
boosts to *retrieve and roughly rank* a few hundred candidates across the shards, then the
expensive model *reranks only the top-K*. You spend your compute where it changes the answer —
the top of page one — and nowhere else.

**Interviewer:** And filters and facets — a user picks "Brand: Apple, Price: <$1000, In
stock". How does that interact with ranking?

**Candidate:** Filters run in ES **filter context**, not query context — that's a real
distinction that matters here. Filter context is a yes/no membership test: it doesn't
contribute to the relevance score, and critically it's **cacheable** because the result of
"in_stock:true" is the same regardless of the query text. So filters cheaply shrink the
candidate set before scoring. Facet counts come from ES **aggregations** over the filtered
set — "Apple (1,240), Samsung (980)…" — computed on `doc_values`. One subtlety: standard
facet UX computes each facet's counts as if that facet weren't applied to itself, so you build
the facet aggregations carefully with post-filtering. Availability I treat as a filter by
default but it's also a ranking signal — out-of-stock items, if shown at all, rank far down.

**Interviewer:** Where does caching help, given results are semi-personalized?

**Candidate:** Layered. The **Result Cache** in Redis caches full result sets keyed by the
normalized query + filters + sort + locale + page. Head queries — "iphone", "shoes", "laptop"
— are a huge fraction of traffic and hugely repetitive, so even a short TTL (tens of seconds)
gets a strong hit rate and shaves the ES fan-out off the hottest queries. Below that, ES's own
**shard-level query cache** and the OS **page cache** (this is why RAM/nodes matter) serve
repeated sub-queries. Filter results cache well as I mentioned. Personalization complicates the
top-level cache — if ranking is per-user I can only cache the retrieval layer, not the final
order, or I cache per-cohort rather than per-user. For most queries a short-TTL result cache on
the non-personalized ranking is the big win, with personalization applied as a light rerank on
top.

> 💡 **Commentary:** The retrieval-vs-ranking separation is the single most important idea in
> search relevance, and the candidate led with it. The two-phase "retrieve cheap, rerank
> expensive top-K" answer is exactly what the interviewer wants when they ask "why not LTR over
> everything" — it shows the candidate understands the cost model, not just the ML buzzword.
> The filter-context / query-context distinction and "filters are cacheable because they're
> query-independent" is a genuine ES-internals signal. And they proactively surfaced the
> caching-vs-personalization tension instead of pretending it away. A weak candidate says "use
> a machine learning model for ranking" and cannot explain retrieval, cost, or filter context.

**Interviewer:** You skipped over spell-correction and synonyms fast. Someone types "nike air
maxx" — walk me through query understanding actually doing something.

**Candidate:** Sure, let me slow down there. Query understanding is a small pipeline of its own,
and I keep it under ~15ms of the budget. Steps: **normalize** (lowercase, strip punctuation,
Unicode-fold so "café" matches "cafe"); **tokenize** per the locale's analyzer — CJK languages
don't split on spaces, so tokenization is locale-specific; **spell-correct** — "maxx" → "max"
via a correction model built from the term dictionary plus query logs (crucially, weighted by
what people *actually search and click*, so corrections are popularity-aware, not just edit
distance); **synonym expansion** — "trainers" ↔ "sneakers", "tv" ↔ "television", handled with a
curated synonym dictionary applied at query time (or index time for stable ones). For "nike air
maxx" I'd correct to "nike air max", recognize "nike" as a known brand token — which can
trigger a brand *filter or boost* rather than just a text match — and preserve the phrase so
"air max" gets phrase-proximity credit. I'd also return the correction in the response metadata
so the UI can show "showing results for *nike air max*," which the requirements' API contract
includes.

**Interviewer:** Where do the synonyms and corrections come from? Who maintains them at this
scale?

**Candidate:** A mix, and I wouldn't hand-curate 10 billion listings' worth. Head synonyms and
known brand/category vocab are **curated** by a relevance/merchandising team — high value,
manageable count. The long tail is **mined from behavior**: query logs, reformulation pairs
(users who searched X then Y then clicked), and co-click data give candidate synonyms and
spell corrections automatically, which humans then sample-audit. It's the same philosophy as
ranking — curate the head, learn the tail, measure everything. And this is one more reason the
offline eval harness I'll mention at the end matters: a bad synonym can quietly wreck relevance
for a whole category.

---

**Interviewer:** Back up to autocomplete — you waved it off as "cheap." At 500K QPS and a
100ms p99, convince me it's actually easy.

**Candidate:** Fair, "cheap" was shorthand; let me justify it. Autocomplete is easy *relative*
to full search because the problem is smaller on every axis. The corpus isn't 10B documents —
it's a **suggestion dictionary** of maybe a few million entries: popular queries, product
names, brands, categories. It's a **prefix** match, not full relevance — I don't run BM25, LTR,
faceting, or scatter-gather over product shards. And the workload is *extremely* repetitive:
the prefixes "i", "ip", "iph", "ipho" are typed millions of times a day by different users, so
cache hit rates are enormous.

Concretely, I'd serve it from a dedicated structure — an ES **`completion` suggester** (an
in-memory FST, a finite-state transducer, that's built for prefix lookup and is extremely fast)
or a purpose-built prefix service backed by `edge_ngram` tokens, with each suggestion weighted
by popularity so "iphone" ranks above "ip camera" for the prefix "ip". In front of that,
aggressive caching — much of the 500K QPS is served from a Redis/edge cache keyed by prefix +
locale without ever touching ES. The suggestion dictionary is rebuilt periodically from query
logs (that's how new trending queries appear), so it doesn't need the NRT write path at all —
a different, slower freshness requirement than product search. So the 500K number is real, but
it's a prefix lookup over millions of cached entries, not 500K full searches. Different problem,
different, lighter subsystem — which is exactly why I split it out in the estimation phase.

> 💡 **Commentary:** The candidate had *claimed* autocomplete was cheap in the estimation phase
> and here they *back it up* on demand — consistency across the interview is itself a signal.
> The FST/completion-suggester detail and "weight suggestions by popularity" show real depth,
> and separating autocomplete's *slower* freshness requirement from product search's ~1s SLA is
> a nuance most candidates miss. The meta-lesson: when you assert something early ("it's a
> separate lighter path"), expect to defend it later — so only assert what you can defend.

### Phase 6 — Deep Dive: Elasticsearch Internals (30–34 min)

**Interviewer:** Let's get lower-level. A search comes in for one shard's data — walk me
through what Lucene actually does, and then: what happens when a shard dies mid-query?

**Candidate:** Bottom-up. Each ES **shard is a Lucene index**, made of immutable **segments**.
The core structure is the **inverted index**: term → posting list of documents containing that
term, plus positions and frequencies. So for "iphone" Lucene looks up the term in the term
dictionary, walks its posting list, and scores each hit with **BM25** — which rewards term
frequency but saturates it, and normalizes by field length (that's the anti-keyword-stuffing
property I mentioned). Sorting and faceting don't use the inverted index; they use
**doc-values**, a columnar per-field store optimized for "give me the price of these 500 docs."
Segments are immutable, so an update is really a delete-and-add; deletes are tombstoned and
reclaimed later by **segment merges**, which compact many small segments into fewer large ones
in the background. **Refresh** makes the in-memory buffer into a new searchable segment (our 1s
freshness knob); **flush/fsync** and the translog give durability.

Now the query lifecycle across the cluster. A search hits a **coordinating node**, which
**scatters** the query to one copy — primary or replica — of every one of the ~600 shards.
Each data node runs the query locally and returns its top results. This is the classic
**scatter-gather / query-then-fetch**: phase one gathers just doc IDs and scores from all
shards, the coordinator merges to the global top-K, then phase two fetches the actual
`_source` for only those K. That's why deep pagination is expensive — page 1,000 means every
shard returns 1,000× page-size candidates to merge — and it's why we use **`search_after`**
cursors instead of `from`+`size` beyond ~page 50.

**On a shard dying mid-query:** it depends when. If the coordinating node hasn't gotten that
shard's response and the shard's node fails, ES **retries the query on a replica** of that
shard — with one replica, there's another copy on another node, and the query transparently
re-routes. The user sees slightly higher latency, not an error. If somehow no copy is
available, ES can return **partial results** with a flag (`_shards.failed` > 0) rather than
failing the whole query — that's a graceful-degradation choice: better to return 95% of results
than a 500. Meanwhile the cluster's master notices the node is gone and **promotes the replica
to primary** and starts rebuilding a new replica elsewhere to restore the replication factor.
The read path stays up throughout because every shard has at least two copies on different
nodes.

**Interviewer:** You said the master promotes a replica. What keeps the masters themselves from
becoming a split-brain problem?

**Candidate:** Dedicated master-eligible nodes — I'd run **3** — and quorum-based election.
A master needs a majority (2 of 3) to be elected, so a network partition can't produce two
masters both making decisions; the minority side steps down and refuses writes. That's the
CP-for-cluster-*coordination* part of the system — cluster metadata is managed consistently
even though the *data/search* path is AP. Separating dedicated masters from data nodes also
means a data-node GC pause or query overload can't knock out cluster coordination.

> 💡 **Commentary:** This is where seniority shows. The candidate goes cleanly from Lucene
> segments and posting lists up through scatter-gather query-then-fetch, and nails the two
> things interviewers probe: *why deep pagination is expensive* (and the `search_after` fix)
> and *shard failure → replica retry → partial results → replica promotion*. The unprompted
> "masters are CP for coordination while data is AP" is a beautiful nuance — it shows the
> candidate understands that "AP vs CP" isn't a single global choice but applies per-subsystem.
> A weak candidate knows "ES uses inverted indexes" as a phrase but can't describe query-then-
> fetch, doesn't connect it to pagination cost, and thinks a shard failure means an error.

---

### Phase 7 — Scaling & Failure (34–41 min)

**Interviewer:** Black Friday. Traffic goes 5× in an hour — well past your 100K peak. What
happens, and what do you do?

**Candidate:** Let me handle it in layers, from "already true" to "emergency lever."

**Provisioning and autoscaling.** Search Service, API Gateway, and the stateless tiers
autoscale on CPU/latency — those scale out in minutes. The expensive, slow-to-scale layer is
**ES data nodes**, because adding a node triggers shard rebalancing that takes time and IO. So
for a *known* event like Black Friday I **pre-scale** — add hot-tier nodes and replicas a day
ahead. More replicas directly buys read throughput: each replica is another copy that can serve
scatter-gather, so read capacity scales with replica count. That's the main dial.

**Caching absorbs the surge.** Black Friday traffic is *more* concentrated on head queries and
head products, not less — everyone's searching the same deals. So the Result Cache hit rate
goes *up* under load, which is exactly when you want it. I'd raise TTLs slightly and make sure
the cache tier is scaled. A good result cache can absorb a large fraction of a 5× spike before
ES ever sees it.

**Load shedding and degradation, in priority order, if we still exceed capacity.** First,
protect the core: **rate-limit** abusive clients at the gateway. Then **shed the expensive
extras before the core function** — e.g., temporarily serve cheaper ranking (skip the LTR
rerank and fall back to BM25 + static popularity), reduce facet computation, cap pagination
depth, degrade autocomplete to cache-only. The principle: **a slightly worse search result for
everyone beats a perfect result for half of users and errors for the other half.** Search
staying *up* is NFR #2. I'd rather return good-enough results at 100% availability than perfect
results at 98%.

**Interviewer:** What about the write side during the surge — prices are changing like crazy
too?

**Candidate:** That's where Kafka earns its keep. The indexing pipeline is already decoupled,
so a write surge just deepens the queue; ES consumes as fast as it can and freshness lag
stretches from ~1s to maybe a few seconds. That's a graceful, invisible degradation — nobody
gets an error, prices are just slightly less fresh for a bit. I'd also, during the event,
possibly *raise* the refresh interval on bulk-heavy indices to trade a little freshness for a
lot of indexing throughput. Reads are sacred; write freshness is the elastic thing I bend.

**Interviewer:** One more failure: an entire region goes down.

**Candidate:** We run **multi-region**, both for latency/locality — serve EU buyers from EU —
and for HA. Each region has a full ES cluster fed by the same Kafka change stream (or
cross-cluster replication). If a region dies, we **fail traffic over to another region** at the
global load-balancer / DNS layer; that region has a complete, independently-serving index.
Because ES is a derived, rebuildable index and the catalog is the real source of truth, we
don't have a cross-region consistency nightmare on the search path — worst case the failed-over
region is slightly staler for its non-local locales. I'd also keep **hot/warm tiers**: hot
nodes hold recent/popular data on fast disks; warm nodes hold older or less-popular locales on
cheaper hardware. That controls cost at 48TB without hurting the queries that matter.

> 💡 **Commentary:** The Black Friday answer is the model answer because it's *layered and
> prioritized*: pre-scale → cache (with the sharp insight that hit rate *rises* under
> concentrated load) → shed extras before core → degrade gracefully. The "worse-for-everyone
> beats errors-for-half" principle is precisely the availability-over-perfection stance senior
> reviewers reward. Bending *write freshness* rather than read availability shows they
> internalized the NFR priority order they set in minute 3 — the whole interview is coherent.
> A weak candidate says "just add more servers" with no notion that ES scales slowly, no
> load-shedding plan, and no idea that caching helps *more* under a concentrated surge.

**Interviewer:** Let me push on the sharding scheme itself. With ~600 shards, how do you decide
what goes on which shard, and can that bite you?

**Candidate:** Default ES routing hashes the document ID to a shard, which spreads data
uniformly — good for avoiding storage hot spots. The thing that can bite is a **routing choice
that concentrates query load**. For instance, if I routed by `market` so each market's listings
sit together, a query scoped to the US market would only hit US shards — great for that query's
efficiency — but US is a huge market, so those shards become hot while a small market's shards
idle. So there's a tension: **routing by locale/market gives query locality and lets me put
cold markets on the warm tier, but risks load skew**; hashing by ID gives even load but every
query fans out to all 600 shards. My default is hash-by-ID for even load with one replica for
throughput, and I use the **hot/warm tiers plus multi-region** to get locality instead of
overloading routing with that job. If profiling showed a specific market dominating, I'd
revisit custom routing for *that* slice. The interview-honest answer is: I'd start simple
(hash), measure shard-level load, and only add routing complexity when data proves I need it.

**Interviewer:** And how do you actually watch for that? What's on your dashboard?

**Candidate:** The SLO metrics I'd wire up on day one: **search latency percentiles** (p50/p95/
p99, and separately for autocomplete against its 100ms budget); **freshness lag** measured as
event-time-to-searchable (CDC commit timestamp to the moment a query can see it) — that's the
direct measure of the ~1s SLA and it's the one most likely to erode silently; **cache hit
rate** on the Result Cache; **`_shards.failed` / partial-result rate** as a health signal for
the cluster; **Kafka consumer lag and DLQ depth** for the write path; and per-shard CPU and
query rate to catch the skew we just discussed. Alerts on p99 breaching budget, freshness lag
exceeding a few seconds, and DLQ depth climbing. Without those, I'm flying blind on exactly the
promises I made in the requirements.

---

### Phase 8 — Trade-offs & a Curveball (41–45 min)

**Interviewer:** Blunt question. Why this whole Elasticsearch cluster? Why not just Postgres
full-text search? We already run Postgres.

**Candidate:** Legitimate question, and for a *small* catalog Postgres full-text is genuinely
the right call — no new system to operate. It breaks down for us on three axes specifically.

**Scale and fan-out.** Postgres full-text doesn't horizontally shard-and-scatter across 600
shards / ~10B documents the way ES does. You'd be sharding Postgres manually and building the
scatter-gather query layer yourself — reinventing ES, worse.

**Relevance and search features.** ES/Lucene give us BM25, analyzers, synonyms, edge-n-gram
autocomplete, `search_after`, and native aggregations for faceting. Postgres `tsvector` has
none of the faceting/aggregation performance, no good typeahead structure, and much weaker
relevance controls. We'd bolt on a search layer anyway.

**Operational fit.** ES is *built* to be the AP, horizontally-scaled, read-heavy derived index
with replica-based read scaling and per-shard failover. Postgres is built to be the
consistent, transactional source of truth — which is exactly the role I've *kept it in* as the
Catalog Service. So it's not ES-vs-Postgres, it's using each for what it's good at: **Postgres
(or the catalog store) is the CP source of truth; Elasticsearch is the AP derived query
engine.** Using Postgres for both would compromise both.

**Interviewer:** And why not build on raw Lucene or Solr, or go with a vector database since
everything's semantic now?

**Candidate:** Raw Lucene means building distribution, replication, and cluster management
ourselves — that's what ES *is*, so no. Solr is a legitimate alternative — comparable
Lucene-based engine; the choice is largely ecosystem, operational familiarity, and features
like ES's LTR and hot/warm tiering; I wouldn't die on that hill. On **vector/semantic search**:
it's complementary, not a replacement. Dense-vector kNN retrieval is fantastic for "queries
that don't share keywords with the doc" — semantic matches, "warm jacket for hiking." But for
e-commerce *head* queries like "iphone 15 128gb" and for exact filter/facet/price behavior,
lexical BM25 is precise, cheap, and debuggable. The mature answer is **hybrid**: run both
lexical and vector retrieval and fuse the candidate sets, then LTR on top. I'd add vectors as an
augmentation to the retrieval layer I already have, not a rip-and-replace. That's a roadmap
item, and I flagged image/visual search earlier as the same category — vector retrieval over
image embeddings.

**Interviewer:** One consistency worry. A seller updates their listing and immediately searches
for it to confirm it's live — but the index is eventually consistent. They see stale data and
file a bug. How do you handle that?

**Candidate:** This is the read-your-own-writes problem, and it's real precisely because the
search path is AP. Two things. First, **set expectations at the product level** — the seller
dashboard shows listing state from the *catalog* (the source of truth), not from search, with
a small "changes may take a few seconds to appear in search results" affordance. Never make the
seller confirm their own edit by running a buyer search against an eventually-consistent index;
read the authoritative store for confirmation UIs. Second, for the genuine need to *preview*
how a listing looks in search, I can offer a preview path that queries with a very fresh /
`refresh=wait_for` semantics for that single document, or hits the catalog-derived view
directly — an exception path, not the hot search path. The key judgment: I don't compromise the
100K-QPS AP search path's design to solve a single-user confirmation flow; I solve that flow
with the source of truth. Bending the whole read path to CP for this would be the wrong trade.

**Interviewer:** Good. If you had one more day on this design, where's the biggest risk you'd
de-risk first?

**Candidate:** Relevance quality, honestly — not the infrastructure. The distributed systems
here are well-trodden; I'm confident the cluster serves 100K QPS at 200ms. The thing most
likely to be quietly *bad* is ranking: getting the LTR features, training data, and
evaluation harness right so we actually improve NDCG and conversion rather than just believing
we do. So day one I'd invest in the **offline eval and A/B framework** — golden query sets,
NDCG measurement, and a clean experimentation path — because without measurement, every ranking
change is a guess. Second risk would be validating the freshness SLA end-to-end under real
write load, since that's the promise most likely to erode silently.

**Interviewer:** That's a great place to stop. Thanks. Any questions for me?

**Candidate:** Yes — two. How does your team currently draw the line between the search team and
the relevance/ML team; who owns the LTR model? And what's the thing about your current search
stack you most wish you could redesign? [*...brief discussion...*]

> 💡 **Commentary:** The trade-offs phase is where the candidate converts knowledge into
> *judgment*. "It's not ES-vs-Postgres, it's using each for what it's good at — and I've kept
> Postgres as the CP source of truth" reframes the gotcha into a coherent thesis. On vectors,
> they neither dismiss the shiny thing nor chase it — "complementary, hybrid, roadmap item" is
> the senior answer. The read-your-writes exchange is a subtle senior test: the trap is to
> "fix" it by making the search path consistent, which would wreck the whole design. The
> candidate instead solves the user problem with the source of truth and *protects the AP
> read path* — knowing which problems to solve *outside* your hot path is judgment. The best
> moment is the final one: asked where the *risk* is, they name **relevance measurement**, not
> infrastructure — showing they know where this class of system actually fails in the real
> world. Thoughtful reverse-questions cap a strong close.

---

## 12.3 Two Moments of Recovery (and why they mattered)

Reviewers care as much about how you handle being wrong as about being right. Two moments in
the transcript deserve a second look — and here's a third, expanded, that's worth practicing.

**Moment 1 — the challenged paradox.** When the interviewer said *"you claimed writes outnumber
reads 3:1 but reads matter more — reconcile that,"* the candidate didn't get defensive or
backpedal. They reframed on two axes (volume vs. latency-criticality) and turned the apparent
contradiction into the *justification* for two separate pipelines. Pushback isn't always a
"gotcha"; often the interviewer is handing you a chance to show depth. Treat challenges as
prompts, not attacks.

**Moment 2 — the reframed gotcha.** "Why not just Postgres" is designed to see if you'll get
defensive or dogmatic. The candidate conceded the valid case ("for a small catalog it's the
right call"), then gave three specific axes, then *reframed* to "use each tool for its job." No
tribalism, no hand-waving.

**A worked recovery from an actual mistake.** Here's an exchange we can insert to model the
graceful-recovery signal explicitly:

**Interviewer:** You said you'd cache the full personalized result set in Redis. Doesn't that
break the moment ranking is per-user?

**Candidate:** You're right — I overstated that, let me correct it. If the final ranking is
per-user, caching the *final ordered result set* per query has a near-zero hit rate and is
wrong. What I should have said: cache the **non-personalized retrieval + base ranking** — which
is shared across users for a given query — and apply personalization as a light rerank *after*
the cache. Or cache per *cohort* rather than per user. The head-query cache win I described
holds for the non-personalized layer; I shouldn't have implied we cache the personalized
output. Thanks — good catch.

> 💡 **Commentary:** This is the single highest-value behavior in the whole interview.
> The candidate (1) *immediately owned it* — "you're right, I overstated that" — with no
> defensiveness and no rambling justification of the wrong thing; (2) *diagnosed* precisely
> what was wrong; (3) *produced the corrected design* on the spot; and (4) preserved the
> valid part of their original point. Interviewers deliberately plant or pounce on small
> errors to test exactly this. A weak candidate either digs in and defends the wrong answer
> (fatal) or collapses and loses confidence for the rest of the interview (also costly). Being
> *correctable* is a senior trait — it's what your future teammates need from you in code
> review. Never argue with a correct correction; never abandon the whole design over a small
> one.

---

## 12.4 Post-Interview Scorecard

Here is how a calibrated interviewer would write this candidate up. Scores are 1–4
(1 = no signal / below bar, 2 = mixed, 3 = solid hire signal, 4 = strong/standout).

| Dimension | Score | Justification |
|---|:---:|---|
| Problem-solving | 4 | Drove the whole session, decomposed an open prompt cleanly, managed the clock, and surfaced the load-bearing "document = listing" decision unprompted. |
| Requirements | 4 | Scoped before building; prioritized conflicting NFRs and named the priority order as *the design*; explicit out-of-scope; got AP-over-CP buy-in early. |
| Estimation | 4 | Derived (not recited) ~3B/day → 35K avg → 100K peak QPS, ~20TB→24/48TB, ~600 shards; and *used* the numbers — "QPS-bound not disk-bound" changed node sizing; separated the 500K autocomplete path. |
| High-level design | 4 | Clean two-path architecture with correct component boundaries; led with the organizing principle; ranked the Kafka justification. |
| Deep dive | 4 | Went genuinely deep on ES internals (segments, doc-values, query-then-fetch), indexing (bulk, refresh, external versioning, DLQ+alias reindex), and relevance (retrieval vs. rank, two-phase LTR, filter context). |
| Scaling | 3.5 | Layered, prioritized Black Friday plan; multi-region + hot/warm; sharp cache-hit-rate insight. Slightly light on concrete replica/node math for the 5× case and on cross-region replication lag specifics. |
| Trade-offs | 4 | Reframed the Postgres gotcha into "right tool per job"; balanced, non-dogmatic take on Solr and vector/hybrid; named *relevance measurement* as the top real risk. |
| Communication | 4 | Signposted phases, legible whiteboard, handled pushback and a genuine mistake with grace, strong reverse-questions. |

**Overall verdict: Strong Hire (senior).** The candidate demonstrated the full arc — scope,
estimate, design, go three levels deep on demand, scale, and reason about trade-offs — and,
crucially, showed the *judgment* signals that separate senior from mid: prioritizing NFRs and
staying consistent with that priority for 45 minutes, knowing which subsystem is CP vs. AP,
knowing where the design's *real* risk lives (relevance, not infra), and correcting a mistake
cleanly.

**What would have made this even stronger:**
- **More concrete math in the scaling phase.** When asked about the 5× surge, quantify it:
  "to serve 500K QPS I need roughly Nx the replicas → M more data nodes; here's why replicas,
  not primaries." Numbers under pressure are a differentiator.
- **A word on observability/SLOs.** Name the metrics you'd watch — search p99 by percentile,
  freshness lag (event-time to searchable), cache hit rate, `_shards.failed` rate, DLQ depth —
  and what alerts on them. It rounds out the "I've operated this" signal.
- **Cross-region consistency, one level deeper.** Acknowledge replication lag between regions
  and how failover interacts with the freshness SLA for non-local locales.
- **One explicit cost sentence.** "~60–100 hot data nodes plus warm tier is the dominant cost;
  hot/warm tiering and the result cache are the two biggest cost levers" ties the design to
  business reality.

These are polish, not gaps — the hire decision doesn't depend on them.

---

## 12.5 How to Practice

Reading a transcript builds recognition; only reps build the skill. Drills, roughly in order:

**Drill 1 — The 5-minute open, out loud, on a timer.** Practice *only* minutes 0–5: clarifying
questions, prioritized NFRs, explicit out-of-scope, and the one load-bearing scoping decision
(here, document = listing). Record yourself. If you can't do a crisp, prioritized open in five
minutes, the rest of the interview runs late. Do this until it's automatic.

**Drill 2 — Estimation from memory, then let it change the design.** Close this book and
re-derive the canonical numbers from just "2B users, 500M DAU, 6 searches/day, 10B docs, 2KB
each": QPS (avg + peak), corpus and disk, shard count, node count. Then say *one design
consequence of each number* out loud — e.g., "500K autocomplete QPS → separate path,"
"QPS-bound → provision for CPU/RAM not disk." Estimation only scores if it moves the design.

**Drill 3 — The whiteboard sketch in 90 seconds.** Draw the two-path diagram (read path + write
path) from memory with the canonical component names, fast and legible. You should be able to
produce it without thinking so your brain is free for the *reasoning*, not the drawing.

**Drill 4 — Deep-dive on demand.** Have a friend point at any box and say "go deeper." Practice
three levels down on each: the Indexing Service (bulk, refresh, versioning, DLQ, reindex-behind-
alias), the Search pipeline (query understanding → retrieval → two-phase rerank → facets →
cache), and ES internals (segment → inverted index/doc-values → scatter-gather query-then-fetch
→ shard failover). If you can't do all three, that's your study list.

**Drill 5 — Curveball rapid-fire.** Answer each of these in under two minutes, out loud:
*How do you keep price fresh? What happens when a shard dies mid-query? Why not Postgres
full-text? How do you survive a 5× Black Friday surge? How do you make "iphone" rank right?
How do you handle an out-of-order price update? What if a bad deploy poisons 2% of docs?*
Every one of these is answered in this transcript — practice giving *your* version.

**Drill 6 — Practice being corrected.** Have your partner deliberately push back on something
you said (right or wrong). Practice the reflex: pause, evaluate honestly, and either *defend
with a reason* or *concede cleanly and produce the fix* — never get defensive, never crumble.
This is the hardest behavior to fake and the easiest to lose points on.

**Drill 7 — Full mock, end to end, on the 45-minute clock, with a human.** No pausing, no
re-dos. Then score yourself against the rubric table above, dimension by dimension, and write
your own "what would have made this stronger" note. Rotate scenarios so you're not memorizing
one answer: same drills work for "design Twitter search," "design a log-search system," "design
type-ahead for a search bar."

---

## 12.6 Key Takeaways

- **The interview is a performance of judgment, not a recital of an answer.** Scope, estimate,
  design, deep-dive, scale, trade-off — *and* manage the clock and communicate.
- **Let the numbers drive the design.** "QPS-bound, not disk-bound" and "500K autocomplete QPS
  → separate path" are the moments estimation earns its score.
- **Lead with organizing principles before boxes:** separate read/write paths joined only by a
  derived, rebuildable index; ES is AP, the catalog is the CP source of truth.
- **Know the ES internals cold** — segments, doc-values, BM25, scatter-gather query-then-fetch,
  `search_after`, refresh, external versioning, replica failover — because the deep dive is
  where senior signal lives.
- **Separate retrieval from ranking**, and rerank only the top-K — the two-phase model is the
  heart of e-commerce relevance.
- **Prioritize availability and stay consistent with that priority:** degrade extras and bend
  write-freshness before you ever sacrifice read availability.
- **Handle pushback and mistakes gracefully.** Concede cleanly, produce the fix, keep the valid
  part. Being correctable is a senior trait.
- **Know where the real risk lives.** For this system it's relevance *measurement*, not the
  distributed-systems plumbing — say so.

*(This chapter's scorecard and "How to Practice" sections stand in for the usual Interview Tips;
the Key Takeaways above close the handbook.)*
