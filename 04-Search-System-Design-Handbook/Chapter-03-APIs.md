# Chapter 3 — APIs

The API is the contract. Everything upstream — mobile apps, the web storefront, partner
integrations, internal ranking experiments — depends on it, and everything downstream — the
Search Service, Elasticsearch, the indexing pipeline — is hidden behind it. In a hyperscale
system this contract is the single most-touched surface in the company: at **~35 K average and
~100 K peak search QPS** (Ch2), plus a typeahead path pushing **~500 K QPS**, an API decision
that costs 2 ms or leaks an implementation detail is multiplied billions of times a day. So we
design the API surface deliberately, before we design the engine behind it.

This chapter specifies the public read APIs (`GET /v1/search`, `GET /v1/autocomplete`), the
internal indexing APIs, and the cross-cutting concerns — auth, versioning, pagination, rate
limiting, and errors — that make the contract usable and stable. We stay faithful to the
canonical API surface in brief §3.

---

## 3.1 API design principles for a search system

Before any endpoint, agree on the principles. In an interview, stating these first signals
seniority; they also justify every concrete choice later.

**1. Client-friendly, not engine-shaped.** The API must not leak that Elasticsearch sits
behind it. A buyer sends `sort=price_asc`, not an ES `sort` clause; they send `filters=brand:Apple`,
not a `bool.filter.term` fragment. This decoupling lets us re-tune the query DSL, swap analyzers,
or even migrate engines without breaking a single client. **Never expose the ES query DSL to the
public.** It is both a security hole (query-of-death, resource exhaustion) and a permanent
coupling you can never undo.

**2. Stable contracts + explicit versioning.** URLs are versioned (`/v1`). Within a version we
only make **additive, backward-compatible** changes: add a new optional query param, add a new
field to the response. We never remove a field, rename it, change its type, or change a default
within a major version. A client written against `/v1` today must keep working unchanged for
years. Breaking changes ⇒ `/v2`, run in parallel, with a deprecation window (see §3.8).

**3. Sensible defaults; every param optional except `q`.** A brand-new client should get a
useful result from `GET /v1/search?q=iphone` alone. Defaults: `sort=relevance`, `page=1`,
`size=24`, `locale`/`market` inferred from the gateway (Accept-Language, geo) if omitted.

**4. Predictable, typed, paginated responses.** Same shape every time. Lists are always arrays
(never "object if one, array if many"). Every list response carries pagination metadata.
Money is always `{amount, currency}`, never a bare float, so currency is never ambiguous.

**5. Read/write separation.** The buyer-facing read API and the internal indexing API are
different surfaces with different auth, different SLAs, and different scaling profiles. We keep
them physically and contractually separate.

**6. Observability baked in.** Every response echoes a `request_id` and query metadata (what we
corrected, which synonyms fired). This is not decoration — it is how support, relevance
engineers, and A/B analysis debug "why did I get these results."

---

## 3.2 `GET /v1/search` — the core endpoint

This is the revenue path. Its p99 budget is **200 ms** end-to-end (Ch2 latency budget). The
contract must be rich enough to power a full search-results page — results, facets, pagination,
and "did you mean" — in a **single round trip**. A results page that needs three API calls to
render is a design failure.

### Query parameters

| Param | Type | Required | Default | Notes |
|---|---|:--:|---|---|
| `q` | string | yes | — | Free-text query. Empty `q` + filters ⇒ a browse/category listing. |
| `filters` | string, **repeatable** | no | none | `field:value`. Repeat for multi-select/AND. See §3.4. |
| `sort` | enum | no | `relevance` | `relevance`, `price_asc`, `price_desc`, `rating`, `newest`, `popularity`. |
| `page` | int | no | `1` | 1-based. Offset paging; valid only up to page ~50 (§3.5). |
| `size` | int | no | `24` | Results per page, **≤ 100** (hard cap). |
| `cursor` | string | no | — | Opaque `search_after` token for deep paging. Mutually exclusive with `page`. |
| `locale` | string (BCP-47) | no | inferred | e.g. `en-US`. Drives analyzer, synonyms, display language. |
| `market` | string | no | inferred | e.g. `US`. Selects catalog partition, currency, shipping rules. |
| `facets` | string, repeatable | no | default set | Which facets to compute (e.g. `brand`, `price`). Fewer facets = cheaper query. |

Design notes worth saying out loud:

- **`locale` vs `market` are distinct.** `locale=en-US` in `market=CA` means "English-speaking
  buyer shopping the Canadian market" — English text analysis, but CAD prices and Canadian
  inventory/shipping. Conflating them is a classic bug.
- **`size ≤ 100` is a hard cap**, enforced at the gateway. It bounds fan-out cost and blast
  radius. A request for `size=100000` gets `400`, not an OOM.
- **`filters` is repeatable, not comma-joined.** `?filters=brand:Apple&filters=brand:Samsung`
  is unambiguous; `filters=brand:Apple,Samsung` forces us to invent escaping rules for values
  containing commas. Repetition is the HTTP-native multi-value idiom.

### Example request

```http
GET /v1/search?q=wireles%20hedphones&filters=brand:Sony&filters=brand:Bose
   &filters=price:50-200&filters=rating:gte:4&sort=relevance&page=1&size=3
   &locale=en-US&market=US
Host: api.globalmart.com
Authorization: Bearer <buyer-JWT>
Accept: application/json
```

Note the deliberately misspelled `q` (`wireles hedphones`) to exercise spell correction and the
mixed filter types (multi-select `brand`, range `price:50-200`, threshold `rating:gte:4`).

### Example response (`200 OK`)

```json
{
  "request_id": "req_9f3c2a7b",
  "query": {
    "original": "wireles hedphones",
    "corrected": "wireless headphones",
    "spelling_corrected": true,
    "synonyms_applied": [
      { "term": "headphones", "expanded_to": ["earphones", "earbuds"] }
    ],
    "locale": "en-US",
    "market": "US",
    "sort": "relevance"
  },
  "pagination": {
    "page": 1,
    "size": 3,
    "total_results": 4187,
    "total_pages": 1396,
    "next_cursor": "eyJzYSI6Wzc5OS4wLCJMLTk0MjExMSJdfQ==",
    "has_more": true
  },
  "results": [
    {
      "listing_id": "L-771043",
      "product_id": "P-55021",
      "title": "Sony WH-1000XM5 Wireless Noise-Cancelling Headphones",
      "brand": "Sony",
      "category_path": ["Electronics", "Audio", "Headphones"],
      "price": { "amount": 349.00, "currency": "USD" },
      "rating": 4.7,
      "review_count": 21843,
      "in_stock": true,
      "seller_id": "S-1201",
      "seller_rating": 4.8,
      "image_url": "https://img.globalmart.com/L-771043.jpg",
      "score": 18.42,
      "highlight": {
        "title": ["Sony WH-1000XM5 <em>Wireless</em> Noise-Cancelling <em>Headphones</em>"]
      }
    },
    {
      "listing_id": "L-889210",
      "product_id": "P-55021",
      "title": "Sony WH-1000XM5 Wireless Headphones (Refurbished)",
      "brand": "Sony",
      "category_path": ["Electronics", "Audio", "Headphones"],
      "price": { "amount": 279.00, "currency": "USD" },
      "rating": 4.5,
      "review_count": 512,
      "in_stock": true,
      "seller_id": "S-4477",
      "seller_rating": 4.4,
      "image_url": "https://img.globalmart.com/L-889210.jpg",
      "score": 15.10
    },
    {
      "listing_id": "L-640318",
      "product_id": "P-71144",
      "title": "Bose QuietComfort Ultra Wireless Headphones",
      "brand": "Bose",
      "category_path": ["Electronics", "Audio", "Headphones"],
      "price": { "amount": 429.00, "currency": "USD" },
      "rating": 4.6,
      "review_count": 8901,
      "in_stock": true,
      "seller_id": "S-1201",
      "seller_rating": 4.8,
      "image_url": "https://img.globalmart.com/L-640318.jpg",
      "score": 14.88
    }
  ],
  "facets": [
    {
      "field": "brand",
      "type": "terms",
      "values": [
        { "value": "Sony", "count": 1204, "selected": true },
        { "value": "Bose", "count": 803, "selected": true },
        { "value": "Sennheiser", "count": 611, "selected": false },
        { "value": "JBL", "count": 559, "selected": false }
      ]
    },
    {
      "field": "price",
      "type": "range",
      "currency": "USD",
      "values": [
        { "from": 0,   "to": 50,   "count": 214 },
        { "from": 50,  "to": 200,  "count": 1876, "selected": true },
        { "from": 200, "to": 500,  "count": 1502 },
        { "from": 500, "to": null, "count": 595 }
      ]
    },
    {
      "field": "rating",
      "type": "terms",
      "values": [
        { "value": "4", "count": 3120, "selected": true },
        { "value": "3", "count": 780 }
      ]
    }
  ]
}
```

Everything the SRP needs is here in one payload: the corrected query (so the UI can show "Showing
results for **wireless headphones** — search instead for *wireles hedphones*"), the ranked
results with a debug `score` and highlight fragments, facets with live counts and a `selected`
flag reflecting the active filters, and pagination that hands back both a page count and a
`next_cursor` for deep paging (§3.5).

---

## 3.3 `GET /v1/autocomplete` — the ultra-low-latency path

Autocomplete is a **separate endpoint with a separate SLA and, in production, a separate service
and datastore.** Why not just call `/v1/search` with a short `q`?

1. **Volume.** Each search is preceded by 5–8 keystrokes, so typeahead runs at **~500 K QPS**,
   roughly 5× the search path. Routing that through the full search stack (query understanding,
   scatter-gather over 600 shards, LTR reranking) would need 5× the fleet for work we don't need.
2. **Latency.** Its budget is **p99 ≤ 100 ms** (half of search) because it fires on every
   keystroke. A full relevance query cannot hit that reliably.
3. **Different data shape.** It returns *suggestions* (query completions, categories, top
   products), not a ranked, faceted result set. It's backed by a purpose-built structure —
   an ES `completion` suggester / `edge_ngram` field or a dedicated prefix store (Ch4, Ch8) —
   not the main scatter-gather query.

Keeping it separate lets us cache aggressively (the top-N completions for a prefix are extremely
skewed and cacheable), scale it independently, and degrade it independently: if autocomplete
falls over, search still works.

### Parameters

| Param | Type | Required | Default | Notes |
|---|---|:--:|---|---|
| `q` | string | yes | — | The prefix typed so far. |
| `locale` | string | no | inferred | Language-specific suggestions. |
| `market` | string | no | inferred | Market-specific popularity ranking. |
| `limit` | int | no | `10` | Suggestions to return, **≤ 10**. |

### Example

```http
GET /v1/autocomplete?q=wireles%20head&locale=en-US&limit=5
Authorization: Bearer <buyer-JWT>
```

```json
{
  "request_id": "req_ac_5521",
  "query": "wireles head",
  "suggestions": [
    { "text": "wireless headphones",       "type": "query",    "score": 0.98 },
    { "text": "wireless headphones sony",   "type": "query",    "score": 0.71 },
    { "text": "wireless headset",           "type": "query",    "score": 0.66 },
    { "text": "Headphones",                 "type": "category", "category_path": ["Electronics","Audio","Headphones"] },
    { "text": "Sony WH-1000XM5",            "type": "product",  "product_id": "P-55021" }
  ]
}
```

Note it still tolerates the typo (`wireles` → `wireless`) and mixes suggestion types (`query`,
`category`, `product`) so the UI can render a rich dropdown. The response is deliberately thin —
no facets, no full documents — to protect the 100 ms budget.

---

## 3.4 Filters & facets on the wire

Filters and facets are two sides of one feature: **facets** describe the refinements available
(and how many results each would yield); **filters** apply the buyer's chosen refinements. The
API represents them symmetrically.

**Filter syntax** is `field:value`, repeatable, with three shapes:

| Shape | Example | Meaning |
|---|---|---|
| Term (exact) | `filters=brand:Sony` | `brand == "Sony"` |
| Multi-select (OR within a field) | `filters=brand:Sony&filters=brand:Bose` | `brand IN (Sony, Bose)` |
| Range | `filters=price:50-200` | `50 ≤ price ≤ 200` (open-ended: `price:500-`) |
| Threshold | `filters=rating:gte:4` | `rating ≥ 4` |

The combining rule matters and should be stated explicitly: **values of the same field OR
together; different fields AND together.** So `brand:Sony`, `brand:Bose`, `price:50-200` means
`(brand=Sony OR brand=Bose) AND (price in 50..200)`. This is exactly what a faceted UI expects —
picking a second brand *widens* results, picking a price band *narrows* them. Internally this maps
to an ES `bool` with one `should`-group per multi-select field, all inside `filter` context (no
scoring, cache-friendly), but the client never sees that.

**Facet counts** are returned per field as an array of `{value, count}` (or `{from, to, count}`
for ranges), plus a `selected` flag. A crucial subtlety senior candidates should raise: **counts
for a multi-select facet are normally computed as if that facet's own filter were not applied**,
so the buyer can still see "Sennheiser (611)" even after selecting Sony+Bose — otherwise the
already-selected facet would collapse to only its selected values and the buyer couldn't add a
third brand. In ES this is the `post_filter` + per-aggregation `filter` pattern (Ch7). The
count is a *would-yield-this-many* number, which is why it can exceed the current `total_results`.

Clients request only the facets they render via the `facets` param; each aggregation costs CPU on
the data nodes, so computing brand+price+rating rather than all 40 attributes keeps the query
inside budget.

---

## 3.5 Pagination in depth

Pagination looks trivial and is a top source of production incidents at scale. We support two
mechanisms and pick between them by depth.

### The two mechanisms

**`page` + `size` (offset paging).** Intuitive: `page=5&size=24` means "skip 96, return 24." The
buyer sees "Page 5 of 1396" and can jump around. **The deep-pagination problem:** in a sharded
engine, to return results 5000–5024 sorted by score, **every one of the ~600 shards must produce
its own top 5024**, ship them to the coordinating node, which merges ~3M candidates and discards
all but 24. Cost grows with `from + size`, *linearly in the offset*, across every shard. ES
enforces this reality with `index.max_result_window` (default 10 000). Deep offset paging is how
you turn one buyer's "jump to page 900" into a cluster-wide latency spike.

**`search_after` (cursor paging).** Instead of an offset, the client passes the **sort values of
the last item on the previous page**. The engine seeks directly to "items sorted after this
point" — cost is independent of depth, because no shard has to build-and-discard a giant prefix.
The trade-off: you can only go forward page-by-page (no random "jump to page 900"), and the
cursor encodes a sort position, not a row number.

**`scroll`** is a third, ES-specific mechanism: it freezes a point-in-time snapshot and streams
the entire result set. It's built for **batch export / reindex jobs, not interactive users** — it
pins segment resources for the scroll's lifetime, which is fine for one nightly job and disastrous
at 100 K QPS. We deliberately **do not** expose it on the public API.

### Our policy

| Depth | Mechanism | Rationale |
|---|---|---|
| Pages 1–~50 (`from+size` ≤ ~1200) | `page`+`size` | Cheap enough; enables page jumps the UI wants. |
| Beyond ~page 50 | `search_after` cursor | Constant cost regardless of depth; the only safe deep-paging option. |
| Full export / reindex | `scroll` / PIT (internal only) | Batch, never buyer-facing. |

In practice buyers almost never go past page 3, so offset paging covers ~99% of traffic; the
cursor exists to keep the long tail (and bots/crawlers) from hurting the cluster. The `size ≤ 100`
cap and the ~page-50 offset ceiling together bound worst-case fan-out cost.

### Cursor token flow

The `next_cursor` is an **opaque, base64-encoded** token — the client must treat it as a magic
string, never parse it. Internally it wraps the `search_after` sort tuple plus context we need to
keep results stable:

```jsonc
// decoded next_cursor (illustrative — clients never see this)
{
  "sa": [349.00, "L-771043"],   // last item's sort values: [price, tiebreaker=listing_id]
  "sort": "price_asc",
  "q_hash": "a91f",             // bind cursor to the original query+filters
  "index": "products-v7"        // pin to the index version for stable paging
}
```

Flow:

```
Client ── GET /v1/search?q=...&size=24 ───────────────▶  page 1 + next_cursor C1
Client ── GET /v1/search?q=...&size=24&cursor=C1 ─────▶  page 2 + next_cursor C2
Client ── GET /v1/search?q=...&size=24&cursor=C2 ─────▶  page 3 + next_cursor C3
                                                          ... has_more:false ⇒ end
```

Two robustness details to mention in an interview: (1) a **tiebreaker** in the sort tuple
(`listing_id`) is mandatory — without a unique last key, items sharing a sort value can be
skipped or duplicated across pages; (2) pinning the cursor to `products-v7` (Ch6's versioned
index behind the `products` alias) keeps a paging session stable even if a reindex swaps the alias
mid-session. When `q` or `filters` change, the client must start over from page 1 — the old
cursor is meaningless against a new query.

---

## 3.6 Internal indexing APIs

These are **not public**. They live under `/internal`, are reachable only inside the service mesh,
and authenticate via **mTLS** (§3.7), never a buyer JWT. They exist so the Indexing Service can
push documents into Elasticsearch — but note the big caveat up front.

> **In production, indexing is CDC/Kafka-driven, not synchronous REST.** The Catalog Service
> (source of truth) emits change events → CDC → Kafka → Indexing Service → ES bulk API (brief §5,
> detailed in **Ch6**). These REST endpoints are the *contract/backfill/manual-correction*
> surface, not the hot path. Nobody's checkout blocks on an HTTP index call.

### Endpoints

| Method + path | Purpose |
|---|---|
| `POST /internal/index/product` | Create a listing (server-assigned or client-supplied `listing_id`). |
| `PUT  /internal/index/product/{listing_id}` | Upsert a listing (full replace). |
| `DELETE /internal/index/product/{listing_id}` | Remove a listing. |
| `POST /internal/index/_bulk` | Batched create/update/delete — the real workhorse. |

### Idempotency & versioning

The catalog change stream is **at-least-once** — the same event can arrive twice, and events can
arrive out of order (a price-up event delayed behind a later price-down). If writes weren't
idempotent and ordered, we'd resurrect stale prices. So every write carries the document's
**external version** (`_version` from the data model, brief §4), and ES applies it with
`version_type=external`: **an incoming write is accepted only if its version is greater than the
stored version; a stale/duplicate write is rejected with `409 Conflict` and dropped.** The
`listing_id` is the idempotency key; `_version` is the ordering guard. This gives us safe,
replayable, out-of-order-tolerant indexing — exactly what a Kafka consumer needs.

```http
PUT /internal/index/product/L-771043
Content-Type: application/json
```
```json
{
  "listing_id": "L-771043",
  "product_id": "P-55021",
  "title": "Sony WH-1000XM5 Wireless Noise-Cancelling Headphones",
  "brand": "Sony",
  "category_path": ["Electronics", "Audio", "Headphones"],
  "price": { "amount": 349.00, "currency": "USD" },
  "in_stock": true, "inventory": 42,
  "rating": 4.7, "review_count": 21843,
  "locale": "en-US", "market": "US",
  "_version": 1699999827000
}
```

`_version` is often a monotonic timestamp (event-time epoch millis) from the source, which makes
"greater version wins" equivalent to "latest change wins" without a central counter.

### Bulk

```http
POST /internal/index/_bulk
Content-Type: application/x-ndjson
```
```
{"index":{"_id":"L-771043","version":1699999827000,"version_type":"external"}}
{"listing_id":"L-771043","price":{"amount":349.00,"currency":"USD"}, ... }
{"delete":{"_id":"L-640001","version":1699999830000,"version_type":"external"}}
```

Bulk is newline-delimited JSON (NDJSON) to amortize per-request overhead — the indexing pipeline
batches thousands of mutations per call (Ch6). The response is **partial-success by design**: a
`200` with a per-item status array, so one bad document (or one `409` stale-version reject)
doesn't fail the batch. The consumer inspects each item, retries the retryable ones, and DLQs the
rest (Ch10).

---

## 3.7 Cross-cutting concerns

### Authentication & authorization

- **Buyer path:** the client presents a short-lived **JWT** (issued after login) as a Bearer
  token. The **API Gateway** validates the signature and expiry, extracts `user_id`/`market`, and
  forwards a trusted, minimal identity header to the Search Service. The Search Service never
  re-validates the raw JWT — the gateway is the trust boundary. Anonymous browse is allowed via a
  restricted app-level token with tighter rate limits.
- **Internal path:** service-to-service calls (Indexing Service → ES, Search Service → ES) use
  **mTLS** — both sides present certificates from an internal CA. There is no JWT here; identity is
  the cert. This is why `/internal/*` is unreachable from the public internet at all.

### Rate limiting

Enforced at the gateway, before any expensive work. Tiered:

| Tier | Limit (illustrative) | Purpose |
|---|---|---|
| Per authenticated buyer | ~10 req/s search, ~50 req/s autocomplete | Autocomplete is keystroke-driven, so its limit is higher. |
| Per IP (anonymous) | Lower, stricter | Bot/crawler defense. |
| Per API key (partner) | Contractual quota | Partner integrations. |

Over-limit ⇒ **`429 Too Many Requests`** with a `Retry-After` header. Limiting at the edge protects
the 600-shard cluster from a stampede turning into a cascading failure (Ch10).

### Error handling

Consistent status codes and a **single error envelope** on every non-2xx:

```json
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "size must be between 1 and 100",
    "field": "size",
    "request_id": "req_9f3c2a7b",
    "retryable": false
  }
}
```

| Status | When | Retryable? |
|---|---|---|
| `200` | Success (incl. zero results — an empty `results` array is **not** an error). | — |
| `400` | Malformed params (`size>100`, bad `filters` syntax). | No |
| `401` / `403` | Missing/invalid JWT / not authorized. | No |
| `404` | Unknown route, or `DELETE` of a non-existent `listing_id`. | No |
| `409` | Stale external version on an index write. | No (drop) |
| `422` | Well-formed but semantically invalid (unknown `sort` value). | No |
| `429` | Rate limited. | Yes, after `Retry-After` |
| `503` | Cluster degraded / circuit open — see graceful degradation, Ch10. | Yes, backoff |

The `request_id` is the golden thread: it appears in the response, the access logs, and the
distributed trace, so any error can be traced end-to-end. `retryable` tells clients whether a
retry could ever help — critical for well-behaved retry/backoff logic.

### Versioning strategy

- **URL-path versioning** (`/v1`, `/v2`) — the most explicit, cache- and log-friendly option, and
  trivial to route at the gateway. We prefer it over header-based versioning for a public API.
- Within `v1`: additive-only. New capability that fits the existing shape ⇒ new optional param or
  new response field. New capability that breaks the shape ⇒ `v2`.
- **v1 and v2 run in parallel**; a published deprecation window (e.g. 12 months) with `Deprecation`
  / `Sunset` headers on `v1` responses gives clients time to migrate. We don't flip a switch.

---

## 3.8 GraphQL / gRPC — and why REST/JSON is the default

Interviewers love to ask "why not GraphQL?" or "why not gRPC?" Answer with trade-offs, not dogma.

**GraphQL** lets clients ask for exactly the fields they want and compose multiple resources in one
round trip — attractive for a heterogeneous storefront. But for *search* it fights us: (1) search
is one operation with a well-known, already-compact response shape, so GraphQL's field-selection
win is marginal; (2) arbitrary client-shaped queries make **caching** (CDN, Redis result cache) far
harder — REST's stable, cacheable URLs are a huge asset when the top queries are extremely skewed;
(3) it complicates cost control and rate limiting (query complexity analysis vs. a simple URL). A
common real-world compromise: a GraphQL **gateway** for the overall app that calls our REST search
service underneath — GraphQL for composition at the edge, REST for the search primitive.

**gRPC** (HTTP/2 + protobuf) is excellent for **internal** service-to-service traffic: binary
framing, lower serialization cost, streaming, strong schemas. It's genuinely a good fit for
Search Service ↔ Ranking Service ↔ ES-facing internals, where every millisecond and every byte of
the 200 ms budget counts. But it's a poor fit for the **public browser-facing** API: patchy native
browser support (needs grpc-web + a proxy), harder to debug/curl/cache at the CDN, and a steeper
integration curve for third parties.

**So:** **REST + JSON over HTTPS is the default for the public API** — universal client support,
human-debuggable, CDN- and cache-friendly, and stateless-scalable, which matters more than raw
byte-efficiency on a path fronted by an edge cache. Internally, we're free to use gRPC between
services where the latency/serialization savings pay off. Best tool per boundary.

---

## Interview Tips

- **Lead with principles, then the endpoint.** Say "client-friendly, versioned, backward-compatible,
  never leak the ES DSL" before you write JSON. It frames every later choice.
- **Design `/v1/search` to power the whole SRP in one call** — results + facets + pagination +
  query metadata. A candidate who returns only a list of IDs and needs three more calls looks junior.
- **Nail the deep-pagination answer.** Explain *why* `from+size` is O(offset) per shard across 600
  shards, then introduce `search_after` as constant-cost, and note the tiebreaker + index-pinning
  details. This is the single most common APIs-chapter follow-up.
- **Justify a separate autocomplete endpoint** with numbers: ~500 K QPS, 100 ms budget, different
  data structure, independent scaling and failure isolation.
- **Explain idempotency crisply:** `listing_id` is the key, external `_version` is the ordering
  guard, stale writes get `409` and are dropped — and tie it to at-least-once Kafka delivery (Ch6).
- **Have a crisp GraphQL/gRPC answer:** REST/JSON public for cacheability and universality, gRPC
  internal for efficiency. Trade-offs, not dogma.
- **Common traps:** exposing raw ES queries; conflating `locale` and `market`; no `size` cap;
  treating zero results as an error; comma-joined filters instead of repeated params; forgetting a
  tiebreaker in cursor sort.

## Key Takeaways

- The public surface is exactly two read endpoints — `GET /v1/search` and `GET /v1/autocomplete` —
  plus a non-public `/internal/index/*` write surface; keep read and write contractually separate.
- `GET /v1/search` returns results, facets (with counts + `selected`), pagination, and query
  metadata (spelling correction, applied synonyms) in **one round trip**.
- Filters are repeatable `field:value`; **same field ORs, different fields AND**; facet counts are
  computed so multi-select stays usable and can exceed `total_results`.
- Use `page`+`size` up to ~page 50, then **`search_after` cursors** (opaque, tiebroken,
  index-pinned) for constant-cost deep paging; `scroll`/PIT is batch-only, never buyer-facing.
- Indexing is idempotent via `listing_id` + external `_version` (stale ⇒ `409`, drop); the REST
  endpoints are a contract/backfill surface — the real path is CDC → Kafka → bulk (Ch6).
- Buyer auth = short-lived JWT at the gateway; internal auth = mTLS. Rate-limit and validate at the
  edge; every response carries a `request_id` and a consistent error envelope.
- **REST/JSON public, gRPC internal, GraphQL at most as an edge gateway** — chosen for
  cacheability, universality, and the 200 ms latency budget.
