# Chapter 8B — Hybrid Search Internals

> **Where we are.** Chapter 7 walked the read path and stopped at a clean two-phase shape:
> retrieve cheaply with BM25, then rerank the survivors. Chapter 8 opened the Elasticsearch box
> and explained *how* BM25 scoring actually works inside a Lucene shard. Both chapters left a
> promise dangling — Chapter 5 and Chapter 11 both said "vectors get layered in later." This is
> later.
>
> This chapter adds a second way to *retrieve* candidates: **semantic search** using vectors.
> Then it shows how to combine the two — keyword search and vector search — into one system.
> That combination is called **hybrid search**. We will build the idea up slowly, in plain
> language, because most of it is new. By the end you should be able to explain, from scratch,
> what an embedding is, how vector search finds nearest neighbors, why the two kinds of scores
> refuse to add up, and how GlobalMart stitches them together without setting fire to its memory
> budget.

---

## 8B.1 Why Hybrid Search Exists

Let us start with the problem, not the solution.

GlobalMart already has a very good keyword search. Chapter 8 explained it: the inverted index
finds documents that contain the query words, and BM25 scores them by how well the words match.
This kind of search has a name. It is called **lexical search**.

> **Lexical search** means matching on the actual words (the letters and tokens) in the query
> and the document. "Lexical" just means "to do with words." BM25 is a lexical scorer.

Lexical search is fast, cheap, and very precise about exact words. But it has a blind spot: **it
only matches words it literally sees.** If the query word and the document word are spelled
differently, lexical search does not connect them, even when they mean the same thing.

Here are three e-commerce examples where pure keyword search fails.

**Example 1 — synonyms it was never told about.** A buyer in the UK types `trainers`. The
seller listed the product as `running shoes` or `sneakers`. The words do not overlap, so BM25
scores the match near zero. Chapter 7 fixes some of this with a hand-maintained synonym list
(`sneakers ↔ trainers`). But a synonym list only knows the synonyms a human wrote down. It will
never contain every phrasing buyers invent.

**Example 2 — intent expressed as a description.** A buyer types `warm jacket for a toddler`.
No product title contains that exact phrase. The good matches are titled things like `insulated
fleece coat, kids 2-3 years` or `winter puffer, infant`. A keyword search sees `warm` and
`jacket` and `toddler` and struggles: the best products do not use any of those three words.
The *meaning* matches, but the *words* do not. Lexical search cannot read meaning.

**Example 3 — a vague, human query.** A buyer types `something to keep coffee hot on my desk`.
The product is a `stainless steel vacuum travel mug`. Zero word overlap. Keyword search returns
junk or nothing.

So keyword search is great when the buyer uses the same words as the catalog, and poor when
they do not. It matches **words**, not **meaning**.

The obvious wish is: "I want search that understands what the buyer *means*, not just the words
they typed." That wish is exactly what **semantic search** provides.

> **Semantic search** means matching on *meaning* rather than on exact words. Two pieces of text
> can score as a strong match even if they share no words at all, as long as they mean similar
> things.

Semantic search is powered by **vectors** (we define these carefully in the next section). It is
wonderful at the three examples above. `warm jacket for a toddler` lands right next to
`insulated fleece coat, kids 2-3 years`, because the system has learned those phrases mean
similar things.

But — and this is the crucial part — **pure semantic search has its own blind spots, and in
e-commerce they are dangerous.** Semantic search returns things that are *similar in meaning*,
and "similar in meaning" is not the same as "the exact thing the buyer wants to buy."

Here are three e-commerce examples where pure vector search fails.

**Example 1 — it ignores exact model numbers.** A buyer types `iPhone 15 128GB`. To a
meaning-based system, `iPhone 15 256GB`, `iPhone 14 128GB`, and even `iPhone 15 Pro` all look
extremely similar — they are all iPhones, all around the same idea. The buyer asked for a very
specific SKU (a specific product variant). Semantic similarity blurs the exact numbers that
matter most. Keyword search, by contrast, nails exact tokens like `128GB` and `15`.

**Example 2 — wrong brand, right vibe.** A buyer types `Nike running shoes`. A vector search may
happily return Adidas and Puma running shoes, because "running shoes" dominates the meaning and
the brand word is just one small part of it. Commercially this is wrong: the buyer named a brand.

**Example 3 — commercially wrong even when semantically right.** A vector search might return a
product that is a perfect meaning-match but is out of stock, is from a banned seller, or costs
ten times the going rate. The vector only knows about *text meaning*. It knows nothing about
price, stock, or business rules.

Now put the two side by side.

| | Lexical (BM25 / keyword) | Semantic (vectors) |
|---|---|---|
| Matches on | Exact words / tokens | Meaning |
| Great at | Exact model numbers, SKUs, brand names, rare tokens | Synonyms, paraphrases, vague "describe-it" queries |
| Bad at | Synonyms it wasn't told about, intent, paraphrase | Exact numbers, precise brand/model, commercial correctness |
| Needs training? | No (BM25 works out of the box) | Yes (needs a trained embedding model) |

Look at that table. The two columns are almost mirror images. Where one is strong, the other is
weak. **That is the entire reason hybrid search exists.** If you run both and combine their
results, the keyword side catches the exact `iPhone 15 128GB` and the brand-specific `Nike`
queries, while the vector side catches `warm jacket for a toddler` and `trainers`. Each covers
the other's blind spot.

> **Hybrid search** means running lexical search and semantic search together and merging their
> results into one ranked list. The rest of this chapter is about how to do that merge well.

A quick mental picture:

```
   query "warm jacket for a toddler"
            │
     ┌──────┴───────┐
     ▼              ▼
  LEXICAL        SEMANTIC
  (BM25 over     (vector kNN over
   inverted       embeddings)
   index)             │
     │                │
   keyword          meaning
   matches          matches
     └──────┬───────┘
            ▼
       FUSE the two lists
            ▼
     one ranked result set
```

---

## 8B.2 Embeddings and Vectors — the Basics

Semantic search rests on one idea: **turn text into numbers so that similar meanings become
nearby numbers.** Let us define that idea carefully, because everything else depends on it.

### What is a vector?

A **vector** is just a list of numbers. That is all. `[0.12, -0.98, 0.34, ...]` is a vector.
The length of the list (how many numbers) is called the number of **dimensions**. A vector with
768 numbers in it is a "768-dimensional vector."

### What is an embedding?

An **embedding** is a vector that captures the *meaning* of a piece of text (or an image). You
feed a sentence into a special model, and it hands you back a list of numbers — the embedding —
that represents what that sentence means.

> **Embedding (plain version):** a list of numbers that captures the meaning of a piece of text,
> produced by a trained model.

The magic property is this: **texts with similar meaning get embeddings whose numbers are
close together; texts with different meaning get embeddings that are far apart.** The model is
trained on huge amounts of text specifically so that this comes out true.

A tiny, made-up illustration in just 2 dimensions (real ones have hundreds):

```
              ^  (dimension 2)
              │
  "puppy" •   │   • "kitten"
              │
   "dog" •    │      • "cat"
──────────────┼──────────────────▶ (dimension 1)
              │
              │           • "airplane"
              │
```

Notice how `dog`, `puppy`, `cat`, `kitten` cluster together (all pets), while `airplane` sits
far away. The model placed them there based on meaning. `warm jacket for a toddler` would land
near `kids winter coat`, and both would be far from `garden hose`.

### "Similar meaning = close in space" — how we measure closeness

If meaning becomes position, then "how similar are these two texts?" becomes "how close are
these two points?" We need a way to measure closeness between two vectors. The common one is
**cosine similarity**.

> **Cosine similarity** measures the *angle* between two vectors, ignoring how long they are.
> If two vectors point in almost the same direction, the angle between them is tiny, and the
> cosine similarity is close to **1** (very similar). If they point in totally unrelated
> directions, it is close to **0**. If they point opposite ways, it is **-1**.

Why the angle and not the plain straight-line distance? Because for text meaning, *direction*
turns out to carry the meaning, and the length of the vector is mostly noise. Two ways of saying
the same thing point the same direction even if one is a longer sentence. (Some systems use
plain straight-line distance, called Euclidean distance, or the dot product instead. The idea is
the same: a single number that says "how close.")

A rule of thumb to hold in your head: **higher cosine = more similar meaning.** `iPhone 15` vs
`Apple smartphone` might score 0.82; `iPhone 15` vs `garden hose` might score 0.05.

### How big are real embeddings?

Real embedding models produce vectors with a few hundred to a couple thousand dimensions.
Common sizes you will hear:

| Dimensions | Typical use |
|---|---|
| 384 | Small, fast models (good default for cost-sensitive scale) |
| 768 | Very common middle ground (BERT-base sized models) |
| 1024 | Larger, more accurate models |
| 1536 / 3072 | Some commercial API models |

More dimensions can capture more nuance but cost more memory and compute. We will feel that cost
sharply at GlobalMart's scale in §8B.7.

### What produces the embedding? The model.

The thing that turns text into a vector is an **embedding model**. The common design for search
is a **bi-encoder**, also called a **sentence-transformer**.

> **Bi-encoder:** a model that encodes (turns into a vector) the query and the document
> *separately* and *independently*. The query becomes one vector; each product becomes its own
> vector. They never "see" each other during encoding.

The word "bi" means "two" — two separate encodings. This separation is what makes it fast and
scalable, because we can embed all 10B products *ahead of time* and reuse those vectors for every
query. (In §8B.5 we will meet its slower, more accurate cousin, the cross-encoder, which does let
query and document see each other.)

There are also **multimodal** models.

> **Multimodal** means the model handles more than one type of input — for example, both text and
> images — and places them in the *same* vector space.

A multimodal model can embed a product *photo* and a text query into the same space, so that the
text `red floral summer dress` lands near the embedding of an actual photo of a red floral dress.
This is what powers image-based product search (search by picture), which we touch on in §8B.10.

### The offline embedding pipeline (ties to Chapter 6)

Here is the practical workflow, and it maps cleanly onto the indexing pipeline from Chapter 6.

**At index time (offline, done once per product):** when the Indexing Service processes a
listing (CDC → Kafka → Indexing Service → Elasticsearch, brief §5), it adds one extra step. It
takes the product's text — usually title plus key attributes — sends it through the embedding
model, gets back a vector, and stores that vector on the document as a field. So every one of
our ~10B listings carries its embedding, computed once, sitting in the index ready to be
searched.

**At search time (online, done once per query):** when a buyer types `warm jacket for a
toddler`, the Search Service sends that query string through the *same* embedding model, gets a
query vector, and uses it to find the nearest product vectors.

```
INDEX TIME (offline, in the Indexing Service — Ch6)
  product text ──▶ embedding model ──▶ vector ──▶ stored on the ES document

SEARCH TIME (online, in the Search Service — Ch7)
  query text ─────▶ embedding model ──▶ query vector ──▶ find nearest product vectors
```

> **The one rule you must never break:** the query and the documents must be embedded by the
> **same model version**. If products were embedded with model v3 and the query is embedded with
> model v4, their vectors live in *different* spaces and "closeness" becomes meaningless. This is
> the vector-search cousin of Chapter 8's cardinal rule that index-time and search-time analysis
> must agree.

### Model versioning and the cost of re-embedding

This "same model" rule has a brutal consequence at scale. Suppose next quarter a better
embedding model comes out. To use it, you must **re-embed every document** — all 10B of them —
because the old vectors are now incompatible with the new query vectors.

Re-embedding 10B products is a massive batch job: 10B model inferences, plus rewriting 10B
documents into the index. It is not free and it is not instant. So embedding models are treated
like a **major, versioned migration**, done with the same alias-and-versioned-index pattern
Chapter 8 used (`products-v7` → `products-v8`): build the new index with new vectors in the
background, then flip the alias. You do not swap embedding models casually. We return to this
under pitfalls (§8B.7) and best practices (§8B.8).

---

## 8B.3 How Vector (kNN) Search Works

We can now turn text into vectors and measure how close two vectors are. But a buyer's query
vector has to be compared against *how many* product vectors? Ten billion. We cannot compare
against all of them one by one for every query. This section is about how vector search finds
the closest vectors quickly.

### k-Nearest-Neighbors, in plain terms

The core operation is called **k-Nearest-Neighbors search**, usually shortened to **kNN**.

> **kNN search:** given a query vector, find the `k` product vectors that are closest to it.
> "k" is just how many you want — for example, "find the 100 nearest products."

That is the whole idea. Embed the query, then find its k nearest neighbors among the product
vectors. Those neighbors are the products whose meaning is most like the query.

### Exact kNN — correct but far too slow

The simplest way to find the nearest neighbors is to compute the distance from the query vector
to *every* product vector, then keep the closest k. This is **exact kNN** (also called
brute-force kNN). It gives the perfectly correct answer.

The problem is arithmetic. Comparing one query against 10B vectors, each with 768 numbers, means
roughly `10,000,000,000 × 768` multiply-and-add operations — trillions of operations — *for a
single query*. At GlobalMart's ~100K queries per second (brief §2), exact kNN is completely
impossible. It would blow the 200 ms latency budget by a factor of thousands.

So at scale we give up on *perfectly* exact and accept *almost* exact in exchange for enormous
speed. That trade-off has a name.

### Approximate Nearest Neighbor (ANN)

> **Approximate Nearest Neighbor (ANN):** a family of techniques that find *most* of the true
> nearest neighbors *most* of the time, but run hundreds or thousands of times faster than exact
> search. You trade a small amount of accuracy for a huge amount of speed.

The accuracy we give up is measured as **recall**.

> **Recall (for ANN):** of the true nearest neighbors, what fraction did the fast approximate
> search actually find? Recall of 0.95 means it found 95 of the true top 100 and missed 5.

At 10B documents, ANN is not optional — it is the only way vector search is viable at all. The
most popular ANN method, and the one Elasticsearch uses, is called **HNSW**.

### HNSW — a graph you walk toward the answer

HNSW stands for **Hierarchical Navigable Small World**. The name is scary; the idea is not.

Here is the intuition first. Imagine every product vector is a city on a map. You want to travel
from where the query lands to the nearest city. If you had to check every city, that is exact
kNN (too slow). Instead, HNSW builds a **road network** ahead of time — it connects each city to
a handful of nearby cities. To find the nearest city, you start somewhere and keep walking to
whichever neighboring city is closer to your target, over and over, until you cannot get any
closer. You only ever visit a tiny fraction of the cities.

Now the "hierarchical" part. HNSW builds **several layers** of this road network, like zoom
levels on a map:

- The **top layer** has very few cities but long-distance highways. You cross huge distances
  fast.
- Each layer down has more cities and shorter roads.
- The **bottom layer** has *every* city and only short local roads.

You start at the top, zoom across the country on highways to get roughly to the right region,
then drop down a layer, refine, drop down again, and finish with careful local steps at the
bottom. This is why it is fast: the top layers get you close in a few big hops, and only the
final layer does fine-grained work.

```
   Layer 2 (few nodes, long hops)     A ───────────────── B
                                       \                 /
   Layer 1 (more nodes)             A ── c ── d ──── e ── B
                                        \    |      /
   Layer 0 (ALL nodes, short hops)  A-c-x-y-d-z-w-e-...-B   ← full graph, land on the answer
```

You "enter" at the top and greedily walk toward the query vector, dropping down layers as you go.

### The three HNSW knobs

HNSW has three tuning knobs, and a good interview answer names all three and what each trades.

| Knob | When it applies | What it controls | Turn it UP → | Turn it DOWN → |
|---|---|---|---|---|
| **M** | Build time | How many neighbor links each node keeps | Higher recall, more memory, slower build | Less memory, lower recall |
| **ef_construction** | Build time | How hard it searches while *building* the graph | Better-quality graph, higher recall, slower build | Faster build, lower-quality graph |
| **ef_search** | Query time | How many candidates it explores while *answering* | Higher recall, slower query | Faster query, lower recall |

The one to really understand is the fundamental three-way tension: **recall vs. speed vs.
memory.** You cannot max out all three.

- Want higher recall? Raise `M` and `ef_search` — but you pay more memory (`M`) and slower
  queries (`ef_search`).
- Want faster queries? Lower `ef_search` — but recall drops, so you silently miss good products.
- Want less memory? Lower `M` — but recall drops again.

The dangerous word there is **silently**. If `ef_search` is too low, vector search still returns
results — it just quietly leaves out some of the best ones, and nothing errors. This is why
recall must be *measured*, not assumed (§8B.7).

### IVF and PQ — the alternative family (one paragraph)

HNSW is graph-based. There is another popular ANN family based on **clustering**, usually
labelled **IVF** and **PQ**. **IVF (Inverted File index)** first groups all the vectors into
clusters (like sorting products into bins by rough meaning); at query time it only searches the
few bins nearest the query, skipping the rest. **PQ (Product Quantization)** is a compression
trick: it chops each vector into small pieces and replaces each piece with a short code from a
small codebook, shrinking memory dramatically at some cost to accuracy. IVF+PQ (popularized by
Facebook's FAISS library) trades a bit more accuracy for much lower memory than HNSW, which is
why very large systems sometimes prefer it. Elasticsearch's default is HNSW, so that is our main
focus.

### How Elasticsearch stores vectors

In Elasticsearch, a vector lives in a field of type **`dense_vector`**, and under the hood
Lucene builds an HNSW graph over those vectors (per segment — remember from Chapter 8 that a
shard is made of immutable segments, and each segment carries its own little HNSW graph).

A minimal mapping:

```jsonc
PUT /products-v8
{ "mappings": { "properties": {
    "title_vector": {
      "type": "dense_vector",
      "dims": 768,               // must match the embedding model's output size
      "index": true,             // build an HNSW graph for fast ANN
      "similarity": "cosine",    // how "closeness" is measured
      "index_options": {
        "type": "hnsw",
        "m": 16,                 // the M knob
        "ef_construction": 100   // the build-time effort knob
      }
    }
} } }
```

That is the whole vector-storage story: a `dense_vector` field, `dims` matching the model,
`similarity` set to cosine, and HNSW parameters. Every product document now carries a
`title_vector`, and Lucene can answer "find the k nearest" quickly.

---

## 8B.4 Scoring — and Why the Two Scores Do Not Mix

We now have two ways to score a product against a query.

1. **BM25** (Chapter 8): a lexical score based on term frequency, inverse document frequency,
   and field length. We will not re-derive it — see Chapter 8, §8.8. The key fact to carry
   forward: a BM25 score is an **unbounded positive number**. A great match might score 12.7;
   another query's great match might score 45.0. There is no fixed maximum. The scale depends on
   the query, the terms' rarity, and the corpus.

2. **Vector similarity** (this chapter): a closeness score. With cosine similarity it lives on a
   **fixed, bounded scale** — roughly 0 to 1 for text embeddings (Elasticsearch shifts cosine
   into a 0–1 range so scores are always positive).

Now look at those two ranges together:

```
   BM25 score:            0 ─────────── 12.7 ────────── 45.0 ──────▶  (no upper bound)
   Vector cosine score:   0 ──── 0.5 ──── 0.82 ── 1.0                 (fixed 0..1)
```

Here is the trap that catches everyone the first time. Suppose a product has a BM25 score of
18.0 and a cosine score of 0.82. Can we just add them? `18.0 + 0.82 = 18.82`? **No.** The BM25
number (18.0) completely dominates; the vector number (0.82) barely moves the total. The two
scores are measured in **totally different units on totally different scales**, like adding a
temperature in Celsius to a distance in miles. The sum is meaningless.

This is the single most important idea in hybrid search:

> **You cannot just add a BM25 score and a vector similarity score.** They live on different
> scales, so naive addition lets one score silently drown out the other. Every hybrid technique
> in the next section is really an answer to the question: *how do we combine two scores that
> refuse to share a scale?*

---

## 8B.5 Combining the Two — the Heart of "Hybrid"

There are three main ways to combine lexical and semantic results, and they get progressively
better. We will go from simplest to best.

### (a) Weighted linear combination — and the normalization problem

The first instinct is: "OK, the scales are different, so let me *rescale* both scores onto the
same range first, then add them with weights I choose." That is a **weighted linear
combination**.

The plan:
1. Take the BM25 scores and squash them onto a 0–1 range.
2. Take the vector scores and squash them onto a 0–1 range.
3. Combine: `final = w × lexical + (1 - w) × semantic`, where `w` is a weight like 0.5.

The squashing step is called **normalization**. Two common ways:

- **Min-max normalization:** find the smallest and largest score in the result set, then rescale
  so the smallest becomes 0 and the largest becomes 1. Formula: `(score - min) / (max - min)`.
- **Z-score normalization:** rescale using the average and the spread (standard deviation) of
  the scores, so the average becomes 0.

This *works*, but it is **fragile**, and it is worth understanding why so you can explain it:

- **The min and max change with every query.** Min-max normalization depends on the highest and
  lowest score *in this particular result set*. A single weird outlier product with a huge BM25
  score stretches the whole scale and squashes everything else near zero. The normalization is
  only as stable as the result set, and result sets are noisy.
- **You have to pick the weight `w` by hand.** Is lexical worth 0.5? 0.7? The right value differs
  per query type (exact SKU queries want more lexical; vague queries want more semantic) and
  drifts as the catalog changes. Tuning one global `w` is guesswork.
- **The distributions are lopsided.** BM25 scores are not shaped like cosine scores, so even
  after rescaling both to 0–1, "0.6 lexical" and "0.6 semantic" do not mean equally good.

So weighted linear combination is understandable but brittle. It exists mostly to motivate a
cleverer idea that sidesteps the scale problem entirely.

### (b) Reciprocal Rank Fusion (RRF) — the common default

The clever idea: **stop looking at the scores at all. Look only at the ranks.**

Here is the intuition. Each search — lexical and semantic — produces a *ranked list*: 1st, 2nd,
3rd, and so on. A rank is just a position. Position 1 from BM25 and position 1 from the vector
search are directly comparable — they both mean "this list's top pick" — even though the raw
scores behind them (18.0 vs 0.82) are not comparable at all. **Ranks share a scale for free;
scores do not.** That is the insight.

**Reciprocal Rank Fusion (RRF)** combines the two ranked lists using only positions.

> **RRF formula.** For each document, add up `1 / (k + rank)` across every list it appears in:
>
> ```
> RRF_score(doc) = Σ   1 / (k + rank_in_list_i)
>                 lists
> ```
>
> `rank` is the document's position in that list (1, 2, 3, …). `k` is a small constant, commonly
> **60**, that softens the difference between the very top ranks. A higher `k` makes the top
> positions matter a little less; 60 is the widely used default.

Let us work a tiny example. A document is ranked **2nd** by the lexical search and **5th** by the
vector search. With `k = 60`:

```
from lexical:  1 / (60 + 2) = 1/62 = 0.01613
from vector:   1 / (60 + 5) = 1/65 = 0.01538
RRF total    = 0.01613 + 0.01538 = 0.03151
```

A document that both lists rank highly gets contributions from both, so it floats to the top. A
document only one list likes still gets a partial score. A document neither likes stays low. And
crucially, **the raw scores never entered the calculation** — so the whole scale-mismatch
problem from §8B.4 simply vanishes.

Why RRF is the common default:
- It needs **no normalization** and **no hand-tuned weight** — just ranks and one constant.
- It is **robust**: a single outlier score cannot distort it, because scores are ignored.
- It is **simple** to implement and reason about.

Its mild downside: because it throws away the actual scores, it ignores *how much* better rank 1
was than rank 2 (a blowout win and a photo-finish win look the same). In practice that loss is
small, and the robustness is worth it. RRF is the standard first choice for fusing hybrid
results, and it is built into Elasticsearch (shown below).

### (c) Retrieve-then-rerank — a cheap net, then a smart judge

The third approach is not really a *fusion* formula. It is a two-stage shape, and it is the same
two-phase idea from Chapter 7 (retrieve cheap, rerank expensive) applied to hybrid retrieval.

Stage 1 — **retrieve** a few hundred candidates cheaply, using *both* lexical and vector search
(often fused with RRF). This casts a wide, cheap net.

Stage 2 — **rerank** just the top candidates (say the top 100–200) with a slow but very accurate
model called a **cross-encoder**. To explain the cross-encoder, contrast it with the bi-encoder
from §8B.2.

> **Bi-encoder** (fast): encodes the query and the document *separately* into two vectors, then
> compares the vectors. Because documents are encoded ahead of time, this is cheap and scales to
> billions of documents. This is what we use for first-stage vector retrieval.
>
> **Cross-encoder** (accurate): takes the query and one document *together, as a pair*, and reads
> them jointly to produce a single relevance score. Because it looks at both at once, it catches
> subtle relationships a bi-encoder misses ("does this jacket really fit a *toddler*, or just a
> child?"). But it must run the model fresh for *every* query-document pair, so it is far too slow
> to run over 10B documents.

The picture:

```
BI-ENCODER (fast, first stage)          CROSS-ENCODER (accurate, rerank stage)

 query ─▶ [model] ─▶ vec_q               ┌─────────────────────────┐
                          ↘ compare      │  query + document        │
 doc  ─▶ [model] ─▶ vec_d ↗ (cheap)      │  read TOGETHER by model  │─▶ one score
   (docs encoded offline, reused)        └─────────────────────────┘
                                          (must run per pair, slow)
```

So the shape is: bi-encoder + BM25 cast the wide cheap net (thousands → top 200), then the
cross-encoder carefully re-scores those 200. You get the cross-encoder's accuracy while only
paying for 200 runs, not 10B. This slots in *before* GlobalMart's business-signal reranking
(LTR), which we cover in §8B.9.

### The concrete Elasticsearch way

Elasticsearch gives you `knn` for vector search and an **RRF retriever** for hybrid fusion. Two
short examples.

**Plain vector (kNN) search** — "embed my query, find the 100 nearest products, but only among
in-stock US items":

```jsonc
POST /products-v8/_search
{
  "knn": {
    "field": "title_vector",
    "query_vector": [0.12, -0.98, 0.34, ...],   // the query, already embedded
    "k": 100,                                    // how many neighbors to return
    "num_candidates": 500,                       // how many to explore (the ef_search lever)
    "filter": [                                  // hard business filters (see §8B.6)
      { "term": { "market": "US" } },
      { "term": { "in_stock": true } }
    ]
  }
}
```

`k` is how many results you want back; `num_candidates` is how many the HNSW walk explores before
picking the best `k` (bigger = higher recall, slower — this is the `ef_search` idea).

**Hybrid search with the RRF retriever** — run a BM25 query *and* a kNN query, then fuse their
ranks with RRF, in one request:

```jsonc
POST /products-v8/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        { "standard": {                          // the lexical side (BM25)
            "query": { "multi_match": {
              "query": "warm jacket for a toddler",
              "fields": ["title^3", "description"] } } } },
        { "knn": {                               // the semantic side (vectors)
            "field": "title_vector",
            "query_vector": [0.12, -0.98, 0.34, ...],
            "k": 100, "num_candidates": 500 } }
      ],
      "rank_constant": 60,        // the "k" in the RRF formula
      "rank_window_size": 100     // how deep into each list RRF looks
    }
  }
}
```

Elasticsearch runs both retrievers, ranks each list, applies the RRF formula from above, and
returns one fused, ranked result set. This one snippet is the practical heart of the whole
chapter.

---

## 8B.6 Filtering with kNN — a Real Gotcha

E-commerce buyers filter constantly: price under $50, brand Nike, in stock only, ships to the
US. In Chapter 7 we put those in `filter` clauses because they are cheap, cacheable, and
non-scoring. With vector search, filtering is trickier than it looks, and it is a favorite
interview gotcha.

The problem: the HNSW graph is built over *all* the vectors. But a filtered query only wants
results that also satisfy the filter (say, in stock). Where do you apply the filter — before or
after the graph walk? Both naive options are broken.

**Post-filter (search first, then filter).** Do the ANN search to get the top 100 nearest
vectors, *then* throw away the ones that fail the filter.

- The danger: if only a tiny fraction of the catalog is in stock, almost all of your 100
  nearest neighbors might be out of stock and get discarded. You asked for 100 results and end
  up with 3. **You can silently return too few results.**

**Pre-filter (filter first, then search).** First find all documents that pass the filter, then
search only among those vectors.

- The danger: HNSW is a *graph you walk*. If you delete most of the nodes first (because they
  failed the filter), the graph's roads get cut. The walk gets stranded — it cannot reach good
  neighbors because the path there ran through now-removed nodes. Recall collapses. **Filtering
  first can wreck ANN accuracy.**

```
Post-filter: [ ANN finds 100 nearest ] → drop out-of-stock → maybe only 3 left  ✗ too few
Pre-filter:  [ remove out-of-stock first ] → walk a graph full of holes → misses good hits  ✗ low recall
```

**How Elasticsearch handles it.** Elasticsearch does neither naive extreme. It performs
**filtered kNN**: the filter is applied *during* the graph walk. As it walks the HNSW graph, it
only *keeps* nodes that pass the filter, but it still *travels through* the full graph structure
to reach them. This keeps the connectivity of the graph (so recall stays high) while ensuring the
returned results all satisfy the filter (so you do not lose them at the end). The `filter` field
inside the `knn` block (shown in §8B.5) is exactly this. If the filter is extremely selective
(almost nothing passes), Elasticsearch can also fall back to exact brute-force kNN over just the
matching documents — which is fine, because if very few documents match, comparing against all of
them is cheap again.

The practical takeaway: **put hard business filters (price, brand, in-stock, market) inside the
`knn` block's `filter`**, and let Elasticsearch apply them during the walk. Do not run the ANN
search wide open and filter afterward in your own code.

---

## 8B.7 Common Issues and Pitfalls with Hybrid Search

Hybrid search adds real power and real complexity. Here are the traps, clearly labelled. If you
can name most of these, you understand the system.

**1. Score-scale mismatch.** As in §8B.4: BM25 and vector scores live on different scales, so
naive addition lets one drown the other. *Mitigation:* fuse on ranks with RRF, or normalize
carefully if you must combine scores.

**2. ANN recall tuning — the silent failure.** With `ef_search`/`num_candidates` set too low,
vector search quietly misses good products and returns *something* anyway, with no error. You can
ship a change that quietly hurts relevance and never see a red light. *Mitigation:* measure recall
against an exact-kNN baseline on a sample, and monitor it (see §8B.8).

**3. Memory and cost at scale — WORK THE NUMBER.** This is the big one for GlobalMart. Vectors
are not free; the whole HNSW graph is held in memory (RAM) for speed. Do the arithmetic:

```
10,000,000,000 docs × 768 dims × 4 bytes (a float32 number)
   = 10e9 × 768 × 4 bytes
   = 30,720,000,000,000 bytes
   ≈ 30 TB   of raw vector data — and that must live in RAM for HNSW to be fast
             (plus more for the graph's neighbor links on top)
```

Thirty terabytes of RAM, just for vectors, before the graph overhead. That is wildly expensive —
far larger than the ~24 TB of on-disk primaries from the brief (§2), and RAM costs vastly more
per terabyte than disk. **Storing raw float32 vectors for 10B documents is economically
impossible.** The fix is **quantization**.

> **Quantization** means storing each number in the vector using *fewer bits*, accepting a small
> loss of precision to save a lot of memory.

The common levels:

| Vector storage | Bytes per number | 10B × 768 total | Accuracy cost |
|---|---|---|---|
| float32 (raw) | 4 bytes | ~30 TB | none (baseline) |
| int8 (byte quantization) | 1 byte | ~7.5 TB | small |
| binary (1 bit per number) | 1 bit | ~0.94 TB | larger, but recoverable |

**int8 quantization** replaces each 4-byte number with a single byte (a whole number from -128
to 127). That is a **4× memory cut** — 30 TB drops to ~7.5 TB — for a small accuracy loss.
**Binary quantization** goes further, keeping just 1 bit per number (is it positive or not?),
for a ~32× cut, at a bigger accuracy loss. Elasticsearch supports these directly (`int8_hnsw`,
`bbq` / binary quantization) and uses a trick called **rescoring**: do the fast search on the
tiny quantized vectors to get candidates, then re-check the top few against fuller-precision
vectors to recover most of the lost accuracy. At GlobalMart scale, **quantization is basically
mandatory** — you simply cannot afford 30 TB of RAM.

**4. Embedding staleness / model drift.** The world changes but old vectors do not. New slang,
new product categories, and new brands appear that the embedding model (trained at some past
date) never learned. Over time the model's understanding "drifts" away from current language.
*Mitigation:* periodically retrain/upgrade the model and re-embed (the expensive migration from
§8B.2).

**5. Chunking long text.** An embedding model can only read so much text at once, and cramming a
1,000-word description into one vector blurs its meaning into mush. For long text you **chunk**
it — split it into smaller passages, embed each, and store several vectors per document. For
GlobalMart this is usually mild (product titles are short), but long descriptions or reviews may
need chunking, which then raises the vector count and the memory bill.

**6. Domain mismatch.** A general-purpose embedding model was trained on the open web, not on
GlobalMart's catalog. It may not know that `128GB` and `256GB` are very different to a buyer, or
that two obscure industrial part numbers are unrelated. On a niche catalog, an off-the-shelf model
can be surprisingly bad. *Mitigation:* fine-tune / domain-adapt the model on your own data
(§8B.8).

**7. Cold start for brand-new products.** A brand-new listing gets embedded fine, but if your
vectors or rankers rely on behavioral signals (clicks, sales) those do not exist yet for a new
product. The embedding itself is available immediately, which actually *helps* cold start versus
behavior-only systems — but any learned reranker still has thin signal for new items.

**8. Two retrievals = more latency and cost.** Hybrid runs *both* a BM25 query and a kNN query,
plus the fusion, plus embedding the query at search time. That is more work than lexical alone,
eating into the 200 ms budget (brief §2). *Mitigation:* run the two retrievals in parallel, cache
query embeddings for head queries, and keep the reranker window small.

**9. "Relevant but wrong" results.** Semantic search returns things that *feel* related but are
commercially wrong — a similar product from the wrong brand, or an accessory instead of the item.
*Mitigation:* keep lexical in the mix (hybrid), and keep hard business rules as filters, never as
soft vector signals.

**10. Evaluation is hard.** It is genuinely difficult to tell whether hybrid beat lexical,
because semantic wins on some queries and loses on others, and human judgment of "is this
relevant?" is fuzzy. *Mitigation:* A/B test on real business metrics (conversion, CTR), not
vibes — tying back to Chapter 7's relevance engineering and Chapter 1's metrics.

---

## 8B.8 Best Practices

A clear checklist for doing hybrid search well.

**1. Start with strong lexical (BM25), add vectors where they measurably help.** BM25 works out
of the box, needs no model, and is already excellent for exact and SKU queries. Get it right
first. Then add vectors and *prove* they help on real queries — do not add vectors just because
they are fashionable.

**2. Use RRF as the default fusion.** No normalization, no hand-tuned weight, robust to score
outliers. Reach for weighted score combination only if you have a specific measured reason and
the appetite to maintain the tuning.

**3. Consider learned sparse retrieval (ELSER / SPLADE) as a middle ground.** There is a third
option between plain keywords and dense vectors.

> **Learned sparse retrieval** produces vectors that are *sparse* — mostly zeros, with weights on
> a handful of actual words plus a few *related* words the model thinks are relevant. It behaves
> like "smart keywords": it still matches on terms (so it stays precise and explainable like
> BM25), but the model *expands* the terms to include related ones (so it catches some synonyms
> like a semantic model). Elasticsearch ships one called **ELSER**; the research method is
> **SPLADE**.

Learned sparse retrieval is attractive because it uses the same inverted-index machinery as BM25
(no giant dense-vector memory bill), needs no separate embedding model at query time in the same
way, and often beats plain BM25 on relevance. It is a strong, cheaper-than-dense option, and it
can itself be one of the retrievers you fuse with RRF.

**4. Fine-tune / domain-adapt the embedding model** on your own catalog and query logs so it
learns *your* domain's meaning (that `128GB` ≠ `256GB` matters, that certain brands are distinct).
This fixes most domain-mismatch pain.

**5. Quantize vectors for scale.** At 10B docs, use int8 (or binary) quantization with rescoring.
The ~30 TB → ~7.5 TB (or less) memory saving is not optional at this size.

**6. Keep hard business filters as filters, never as soft signals.** Price, availability, market,
banned sellers — these are non-negotiable business rules. Put them in the `knn` `filter` (§8B.6)
and in the lexical `filter` clause. Never hope the vector "kind of understands" that out-of-stock
is bad. It does not.

**7. Do retrieve-then-rerank for the top results.** Cast the cheap wide net (BM25 + kNN, fused),
then apply a cross-encoder over the top ~100–200 for a big precision boost at a small, bounded
cost.

**8. A/B test with NDCG, click-through, and conversion.** Judge hybrid by outcomes, not opinions.
Use NDCG offline as a pre-filter and conversion/CTR online as the arbiter — exactly the loop from
Chapter 7 (§7.10) and the metrics from Chapter 1.

**9. Monitor recall and latency continuously.** Because ANN can fail silently (pitfall #2),
track ANN recall against an exact baseline on a sample, and watch p99 latency so the extra
retrieval does not blow the budget.

**10. Version embedding models like code.** Pin a model version, tie each index to the model that
built it, and roll a new model via the alias-and-versioned-index migration (Chapter 8). Never let
query-time and index-time model versions drift apart.

---

## 8B.9 At GlobalMart Scale

Let us connect all of this to the handbook's canonical system.

**The ~30 TB vector-memory reality (ties to Ch2 capacity, Ch9 scaling).** The brief pegs the
corpus at ~10B listings and ~24 TB of on-disk primaries, and describes the cluster as
**QPS-bound, not storage-bound** (brief §2). Vectors change that calculus. Raw float32 vectors
for 10B docs would be ~30 TB of *RAM* — bigger than the entire on-disk primary set and far more
expensive per terabyte. So at GlobalMart, dense vectors are only viable with **int8/binary
quantization** (~7.5 TB or less) plus rescoring, and the vector memory becomes a first-class
capacity input in Chapter 2's sizing and Chapter 9's node planning. In practice GlobalMart may
also apply vectors only where they earn their keep — for example on the head of the catalog or on
categories where "describe-it" queries are common — rather than embedding all 10B listings
blindly.

**Where hybrid slots into the search pipeline (ties to Ch7).** Chapter 7's pipeline had a
retrieval stage (candidate generation) feeding a ranking stage. Hybrid retrieval *replaces the
retrieval stage's inside*: instead of BM25 alone generating candidates, we run BM25 **and** kNN
and fuse them with RRF to build the candidate set. Everything upstream (Query Understanding,
which now also embeds the query) and everything downstream is unchanged in shape. The query
embedding step lives in the Search Service and must fit the 15 ms Query Understanding budget, so
head-query embeddings are cached.

**How it interacts with LTR reranking (ties to Ch7 §7.4).** This is the key ordering to state
clearly:

```
STAGE 1 — first-stage retrieval (this chapter)
   BM25  ─┐
          ├─ RRF fuse ─▶ ~500 candidates    (hybrid does candidate generation)
   kNN   ─┘

STAGE 2 — reranking (Chapter 7)
   optional cross-encoder rerank of top ~200
          ▼
   LTR / function_score with BUSINESS signals
   (popularity, seller_rating, price competitiveness, availability, freshness)
          ▼
   final top-24 returned
```

Hybrid search decides *which products are in the running* (recall). LTR and business-signal
ranking still run *after* it and decide the *final order* (precision + business goals). Hybrid
does **not** replace LTR — it feeds it better candidates. A common mistake is to think adding
vectors means throwing away the ranking model; in fact the ranking model matters *more*, because
now it must order a more diverse candidate set that mixes exact and semantic matches.

---

## 8B.10 Other Important Things to Know

A few extras worth having in your pocket.

**Cross-encoder rerankers (recap and when to use).** We met these in §8B.5. Use a cross-encoder
as the *last* retrieval-quality step, over just the top ~100–200 candidates, when relevance
really matters and you can afford ~10–30 ms extra. Never run it over the full match set — that is
what the cheap first stage is for. It is the natural "precision booster" sitting between hybrid
retrieval and business-signal LTR.

**Multimodal (image) search for products.** Product photos are a huge signal in e-commerce, and
the brief lists image search as an extension (brief §1, out of scope but "mention as
extension"). A multimodal model (§8B.2) embeds both product images and text queries into one
shared space, so a buyer can search by uploading a photo ("find me this dress") or a text query
can match against image content, not just the seller's text. It uses the exact same kNN
machinery — the vectors just came from images instead of, or in addition to, text.

**Multilingual embeddings for global locales.** GlobalMart spans many markets and locales (brief:
`locale`, `market`). A **multilingual embedding model** places text from different languages into
the *same* space, so a German query can match an English product and vice versa, and you do not
need a separate vector index per language. This complements Chapter 8's per-locale *lexical*
analyzers: lexical stays language-specific (German stemming, Japanese tokenization), while the
vector side can bridge languages. For truly global relevance you often want both.

**When NOT to bother with vectors.** Vectors are not always worth their cost. Skip them when:
- The **catalog is small** — with a few thousand products, plain BM25 plus a good synonym list is
  plenty, and the vector infrastructure is overkill.
- The queries are **pure keyword or SKU lookups** — part numbers, barcodes, exact model codes.
  Here semantic similarity actively *hurts* (it blurs the exact codes), and lexical is both
  cheaper and better.
- You have **no way to evaluate** the change. If you cannot A/B test, you cannot tell if vectors
  helped, and you will just be paying for RAM and complexity on faith.

The honest senior answer is: "Vectors are a tool for the *meaning-mismatch* problem. If your
users' problem is not meaning mismatch, do not reach for them."

---

## Interview Tips

- **Lead with *why* hybrid exists — the mirror-image weakness.** Say it in one breath: "Lexical
  (BM25) matches exact words and nails SKUs and brands but misses synonyms and intent; vectors
  match meaning and nail 'warm jacket for a toddler' but blur exact model numbers and can return
  commercially wrong items. Their strengths and weaknesses are mirror images, so you run both and
  fuse." This framing instantly signals you understand the trade-off, not just the buzzword.

- **Name the score-scale problem explicitly, then name RRF as the fix.** "BM25 is unbounded, cosine
  is 0–1, so you can't just add them." Then: "Reciprocal Rank Fusion combines the *ranks*, not the
  scores — `sum of 1/(k+rank)`, k usually 60 — which sidesteps the scale problem entirely and needs
  no tuning." Reciting the RRF formula and *why* rank-based fusion works is a strong senior signal.

- **Work the vector-memory number out loud.** "10B docs × 768 dims × 4 bytes ≈ 30 TB of RAM for
  raw vectors — that's more than our entire on-disk index, so quantization (int8 gives a 4× cut to
  ~7.5 TB, binary more) is mandatory at this scale." Interviewers love a candidate who reaches for
  the arithmetic and lands on quantization.

- **Explain HNSW simply and name the three knobs.** "A multi-layer graph you walk toward the query;
  top layers are highways, bottom layer is every node. M controls links (memory vs recall),
  ef_construction is build-time effort, ef_search is query-time effort (speed vs recall)." Then add
  that ANN can **fail silently** — low ef_search quietly drops good results with no error.

- **Say "retrieve-then-rerank" and distinguish bi- vs cross-encoder.** "Bi-encoder embeds query and
  doc separately, so it's cheap and precomputable for 10B docs; cross-encoder reads the pair
  together, far more accurate but too slow for more than the top ~200." That contrast is a reliable
  depth check.

- **Nail the filtering gotcha.** Post-filter can return too few results; pre-filter can wreck HNSW
  recall by cutting the graph; Elasticsearch applies the filter *during* the walk. Keep hard
  business filters (price, in-stock) as filters, never as soft vector signals.

- **Place hybrid correctly in the pipeline.** "Hybrid is *first-stage retrieval* — it feeds better
  candidates to the LTR/business-signal reranker, it does not replace it." Getting this ordering
  right shows you see the whole system (Ch7), not just the shiny part.

- **Common traps to avoid:** adding BM25 and cosine scores directly; forgetting query and documents
  must use the same embedding model version; assuming vectors replace the ranking model; running a
  cross-encoder over the full match set; storing raw float32 vectors at 10B scale; treating "in
  stock" as something the vector will "understand."

## Key Takeaways

- **Lexical and semantic search have mirror-image strengths.** Keyword/BM25 wins on exact words,
  SKUs, and brands; vectors win on synonyms, intent, and vague "describe-it" queries. Hybrid runs
  both so each covers the other's blind spot.

- **An embedding is a list of numbers that captures meaning.** Similar meaning → vectors that point
  the same way → high cosine similarity. Products are embedded offline at index time; the query is
  embedded at search time; both must use the **same model version**.

- **kNN finds the k closest vectors.** Exact kNN is too slow at 10B docs, so we use **ANN**
  (approximate) — mainly **HNSW**, a multi-layer graph you walk toward the query. Its knobs (M,
  ef_construction, ef_search) trade recall vs. speed vs. memory, and low settings fail *silently*.

- **BM25 scores and vector scores live on different scales**, so you cannot just add them. Every
  fusion method is an answer to that scale problem.

- **RRF is the default fusion:** combine on *ranks* (`sum of 1/(k+rank)`, k≈60), not scores — no
  normalization, no hand-tuned weights, robust to outliers. Elasticsearch exposes it via the RRF
  retriever; kNN via the `knn` block.

- **Retrieve-then-rerank** adds a slow, accurate **cross-encoder** over just the top ~100–200
  candidates for a precision boost, versus the fast **bi-encoder** used for first-stage retrieval.

- **Filtered kNN is a real gotcha:** post-filter returns too few, pre-filter wrecks recall;
  Elasticsearch filters *during* the graph walk. Keep hard business rules as filters.

- **At 10B docs, raw vectors ≈ 30 TB of RAM — quantization (int8/binary) is mandatory.** Other
  pitfalls: silent ANN recall loss, model drift/staleness, chunking, domain mismatch, cold start,
  extra latency, "relevant but wrong" results, and hard evaluation.

- **Best practices:** strong BM25 first; RRF by default; consider **learned sparse retrieval
  (ELSER/SPLADE)** as a cheaper middle ground; fine-tune the model to your domain; quantize; keep
  business filters as filters; rerank the top; A/B test on conversion/NDCG; monitor recall and
  latency; version models like code.

- **In the GlobalMart pipeline, hybrid is first-stage retrieval** — it generates candidates and
  feeds them to the LTR/business-signal reranker (Ch7), which still decides the final order. Vectors
  make the ranker matter *more*, not less. And when the problem is not meaning-mismatch (small
  catalogs, pure SKU lookups), do not reach for vectors at all.

---

*This chapter extends Chapter 7's retrieval stage and Chapter 8's scoring internals. Hybrid
retrieval sits inside the "candidate generation" box of the search pipeline; BM25 (Ch8) is one
retriever, vector kNN is the other, RRF fuses them, and the LTR reranker (Ch7) still has the final
word on order.*
