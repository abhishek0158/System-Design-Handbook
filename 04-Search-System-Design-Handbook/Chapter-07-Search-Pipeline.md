# Chapter 7 — Search Pipeline

> **Where we are.** Chapter 5 gave us the end-to-end architecture and Chapter 6 walked the
> *write* path (CDC → Kafka → Indexing Service → Elasticsearch). This chapter is the mirror
> image: the **read path**. From the moment a buyer's keystroke leaves the browser to the
> moment a ranked, faceted result set comes back, what happens — and how do we do it inside a
> **200 ms p99** budget at **~100 K QPS peak**?
>
> This is a flagship chapter. Query understanding, retrieval, two-phase ranking, faceting,
> caching, personalization, autocomplete, and relevance measurement are exactly the topics a
> senior interviewer drills into once you've drawn the boxes. We'll spend our words on the
> *why* behind each stage, then show the concrete Elasticsearch DSL.

---

## 7.1 The Query Lifecycle at a Glance

A search request is not one operation; it is a **pipeline of stages**, each with its own
latency envelope. The golden rule of low-latency search is that the budget is *spent
sequentially*, so every stage's cost is additive on the critical path unless we deliberately
parallelize or cache it.

Recall the canonical latency budget from the brief (search path, target **p99 200 ms**):

| Stage | Budget (p99) | What happens |
|---|---:|---|
| Edge / LB | 5 ms | TLS termination, routing at CDN/Edge |
| API Gateway + auth | 10 ms | JWT validation, rate-limit, request parse |
| Query Understanding | 15 ms | tokenize, normalize, spell-correct, expand synonyms, detect intent |
| ES query (scatter-gather) | 120 ms | fan-out to ~600 shards, BM25 retrieval, aggregations |
| Ranking / rerank | 30 ms | LTR rerank of top-K, business-signal blending |
| Serialization | 10 ms | build JSON response, facets, metadata |
| Buffer | 10 ms | slack for GC pauses, tail variance |
| **Total** | **200 ms** | |

Here is the full lifecycle from raw string to ranked results:

```
                      ┌──────────────────────────── SEARCH SERVICE ────────────────────────────┐
                      │                                                                          │
 Client              │   ┌─────────────────┐    ┌───────────────────────┐                       │
 "iphn 128gb"        │   │ 1. Result Cache │    │ 2. Query Understanding│                       │
   │                 │   │    (Redis)      │    │  tokenize→normalize→  │                       │
   ▼                 │   │  key = hash(    │miss│  spell→synonyms→      │                       │
CDN/Edge ─▶ API GW ──┼──▶│  q,filters,     │───▶│  intent/category      │                       │
 (5ms)     (10ms)    │   │  sort,locale,pg)│    │  detection (15ms)     │                       │
                      │   └────────┬────────┘    └───────────┬───────────┘                       │
                      │       hit  │                          │ "iphone" + storage:128GB          │
                      │            │                          ▼                                    │
                      │            │             ┌────────────────────────────┐                   │
                      │            │             │ 3. Retrieval (candidate gen)│                   │
                      │            │             │  ES bool query:            │                   │
                      │            │             │  filter (cacheable) +      │                   │
                      │            │             │  multi_match should/must   │                   │
                      │            │             └──────────────┬─────────────┘                   │
                      │            │                             │ builds request                 │
                      │            │                             ▼                                 │
                      │            │        ┌────────────────────────────────────────┐            │
                      │            │        │  Elasticsearch (coordinating → data)   │            │
                      │            │        │  Phase A: BM25 top-N per shard         │──scatter──▶ │
                      │            │        │  + terms/range aggregations (facets)   │◀──gather──  │
                      │            │        └───────────────────┬────────────────────┘  (120ms)   │
                      │            │                             │ top-N + facet counts            │
                      │            │                             ▼                                 │
                      │            │        ┌────────────────────────────────────────┐            │
                      │            │        │ 4. Ranking / LTR reranker              │            │
                      │            │        │  Phase B: rerank top-K with            │            │
                      │            │        │  function_score / LTR model            │            │
                      │            │        │  (business signals) (30ms)             │            │
                      │            │        └───────────────────┬────────────────────┘            │
                      │            │                             │                                 │
                      │            ▼                             ▼                                 │
                      │   ┌──────────────────────────────────────────────────┐                   │
                      │   │ 5. Assemble response: results + facets + metadata │ (serialize 10ms)  │
                      │   │    (corrected spelling, applied synonyms)         │                   │
                      │   └────────────────────────┬─────────────────────────┘                   │
                      │                             │ write-through cache (TTL)                    │
                      └─────────────────────────────┼───────────────────────────────────────────┘
                                                    ▼
                                                 Client
```

Two structural decisions dominate the rest of the chapter:

1. **Cache before you compute.** The Result Cache (Redis) sits *in front of* query
   understanding and ES. A cache hit collapses the whole pipeline to a Redis round trip
   (single-digit ms). This is why cacheability is a first-class design concern (§7.7).
2. **Cheap-then-expensive ranking.** We never run an expensive model over 10 B documents.
   We retrieve cheaply (BM25) to a few hundred candidates, then rerank expensively over a few
   dozen (§7.4). Latency math forces this shape.

---

## 7.2 Query Understanding (FR6)

The raw query string is a hostile artifact: typos, mixed case, accents, extra whitespace,
stopwords, ambiguous intent, and multiple locales. **Query Understanding** is the stage that
turns `"iphn 128gb"` into a structured, expanded, corrected query the retrieval layer can
execute well. It runs in the Search Service (brief §5) inside a **~15 ms** budget, so every
sub-step must be cheap — mostly dictionary/FST lookups and cached models, never a network
call to a heavy service on the hot path.

### 7.2.1 Tokenization and normalization

Before anything semantic, we normalize the surface form. This mirrors (and must stay
consistent with) the **index-time analyzer** from Chapter 4 — if query-time and index-time
analysis diverge, terms won't match.

- **Lowercasing:** `"iPhone"` → `"iphone"`.
- **Unicode normalization (NFKC) + ASCII folding:** `"café"` → `"cafe"`, full-width →
  half-width. Critical for a multi-locale catalog.
- **Tokenization:** split on whitespace/punctuation, but preserve meaningful units. `"128gb"`
  should survive as a token (or be split into `128` + `gb` *and* kept as `128gb` via a
  word-delimiter filter so both `"128 gb"` and `"128gb"` queries match).
- **Trim / collapse whitespace, strip control chars.**

> **Interview trap:** candidates say "just lowercase and split." The subtlety is that
> **query-time analysis must match index-time analysis**. If the index used `edge_ngram` on
> title but the query analyzer also applies `edge_ngram`, you get a token explosion and
> garbage matches. The rule: **index with ngrams, search with a `search_analyzer` that does
> *not* re-ngram.** State this explicitly.

### 7.2.2 Spell correction (FR6)

`"iphn"` is not in the term dictionary. We correct it before retrieval, because searching the
misspelled token yields zero or junk results.

Techniques, cheapest first:

1. **Dictionary + edit distance (Levenshtein automaton).** Build a term dictionary from the
   corpus (high-frequency terms per locale). For an out-of-vocabulary token, find candidates
   within edit distance 1–2. Elasticsearch exposes this via the **term/phrase suggester** and
   fuzzy queries, but on the hot path we prefer a **precomputed correction dictionary** in the
   Query Understanding service so we don't pay an extra ES round trip.
2. **Frequency-weighted ranking of candidates.** `"iphn"` is within edit distance 1 of both
   `"iphone"` and (hypothetically) `"phon"`. Pick the candidate with the highest corpus
   frequency *and* the highest historical CTR. This is a noisy-channel model:
   `score(correction) ∝ P(term) × P(typo | term)`.
3. **Context / bigram model.** `"iphn case"` disambiguates toward `"iphone case"`. A small
   bigram language model over query logs resolves corrections that a per-token model can't.
4. **"Did you mean" vs. auto-correct.** For high-confidence corrections, silently rewrite and
   return results with a `spell_corrected: {from, to}` field in metadata. For low-confidence,
   return original results plus a suggestion. Amazon-style UX: auto-correct with a
   "Showing results for **iphone**. Search instead for *iphn*" affordance.

If a corrected query still returns few results, fall back to a **fuzzy `multi_match`**
(`fuzziness: AUTO`) at retrieval time — but fuzzy queries are expensive (they expand each term
to many variants), so use them as a **fallback**, not the default.

### 7.2.3 Synonym expansion (FR6)

`"trainers"` and `"sneakers"` are the same product; `"nb"` means `"new balance"`. Synonyms let
one query match documents phrased differently.

Two placement choices, with a clear recommendation:

| Approach | Where | Pros | Cons |
|---|---|---|---|
| **Index-time expansion** | Analyzer at ingest | Fast queries; synonyms baked into inverted index | Re-index the world to change the synonym list; index bloat |
| **Query-time expansion** | `synonym_graph` filter at search | Update synonyms without reindex; smaller index | Slightly heavier queries; multi-word synonyms need `synonym_graph` |

**Recommendation:** query-time `synonym_graph` for anything volatile (marketing terms, brand
aliases, seasonal), because synonym lists change weekly and reindexing 10 B docs is not free
(Chapter 6). Reserve index-time synonyms for a stable, curated core set. Use the
**`synonym_graph` token filter** (not the legacy `synonym`) so multi-word synonyms
(`"ny" → "new york"`) tokenize correctly.

> Synonyms are directional and dangerous. `"apple" → "iphone"` looks helpful until someone
> searches for Apple-brand *headphones* or literal apples (grocery). Curate synonyms
> **per-category** and measure with the relevance harness in §7.10 before shipping.

### 7.2.4 Stopword handling

`"the", "a", "of", "for"` carry little signal and inflate posting-list traversal. But **do not
strip stopwords blindly** — `"the north face"` and `"it" (the movie/product)` are real
queries where the stopword is load-bearing. Modern practice:

- Keep stopwords in the index (cheap with BM25's term saturation), and
- Down-weight them at query time rather than removing them, or
- Use `"common terms"`-style handling / `minimum_should_match` so a missing stopword doesn't
  drop an otherwise-perfect match.

### 7.2.5 Category / intent detection

Knowing that `"iphone 128gb"` targets **Electronics › Phones › Smartphones** lets us (a) apply
a category boost or soft filter, (b) surface the right facets (storage, color — not shoe
size), and (c) pick category-specific synonyms/rankers. Techniques:

- **Query classification model** (lightweight logistic regression / gradient-boosted trees or
  a small distilled transformer) trained on `(query → clicked category)` from logs. Output: a
  probability distribution over top categories. Cached by normalized query string.
- **Attribute extraction / entity recognition:** parse `"128gb"` → `attributes.storage=128GB`,
  `"black"` → `attributes.color=black`, `"under $500"` → `price.amount < 500`. This turns free
  text into structured filters, dramatically improving precision.
- **Intent flags:** navigational ("nike air max 90" → specific product), broad ("shoes"),
  or transactional ("buy iphone"). Broad queries lean on business signals for ranking; narrow
  queries lean on textual match.

### 7.2.6 Multi-locale queries

The corpus spans locales (`en-US`, `de-DE`, `ja-JP`, …). Query Understanding must:

- **Route by `locale`/`market`** params (brief §3) so we search the right analyzers and the
  right documents. A German query needs German stemming, decompounding
  (`"waschmaschine"` → `"wasch" + "maschine"`), and German synonyms.
- **Handle CJK** with dedicated tokenizers (ICU / kuromoji for Japanese, no whitespace to
  split on).
- **Cross-locale fallback:** brand/model tokens (`"iphone"`, `"sony"`) are locale-agnostic and
  should match across locales; descriptive words should not. Keep a locale-neutral `brand`
  subfield.

### 7.2.7 Worked example: `"iphn 128gb"`

```
RAW:            "iPhn  128GB"
─ normalize:    "iphn 128gb"                     (lowercase, collapse ws)
─ tokenize:     ["iphn", "128gb"]  → also ["128","gb"] via word-delimiter
─ spell-correct:"iphn" → "iphone"  (edit-distance 1, high corpus freq + CTR)
─ attr-extract: "128gb" → attributes.storage = "128GB"
─ synonym:      "iphone" → {iphone, "apple phone"}   (query-time graph, Electronics category)
─ intent:       category = Electronics›Phones›Smartphones (p=0.94), intent=navigational
─ RESULT (structured query):
     text match: (iphone OR "apple phone")  over title^3, brand^2, description^1
     soft filter: attributes.storage = 128GB   (boost, not hard filter — see §7.3)
     category boost: Smartphones
     metadata returned to client: { spell_corrected: {from:"iphn", to:"iphone"} }
```

That structured object — not the raw string — is what the retrieval layer executes.

### 7.2.8 Keeping Query Understanding inside 15 ms

Every sub-step above is a potential network call or model inference, and we have **15 ms** for
all of them combined. The discipline that makes this budget real:

- **Everything is a local lookup or an in-process model.** Spell dictionaries, synonym graphs,
  and the correction/intent models are loaded into the Search Service's memory (or a co-located
  sidecar), refreshed periodically out-of-band. No per-query hop to a "spell service."
- **Cache the *understanding*, not just the results.** The mapping `normalized_query →
  {corrected, expanded, intent}` is itself cached in Redis with a longer TTL than the result
  cache (understanding of `"iphn"` doesn't change minute-to-minute even when prices do). On a
  result-cache miss, we still usually get a *understanding-cache* hit, so we skip the models.
- **Degrade, don't block.** If the intent model times out (say >5 ms), skip it and proceed with
  plain retrieval — a slightly less-targeted result beats a blown latency budget. Query
  Understanding is *enhancement*; retrieval must never hard-depend on it (Chapter 10).

---

## 7.3 Retrieval — Candidate Generation

Retrieval's job is **recall, cheaply**: get *all plausibly relevant* documents into a
candidate set of a few hundred, fast. We do **not** try to get the final order right here —
that's ranking's job (§7.4). This split is the single most important idea in the chapter.

### 7.3.1 The `bool` query: must / should / filter

Elasticsearch's `bool` query has four clause types; three matter here:

| Clause | Scoring? | Cacheable? | Semantics |
|---|---|---|---|
| `filter` | **No** (yes/no) | **Yes** | Hard constraint. Must match, contributes 0 to score. |
| `must` | Yes | No | Must match *and* contributes to relevance score. |
| `should` | Yes | No | Optional; boosts score. `minimum_should_match` can force ≥N. |
| `must_not` | No | Yes | Exclusion. |

The design pattern for e-commerce search: **hard constraints in `filter`, relevance in
`must`/`should`.**

```json
GET /products/_search
{
  "size": 24,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "market": "US" } },
        { "term":  { "in_stock": true } },
        { "range": { "price.amount": { "lte": 1000 } } },
        { "terms": { "category_path": ["Electronics","Phones","Smartphones"] } }
      ],
      "must": [
        {
          "multi_match": {
            "query": "iphone",
            "type": "best_fields",
            "fields": ["title^3", "brand^2", "description^1"],
            "fuzziness": "AUTO",
            "minimum_should_match": "2<70%"
          }
        }
      ],
      "should": [
        { "match": { "title.raw": { "query": "iphone", "boost": 2 } } },
        { "term":  { "attributes.storage": { "value": "128GB", "boost": 1.5 } } }
      ]
    }
  }
}
```

Note that `attributes.storage = 128GB` from our worked example is a **`should` boost**, not a
`filter`. Why? Because a hard filter would drop a perfect 256 GB match the buyer might happily
buy; a soft boost surfaces 128 GB first while keeping alternatives. Reserve hard `filter` for
constraints the buyer explicitly set via the UI (facets: "In stock only", "Under $1000").

### 7.3.2 `multi_match` and field boosting

Titles are the strongest relevance signal, brand next, description weakest (it's keyword-stuffed
by sellers). The `^N` syntax boosts a field's contribution: `title^3` means a title match is
worth 3× a description match. `multi_match` types worth knowing:

- **`best_fields`** (default): score = best single field's score. Good when a match in *one*
  field (the title) is what matters. This is our default.
- **`most_fields`**: sum across fields. Good when the same text appears in multiple analyzed
  fields (e.g., `title` + `title.stemmed`).
- **`cross_fields`**: treats multiple fields as one big field, term-centric. Good for
  `"first last"` name-style queries spanning fields.

Field boosts are a blunt instrument and a classic **over-tuning trap** — hand-tuned `^` values
are the thing LTR (§7.4) later learns properly. Set sane defaults, then let the model earn the
weights from click data.

### 7.3.3 Query context vs. filter context — why filters are cheap

This distinction is worth its own section because interviewers love it.

- **Query context** answers *"how well does this document match?"* — it computes a `_score`
  (BM25). Scoring touches term frequencies, field lengths, and IDF; it cannot be reused across
  queries because the score depends on the specific query terms.
- **Filter context** answers *"does this document match, yes or no?"* — no score. Because the
  answer is a pure boolean, Elasticsearch caches it as a **bitset** in the **node query cache**
  (the "filter cache"), keyed by the filter clause. The next query with `in_stock: true`
  reuses the precomputed bitset — an O(1) AND of bitsets instead of re-evaluating the clause.

Consequences we exploit:

1. **Put every non-scoring constraint in `filter`.** `market`, `in_stock`, `price range`,
   `category`, `seller` — all boolean, all cacheable, all cheap.
2. **Filters compose by bitset intersection**, which is astonishingly fast (SIMD-friendly
   popcount/AND over roaring bitmaps). A query filtered to `market:US AND in_stock:true`
   narrows 10 B docs to the candidate universe before any scoring happens.
3. **Stable filters get warm caches.** `market:US` is on nearly every US query, so its bitset
   is essentially always hot. This is why faceted navigation (lots of repeated filters) stays
   cheap.

> **Rule of thumb:** *If a clause doesn't need to affect the order of results, it belongs in
> `filter`.* Scoring is the expensive part; do it on as few documents as possible.

### 7.3.4 Sorting and pagination interplay (FR4, FR5)

Retrieval interacts with two request params from the API surface (brief §3): `sort` and
`page`/`size`.

- **`sort=relevance` (default)** uses the two-phase scored order of §7.4. This is the only sort
  where the expensive rerank matters.
- **`sort=price|rating|newest|popularity`** replaces the score with a **`doc_values` sort** on
  that field. When the user sorts by price, relevance ordering is discarded — but *the query and
  filters still define the candidate set*. A subtle consequence: sorting by a field lets ES skip
  scoring entirely (it can short-circuit once it has the top-N by that field), so these sorts are
  often *cheaper* than relevance sort. Keep the LTR rescore **only** on the relevance path;
  running it under `sort=price` is wasted work.
- **Deep pagination** must use **`search_after`** cursors, not `from`+`size`, beyond ~page 50
  (brief §3, §6). `from`+`size` forces every shard to build and ship a priority queue of
  `from+size` hits to the coordinating node — at `from=10000` that's 10 K hits × ~600 shards
  gathered and re-sorted, an O(from) memory blow-up that threatens the 120 ms ES budget.
  `search_after` instead passes the sort values of the last item as a cursor, so each shard
  resumes from a point and only returns the next `size` hits — O(size), constant regardless of
  depth. The Result Cache key (§7.7) therefore includes the page/cursor.

> Tell the interviewer: *"Non-relevance sorts skip scoring and rerank entirely; deep pages use
> `search_after` because `from`+`size` is O(from) per shell across 600 shards."* Both are quick
> senior signals.

---

## 7.4 Two-Phase Ranking

We have a candidate set. Now we order it. The core architectural move: **two phases** — a
cheap first pass over many documents, an expensive second pass over few.

### 7.4.1 Why two phases? The latency argument

Suppose the query matches 2 M documents (common for a broad query like `"shoes"`). We want the
best 24. Options:

- **Score all 2 M with an expensive model** → correct order, but a heavy LTR model at, say,
  50 µs/doc × 2 M = **100 seconds per shard**. Absurd. Blows the 120 ms ES budget by
  three orders of magnitude.
- **Score all 2 M with cheap BM25, then rerank the top few with the expensive model** → BM25
  is a single dot-product over a handful of query terms per doc (nanoseconds, and it only
  visits docs on the query's posting lists, not all 2 M). Then run the expensive model on just
  the top **K ≈ 100–500** survivors.

The math is the whole point: **expensive_cost × N is infeasible; expensive_cost × K is cheap.**
This is why every production search stack — GlobalMart included — is **retrieve-then-rerank**.

```
   2,000,000 matches
        │  Phase A: BM25 (cheap, per-shard, top-N each)      ~120ms in ES budget
        ▼
   top-N ≈ 500 candidates (gathered at coordinating node)
        │  Phase B: LTR / function_score rerank (expensive)  ~30ms in rerank budget
        ▼
   top-K ≈ 24 returned, correctly ordered
```

Elasticsearch supports this natively with **`rescore`** (rerank only the top window returned by
the query) and with the **Learning-to-Rank plugin** (`sltr` / `ltr` rescorer). The rescore
window is exactly our K.

### 7.4.2 Phase A — cheap retrieval (BM25)

Phase A is the `bool` query from §7.3. BM25 gives each candidate a textual relevance score.
It runs **per-shard** (each of ~600 shards returns its local top-N), and the coordinating node
merges them (scatter-gather, Chapter 8). BM25 is cheap because it only touches documents on the
query terms' posting lists and uses precomputed term statistics.

### 7.4.3 Phase B — expensive rerank (LTR / function_score)

Two implementation styles, often combined:

**(a) `function_score` — rule-based blending.** Blend BM25 with business signals via explicit
functions. Good baseline, fully explainable, no training pipeline required:

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { "bool": { /* the §7.3 retrieval query */ } },
      "functions": [
        { "field_value_factor": {
            "field": "popularity", "factor": 1.2, "modifier": "sqrt", "missing": 0.1 } },
        { "field_value_factor": {
            "field": "seller_rating", "factor": 1.0, "modifier": "ln1p", "missing": 3.0 } },
        { "filter": { "term": { "in_stock": true } }, "weight": 1.5 },
        { "gauss": {
            "updated_at": { "origin": "now", "scale": "30d", "decay": 0.5 } } }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

`boost_mode: multiply` means final = `BM25_score × combined_business_factor`, so textual
relevance sets the baseline and business signals scale it. `score_mode: sum` combines the
individual functions.

**(b) Learning-to-Rank (LTR) — model-based.** Train a model (LambdaMART / gradient-boosted
trees, or a neural ranker) to predict *click/conversion* from a **feature vector** per
`(query, document)` pair. Features:

| Feature category | Examples |
|---|---|
| Textual relevance | BM25 on title, BM25 on description, phrase-match flag, `title.raw` exact match |
| Document quality | `popularity`, `rating`, `review_count`, `seller_rating` |
| Business | `price` competitiveness (vs. category median), margin, `in_stock`, shipping speed |
| Query-doc match | category match probability, attribute overlap (storage/color) |
| Behavioral | historical CTR / conversion for this doc on this query |

At query time, the LTR rescorer computes features for the **top-K** candidates only and applies
the model:

```json
"rescore": {
  "window_size": 200,
  "query": {
    "rescore_query": {
      "sltr": {
        "params": { "query_string": "iphone" },
        "model": "globalmart_ltr_v7"
      }
    },
    "query_weight": 0.0,
    "rescore_query_weight": 1.0
  }
}
```

`window_size: 200` is our K — the model runs on 200 docs, not the full match set. This is the
concrete realization of §7.4.1's latency argument, and it fits the 30 ms rerank budget.

**LTR vs. function_score:** start with `function_score` (ship in a week, explainable, no
training loop). Graduate to LTR once you have enough labeled click data and a training/serving
pipeline — LTR learns feature weights (including the field boosts of §7.3.2) from real behavior
instead of guesswork, and typically lifts NDCG/conversion meaningfully. Keep `function_score`
as the fallback if the model service is degraded (§7.10, graceful degradation).

### 7.4.4 Feature logging and the training loop

An LTR model is only as good as its labels, and the label source that scales is **behavioral
data**, not human raters. The loop the Ranking Service closes:

1. **Log features at serve time.** For every query, log the feature vector of the top-K
   candidates *as computed* — the exact BM25 sub-scores, popularity, price competitiveness, etc.
   Logging features at inference (not recomputing them later from a snapshot) avoids
   **training/serving skew**, where the model trains on features that differ from what
   production produces.
2. **Join with judgments.** Attribute clicks, add-to-carts, and conversions back to the query
   and the shown position. **Debias for position** — item #1 gets clicked because it's #1, not
   only because it's best; a click model (e.g., inverse-propensity weighting) corrects this or
   the model just learns "rank whatever we already ranked first."
3. **Train offline** (LambdaMART optimizing NDCG is the pragmatic default), validate against a
   held-out judgment set via `_rank_eval`, then A/B online (§7.10).
4. **Version and roll.** Models are versioned (`globalmart_ltr_v7`) like indices; a new model
   ships behind a flag with instant rollback to the previous version or to `function_score`.

The reranker itself is a **stateless service** (brief §5, Ranking Service / LTR reranker) fed
the candidate feature vectors from the coordinating node; it holds the model in memory and
scores the top-K synchronously inside the 30 ms budget. Because it's stateless, we scale it
horizontally with QPS and it's trivially cacheable behind the segment-level Result Cache.

---

## 7.5 Ranking Signals — Combining Relevance and Business

Textual relevance answers *"is this what they asked for?"*; business signals answer *"is this a
good thing to show them?"* A perfectly-matching listing from a 2-star seller who's out of stock
is a bad result. Blending is the art.

### 7.5.1 The signal palette (from the canonical data model)

| Signal | Field | Direction | Notes |
|---|---|---|---|
| Textual relevance | BM25 | ↑ | The baseline; everything scales it |
| Popularity | `popularity` (0–1) | ↑ | Normalized sales/CTR; use `sqrt`/`log` to tame heavy tail |
| Seller quality | `seller_rating` | ↑ | Trust signal |
| Product rating | `rating`, `review_count` | ↑ | Weight by review count (Bayesian avg to avoid 1-review 5.0★) |
| Price competitiveness | `price.amount` | context | Cheaper *within a product* is better; compare to category/product median |
| Availability | `in_stock`, `inventory` | ↑ | Demote/hide out-of-stock |
| Freshness | `updated_at`, `created_at` | ↑ (decay) | New listings get a temporary boost |

### 7.5.2 Combination strategies

- **Multiplicative (`boost_mode: multiply`)** — business factors *scale* relevance. A great-
  quality listing that's a poor textual match still can't outrank a good match, because a low
  BM25 baseline caps it. Preferred default: it preserves relevance primacy.
- **Additive (`sum`)** — risk: a hugely popular but irrelevant item floods results. Use only
  for small, bounded boosts.
- **`script_score`** — when you need a custom formula ES functions can't express. Full control,
  but you own the performance:

```json
"script_score": {
  "script": {
    "source": """
      double rel = _score;
      double pop = Math.sqrt(doc['popularity'].value);
      double stock = doc['in_stock'].value ? 1.0 : 0.2;   // demote OOS, don't drop
      double fresh = decayDateGauss('updated_at', params.now, '30d');
      return rel * (1 + 0.5*pop) * stock * (1 + 0.1*fresh);
    """,
    "params": { "now": "2026-07-15T00:00:00Z" }
  }
}
```

`script_score` is flexible but skips some Lucene optimizations, so keep it on the **rescore
window (top-K)**, never the full query.

### 7.5.3 Freshness and availability, concretely

- **Freshness boost** with a Gaussian/exponential **decay** (`gauss` on `updated_at`): a
  listing updated today scores ~2×; the boost decays to ~1× over 30 days. This surfaces new
  listings (FR8's NRT work is wasted if new docs never rank) without permanently privileging
  churn.
- **Demote, don't delete, out-of-stock.** Multiply OOS scores by ~0.2 rather than filtering
  them out entirely — a buyer searching a specific model may still want to see it (and click
  "notify me"). *Unless* the buyer set the "In stock only" facet, in which case it becomes a
  hard `filter` (§7.3). This is a UX/relevance decision to state explicitly in an interview.

---

## 7.6 Faceting and Aggregations (FR3)

Alongside results, the response carries **facets**: the brand list with counts, price
histogram, rating buckets, availability toggle. These come from **Elasticsearch aggregations**
computed in the *same* request as the search (one round trip, not two).

### 7.6.1 Facet aggregations

```json
GET /products/_search
{
  "query": { "bool": { /* retrieval + filters */ } },
  "size": 24,
  "aggs": {
    "brands":   { "terms": { "field": "brand", "size": 20 } },
    "categories": { "terms": { "field": "category_path", "size": 15 } },
    "ratings":  { "range": { "field": "rating",
        "ranges": [ {"from":4}, {"from":3,"to":4}, {"from":2,"to":3} ] } },
    "price":    { "histogram": { "field": "price.amount", "interval": 100 } },
    "in_stock": { "terms": { "field": "in_stock" } }
  }
}
```

- **`terms` agg** → discrete facets (brand, category, color). Backed by **`doc_values`**
  (columnar, on-disk, Chapter 4/8), not the inverted index.
- **`range` / `histogram` agg** → price buckets, rating bands.

### 7.6.2 The cost of aggregations, and mitigation

Aggregations are often **more expensive than the query itself** because they must visit *every
matching document's* doc_values to build buckets, whereas the query only needs the top-N. On a
broad query matching millions of docs, this dominates latency.

Mitigations, in order of impact:

1. **Compute facets in filter context.** Facet counts should reflect the query + *other*
   facets, but crucially they run over the filtered candidate set, which the filter-cache
   bitsets (§7.3.3) already narrowed cheaply.
2. **Limit `size`** on `terms` aggs. You don't need 10,000 brands; the UI shows ~20. Smaller
   `size` = smaller priority queues per shard.
3. **`execution_hint`** and eager global ordinals for high-cardinality keyword fields (brand,
   seller) so ordinals are prebuilt at refresh, not per query.
4. **Sampling for approximate facets** on ultra-broad queries: run aggs on a
   `sampler`/`diversified_sampler` of top docs when exact counts aren't worth the latency. Facet
   counts are a hint, not an invoice — "About 1,200 in Nike" is fine.
5. **Cache facet blocks in the Result Cache** for popular query+filter combinations (§7.7).
6. **Separate agg-heavy from result-heavy paths** if needed: `size:0` agg-only requests are
   cacheable and can be served from the ES **request cache** (which only caches `size:0` /
   aggregation responses per shard).

> **Interview point:** the honest answer to "why is faceting expensive?" is *"aggregations scan
> all matching docs' doc_values, not just the top-N — so their cost scales with match-set size,
> not page size."* Then list the mitigations.

---

## 7.7 Caching — Redis Result Cache and the ES Request Cache

Caching is how we survive 100 K QPS. There are **two** caches on the read path; keep them
straight.

### 7.7.1 The two caches

| Cache | Scope | Caches what | Keyed by |
|---|---|---|---|
| **Result Cache (Redis)** | Search Service, cluster-wide | Fully assembled response (results + facets + metadata) | hash of normalized(q, filters, sort, page, size, locale, market) |
| **ES request cache** | Per data node, per shard | Shard-local `size:0` / aggregation results | shard + full request body |
| **ES node query cache** | Per data node | Filter bitsets (§7.3.3) | filter clause |

The **Redis Result Cache** is the big lever: a hit skips query understanding, ES, *and*
ranking. At e-commerce head-query skew (a small set of queries — `"iphone"`, `"airpods"`,
`"shoes"` — is a large fraction of traffic), a well-tuned result cache serves **50–70%** of
queries from Redis in single-digit ms, slashing ES load and tail latency.

### 7.7.2 Cache key design

The key must capture **everything that changes the response**:

```
key = sha1( normalized_query + "|" + sorted(filters) + "|" + sort +
            "|" + page + "|" + size + "|" + locale + "|" + market )
```

Subtleties:

- **Normalize before hashing** so `"iPhone"`, `"iphone "`, `"iphone"` share one entry. Do the
  §7.2.1 normalization first — this is a big hit-rate multiplier.
- **Sort filter values** so `brand:nike,color:red` and `color:red,brand:nike` collide.
- **Exclude volatile-but-irrelevant params** (request id, client timestamp).
- **Personalization busts the key** — see §7.8. Only cache the **non-personalized** response;
  fold personalization in *after* the cache read.

### 7.7.3 TTL vs. NRT freshness — the core tension

FR8 demands price/inventory freshness within ~1 s. A cache with a 5-minute TTL directly fights
that. Resolve it with **short TTLs plus targeted invalidation**:

- **Short TTL (30–60 s) on the full result cache.** Bounds staleness to under a minute for the
  *ranked set*. Prices shown from a 60 s-old cache can be briefly stale — acceptable for
  display, but re-validated at add-to-cart (checkout is out of scope but this is the seam).
- **Tier by volatility.** The *set and order* of results changes slowly; *price/inventory*
  change fast. One pattern: cache the **ranked list of `listing_id`s** with a short TTL, then
  **hydrate price/stock** from a fast key-value view (or a `_mget` to ES) at serve time. This
  keeps the expensive ranked list cached while prices stay ~1 s fresh. It trades a small
  hydration cost for freshness — worth it for price-sensitive categories.
- **Event-driven invalidation.** The Indexing Service (Chapter 6) already consumes the CDC/Kafka
  change stream. Emit cache-invalidation events for hot listings/queries — e.g., a flash-sale
  price drop publishes an invalidation for affected query keys. Full precise invalidation of
  query-level caches is hard (which queries return listing L?), so in practice we rely on
  **short TTL as the backstop** and event invalidation only for known-hot, high-value changes.

> The senior framing: **"We accept bounded staleness. Search is AP (brief NFR6); the cache TTL
> is just a knob on *how* eventual the consistency is. We keep the ranked set cached and
> hydrate the volatile fields."**

### 7.7.4 Stampede protection and warming

Two operational hazards at 100 K QPS:

- **Cache stampede (thundering herd).** When a hot key (`"iphone"`) expires, thousands of
  concurrent requests miss simultaneously and all hammer ES with the identical expensive query.
  Mitigate with **request coalescing / single-flight**: the first miss acquires a short lock
  (or a Redis `SETNX` sentinel), executes the query, and populates the cache; concurrent misses
  wait briefly for the result instead of duplicating work. A softer variant is **probabilistic
  early expiration** — refresh a hot entry *before* its TTL with a probability that rises as it
  ages, so the herd never all expire at once.
- **Cold cache after deploy/failover.** A flushed Redis or a failed-over region starts at 0%
  hit rate and can overwhelm ES. **Warm** the cache by replaying the top-N head queries
  (derived from query logs, the same source feeding autocomplete) at startup, and roll deploys
  so the cache tier is never fully cold. This ties into Chapter 10's graceful-degradation and
  Chapter 9's multi-region caching tiers.

### 7.7.5 What is *not* cacheable

Be explicit about it: **fully personalized responses** (§7.8), **cursor-deep tails** (few
repeat hits, low value to cache), and **rare long-tail queries** (cache pollution — they evict
hot entries for a one-time hit). Cache admission should favor the head; an LFU/LRU-with-
admission policy or simply *only caching queries seen ≥N times in a window* keeps the working
set small and the hit rate high.

---

## 7.8 Personalization at Query Time (FR7) — Without Wrecking Cacheability

Personalization (rank results using *this buyer's* history, location, affinity) collides
head-on with caching: a fully personalized response is unique per user, so the Result Cache hit
rate collapses toward 0. The design goal is **most of the benefit of personalization with most
of the cacheability**.

Strategies, from most cacheable to least:

1. **Cohort / segment personalization (recommended default).** Bucket users into a small number
   of **segments** (e.g., locale × price-tier × top-affinity-category, or a clustering of
   behavior). Personalize per *segment*, not per *user*. The cache key gains one low-cardinality
   dimension (`segment_id`), so hit rate stays high (dozens of segments, not 500 M users).
   Covers the majority of the lift.
2. **Post-retrieval re-ranking of a cached candidate set.** Serve the cached
   *non-personalized* top-K from Redis, then apply a **cheap per-user reorder** in the Search
   Service (boost items in the user's affinity categories, demote already-purchased). The
   expensive retrieval is shared/cached; only a lightweight reorder is per-user. This is the
   sweet spot: cache the candidates, personalize the presentation.
3. **Feature injection into LTR.** Add user features (affinity vectors, recent categories) to
   the LTR feature vector (§7.4.3). Fully personalized ranking, but only cacheable at the
   segment level; reserve for logged-in high-value sessions.
4. **Personalized boosts as query params** — small `function_score` boosts for the user's
   preferred brands/categories, computed from a compact user profile fetched in parallel with
   (not before) the ES call so it's off the critical path.

```
Redis hit: cached NON-personalized top-K (shared across a segment)
        │
        ▼
Search Service: cheap per-user reorder
   for each candidate:
       score' = score × (1 + 0.15·affinity(user, category))
                       × (already_purchased ? 0.3 : 1.0)
        │
        ▼  (adds ~2–3 ms, no extra ES load)
   personalized order returned
```

> **Interview trap:** candidates propose per-user ML ranking and forget it destroys the cache.
> The strong answer names the tension explicitly and resolves it with **segment-level caching +
> post-cache re-ranking**, keeping personalization *off* the expensive shared path.

---

## 7.9 The Autocomplete Path (FR2)

Autocomplete is a **separate system**, not a mode of the search path — a deliberate
architectural split. The reasons:

- **Volume.** ~5–8 keystrokes per search means autocomplete peaks at **~500 K QPS** (brief §2) —
  5× the search peak. It cannot share the search cluster's budget.
- **Latency.** Its p99 target is **100 ms** (brief NFR1), and it fires on *every keystroke*, so
  it must feel instantaneous — tighter than search's 200 ms.
- **Different problem.** It matches **prefixes of short strings** (queries/product names), not
  full relevance over long documents. That's a different index and a different query.

### 7.9.1 Two implementation options

**(a) Completion Suggester (FST-based).** Elasticsearch's `completion` field type builds an
in-memory **Finite State Transducer** — essentially a prefix trie — optimized for
"complete-this-prefix." Sub-millisecond lookups, but the FST lives in heap and updates require
rebuilds, so it suits a **curated suggestion dictionary** (top queries, product/brand names)
rather than the full 10 B-doc corpus.

```json
PUT /suggestions/_mapping
{ "properties": {
    "suggest": { "type": "completion", "analyzer": "simple",
                 "contexts": [ { "name": "locale", "type": "category" } ] } } }

POST /suggestions/_search
{ "suggest": { "s": {
    "prefix": "iph",
    "completion": { "field": "suggest", "size": 10, "fuzzy": { "fuzziness": 1 },
                    "contexts": { "locale": ["en-US"] } } } } }
```

The **completion contexts** let one index serve per-locale suggestions (FR: multi-locale). Note
`fuzzy` for typo-tolerance even while typing (`"iph"` still works after `"ipf"`).

**(b) `edge_ngram` index.** Index each suggestion tokenized into prefixes at index time:
`"iphone"` → `["i","ip","iph","ipho","iphon","iphone"]` (edge_ngram, min 1 max ~20). At query
time, a simple `match` on the prefix (with a `search_analyzer` that does **not** re-ngram, per
§7.2.1) hits the inverted index directly. More flexible than the FST (supports full ranking,
mid-word matching), slightly slower, disk-backed rather than heap-bound.

| | Completion Suggester (FST) | edge_ngram |
|---|---|---|
| Speed | Fastest (in-memory FST) | Fast (inverted index) |
| Ranking | Weight field only | Full BM25 + function_score |
| Updates | Rebuild-ish, heap cost | Normal indexing |
| Best for | Curated top-query/name dictionary | Ranked suggestions with signals |

**GlobalMart choice:** a dedicated **suggestions index** (its own small cluster/tier), populated
from **query logs** (top queries by frequency × CTR) and top product/brand names, using
`edge_ngram` when we want to rank suggestions by popularity/conversion, backed by the
completion suggester for the ultra-hot curated head. Ranking suggestions by **popularity and
historical CTR** (not just prefix match) is what makes autocomplete feel smart — `"i"` →
`"iphone"` because that's what people click, even though `"ink"`, `"ice"` also match.

### 7.9.2 Why it's fast

The suggestions corpus is **millions of entries, not 10 B** — it fits in memory, on far fewer
nodes, with a trivially cacheable and heavily-hit head (`"i"`, `"ip"`, `"iph"` are the same for
everyone in a locale). Aggressive Redis/CDN caching of prefix→suggestions plus the FST's
in-heap lookups is what buys the 100 ms p99 at 500 K QPS.

---

## 7.10 Relevance Engineering — Measuring and Improving

You cannot improve what you don't measure. Ranking changes must be judged by **relevance
metrics and business outcomes**, never by eyeballing a few queries (NFR5).

### 7.10.1 Metrics

- **NDCG (Normalized Discounted Cumulative Gain)** — the workhorse offline metric. Rewards
  putting highly-relevant docs near the top; discounts gains logarithmically by position.
  Requires **graded relevance judgments** (0–4) per `(query, doc)`, from human raters or
  derived from clicks/conversions. Elasticsearch's **Ranking Evaluation API** (`_rank_eval`)
  computes NDCG/MAP/precision against a judgment set in CI.
- **MRR / Precision@k / Recall@k** — simpler, for navigational and zero-result analysis.
- **Online behavioral metrics** — the ones that pay salaries:
    - **CTR** (click-through rate) — did they click a result?
    - **Conversion rate** — did the search lead to a purchase? The north-star.
    - **Add-to-cart rate**, **revenue per search**, **null-result rate**, **reformulation rate**
      (a proxy for dissatisfaction — users re-typing means we failed).

> Offline NDCG and online conversion don't always agree. Offline is fast and safe for
> pre-filtering bad changes; **online A/B is the arbiter** for anything shipping.

### 7.10.2 A/B testing ranking changes

- **Bucket by user** (stable hash of user id → variant) so a user sees a consistent experience,
  and route a slice (e.g., 5%) to the new ranker. Compare conversion / revenue-per-search with
  significance testing; watch guardrail metrics (latency, null-rate) so you don't win
  conversion while wrecking p99.
- **Interleaving** (TeamDraft) is more sensitive than A/B for ranking: for one query, blend
  results from ranker A and B and see which side's results get clicked more. Needs far less
  traffic to reach significance — useful given ranking iterations are frequent.
- **Ship behind flags**, roll forward gradually, keep the previous ranker as instant rollback
  (ties to §7.4's function_score fallback).

### 7.10.3 Zero-result queries

Zero results is the worst outcome — a dead end and lost revenue. Handle at multiple layers:

1. **Prevent:** the §7.2 pipeline (spell-correct, synonyms, fuzzy fallback) is the first line —
   most zero-result queries are typos or vocabulary mismatch.
2. **Relax:** if the strict query returns 0, progressively **drop clauses** — remove the weakest
   `must` term, convert hard filters to soft boosts, widen fuzziness. "No exact match, showing
   related results."
3. **Fallback:** category-level or popular results for the detected intent, so the page is never
   empty.
4. **Mine:** log zero-result queries; they are a **direct backlog** for synonym additions,
   catalog gaps ("customers want X, we don't stock it"), and analyzer fixes. A weekly
   zero-result review is one of the highest-ROI relevance activities.

---

## Interview Tips

- **Lead with the pipeline diagram and the latency budget.** Saying "200 ms p99 is spent
  sequentially, so cache first and do expensive work on few documents" instantly signals
  seniority. Tie every design choice back to a line in the budget table.
- **Nail retrieve-then-rerank and be able to defend it with math.** "Running the LTR model over
  2 M matches is ~100 s/shard; over the top-200 it's milliseconds" is the single most convincing
  thing you can say about ranking. Interviewers who hear it stop probing.
- **Own the filter vs. query context distinction.** State that filters are non-scoring, cached
  as bitsets, and composable — so *every non-scoring constraint goes in `filter`*. It's a
  reliable "does this person actually know ES" checkpoint.
- **Name the caching-vs-freshness tension out loud, then resolve it.** Short TTL + event
  invalidation + "cache the ranked id list, hydrate price/stock" is the answer that separates
  senior from mid-level.
- **Name the personalization-vs-cacheability tension out loud, then resolve it** with
  segment-level caching + post-cache re-ranking. Proposing per-user ML ranking without noticing
  it destroys the cache is a classic trap.
- **Explain why autocomplete is a *separate* system** (500 K QPS, 100 ms, prefix-matching a
  small curated corpus). Candidates who try to serve typeahead from the main search path lose
  points.
- **Insist on measurement.** "I'd A/B test with conversion as the north-star and NDCG offline as
  a pre-filter, and treat zero-result queries as a backlog." Relevance without metrics is an
  opinion.
- **Common traps to avoid:** hand-tuning `^` boosts forever (that's LTR's job); putting
  `in_stock`/`price` in `must` instead of `filter`; running `script_score` or aggregations over
  the full match set instead of the rescore window; forgetting query-time analysis must match
  index-time analysis.

## Key Takeaways

- The search pipeline is a **sequential budget**: Edge 5 · API 10 · Query Understanding 15 · ES
  120 · rerank 30 · serialize 10 · buffer 10 = **200 ms p99**. Cache in front of it; do
  expensive work last and on the fewest documents.
- **Query Understanding (FR6)** turns a hostile raw string into a structured query: normalize →
  spell-correct → synonym-expand (query-time `synonym_graph`) → attribute-extract → intent/
  category detect, all locale-aware. `"iphn 128gb"` → `iphone (OR "apple phone")` + soft
  `storage:128GB` + Smartphones category + `spell_corrected` metadata.
- **Retrieval optimizes recall cheaply** with a `bool` query: hard constraints in `filter`
  (non-scoring, bitset-cached, composable), relevance in `must`/`should` via `multi_match` with
  field boosts (`title^3, brand^2, description^1`).
- **Two-phase ranking** is non-negotiable at scale: cheap **BM25** first pass over the whole
  match set, expensive **LTR / function_score** rerank over only the top-K (`rescore
  window_size ≈ 200`). Justified purely by latency math.
- **Business signals** (popularity, seller_rating, price competitiveness, availability,
  freshness) blend **multiplicatively** with relevance (`boost_mode: multiply`) so relevance
  stays primary; out-of-stock is **demoted, not dropped** (unless the buyer filters for it).
- **Faceting** via `terms`/`range` aggregations is often costlier than the query (it scans all
  matching docs' doc_values); mitigate with filter-context reuse, small `size`, global
  ordinals, sampling, and request caching.
- **Two caches:** the **Redis Result Cache** (whole response, short TTL ~30–60 s, keyed on
  normalized q+filters+sort+page+locale) serves the head-query majority; the **ES request/query
  caches** handle shard-local aggs and filter bitsets. Resolve the **TTL-vs-NRT** tension with
  short TTL + event invalidation + hydrate-volatile-fields.
- **Personalization (FR7)** preserves cacheability via **segment-level caching** and
  **post-cache per-user re-ranking**, keeping per-user work off the expensive shared path.
- **Autocomplete (FR2)** is a **separate ultra-fast path** (~100 ms p99, ~500 K QPS): a small
  suggestions index built from query logs, served by the FST completion suggester / `edge_ngram`,
  ranked by popularity and CTR.
- **Relevance is engineered, not guessed:** NDCG offline (`_rank_eval`) as a pre-filter,
  **conversion/CTR via A/B or interleaving** as the arbiter, and **zero-result queries** treated
  as a prevention-relaxation-fallback problem and a standing backlog.

---

*Next: **Chapter 8 — Elasticsearch Internals**, where we open the box under this pipeline —
Lucene segments, the inverted index, BM25 scoring internals, refresh/merge, and how
scatter-gather actually executes across our ~600 shards.*
