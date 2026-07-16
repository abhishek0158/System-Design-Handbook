# Chapter 1 — Requirement Gathering

> "Weeks of coding can save you hours of planning." The joke lands in system design
> interviews too. The candidates who fail rarely fail because they don't know
> Elasticsearch — they fail because they solve the *wrong problem* very impressively.
> This chapter is about making sure you solve the right one.

---

## 1. Why Requirement Gathering Is the Highest-Leverage Phase

A 45-minute system design interview is a compressed simulation of a real design review.
Everything you draw later — the shard math in Chapter 2, the API contracts in Chapter 3,
the read/write split in Chapter 5 — is downstream of the requirements you pin down in the
first 5–8 minutes. Get the requirements wrong and every subsequent minute compounds the
error. Get them right and the rest of the interview becomes a guided tour through a problem
you have already framed to your advantage.

There are three reasons this phase has the highest leverage:

- **It sets the scoring rubric.** Interviewers grade against the requirements *you* stated,
  not some hidden ideal. If you declare that autocomplete is in scope and NRT indexing must
  hit a ~1 s inventory lag, you have handed yourself the criteria you'll be judged against —
  and shown you know what matters. If you never mention freshness, you can't be rewarded for
  the elegant CDC pipeline you build in Chapter 6.
- **It bounds the search space.** "Design search" is unbounded. "Design a read path that
  serves 100K QPS at p99 ≤ 200 ms over 10B eventually-consistent documents, where a stale
  price for one second is acceptable but a wrong result is not" is a problem with a tractable
  set of correct answers. Requirements convert an essay question into an engineering problem.
- **It signals seniority.** Junior engineers implement the spec they're given. Senior
  engineers *interrogate* the spec — they know that a one-word answer ("yes, personalized")
  can swing the design by two orders of magnitude of complexity. The clarifying questions you
  ask in §5 are, by themselves, a large fraction of your score.

### How to open a search-system interview

Do **not** open with "So I'll use Elasticsearch." (See Interview Tips for why this is the
single most common trap.) Open by *restating the problem in your own words and naming the
axes you'll scope along*. Something like:

> "Let me make sure I understand the problem. We're building product search for GlobalMart, a
> global marketplace at Amazon/AliExpress scale. Buyers type queries into a search box and get
> ranked product listings back, with filters, facets, and autocomplete. Before I design
> anything, I want to nail down four things: the functional scope — what features count as
> 'search'; the scale — QPS, corpus size, write rate; the non-functional targets — latency,
> availability, freshness; and any hard constraints. Can I walk through those and check my
> assumptions with you?"

That single paragraph does four things: it proves you heard the problem, it front-loads a
structure, it invites collaboration, and it buys you permission to ask questions. From here
you drive.

---

## 2. A Repeatable Scoping Framework

Use the same four-bucket framework every time. It's memorable under pressure and it maps
cleanly onto how real design docs are organized.

| Bucket | Question it answers | Examples for GlobalMart |
|---|---|---|
| **Functional** | *What does the system do?* | Full-text search, autocomplete, filtering/faceting, sorting, spell correction |
| **Non-functional** | *How well must it do it?* | p99 ≤ 200 ms, 99.99% availability, ~1 s freshness, NDCG targets |
| **Constraints** | *What's fixed / given?* | Elasticsearch is the engine; multi-region; catalog is the source of truth; budget & team |
| **Scale** | *How much of it?* | 10B docs, 100K peak QPS, 100K writes/s, 20 TB corpus |

Two disciplines make this framework work:

1. **Separate the *what* from the *how*.** A requirement is "results in ≤ 200 ms," not "use a
   Redis cache." The cache is a *design decision* you make later to *satisfy* the requirement.
   Interviewers who hear you conflate the two assume you're pattern-matching to a memorized
   architecture rather than reasoning from goals. Keep requirements solution-agnostic in this
   phase — even though you and the interviewer both know Elasticsearch is coming (it's in the
   brief and the handbook is built around it), you *earn* it by deriving it in Chapter 11, not
   by asserting it in minute one.
2. **Quantify everything you can, and label the rest as an assumption.** "Low latency" is a
   wish; "search p99 ≤ 200 ms, autocomplete p99 ≤ 100 ms" is a requirement you can design and
   test against. When you don't have a number, *state the assumption out loud* (§6) so the
   interviewer can correct it cheaply.

The rest of the chapter walks each bucket in depth, then turns the whole thing into a
**design contract** (§8) that the remaining chapters must satisfy.

---

## 3. Functional Requirements (FR1–FR8)

These are the eight capabilities that define "search" for GlobalMart. For each, I give the
concrete behavior, an e-commerce example, and the design pressure it creates downstream.

### FR1 — Full-text search
Free-text query matched over `title`, `brand`, `description`, and `attributes`. This is the
core retrieval problem: tokenize the query, match against an inverted index, score by textual
relevance (BM25) blended with business signals.

> *Example:* A buyer types `wireless noise cancelling headphones`. We must match listings
> whose title says "Wireless Noise-Cancelling Over-Ear Headphones" *and* ones whose title says
> "Bluetooth ANC Headphones" where "ANC" is a synonym-expanded match on "noise cancelling."

Design pressure: drives the analyzer/mapping choices in Chapter 4 and the retrieval stage in
Chapter 7. Multi-field matching with field boosts (title matters more than description) is the
baseline; matching only exact strings would be an instant fail.

### FR2 — Autocomplete / typeahead
Suggestions rendered *as the user types*, ideally within a keystroke's worth of time. This is
a **separate, lighter path** from full search — different latency budget (p99 ≤ 100 ms),
different data structure (completion/edge-ngram, Chapter 8), and dramatically higher call
volume (~500K QPS, since each search generates 5–8 keystrokes).

> *Example:* Typing `sam` returns `samsung`, `samsung galaxy s24`, `samsung tv 55 inch` —
> ranked by popularity, locale-aware, and typo-tolerant enough that `samsng` still works.

Design pressure: you cannot serve 500K QPS through the same heavy scatter-gather path you use
for search. Calling this out early demonstrates you understand that "search" is really two
systems with two SLOs.

### FR3 — Filtering & faceting
Users narrow results by `category`, `brand`, price range, `rating`, `availability`,
`seller`, shipping, and attributes (size/color/etc.). Crucially, facets carry **counts** — the
UI shows "Brand: Samsung (1,204) · Apple (987)" — which means filtering isn't just a boolean
mask, it's an aggregation over the matching set.

> *Example:* Query `running shoes`, then filter `category:Footwear`, `price:50-100`,
> `brand:Nike`, and see live counts update for `color` and `size` facets.

Design pressure: facet counts are aggregations that must be fast at 100K QPS; this drives
`doc_values`, filter caching, and the decision about which fields are `keyword` vs `text`
(Chapters 4 and 7). Filters should also be *cacheable and order-independent*.

### FR4 — Sorting
Result ordering by **relevance (default)**, price (asc/desc), rating, newest, or popularity.
Relevance is a computed score; the others are sorts on stored numeric/date fields.

> *Example:* "Sort by: Price — Low to High" must be a stable, correct sort over the *entire*
> matching set, not just the current page — a classic distributed-sort subtlety when results
> are scattered across ~600 shards.

Design pressure: non-relevance sorts need `doc_values` and interact with pagination (§FR5).
Relevance sort interacts with the ranking/LTR stage (FR7).

### FR5 — Pagination
Navigate through result pages. The brief is explicit: use **cursor-based `search_after`**, not
`from + size`, beyond roughly page 50. Deep `from + size` forces every shard to build and
transmit a giant top-N list, which blows up memory and latency.

> *Example:* A buyer scrolling "page 3 of 200" for `phone case` — deep enough that offset
> pagination degrades, so we hand back an opaque cursor token.

Design pressure: shapes the API contract in Chapter 3 and constrains sorting (the cursor
encodes the sort key of the last item).

### FR6 — Spell correction & synonyms
`iphn` → `iphone` (correction); `trainers` ↔ `sneakers`, `laptop` ↔ `notebook` (synonyms,
often locale-specific). This lives in the **Query Understanding** stage.

> *Example:* A UK buyer searching `trainers` must find US listings titled "sneakers"; a
> misspelled `blutooth speker` must still return Bluetooth speakers, ideally with a "showing
> results for bluetooth speaker" note in the response metadata.

Design pressure: needs a dictionary/analyzer strategy and a decision on *when* to correct
(pre-retrieval rewrite vs. did-you-mean fallback on zero results). Directly attacks the
zero-result-rate metric (§7).

### FR7 — Personalization / ranking signals
The final ordering blends textual relevance with **business signals**: `popularity`,
conversion, `seller_rating`, price competitiveness, and `in_stock` availability. Optionally
personalized to the user.

> *Example:* Two listings match `air fryer` equally well textually, but one is in stock, has
> 4.7 stars and 5,000 sales, and the other is out of stock with no reviews. The first must rank
> higher. That's a Learning-to-Rank reranker applied to the top-N candidates.

Design pressure: this is the Ranking Service / LTR reranker in the architecture and the 30 ms
"ranking/rerank" slice of the latency budget. It's also where relevance *quality* (NDCG,
conversion) is won or lost. **Ask whether personalization is in scope** — full per-user
personalization pulls in a feature store and user-history joins, which is a huge scope
expansion (see §5).

### FR8 — Near-real-time indexing
New/updated listings become searchable **within seconds**; price/inventory changes propagate
within **~1 s**. This is the write path: Catalog → CDC → Kafka → Indexing Service → ES bulk.

> *Example:* A seller drops the price of a listing during a flash sale. Within ~1 s, search
> results (and any price-sort/price-filter) must reflect the new price, or buyers see and click
> a price that's already stale — a trust and conversion problem.

Design pressure: this is the entire Chapter 6 pipeline and the freshness NFR. Note the *tiered*
freshness: full re-index of a new listing (seconds) vs. a cheap partial update of price/stock
(~1 s) — they don't need the same mechanism.

---

## 4. Non-Functional Requirements and How to Prioritize Them

Functional requirements say *what*; NFRs say *how well*, and they are where senior candidates
separate themselves — because NFRs **trade against each other**, and naming the trade is the
skill being tested.

| NFR | Target (canonical) | Why it matters | Primary tension |
|---|---|---|---|
| **Latency** | search p99 ≤ 200 ms; autocomplete p99 ≤ 100 ms | Every 100 ms of latency measurably drops conversion | vs. relevance depth, vs. freshness (refresh cost) |
| **Availability** | 99.99% on read/search path | Search is revenue-critical; downtime = lost sales | vs. consistency (favor AP), vs. cost (redundancy) |
| **Scalability** | 10B docs, 100K peak QPS, horizontal | Traffic is spiky (sales events, 3× peak) | vs. cost, vs. operational complexity |
| **Freshness** | NRT; inventory/price lag ≤ ~1 s | Stale price/stock erodes trust & conversion | vs. latency & throughput (frequent refresh/merge cost) |
| **Relevance quality** | measured by NDCG / conversion | Bad ranking = users bounce even if fast & up | vs. latency (deeper ranking costs ms) |
| **Consistency** | eventual (AP over CP) is acceptable | The index is derived and rebuildable | traded *away* deliberately to buy A and latency |

### The latency budget (make it concrete)

Latency isn't one number — it's a *budget* you spend across the request path. The brief fixes
it, and quoting it verbatim is a strong move:

```
Search path p99 = 200 ms total:
  Edge / LB ............... 5 ms
  API + auth ............. 10 ms
  Query understanding .... 15 ms   (spell, synonyms, tokenize)
  ES query (scatter-gather) 120 ms  (fan-out to ~600 shards, gather)
  Ranking / rerank ....... 30 ms   (LTR over top-N)
  Serialization .......... 10 ms
  Buffer ................. 10 ms
```

Two insights fall straight out of this table: (1) the ES scatter-gather dominates, so most of
your scaling effort (Chapter 9) targets that 120 ms; and (2) there's almost no slack, so you
cannot afford a synchronous DB lookup or an un-cached facet aggregation on the hot path.

### How to prioritize — the ordering and the trades

State the priority order explicitly and justify it:

1. **Latency** first, because it's the most visible and most directly tied to revenue, and
   because it's the tightest constraint (200 ms with no slack).
2. **Availability** second (99.99%), because search is revenue-critical and reads are the SLA
   path — but it's second because a fast system that's occasionally *slightly* stale beats a
   perfectly-consistent system that's slow.
3. **Scalability** third — a means to the first two at 100K QPS, not an end in itself.
4. **Freshness** fourth — important, but we can *tolerate seconds*, which is what lets us pick
   an asynchronous, eventually-consistent pipeline instead of synchronous writes.
5. **Relevance quality** — continuously optimized, but measured offline/online rather than as a
   hard SLO gate; a 1% NDCG regression doesn't page anyone the way a latency breach does.
6. **Consistency** is deliberately traded *down* to eventual.

The headline trade to say out loud: **we choose AP over CP on the search path** (Chapter 11's
CAP discussion). Because Elasticsearch is a *derived, rebuildable* index and the Catalog
service is the source of truth, a one-second-stale price or a replica that briefly lags is
acceptable — that's the price we happily pay for 99.99% availability and sub-200 ms latency.
The opposite choice (strong consistency on every read) would be architecturally absurd for a
search index and betrays a misunderstanding of what the index *is*.

The other trade worth naming: **freshness vs. throughput.** Elasticsearch's `refresh`
interval controls how quickly new docs become visible; refreshing every 1 s costs segment
churn and merge pressure. So freshness is tunable per tier — 1 s on hot data, higher during
bulk loads (Chapter 8).

---

## 5. Clarifying Questions to Ask the Interviewer

This is where you earn a large share of your score. Ask grouped questions, and — critically —
**explain why each answer changes the design.** Below, each question is paired with the design
fork it opens.

### Scope
| Question | Why it changes the design |
|---|---|
| Is this only *product* search, or also orders/sellers/content search? | Multiple corpora → multiple indices, federated ranking, more complex query routing. |
| Is autocomplete in scope? | If yes, it's a whole second low-latency subsystem at ~500K QPS (FR2). |
| Do we need image / visual search? | Pulls in embeddings + a vector index — big scope jump; brief says *mention as extension only*. |
| Is checkout/cart/recommendations part of this? | Explicitly **out of scope** — confirm so you don't waste minutes. |

### Scale
| Question | Why it changes the design |
|---|---|
| How many documents, and what's the growth rate? | 10B docs → ~600 shards, ~48 TB — drives all of Chapter 2. |
| Peak vs. average QPS, and how spiky? | 100K peak (~3× avg) → provision for peak + autoscale for sale events. |
| Write rate and its composition? | 100K writes/s, mostly price/inventory mutations → partial-update path, not full re-index. |
| Average document size? | ~2 KB → 20 TB raw → memory and node-count math. |

### Users
| Question | Why it changes the design |
|---|---|
| Global or single-region users? | Global → multi-region ES, locality routing, data residency (Chapter 9). |
| How many locales/markets, and is content localized? | Drives the `locale`/`market` fields, per-locale synonyms, and why one product yields many listing docs. |
| Logged-in vs. anonymous mix? | Determines whether personalization (FR7) is even feasible/worthwhile. |

### Consistency & freshness
| Question | Why it changes the design |
|---|---|
| Is eventual consistency acceptable on the index? | Yes → async CDC/Kafka pipeline, AP design. A "must be immediately consistent" answer would force a radically different (and worse) architecture. |
| How stale can price/inventory be? | ~1 s → tiered freshness with a fast partial-update path (FR8). |
| Is it OK to show an out-of-stock item (ranked low) vs. hide it entirely? | Changes whether availability is a *filter* or a *ranking signal*. |

### Ranking / relevance
| Question | Why it changes the design |
|---|---|
| Is personalization per-user required, or is popularity-based ranking enough? | Per-user → feature store + online features + user-history joins; a major complexity multiplier. |
| Which business signals matter (popularity, conversion, seller quality, price)? | Defines the LTR feature set and the Ranking Service (FR7). |
| How do we measure relevance success? | Anchors the metrics in §7 — NDCG offline, CTR/conversion online. |

### Constraints
| Question | Why it changes the design |
|---|---|
| Any mandated tech (must we use Elasticsearch)? | The brief anchors on ES, but *asking* shows you'd otherwise justify the choice (Chapter 11). |
| Budget / team size / operational maturity? | A 3-person team can't run a 100-node multi-region cluster the way a platform org can. |
| Compliance / data residency (GDPR, regional)? | May force regional data isolation, affecting multi-region topology. |

If you only have time for a handful, ask the ones with the biggest design forks:
**personalization? eventual consistency? autocomplete in scope? global multi-region?** Each of
those single answers moves the design by an order of magnitude.

---

## 6. Stating Assumptions and Defining Out-of-Scope

You will never get every question answered. The professional move is to **make an explicit
assumption and move on**, flagging it so the interviewer can veto it cheaply:

> "I'll assume ranking is popularity/conversion-based with optional light personalization, not
> full per-user models — tell me if you want deep personalization and I'll expand the ranking
> section. I'll also assume eventual consistency on the index is fine given the catalog is the
> source of truth."

Assumptions to state for GlobalMart (all consistent with the brief):

- Elasticsearch is the core engine; the Catalog service (sharded SQL / NoSQL) is the **source
  of truth**, and ES is a **derived, rebuildable index** — never authoritative.
- Eventual consistency on the index is acceptable (AP over CP on the read path).
- Read path is the SLA-critical, latency-sensitive path; the write path can be asynchronous.
- Traffic is spiky with ~3× peaks around sales events; we provision for ~100K QPS peak.
- Freshness is tiered: new listings visible in seconds, price/inventory in ~1 s.

**Out of scope (and *why*), so you don't burn minutes:**

| Out of scope | Why it's excluded |
|---|---|
| Checkout, payments, cart | Transactional systems with different consistency needs; not search. |
| Recommendations / home feed | Related but a distinct ML/serving problem (no query intent). |
| Image / visual search | Vector-search extension; *mention* it as future work, don't design it. |
| Fraud detection | Orthogonal domain. |
| Seller onboarding UI | We consume the catalog; we don't build its authoring tools. |

Naming what you're *not* building is not a cop-out — it's evidence you can bound a problem,
which is exactly the senior signal being assessed.

---

## 7. Success Metrics — Measuring Quality, Not Just Uptime

A search system that is fast and up but returns bad results is a *failed* search system. So
your metrics must cover **relevance quality**, not only operational health. This is a point
most candidates miss — raising it unprompted is a strong differentiator.

| Metric | What it measures | Type | Target / signal |
|---|---|---|---|
| **NDCG** (Normalized Discounted Cumulative Gain) | Ranking quality vs. ideal order, position-weighted | Offline (graded judgments) | Higher is better; primary ranking KPI |
| **MRR** (Mean Reciprocal Rank) | How high the *first* relevant result appears | Offline/online | Good for known-item / navigational queries |
| **CTR** (Click-Through Rate) | Do users click results? | Online | Proxy for perceived relevance |
| **Conversion rate** | Do searches lead to purchases? | Online (business) | The metric the business actually cares about |
| **Zero-result rate** | % of queries returning nothing | Online | Lower is better; attacked by spell/synonyms (FR6) |
| **Latency SLOs** | search p99 ≤ 200 ms; autocomplete p99 ≤ 100 ms | Operational | Hard SLO; breach = page |
| **Indexing lag** | Time from catalog write → searchable | Operational | ≤ seconds; price/inventory ≤ ~1 s (FR8) |
| **Availability** | Uptime of read/search path | Operational | 99.99% |

A few teaching points:

- **NDCG is the workhorse of relevance.** It rewards putting the most-relevant items at the
  *top* (gains are discounted by position) and normalizes so scores are comparable across
  queries. You compute it offline against human-graded query/result judgments before shipping a
  ranking change — this is how you catch a relevance regression that latency dashboards can't
  see.
- **Offline vs. online.** NDCG/MRR are offline gates on graded data; CTR/conversion/zero-result
  are measured live, typically via **A/B tests** and **interleaving**. You need both loops: an
  offline gate to avoid shipping regressions, and an online signal to know real users are
  better served.
- **Zero-result rate is a product metric, not just a quality one.** A query that returns
  nothing is a dead end and a lost sale; it's the direct scoreboard for your spell-correction
  and synonym work (FR6), and the trigger for a "did you mean / related results" fallback.
- **Indexing lag closes the loop on freshness.** It's the *measurable* form of FR8/the
  freshness NFR — instrument the CDC→Kafka→ES pipeline end-to-end so you can alert when the
  price/inventory lag exceeds ~1 s.

Say explicitly: *"Uptime tells me the system is running; NDCG and conversion tell me it's
actually doing its job. I'd track both, with latency and indexing-lag as hard SLOs and
relevance metrics as ship/no-ship gates on ranking changes."*

---

## 8. From Requirements to a Design Contract

The output of this phase is a **design contract**: a compact, quantified spec that every later
chapter must satisfy. Writing it down (or stating it) closes the requirements phase cleanly and
gives you a checklist to validate the design against at the end (Chapter 12 does exactly this).

```
GLOBALMART SEARCH — DESIGN CONTRACT
-----------------------------------
FUNCTIONAL (must support)
  FR1 full-text  FR2 autocomplete  FR3 filter/facet+counts  FR4 sort
  FR5 cursor pagination  FR6 spell+synonyms  FR7 ranking signals  FR8 NRT index

SCALE (must handle)
  10B docs · ~20 TB raw · 100K peak QPS search · ~500K QPS autocomplete
  100K writes/s (mostly price/inventory) · 50M new listings/day · global/multi-region

NON-FUNCTIONAL (must meet)
  search p99 ≤ 200 ms · autocomplete p99 ≤ 100 ms
  99.99% availability on read path
  freshness: listings in seconds, price/inventory ≤ ~1 s
  consistency: eventual (AP over CP); Catalog = source of truth, ES = derived index

MEASURED BY
  relevance: NDCG (offline gate), CTR/conversion (online), zero-result rate
  ops: latency p99 SLO, indexing lag, availability

OUT OF SCOPE
  checkout/cart/payments · recommendations feed · image search (extension only)
  · fraud · seller onboarding UI
```

Every downstream chapter now has a target to hit and a target to be judged against:

- **Chapter 2 (Capacity Estimation)** turns the SCALE line into shards, nodes, RAM, bandwidth.
- **Chapter 3 (APIs)** encodes FR1–FR5 into `/v1/search` and `/v1/autocomplete` contracts,
  including `search_after` pagination.
- **Chapter 4 (Data Model)** realizes the document schema and the source-of-truth-vs-index rule.
- **Chapters 5–7** build the read and write paths to hit the latency budget and freshness targets.
- **Chapters 8–9** make the ES scatter-gather meet 120 ms at 100K QPS.
- **Chapters 10–11** defend the availability and consistency (AP) choices under failure.

A requirement that isn't in the contract can't be scored; a design choice that isn't traceable
to the contract is gold-plating. The contract keeps you honest in both directions.

---

## Interview Tips

- **Do not say "I'll use Elasticsearch" in the first two minutes.** This is the #1 trap. It
  signals you're replaying a memorized solution instead of reasoning from requirements. The
  engine is a *conclusion* you derive in Chapter 11 from the requirements you gather here. Let
  the interviewer see the derivation.
- **Restate the problem before you design.** Thirty seconds of "here's what I think we're
  building" prevents ten minutes of solving the wrong thing, and immediately reads as senior.
- **Lead with the four buckets** (functional / non-functional / constraints / scale). Announcing
  the structure makes you easy to follow and shows you have a repeatable method.
- **Ask clarifying questions *with their consequences attached*.** "Is personalization in
  scope? Because if it is, I'll need a feature store and user-history joins, which roughly
  doubles the ranking complexity." The *because* is what scores — anyone can ask a question.
- **Quantify or flag.** Turn every "fast/reliable/fresh" into a number, or say "I'll assume X;
  correct me." Vague NFRs are un-designable and un-scorable.
- **Name the trade-offs out loud**, especially AP-over-CP and freshness-vs-throughput. Interviewers
  are explicitly listening for whether you know what you're giving up.
- **Bring up relevance metrics unprompted.** Most candidates only mention latency/uptime.
  Saying "NDCG and conversion, not just p99" is a cheap, high-value differentiator.
- **Bound the scope proactively.** Stating what's *out* of scope (and why) is as valuable as
  stating what's in. It shows you can ship, not just enumerate.
- **Watch the clock.** Spend ~5–8 minutes here, then move on. Requirement gathering is
  high-leverage, but a candidate still stuck scoping at minute 20 hasn't shown they can build
  anything.
- **Don't over-clarify.** If the interviewer isn't engaging, make reasonable assumptions and
  proceed. The skill is knowing which 3–4 questions actually move the design.

## Key Takeaways

- Requirement gathering is the **highest-leverage phase**: it sets the rubric, bounds the
  search space, and is where seniority shows.
- Scope along **four buckets** — functional, non-functional, constraints, scale — and keep
  requirements **solution-agnostic** (the *what*, not the *how*).
- GlobalMart's functional core is **FR1–FR8**: full-text, autocomplete, filter/facet, sort,
  pagination, spell/synonyms, ranking signals, NRT indexing.
- The NFR targets are fixed: **search p99 ≤ 200 ms, autocomplete p99 ≤ 100 ms, 99.99%
  availability, ~1 s freshness**, over **10B docs / 100K peak QPS**.
- NFRs **trade against each other**; the defining choice is **AP over CP** — eventual
  consistency on a derived, rebuildable index, with the Catalog as source of truth.
- Ask **clarifying questions with consequences attached**; state **assumptions explicitly**;
  declare what's **out of scope** and why.
- Measure **relevance quality (NDCG, MRR, CTR, conversion, zero-result rate)**, not just
  latency and uptime.
- Crystallize everything into a **design contract** — the quantified spec every later chapter
  must satisfy and be judged against.
