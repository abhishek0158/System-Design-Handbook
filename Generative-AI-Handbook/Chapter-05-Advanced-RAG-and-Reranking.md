# Chapter 5 — Advanced RAG & Re-ranking

In Chapter 4, we built a basic RAG (Retrieval-Augmented Generation) pipeline for the GlobalMart
Assistant. RAG means: find relevant text, then give it to the model as context before asking a
question. Our first version worked. But it has real weaknesses. This chapter fixes them.

*RAG* stands for Retrieval-Augmented Generation. The model does not just answer from memory. It
first retrieves relevant text, then generates an answer using that text.

## 1. Why Naive RAG Fails

The Chapter 4 pipeline did three things: split documents into chunks, embed each chunk as a
vector, and retrieve the top-k chunks by vector similarity. An *embedding* is a list of numbers
that captures the meaning of text. This works for simple questions. But it breaks in common
cases.

**Problem 1: Keyword-vs-meaning mismatch.** Vector search matches by meaning, not exact words.
If a user asks "What is SKU-4471's warranty?", the embedding model may not treat "SKU-4471" as
special. It is just one token among many. A document that repeats "SKU-4471" many times but uses
different wording elsewhere might rank lower than a document that "feels" similar in topic but
never mentions that exact SKU. Exact codes, IDs, and rare terms are where pure vector search is
weakest.

**Problem 2: Wrong chunks retrieved.** Chunking cuts documents into fixed-size pieces, often
without respect for meaning. A policy sentence can get split across two chunks. The retriever
then finds a chunk with half the answer, missing the other half.

**Problem 3: Missing context.** Even when the right chunk is retrieved, it might be too small to
make sense on its own. A chunk that says "In this case, the fee is waived" is useless without
the earlier sentence that defines "this case."

**Problem 4: No scoping.** Naive RAG searches the entire document collection every time. If the
user is asking about electronics returns, there is no reason to search furniture policies too.
Without filters, irrelevant chunks compete for the top-k slots.

**Problem 5: One-shot retrieval.** Naive RAG retrieves once, then answers. Some questions need
more than one lookup. "Is the manager of the person who approved order 5521 still employed?"
needs two facts from two different lookups, not one.

The rest of this chapter fixes each problem, one upgrade at a time. Table 5.1 gives an overview.

| Problem | Upgrade |
|---|---|
| Keyword vs. meaning mismatch | Hybrid search (§2) |
| Retriever ranks the wrong chunk highest | Re-ranking (§3) |
| User's question is vague or too narrow | Query rewriting / expansion (§4) |
| Search returns irrelevant categories | Metadata filtering (§5) |
| Retrieved chunk is too small for context | Small-to-big / parent-document retrieval (§6) |
| Question needs more than one fact | Multi-hop retrieval (§7) |

## 2. Hybrid Search: Keyword + Vector

*Hybrid search* combines two search methods and merges their results. One method is keyword
search. The other is vector search.

**Keyword search** (often done with **BM25**) ranks documents by exact word overlap, weighted by
how rare and how important each word is. BM25 is a scoring formula used by search engines like
Elasticsearch and OpenSearch. It is good at exact matches: product codes, names, numbers.

**Vector search** ranks chunks by meaning, using embeddings. It is good at paraphrases and
synonyms. It is weak at exact tokens like "SKU-4471".

Hybrid search runs both searches, then fuses the two ranked lists into one. A common fusion
method is **Reciprocal Rank Fusion (RRF)**. RRF gives each result a score based on its rank
position in each list (not its raw score, since BM25 and cosine-similarity scores are not on the
same scale). It adds the two rank-based scores together.

```python
from collections import defaultdict

def reciprocal_rank_fusion(ranked_lists: list[list[str]], k: int = 60) -> list[str]:
    """
    ranked_lists: each item is a list of document IDs, best first.
    k: a constant that softens the effect of very high ranks. 60 is a common default.
    Returns: a single fused ranking of document IDs, best first.
    """
    scores = defaultdict(float)
    for ranked_list in ranked_lists:
        for rank, doc_id in enumerate(ranked_list, start=1):
            scores[doc_id] += 1 / (k + rank)
    fused = sorted(scores.items(), key=lambda pair: pair[1], reverse=True)
    return [doc_id for doc_id, _ in fused]
```

Each document gets `1 / (k + rank)` from each list it appears in. A document ranked #1 in both
lists gets a high combined score. A document that only shows up in one list still gets counted,
but with a smaller score.

For a full deep-dive on how hybrid search works inside a real search engine — index structures,
score normalization, and production tuning — see the Search System Design Handbook, Chapter 8B
(*Hybrid Search Internals*). This chapter keeps things at the RAG-application level: how to call
both search types and merge the output.

```python
# BM25 needs a plain keyword index. rank_bm25 is a small, dependency-free library
# good for prototypes. For production, use Elasticsearch/OpenSearch's built-in BM25.
from rank_bm25 import BM25Okapi

class KeywordIndex:
    def __init__(self, chunks: list[dict]):
        self.chunks = chunks  # each chunk: {"id": ..., "text": ..., "metadata": {...}}
        tokenized = [c["text"].lower().split() for c in chunks]
        self.bm25 = BM25Okapi(tokenized)

    def search(self, query: str, top_k: int = 20) -> list[str]:
        scores = self.bm25.get_scores(query.lower().split())
        ranked = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)
        return [self.chunks[i]["id"] for i in ranked[:top_k]]
```

The vector side is the Chroma retriever from Chapter 4. Combining them:

```python
def hybrid_search(query: str, keyword_index: KeywordIndex, vector_collection, top_k: int = 20):
    keyword_ids = keyword_index.search(query, top_k=top_k)

    vector_results = vector_collection.query(query_texts=[query], n_results=top_k)
    vector_ids = vector_results["ids"][0]

    fused_ids = reciprocal_rank_fusion([keyword_ids, vector_ids])
    return fused_ids[:top_k]
```

Now "SKU-4471" gets found through the keyword path, and paraphrased questions still get found
through the vector path. Neither method has to be perfect alone.

## 3. Re-ranking: A Second, Smarter Look

Hybrid search gives us a wider, better set of candidates. But the fusion step is crude — it does
not deeply read each chunk against the question. **Re-ranking** adds a second pass: take the top
candidates (say, top 20) and use a more careful model to reorder them, then keep only the best
few (say, top 5).

To understand why this second pass helps, compare two model types:

- A **bi-encoder** (used for embeddings and vector search) turns the query and each document
  into a vector *separately*. It never lets the query and document "look at" each other while
  encoding. This is what makes it fast: you can pre-compute every document's vector once, store
  it, and only encode the query at search time. But this speed costs accuracy.
- A **cross-encoder** (used for re-ranking) takes the query and one document *together*, as a
  single input, and outputs one relevance score for that pair. It can compare specific words and
  phrases directly. This is far more accurate. But it is also slower — you cannot pre-compute
  anything, because the score depends on both texts at once. It does not scale to millions of
  documents.

The pattern that works: use the fast bi-encoder (embeddings + vector search) to narrow millions
of chunks down to a shortlist of 20–50. Then use the slow, accurate cross-encoder to re-rank just
that shortlist. You get both speed and accuracy.

```
[All chunks: 100,000+]
        │  bi-encoder (fast, approximate)
        ▼
[Candidates: top 20-50]
        │  cross-encoder re-ranker (slow, accurate)
        ▼
[Final: top 3-5]  →  sent to Claude as context
```

For re-ranking, use a dedicated re-rank model. Do not invent an Anthropic re-rank endpoint —
Claude does not have one. Voyage AI (the embeddings provider used in this handbook) also offers a
re-rank model called `rerank-2`:

```python
import voyageai

voyage_client = voyageai.Client()  # reads VOYAGE_API_KEY from env

def rerank_candidates(query: str, candidate_texts: list[str], top_k: int = 5) -> list[dict]:
    result = voyage_client.rerank(
        query=query,
        documents=candidate_texts,
        model="rerank-2",
        top_k=top_k,
    )
    # result.results is a list of objects with .index (position in candidate_texts)
    # and .relevance_score (higher is more relevant)
    return [
        {"text": candidate_texts[r.index], "score": r.relevance_score}
        for r in result.results
    ]
```

If you prefer a fully local, open-source option, `sentence-transformers` ships cross-encoder
models built for this exact job:

```python
from sentence_transformers import CrossEncoder

local_reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank_local(query: str, candidate_texts: list[str], top_k: int = 5) -> list[str]:
    pairs = [(query, text) for text in candidate_texts]
    scores = local_reranker.predict(pairs)
    ranked = sorted(zip(candidate_texts, scores), key=lambda pair: pair[1], reverse=True)
    return [text for text, _ in ranked[:top_k]]
```

Re-ranking typically gives the biggest single quality jump of any upgrade in this chapter,
because it directly fixes "the right chunk was retrieved, but ranked too low to make the
top-k cut."

## 4. Query Rewriting and Expansion

Users type short, messy questions. "return policy electronics broke" is a real question a user
might type. This is a weak search query: no clear subject, no context. **Query rewriting** uses
an LLM call to clean up or expand the question before it is used for search.

Two common patterns:

1. **Rewrite for clarity.** Turn a vague or shorthand question into a clear, complete one.
2. **Expand with related terms.** Add synonyms or related phrases so keyword search has more
   chances to match.

```python
import anthropic

client = anthropic.Anthropic()

def rewrite_query(raw_query: str) -> str:
    resp = client.messages.create(
        model="claude-haiku-4-5",  # cheap and fast — this is a simple sub-task
        max_tokens=200,
        system=(
            "Rewrite the user's question into a clear, complete search query. "
            "Fix typos. Expand abbreviations. Add likely related terms in "
            "parentheses. Return only the rewritten query, nothing else."
        ),
        messages=[{"role": "user", "content": raw_query}],
    )
    for block in resp.content:
        if block.type == "text":
            return block.text.strip()
    return raw_query  # fallback: use the original if something went wrong
```

Example: `"return policy electronics broke"` might become `"What is the return policy for a
broken or defective electronics product? (warranty, refund, replacement, damaged item)"`. This
gives both the vector search and the BM25 search much more to work with.

Use a cheap, fast model like `claude-haiku-4-5` for this step. It runs on every single query, so
cost and latency matter. Do not use `claude-opus-4-8` here — it is overkill for rewriting a
sentence, and it slows down every request.

One more variant: **multi-query expansion**. Ask the model to generate 2–3 different phrasings of
the same question, run retrieval for each, then merge the results (with RRF, from §2). This
covers more ground when one rewrite is not enough.

## 5. Metadata Filtering

*Metadata* is structured information attached to each chunk: category, product line, date,
document type, region. **Metadata filtering** restricts search to chunks matching specific field
values, before or during the similarity search. This scopes retrieval and removes irrelevant
categories entirely — it does not rely on the model to notice they are irrelevant.

Store metadata alongside each chunk when you index it:

```python
collection.add(
    ids=["doc_42_chunk_3"],
    documents=["Electronics returns must be initiated within 30 days of delivery..."],
    metadatas=[{
        "category": "electronics",
        "doc_type": "return_policy",
        "region": "US",
        "last_updated": "2026-06-01",
    }],
)
```

Then filter at query time. Chroma supports a `where` clause for this:

```python
def filtered_search(query: str, collection, category: str, top_k: int = 10):
    return collection.query(
        query_texts=[query],
        n_results=top_k,
        where={"category": category},   # only search chunks tagged "electronics"
    )
```

Where do filter values come from? Sometimes the user picks them (a category dropdown in the UI).
Sometimes the LLM extracts them from the question. You can combine this with query rewriting: ask
the model to also output a structured filter alongside the rewritten query.

```python
from pydantic import BaseModel
from typing import Optional

class SearchPlan(BaseModel):
    rewritten_query: str
    category: Optional[str] = None  # e.g. "electronics", "furniture", or None if unclear

def plan_search(raw_query: str) -> SearchPlan:
    resp = client.messages.parse(
        model="claude-haiku-4-5",
        max_tokens=300,
        system="Extract a clean search query and a product category filter, if one is implied.",
        messages=[{"role": "user", "content": raw_query}],
        output_format=SearchPlan,
    )
    return resp.parsed_output
```

Metadata filtering is cheap and precise. Always apply it when you have a reliable filter value.
It removes whole categories of noise before ranking even starts.

## 6. Better Chunking: Small-to-Big (Parent-Document) Retrieval

Chapter 4 used fixed-size chunking: cut every N tokens. This causes Problem 3 from §1 — chunks
too small to make sense alone, or split mid-sentence.

**Small-to-big retrieval** (also called **parent-document retrieval**) fixes this with a simple
idea: index small chunks for accurate search, but hand the model a bigger surrounding block of
text for actual generation. Small chunks make search precise, because they match narrow, specific
meaning. Big blocks make generation good, because they carry full context.

```
Document
├── Parent block 1 (800 tokens) ──┬── Small chunk 1a (150 tokens)  ← indexed & searched
│                                  ├── Small chunk 1b (150 tokens)  ← indexed & searched
│                                  └── Small chunk 1c (150 tokens)  ← indexed & searched
└── Parent block 2 (800 tokens) ──┬── Small chunk 2a (150 tokens)
                                   └── Small chunk 2b (150 tokens)
```

Each small chunk stores a pointer to its parent block. Search finds the best small chunk. Then
you look up its parent and send the parent — not the small chunk — to Claude.

```python
def build_small_to_big_index(document_text: str, parent_size: int = 800, child_size: int = 150):
    words = document_text.split()
    parents = [
        " ".join(words[i : i + parent_size])
        for i in range(0, len(words), parent_size)
    ]

    child_chunks, child_metadatas = [], []
    for parent_id, parent_text in enumerate(parents):
        parent_words = parent_text.split()
        for j in range(0, len(parent_words), child_size):
            child_text = " ".join(parent_words[j : j + child_size])
            child_chunks.append(child_text)
            child_metadatas.append({"parent_id": parent_id})

    return parents, child_chunks, child_metadatas

def retrieve_with_parents(query: str, collection, parents: list[str], top_k: int = 5) -> list[str]:
    results = collection.query(query_texts=[query], n_results=top_k)
    parent_ids = {meta["parent_id"] for meta in results["metadatas"][0]}
    return [parents[pid] for pid in parent_ids]
```

The model now sees full paragraphs, with the sentence that defines "this case" included, instead
of an isolated fragment.

## 7. Multi-Hop Retrieval

Some questions cannot be answered from one retrieval. **Multi-hop retrieval** breaks a question
into steps, where each step's answer feeds the next search.

Example: "Did the customer who filed ticket #8891 also place a return last month?" This needs two
facts: who filed ticket #8891, and that customer's return history. One vector search over policy
documents will not find either — this needs an agent-style loop, not a single RAG call.

A simple pattern: let the model decide, one step at a time, whether it has enough information or
needs another search.

```python
def multi_hop_answer(question: str, search_fn, max_hops: int = 3) -> str:
    known_facts = []
    for hop in range(max_hops):
        resp = client.messages.create(
            model="claude-sonnet-5",
            max_tokens=500,
            system=(
                "You answer questions using search. Given the question and facts "
                "found so far, either say 'ENOUGH' followed by the final answer, "
                "or say 'SEARCH' followed by one specific search query for the "
                "next missing fact."
            ),
            messages=[{
                "role": "user",
                "content": f"Question: {question}\nFacts so far: {known_facts}",
            }],
        )
        text = "".join(b.text for b in resp.content if b.type == "text")

        if text.startswith("ENOUGH"):
            return text.removeprefix("ENOUGH").strip()

        next_query = text.removeprefix("SEARCH").strip()
        found = search_fn(next_query)
        known_facts.append({"query": next_query, "result": found})

    return "Could not find a complete answer within the hop limit."
```

This is a small step toward the agent patterns in Chapter 7. For now, note the key idea: retrieval
does not have to be one shot. It can loop, with the model deciding what to search for next.

## 8. Hands-On: Upgrading the GlobalMart RAG Pipeline

Let's put the pieces together. We upgrade the Chapter 4 GlobalMart RAG pipeline with hybrid
search, re-ranking, and metadata filtering — the three upgrades that give the best return for a
Q&A assistant like this one.

```python
import anthropic
import voyageai
from rank_bm25 import BM25Okapi

anthropic_client = anthropic.Anthropic()
voyage_client = voyageai.Client()


class AdvancedGlobalMartRAG:
    def __init__(self, chunks: list[dict], vector_collection):
        """
        chunks: list of {"id", "text", "metadata": {"category": ..., "doc_type": ...}}
        vector_collection: a Chroma collection already populated with these chunks
        """
        self.chunks_by_id = {c["id"]: c for c in chunks}
        self.vector_collection = vector_collection
        tokenized = [c["text"].lower().split() for c in chunks]
        self.bm25 = BM25Okapi(tokenized)
        self.chunk_order = [c["id"] for c in chunks]

    def _rewrite_query(self, raw_query: str) -> str:
        resp = anthropic_client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=150,
            system="Rewrite this into a clear, complete search query. Return only the query.",
            messages=[{"role": "user", "content": raw_query}],
        )
        for block in resp.content:
            if block.type == "text":
                return block.text.strip()
        return raw_query

    def _keyword_search(self, query: str, top_k: int) -> list[str]:
        scores = self.bm25.get_scores(query.lower().split())
        ranked = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)
        return [self.chunk_order[i] for i in ranked[:top_k]]

    def _vector_search(self, query: str, top_k: int, category: str | None) -> list[str]:
        where = {"category": category} if category else None
        results = self.vector_collection.query(
            query_texts=[query], n_results=top_k, where=where,
        )
        return results["ids"][0]

    def _fuse(self, list_a: list[str], list_b: list[str], k: int = 60) -> list[str]:
        scores = {}
        for ranked_list in (list_a, list_b):
            for rank, doc_id in enumerate(ranked_list, start=1):
                scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
        return [doc_id for doc_id, _ in sorted(scores.items(), key=lambda p: p[1], reverse=True)]

    def _rerank(self, query: str, candidate_ids: list[str], top_k: int) -> list[str]:
        texts = [self.chunks_by_id[cid]["text"] for cid in candidate_ids]
        result = voyage_client.rerank(query=query, documents=texts, model="rerank-2", top_k=top_k)
        return [candidate_ids[r.index] for r in result.results]

    def retrieve(self, raw_query: str, category: str | None = None, top_k: int = 4) -> list[dict]:
        query = self._rewrite_query(raw_query)

        keyword_ids = self._keyword_search(query, top_k=20)
        vector_ids = self._vector_search(query, top_k=20, category=category)
        fused_ids = self._fuse(keyword_ids, vector_ids)[:20]

        reranked_ids = self._rerank(query, fused_ids, top_k=top_k)
        return [self.chunks_by_id[cid] for cid in reranked_ids]

    def answer(self, raw_query: str, category: str | None = None) -> str:
        top_chunks = self.retrieve(raw_query, category=category)
        context = "\n\n".join(f"[Source {i+1}] {c['text']}" for i, c in enumerate(top_chunks))

        resp = anthropic_client.messages.create(
            model="claude-opus-4-8",
            max_tokens=800,
            system=(
                "Answer using only the sources below. Cite sources like [Source 1]. "
                "If the sources do not contain the answer, say so.\n\n" + context
            ),
            messages=[{"role": "user", "content": raw_query}],
        )
        return "".join(b.text for b in resp.content if b.type == "text")
```

**Quality comparison.** With the Chapter 4 naive pipeline, the question "warranty on SKU-4471"
often missed the right chunk: the embedding model treated the SKU code as noise, and the top-5
vector results were topically close but factually wrong. With the upgraded pipeline:

- Hybrid search brings back the SKU-specific chunk through the BM25 path, even though its vector
  similarity was mediocre.
- Metadata filtering (`category="electronics"`) removes competing chunks from other product
  lines before ranking starts.
- Re-ranking pushes the exact-match chunk to position 1, ahead of chunks that were only
  topically similar.

The net effect: the right chunk reaches the top of the final list far more often. This is the
single biggest lever for RAG answer quality — better context in, better answers out. Chapter 6
shows how to measure this improvement with numbers, instead of eyeballing examples.

## What You Built / Learned

- Why naive vector-only RAG fails: keyword mismatch, wrong chunk ranked highest, missing
  surrounding context, no scoping, and one-shot retrieval limits.
- Hybrid search: combine BM25 keyword search and vector search, fuse results with Reciprocal
  Rank Fusion.
- Re-ranking: bi-encoders retrieve fast at scale; cross-encoders re-score a small shortlist
  accurately. Use a re-rank model (Voyage `rerank-2` or a local cross-encoder) for the second
  pass — never an Anthropic rerank endpoint, since one does not exist.
- Query rewriting and expansion: use a cheap model (`claude-haiku-4-5`) to clean up and expand
  vague user questions before searching.
- Metadata filtering: scope search to the right category, date range, or document type using
  structured fields stored with each chunk.
- Small-to-big (parent-document) retrieval: index small chunks for precise search, but return
  the larger parent block for full context.
- Multi-hop retrieval: some questions need more than one search; let the model decide when to
  search again.
- Combined all of the above into an upgraded `AdvancedGlobalMartRAG` class, replacing the
  Chapter 4 naive pipeline.

## Production Notes & Pitfalls

- **Order of operations matters.** Filter first (cheap, exact), then hybrid-fuse a wide
  candidate set (recall), then re-rank down to a few (precision). Doing re-ranking on too many
  candidates is slow and expensive; doing it on too few loses recall.
- **Re-ranking cost adds up.** A cross-encoder call scores every (query, candidate) pair. At
  high query volume, re-ranking 20 candidates per query is real, measurable cost and latency.
  Keep the pre-rerank candidate list as small as recall allows — do not rerank 200 chunks "to be
  safe."
- **Query rewriting can drift from user intent.** An LLM rewrite occasionally adds a wrong
  assumption (for example, turning "return" into a policy question when the user meant a
  product return, an object, not the word). Log the raw query alongside the rewritten one so you
  can debug this later.
- **Metadata is only as good as your tagging pipeline.** If category tags are inconsistent or
  missing at ingestion time, filtering silently drops relevant chunks with no error. Treat
  metadata extraction as seriously as chunking.
- **RRF's constant `k` and top-k sizes are tuning knobs.** Defaults (like `k=60`) work
  reasonably out of the box, but always validate against your own evaluation set (Chapter 6)
  rather than assuming defaults transfer.
- **Multi-hop retrieval can loop or hallucinate a stopping point.** Always cap the number of
  hops (`max_hops`) and log every intermediate search — silent infinite loops are a common bug
  in early agentic RAG code.
- **Keep the simplest thing that works.** Not every query needs hybrid search, re-ranking, and
  multi-hop. Simple factual questions often do fine with plain vector search. Add each upgrade
  only where evaluation (Chapter 6) shows it helps — more moving parts means more to monitor and
  more that can break in production.
