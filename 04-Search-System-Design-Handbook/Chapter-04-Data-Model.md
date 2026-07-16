# Chapter 4 — Data Model

In Chapter 3 we fixed the API contracts — what a caller sends and what comes back. This chapter answers the question that sits directly underneath those contracts: *what does a single searchable thing look like on disk, and where does the truth about it actually live?* Get this wrong and every later chapter suffers. A sloppy mapping means you cannot facet efficiently, cannot sort without blowing up heap, cannot reindex without downtime, and cannot reason about consistency. Get it right and the indexing pipeline (Ch6), the search pipeline (Ch7), and scaling (Ch9) all fall out naturally.

We are modeling the **product-listing document** — one product offered by one seller in one locale — the unit fixed in brief §0. Because the same SKU is listed by many sellers across many markets, we have ~1.5 B distinct products fanning out to **~10 B listings**, ~20 TB of raw corpus. Every modeling decision below is made at that scale, not at toy scale.

## 4.1 Source of Truth vs. the Derived Index

The single most important principle in this entire design: **Elasticsearch is never the system of record.** The source of truth is the **Catalog Service's transactional store** — a sharded SQL cluster or a NoSQL document store that owns products, sellers, prices, and inventory with ACID guarantees, referential integrity, and an audit trail. Elasticsearch holds a **derived, rebuildable projection** of that data, optimized purely for retrieval.

Why draw the line so hard?

| Concern | Catalog store (truth) | Elasticsearch (derived) |
|---|---|---|
| Consistency | Strong, transactional | Eventual (AP on search path, brief NFR6) |
| Durability model | Authoritative | Reconstructable from source |
| Write pattern | Row/document mutations, FK integrity | Bulk, append-heavy segments |
| What breaks if lost | Business stops | Rebuild from truth; degraded, not dead |
| Query shape | Point lookups, transactions | Full-text, faceting, ranking, scatter-gather |

Elasticsearch is a Lucene-backed search engine, not a database. It has no cross-document transactions, no foreign keys, no join engine worth the name, and — critically — a *versioned, segment-merging storage model* where the on-disk representation is expected to be thrown away and rebuilt (the `products-vN` reindex pattern from brief §6). Treating it as the record of a $799 price would be a correctness bug waiting to happen: a dropped Kafka message, a mapping migration, or a cluster restore-from-snapshot could silently roll a price backwards.

The operational payoff of this discipline:

- **You can always rebuild.** If the index is corrupted, a mapping is wrong, or you need a new analyzer, you reindex from Catalog into `products-v(N+1)` and flip the `products` alias (brief §6). No data is at risk because none of it *originated* in ES.
- **Freshness is a pipeline property, not a storage property.** The path is `Catalog → CDC → Kafka → Indexing Service → ES bulk` (brief §5). ES lag (target ≤1 s for price/inventory, brief FR8) is something you *tune*, not something you *trust for money*.
- **Writes never go to ES directly from product code.** The public API has no synchronous "write to ES" call; indexing is CDC-driven (brief §3). This keeps ES a pure read replica of intent.

Say this out loud in an interview: *"ES is a rebuildable derived index; the Catalog transactional store is the source of truth."* It signals you understand the difference between a search engine and a database, and it justifies half the decisions in Ch6 and Ch10.

## 4.2 The Product-Listing Document, Field by Field

Here is the canonical document from brief §4, annotated by the *role* each field plays. A field can serve more than one role, and that is exactly what drives the mapping choices in §4.3. The five roles:

- **Search** — participates in full-text matching (analyzed).
- **Filter** — narrows the result set with an exact predicate (`term`/`range`).
- **Facet** — powers aggregation counts shown in the UI (needs `doc_values`).
- **Sort** — orders results (needs `doc_values`).
- **Display** — returned to the client but never queried.
- **Ranking signal** — feeds relevance scoring / the LTR reranker (Ch7).

| Field | Type | Roles | Notes |
|---|---|---|---|
| `listing_id` | keyword | filter, display, dedupe/version key | Primary key = product × seller × locale; also the ES `_id`. |
| `product_id` | keyword | filter, facet (grouping) | Shared across sellers; used to collapse duplicate offers. |
| `title` | text + `title.raw` keyword | search, sort, display | Multi-field: analyzed for search, `.raw` for exact sort/aggs. |
| `description` | text | search, display | Analyzed only; long free text, never sorted or faceted. |
| `brand` | keyword + `brand.text` | filter, facet, search | Exact for facets; analyzed sub-field for "sony" matching "Sony". |
| `category_path` | keyword array | filter, facet | Hierarchy `["Electronics","Phones","Smartphones"]`. |
| `attributes` | object of keywords (see §4.4) | filter, facet | color/size/storage; open-ended per category. |
| `price.amount` | scaled_float / double | filter (range), sort, ranking | Money; filterable and sortable (§4.8). |
| `price.currency` | keyword | filter, display | Drives currency-aware handling (§4.8). |
| `seller_id` | keyword | filter, facet | Which seller offers this listing. |
| `seller_rating` | float | ranking, filter | Denormalized from Seller service (§4.7). |
| `in_stock` | boolean | filter | Cheap availability gate. |
| `inventory` | integer | filter, display | Exact count; changes constantly (hot mutation). |
| `rating` | float | sort, filter, ranking | Product star rating. |
| `review_count` | integer | sort, ranking, display | Confidence weight for `rating`. |
| `popularity` | float (0–1) | ranking, sort | Normalized sales/CTR signal. |
| `locale` | keyword | filter (routing) | e.g. `en-US`; selects analyzer & language. |
| `market` | keyword | filter (routing) | e.g. `US`; often the shard-routing key. |
| `image_url` | keyword, `index:false` | display only | Stored, never searched. |
| `created_at` / `updated_at` | date | sort, filter (range) | "Newest" sort (FR4), freshness signals. |
| `_version` | long (external version) | idempotency | Monotonic version for NRT updates (§4.7, Ch6). |

Two things to internalize. First, **most fields carry multiple roles**, and the mapping must satisfy all of them simultaneously — that is the entire reason multi-fields exist. Second, some fields are *display-only* (`image_url`); marking them `index: false` saves inverted-index space across 10 B docs, which is real money at 20 TB scale.

## 4.3 Mappings: `text` vs `keyword`, and Multi-Fields

This is the decision that trips up most candidates, so let us be precise.

- **`text`** — the value is run through an **analyzer** (tokenized, lowercased, stemmed, etc., §4.5) and stored in the inverted index. You get full-text matching: `match` queries, phrase queries, relevance scoring. You **cannot** sort or aggregate on it efficiently (that requires `fielddata`, which is dangerous — §4.6). A `text` field is not stored in a way that lets you ask "give me the exact string back for sorting."
- **`keyword`** — the value is stored **verbatim**, un-analyzed, as a single token. You get exact-match `term` queries, `range`, sorting, and aggregations (facets) via `doc_values`. You **cannot** do partial/full-text matching — `keyword: "Apple iPhone 15"` matches only the whole string `"Apple iPhone 15"`, not `iphone`.

The tension: `title` needs *both*. A buyer typing "iphone" must match "Apple iPhone 15 Pro" (that is `text`), but the UI may want to sort alphabetically or show a "top titles" aggregation (that is `keyword`). The answer is a **multi-field**: index the same source value two ways under one field name.

```jsonc
"title": {
  "type": "text",
  "analyzer": "en_text",          // analyzed for search
  "fields": {
    "raw": { "type": "keyword", "ignore_above": 256 },  // exact: sort/agg
    "suggest": {                    // autocomplete (§4.5, Ch7)
      "type": "text",
      "analyzer": "edge_ngram_analyzer",
      "search_analyzer": "standard"
    }
  }
}
```

Now `title` is full-text searchable, `title.raw` is sortable/aggregatable, and `title.suggest` powers prefix autocomplete — from **one** source field, indexed three ways. `ignore_above: 256` on the keyword protects against a pathological 40 KB "title" bloating doc_values.

Here is a **full products index mapping** consolidating the schema. This is the artifact an interviewer wants to see you able to sketch.

```jsonc
PUT /products-v1
{
  "settings": {
    "number_of_shards": 600,        // brief §2/§6: ~600 primaries
    "number_of_replicas": 1,
    "refresh_interval": "1s",       // NRT on hot tier (Ch8)
    "analysis": { /* analyzers defined in §4.5 */ }
  },
  "mappings": {
    "dynamic": "strict",            // reject unexpected fields; catch bugs early
    "properties": {
      "listing_id":  { "type": "keyword" },
      "product_id":  { "type": "keyword" },
      "title": {
        "type": "text", "analyzer": "en_text",
        "fields": {
          "raw":     { "type": "keyword", "ignore_above": 256 },
          "suggest": { "type": "text", "analyzer": "edge_ngram_analyzer",
                       "search_analyzer": "standard" }
        }
      },
      "description": { "type": "text", "analyzer": "en_text" },
      "brand": {
        "type": "keyword",
        "fields": { "text": { "type": "text", "analyzer": "en_text" } }
      },
      "category_path": { "type": "keyword" },   // array of hierarchy tokens
      "attributes": {
        "type": "object",
        "properties": {
          "color":   { "type": "keyword" },
          "storage": { "type": "keyword" },
          "size":    { "type": "keyword" }
        }
      },
      "price": {
        "properties": {
          "amount":   { "type": "scaled_float", "scaling_factor": 100 },
          "currency": { "type": "keyword" }
        }
      },
      "seller_id":     { "type": "keyword" },
      "seller_rating": { "type": "float" },
      "in_stock":      { "type": "boolean" },
      "inventory":     { "type": "integer" },
      "rating":        { "type": "float" },
      "review_count":  { "type": "integer" },
      "popularity":    { "type": "float" },
      "locale":        { "type": "keyword" },
      "market":        { "type": "keyword" },
      "image_url":     { "type": "keyword", "index": false },
      "created_at":    { "type": "date" },
      "updated_at":    { "type": "date" },
      "_ver":          { "type": "long" }
    }
  }
}
```

Note `dynamic: "strict"`. At 10 B docs and 50 M new listings/day, silent field explosion (a "mapping explosion") from an upstream bug can bloat the cluster state and OOM masters. Being strict forces schema changes to be deliberate — an intentional reindex into `products-vN+1`.

## 4.4 Attributes: Object vs Nested vs Flattened

`attributes` is the hardest sub-problem because it is *open-ended*: phones have `storage`, shirts have `sleeve_length`, and you cannot pre-declare every key. Three ES modeling options, each with a sharp trade-off.

**1. `object` (default).** ES flattens `{ "color":"black", "storage":"128GB" }` into dotted paths `attributes.color`, `attributes.storage`. Cheap, fast, `doc_values`-friendly for faceting. This is the right default when attributes are simple key→single-value pairs and you never need to preserve the correlation *between* keys within one nested entry. Its danger appears only with **arrays of objects**.

**2. `nested`.** Needed when an attribute is a *list of objects* and you must keep each object's fields correlated. The classic pitfall: model variants as

```jsonc
"variants": [ { "color":"red", "size":"L" }, { "color":"blue", "size":"S" } ]
```

as a plain `object`, and ES flattens it to `color: [red, blue]`, `size: [L, S]` — losing which color went with which size. A query for `color=red AND size=S` would **wrongly match** because both values exist *somewhere* in the doc. `nested` fixes this by indexing each object as a hidden sub-document, so `nested` queries match field combinations within a *single* entry.

The **nested-query pitfall** you must name in an interview: nested docs are separate Lucene documents, so they cost extra storage, require special `nested` queries and `nested` aggregations, and — the killer at our scale — a single parent with N nested entries becomes N+1 documents. For 10 B listings with many variants, indiscriminate `nested` multiplies your doc count and merge cost. **Use `nested` only when correlation genuinely matters**; for most GlobalMart facets (color, storage, brand), a flat keyword is correct and far cheaper.

**3. `flattened`.** The entire `attributes` sub-object is indexed as a single field of keyword-ish values, with no mapping per key. Perfect for sparse, high-cardinality, unpredictable attribute bags because it *cannot* cause mapping explosion. The cost: everything is a keyword — no per-key analysis, no numeric ranges, limited aggregation nuance. A pragmatic hybrid: promote the ~20 high-value, faceted attributes (color, size, storage) to explicit typed `keyword` fields, and dump the long tail into a `flattened` field.

**Why `category_path` is a keyword array, not nested.** Faceting on a hierarchy just needs each level as an exact, aggregatable token:

```jsonc
"category_path": ["Electronics", "Electronics/Phones", "Electronics/Phones/Smartphones"]
```

Storing the *cumulative* paths (not just leaf tokens) lets one `terms` aggregation produce counts at every level, and a single `term` filter (`category_path: "Electronics/Phones"`) selects a whole subtree. There is no correlation between siblings to preserve, so `nested` would add cost for nothing. Keyword array = correct, `doc_values`-backed, cheap faceting.

## 4.5 Analyzers and the Analysis Chain

Analysis is what turns `text` into searchable tokens, and it runs at **two** times: **index time** (building the inverted index) and **query time** (tokenizing the query). They must agree, or matches silently fail. An analyzer is a three-stage pipeline:

```
raw text ─▶ [char filters] ─▶ [tokenizer] ─▶ [token filters] ─▶ terms
            strip HTML,        split into      lowercase, stem,
            map & → and        tokens          stopwords, synonyms
```

- **Char filters** — pre-tokenization string surgery: strip HTML from seller descriptions, normalize `"&"→"and"`, fold `"™"`.
- **Tokenizer** — splits the stream. `standard` (Unicode word boundaries) is the workhorse; CJK languages need `icu_tokenizer` or a language-specific tokenizer because Chinese/Japanese have no spaces.
- **Token filters** — transform the token stream: `lowercase` (so "iPhone" matches "iphone"), `stemmer` (so "running" matches "run"), `stop` (drop "the", "a"), and `synonym`.

**Standard vs language-specific.** The `standard` analyzer is language-agnostic: it tokenizes and lowercases but does **not** stem. For a single-language corpus that is a relevance miss — an English buyer expects "shoes" to match "shoe". So we define per-language analyzers:

```jsonc
"analysis": {
  "filter": {
    "en_stop":     { "type": "stop", "stopwords": "_english_" },
    "en_stemmer":  { "type": "stemmer", "language": "english" },
    "syn_ecom":    { "type": "synonym_graph",
                     "synonyms": ["trainers, sneakers", "tv, television"] }
  },
  "analyzer": {
    "en_text": {
      "char_filter": ["html_strip"],
      "tokenizer": "standard",
      "filter": ["lowercase", "syn_ecom", "en_stop", "en_stemmer"]
    },
    "edge_ngram_analyzer": {
      "tokenizer": "standard",
      "filter": ["lowercase", "edge_ngram_3_20"]
    }
  }
}
```

**Synonyms** (brief FR6: "trainers ↔ sneakers") are best applied at **query time** via `synonym_graph`, not baked into the index — otherwise every synonym change forces a reindex of 10 B docs. Put the synonym filter in the *search* analyzer, keep the index analyzer synonym-free. (More in Ch7.)

**Autocomplete: `edge_ngram` vs the completion suggester.** Two approaches, deliberately different, and we forward-reference them:

| | `edge_ngram` field | `completion` suggester |
|---|---|---|
| Mechanism | Index prefixes as tokens ("iph","ipho",…) | In-memory FST built at index time |
| Query | Normal `match` on `title.suggest` | Dedicated `_search` suggest / `completion` |
| Flexibility | Full query DSL, filters, ranking | Prefix only, very limited filtering |
| Speed | Fast (inverted index) | Fastest (RAM FST), sub-ms |
| Cost | Larger index (many prefix tokens) | Heap for the FST |

For GlobalMart's typeahead peak of ~**500 K QPS** (brief §2) we lean on the completion suggester's FST speed for the pure prefix path, and use `edge_ngram` where we need to blend suggestions with filters and business ranking. We design this properly in **Ch7**.

## 4.6 `doc_values`: Sorting, Aggregations, Faceting — and Why `fielddata` Is Dangerous

Full-text search wants an **inverted index** (term → list of docs). But sorting, faceting, and aggregations want the **opposite**: given a doc, what is its value for this field? That access pattern is columnar, and ES provides it via **`doc_values`** — an on-disk, column-oriented store built at index time, memory-mapped and OS-page-cached at query time.

`doc_values` are **on by default for every non-`text` field** (`keyword`, numerics, `date`, `boolean`). This is precisely why `price.amount`, `rating`, `popularity`, `created_at` (sorts, brief FR4) and `brand`, `category_path`, `attributes.*` (facets, FR3) work efficiently and off-heap — the OS page cache absorbs them, so they scale to 10 B docs without exploding the JVM heap.

Now the trap. **`text` fields have `doc_values` disabled** — they cannot, because their content is analyzed into many tokens, not one sortable value. If you *try* to sort or aggregate a `text` field, ES will refuse unless you enable **`fielddata: true`**, which builds the doc-values-equivalent structure **on the JVM heap, at query time, for the entire field, across every matching segment.** At our scale this is a guaranteed circuit-breaker trip or an OOM'd data node. **Never enable `fielddata` on a high-cardinality `text` field.** The correct fix is always the multi-field: sort/aggregate on `title.raw` (a `keyword`, doc_values-backed), never on `title`.

Rule of thumb to state in an interview: *"Search on `text`, sort and facet on `keyword`/numeric via doc_values. If I ever reach for `fielddata`, I've modeled the field wrong."*

## 4.7 Denormalization: Embed or Join?

Elasticsearch has no real join. So signals that live in *other* services — `seller_rating` (Seller service), `popularity` (analytics), `rating`/`review_count` (Reviews service) — must be **denormalized** (embedded) into the listing document at index time by the Indexing Service (Ch6). We accept redundancy to make the read path a single-document lookup with no join.

The trade-off is **update amplification**. If a seller with 200,000 active listings changes their rating from 4.6 to 4.7, that one logical fact must be written to **200,000 documents**. Contrast the alternatives:

| Strategy | Read cost | Write/update cost | Verdict |
|---|---|---|---|
| Embed (denormalize) | 1 doc, no join — fast | Fan-out: N docs per source change | **Chosen** — reads are the SLA-critical path (brief §1) |
| `join`/parent-child | Slow (join at query) | Cheap (1 update) | Rejected — kills the 200 ms p99 budget |
| App-side join (fetch from Catalog per hit) | Extra network hop per result | Cheap writes | Rejected — latency + coupling |

We embed, because reads dominate the SLA and must stay a single scatter-gather (brief §2 latency budget: ES query 120 ms of the 200 ms). We *manage* the amplification, we don't avoid it:

- **Tier signals by volatility.** `title`/`brand` change rarely; `seller_rating`/`popularity` change slowly (batch-refreshable); `price`/`inventory` change constantly (~100 K writes/sec, brief §2). Route each through the right pipeline cadence — high-churn fields get partial updates, slow signals get periodic bulk refreshes (Ch6/Ch11).
- **Partial updates + external versioning.** Use `_version` (brief §4) with external versioning so an out-of-order price update from Kafka cannot overwrite a newer one — the older version is simply rejected. This is what makes NRT updates idempotent (brief §3).
- **Bound the fan-out.** A seller-rating change becomes a bulk job keyed by `seller_id`, executed off-peak, at a controlled rate so it never starves buyer-facing indexing.

This is a genuine engineering trade-off, not a free lunch — call it out explicitly. We revisit the update-amplification mechanics in **Ch6** and weigh the full decision in **Ch11**.

## 4.8 Multi-Locale and Multi-Currency Modeling

The listing unit is *product × seller × locale*, so locale is baked into identity, not bolted on. Two dimensions to model: **language** (affects analysis/relevance) and **market/currency** (affects filtering/sorting/display).

**Language / per-locale analyzers.** "iPhone" tokenizes fine with `standard`, but German compound words, French accents, and CJK word segmentation each need their own analysis chain. We do **not** cram all languages into one field. Two viable patterns:

1. **Per-language sub-fields** on the analyzed field — `title` (default), `title.de` (German analyzer), `title.fr` — and query the sub-field matching the request `locale`. Simple, but every doc carries fields for languages it doesn't use.
2. **Per-locale indices** behind the `products` alias — `products-de-v1`, `products-us-v1` — each with locale-appropriate analyzers, routed by the `locale`/`market` filter. This is the cleaner fit at 10 B docs: it isolates analysis, lets the **data-warm tier** hold low-traffic locales (brief §6), and aligns with multi-region locality (Ch9). GlobalMart uses per-locale/market indices as the primary partitioning, with per-language analyzers inside.

Because `locale` and `market` are indexed keywords, they double as **routing keys** — a US buyer's query is filtered (and physically routed, Ch9) to US-market shards, cutting scatter-gather fan-out and latency.

**Currency.** Price must stay **filterable and sortable** (FR3/FR4), so it is a numeric, not a formatted string. We store `price.amount` as **`scaled_float`** (`scaling_factor: 100`) — it behaves like a double for `range` and `sort` but is stored compactly as a scaled long, ideal for money where two decimal places suffice. `price.currency` is a keyword filter/display field.

The subtlety: **you cannot correctly range-filter or sort across mixed currencies.** "$50–$100" is meaningless if some listings are in EUR and some in JPY. Two options:

- **Query within a single currency** — because each listing is locale-scoped and each market has a canonical currency, "price 50–100 USD" filters only USD listings. Simplest, and the default.
- **Store a normalized comparison price** — add `price.amount_usd`, a periodically FX-converted value, so cross-market sorting/filtering uses one common numeraire. The FX rate is a *derived signal from the source of truth* (§4.1), refreshed on a schedule — never treat the converted value as authoritative money.

GlobalMart keeps the display/transaction price in native currency (`price.amount` + `price.currency`) and adds a `price.amount_usd` normalized field for any cross-locale ranking or comparison — the same denormalization discipline as §4.7.

## Interview Tips

- **Lead with the source-of-truth statement.** Before any mapping talk, say: *"Catalog's transactional store is the system of record; ES is a rebuildable derived index."* It frames every later trade-off and shows senior judgment.
- **`text` vs `keyword` is a guaranteed question.** Answer with the multi-field pattern (`title` + `title.raw`) and *why*: search on `text`, sort/facet on `keyword`. If you can sketch the mapping JSON from memory, you're ahead of most candidates.
- **Name the `fielddata` trap unprompted.** "I'd never sort/aggregate on a `text` field — that forces `fielddata` onto the heap and OOMs the node at 10 B docs. I sort on the `.raw` keyword via doc_values." This one sentence signals real ES operational experience.
- **Know the nested pitfall cold.** Arrays of objects flatten and cross-match (`red`+`S` matching the wrong variant); `nested` fixes correctness but multiplies doc count and merge cost. Use it *only* when correlation matters — most facets don't need it.
- **Frame denormalization as a trade, not a default.** "I embed `seller_rating` for single-doc reads because reads are the SLA-critical path; the cost is update amplification — a seller change fans out to N listings, which I manage with versioning and off-peak bulk jobs." Mention Ch6/Ch11.
- **Common traps to avoid saying:** "just store price as a string" (kills range/sort); "put everything in one field" (kills per-language analysis); "let mapping be dynamic" (mapping explosion at scale — use `dynamic: strict` + reindex).
- **Always connect a field to a role.** When you list a field, say whether it's for search, filter, facet, sort, display, or ranking — interviewers want to see you think in access patterns, not attributes.

## Key Takeaways

- **ES is a derived, rebuildable index; the Catalog transactional store is the source of truth.** Never treat ES as the record for price, inventory, or anything else. Rebuild via `products-vN` reindex + alias flip.
- **Every field has a role** (search / filter / facet / sort / display / ranking). Multi-role fields drive the multi-field mapping pattern.
- **`text` = analyzed, searchable, not sortable; `keyword` = verbatim, exact, sortable/aggregatable.** Use multi-fields (`title` + `title.raw` + `title.suggest`) to get all behaviors from one source value.
- **`doc_values` (columnar, off-heap) power sorting, faceting, and aggregations** and are on by default for non-text fields. **`fielddata` on `text` is dangerous** — it's on-heap and will OOM at scale; sort/facet on a keyword instead.
- **Attributes:** default to flat `keyword` fields; use `nested` only when field correlation within an object matters (and accept the doc-count/merge cost); use `flattened` for the sparse long tail to avoid mapping explosion. `category_path` is a keyword array of cumulative paths for cheap hierarchical faceting.
- **Analysis is a char-filter → tokenizer → token-filter chain** run at index *and* query time. Use per-language analyzers (stemming, stopwords), apply synonyms at query time to avoid reindexing, and choose `edge_ngram` vs the completion suggester per autocomplete need (Ch7).
- **Denormalize business signals into the listing doc** for single-doc reads, accepting update amplification managed by volatility tiering, external `_version` idempotency, and rate-controlled bulk fan-out (Ch6/Ch11).
- **Model locale into identity:** per-locale/market indices with per-language analyzers; store money as numeric `scaled_float` with an explicit currency, and add a normalized `amount_usd` for cross-market comparison — never range-filter across mixed currencies.
