# Chapter 4 — RAG Fundamentals

## 4.1 The problem: Claude does not know your documents

Claude is trained on a large, general set of text. It does not know GlobalMart's return
policy. It does not know GlobalMart's shipping rules for electronics, or the refund window
for marketplace sellers. These documents were written after training, or they are private.
Claude has never seen them.

There is a simple first idea: paste the whole document into the prompt. Ask the question
after it. This works for one short document. It breaks fast in the real world, for three
reasons.

- **Size.** GlobalMart has hundreds of policy documents. They do not fit in one prompt,
  even with a large context window.
- **Cost.** Every token you send costs money and adds latency. Sending 200 pages to answer
  one question about a return window is wasteful.
- **Noise.** When you stuff in everything, Claude must search through mostly irrelevant
  text to find the one paragraph that matters. Answer quality drops. This is sometimes
  called "needle in a haystack" — the needle is the right paragraph, the haystack is
  everything else you sent.

We need a way to find only the few paragraphs that answer the question, and send just
those. This is what **RAG** does.

## 4.2 What RAG is, in plain words

**RAG** stands for **Retrieval-Augmented Generation**. It has three steps, in plain words:

1. **Retrieve** — search your own documents for the small number of passages that are
   likely to answer the question.
2. **Augment** — add those passages to the prompt, next to the user's question.
3. **Generate** — ask Claude to answer the question, using the passages as its source.

That's it. RAG does not change Claude. It changes what you put in the prompt. You are
giving Claude a short, relevant "open book" instead of asking it to work from memory, or
forcing it to read the entire library.

This matters for the GlobalMart Assistant (our running example through this book). When a
customer asks "What is the return policy for electronics?", the assistant should not
guess. It should look up the real policy document, find the electronics return section,
and answer from that text.

The hard engineering question in RAG is: **how do we find the right passages, fast, out of
thousands of documents?** The answer uses two ideas: embeddings, and a vector database.

RAG is not the only way to reduce a large document set to something a prompt can hold.
Some teams use plain keyword search (matching exact words, like a search engine from
before AI). Keyword search is fast and easy to debug, but it fails when the customer's
words do not match the document's words — "broken" versus "damaged," "return" versus
"send back." RAG's embedding-based search, described next, is built to catch exactly this
kind of mismatch, which is why it is the default approach in this handbook. Chapter 5
shows that the strongest systems often combine both: keyword search for exact terms, and
embedding search for meaning, in what is called hybrid search.

## 4.3 Embeddings: turning text into numbers that capture meaning

An **embedding** is a list of numbers that captures the meaning of a piece of text. A
sentence like "How do I return a broken laptop?" gets turned into a list of, say, 1024
numbers. Sentences with similar meaning get similar lists of numbers, even if they use
different words.

This is the key trick. "How do I return a broken laptop?" and "What is the process for
sending back a damaged computer?" use almost no shared words. But they mean nearly the same
thing. A good embedding model puts their number-lists close together. "What is the shipping
cost to Canada?" means something different, so its number-list lands far away.

We measure "how close" two embeddings are with **cosine similarity**. This is a number
between -1 and 1 that says how similar two directions are, ignoring length. A score near 1
means very similar meaning. A score near 0 means unrelated. Here is the idea in code,
before we bring in any AI model:

```python
import numpy as np

def cosine_similarity(vec_a: list[float], vec_b: list[float]) -> float:
    """Return a score from -1 to 1. Higher means more similar meaning."""
    a, b = np.array(vec_a), np.array(vec_b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

Now let's create real embeddings. **The Claude API does not create embeddings.** Claude is
a chat and reasoning model, called through `client.messages.create(...)`. There is no
`client.embeddings...` method — that call does not exist on the Anthropic client. For
embeddings, we use a separate, dedicated embeddings model. This handbook's default is
**Voyage AI** (`voyageai` package, model `voyage-3`), which Anthropic recommends for use
with Claude. If you want a free, local option with no API calls, use the open-source
`sentence-transformers` library instead — we show both.

```python
import os
import voyageai

# Reads the VOYAGE_API_KEY environment variable.
voyage = voyageai.Client()

texts = [
    "How do I return a broken laptop?",
    "What is the process for sending back a damaged computer?",
    "What is the shipping cost to Canada?",
]

# input_type="document" tells the model these are texts we will search over later.
result = voyage.embed(texts, model="voyage-3", input_type="document")
embeddings = result.embeddings  # a list of three number-lists (vectors)

print(cosine_similarity(embeddings[0], embeddings[1]))  # high — same meaning
print(cosine_similarity(embeddings[0], embeddings[2]))  # low — different topic
```

If you run this, the first pair scores much higher than the second. That is the whole idea
of semantic search: compare meaning, not exact words.

**How big is an embedding?** The number of values in the list is called its
**dimension**. `voyage-3` produces 1024 numbers per piece of text, no matter how long the
text is — one sentence and one paragraph both become a single 1024-number list. Other
embedding models use different dimensions (some are 384, some are 1536, some more). This
number is fixed per model. You cannot compare an embedding from one model against an
embedding from a different model — they do not share a coordinate system, even if the
lists happen to be the same length.

**Local alternative.** If you cannot call an external embeddings API, `sentence-transformers`
runs a small model on your own machine, for free:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")
embeddings = model.encode(texts).tolist()
```

The rest of this chapter uses Voyage, because it is the default in this handbook's stack.
Swap in `sentence-transformers` anywhere you see `voyage.embed(...)` if you want a local,
free setup.

## 4.4 The vector database: where embeddings live

An embedding by itself is not useful. You need a place to store thousands of them, and a
fast way to ask: "which stored embeddings are closest to this new one?" That is what a
**vector database** does. A vector database stores embeddings (called **vectors**) plus the
original text and any extra information (called **metadata**, such as the document name).
It can search millions of vectors for the closest matches in milliseconds.

This handbook's default vector database is **Chroma** (`chromadb` package). Chroma is open
source, and it can run fully in your own process — no separate server needed for
development. For production systems with heavy traffic or very large document sets,
**pgvector** (a Postgres extension) is a common choice, because it lets you keep vectors in
the same database as the rest of your application data. We use Chroma in this chapter's
code; the ideas carry over directly to pgvector.

A vector database entry usually has four parts:

| Part | Example |
|---|---|
| `id` | `"electronics-returns-chunk-2"` |
| `embedding` | `[0.014, -0.221, 0.083, ...]` (1024 numbers) |
| `document` (the text) | "Electronics may be returned within 30 days if unopened..." |
| `metadata` | `{"source": "electronics_returns.md", "chunk": 2}` |

When you search, you give the database a query embedding. It returns the `k` closest
stored entries — this is called **top-k retrieval**. "Top-5" means "give me the 5 closest
matches."

"Closest" needs a distance rule. Cosine similarity, from §4.3, is one option. Another
common one is **L2 distance** (also called Euclidean distance) — plain straight-line
distance between two points, treating each embedding as a point in space. For most text
embedding models, including Voyage's, cosine similarity is the right default, because it
ignores the length of the vector and looks only at its direction. Chroma lets you set the
distance function per collection; unless you have a specific reason to change it, leave it
at the default recommended by your embedding model's documentation.

## 4.5 Chunking: why we split documents before storing them

We do not store a whole 10-page policy document as one embedding. We split it into smaller
pieces first. This is called **chunking**. Each small piece is a **chunk**.

**Why chunk at all?**

- A single embedding is a rough summary of everything in the text it covers. A 10-page
  document mixes many topics — electronics returns, clothing returns, refund timing,
  seller responsibilities. One embedding for all of it is too blurry to match a specific
  question well.
- Claude's prompt has a limited useful size, and we pay for every token we send. We want to
  send only the 3–5 paragraphs that matter, not the whole document.
- Smaller chunks give more precise retrieval: a chunk about "electronics returns" can be
  found and ranked separately from a chunk about "clothing returns," even though they sit
  in the same file.

**Chunk size and overlap.** A common starting point is 300–800 tokens per chunk. Too small
(one sentence) and each chunk loses context — "this applies only within the state where
the item was purchased" means nothing without the sentence before it. Too large (a whole
document) and we are back to the blurry-embedding problem.

We also add **overlap** — a small shared slice of text between one chunk and the next
(commonly 10–20% of the chunk size). Without overlap, a sentence that explains an important
rule can get cut in half, split across the end of one chunk and the start of the next, so
neither chunk contains the full idea. Overlap makes it more likely that any single idea is
fully contained in at least one chunk.

**Common chunking mistakes:**

- **Splitting mid-sentence or mid-table** by a fixed character count. This cuts meaning in
  half. Prefer splitting on paragraph or section breaks first, and only fall back to a hard
  cut if a paragraph is unusually long.
- **No overlap at all.** Small, easy fix — always keep at least some overlap.
- **Chunking everything the same way.** A FAQ document (many short question-answer pairs)
  should probably be chunked one Q&A per chunk, not by a fixed token count that mixes two
  unrelated questions into one chunk.
- **Losing the source.** Always store, in metadata, which document and section a chunk came
  from. You will need this for citations (§4.8) and for debugging bad answers.
- **Ignoring headers and structure.** A section heading like "Electronics Return Policy"
  carries meaning that helps retrieval. If your chunker throws away headings and keeps only
  body text, later chunks from the same section lose that context. A simple fix: prepend
  the nearest heading to every chunk taken from under it.

Here is a simple, paragraph-aware chunker with overlap:

```python
def chunk_text(text: str, max_chars: int = 800, overlap_chars: int = 150) -> list[str]:
    """Split text into overlapping chunks, preferring paragraph breaks."""
    paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]
    chunks: list[str] = []
    current = ""

    for para in paragraphs:
        if len(current) + len(para) + 1 <= max_chars:
            current = f"{current}\n\n{para}".strip()
        else:
            if current:
                chunks.append(current)
            # Start the next chunk with the tail of the previous one (the overlap).
            current = (current[-overlap_chars:] + "\n\n" + para).strip()

    if current:
        chunks.append(current)
    return chunks
```

This is intentionally simple. Real systems often chunk by tokens (using the model's own
tokenizer) instead of characters, and may use smarter splitters for tables, code, or
Markdown headers. The idea stays the same: keep related sentences together, and keep a
small bridge between neighboring chunks.

## 4.6 Two phases of a RAG system

Every RAG system has two separate phases that run at different times.

**Phase 1: Offline indexing.** This runs ahead of time, whenever documents are added or
changed. It does not involve the user's question at all.

```
Load documents → Split into chunks → Embed each chunk → Store in vector database
```

**Phase 2: Query time.** This runs for every user question, in real time.

```
User question → Embed the question → Retrieve top-k closest chunks
              → Put chunks + question into a prompt → Claude generates the answer
```

Here is the whole system as one diagram:

```
                    OFFLINE INDEXING (run once, or on document update)
   ┌───────────┐   ┌──────────┐   ┌───────────┐   ┌────────────────────┐
   │ Documents │──▶│ Chunking │──▶│ Embedding │──▶│  Vector database    │
   │ (.md/.txt)│   │          │   │ (Voyage)  │   │  (Chroma)           │
   └───────────┘   └──────────┘   └───────────┘   └──────────┬─────────┘
                                                              │ stored
                                                              ▼
                        QUERY TIME (run per user question)
   ┌───────────┐   ┌───────────┐   ┌──────────────┐   ┌─────────────┐
   │ Question  │──▶│ Embedding │──▶│  top-k search │──▶│ Prompt with │
   │           │   │ (Voyage)  │   │  in Chroma    │   │ chunks +    │
   └───────────┘   └───────────┘   └──────────────┘   │ question    │
                                                        └──────┬──────┘
                                                               ▼
                                                        ┌─────────────┐
                                                        │   Claude    │
                                                        │  generates  │
                                                        │  the answer │
                                                        └─────────────┘
```

Notice: embedding happens twice — once for every chunk (offline), and once for every
question (at query time). Both must use the **same embedding model**. If you embed
documents with `voyage-3` and questions with a different model, the number-lists live in
different "spaces," and similarity scores become meaningless.

## 4.7 Building a naive RAG system for GlobalMart

Let's build this end to end. We will index a few GlobalMart policy documents, then answer
questions about them. This is a **naive RAG** system — the simplest version that works. It
retrieves once, with no re-ranking or query rewriting. Chapter 5 improves on it.

First, some sample policy documents. In a real system these would be loaded from files or a
document store; here we keep them as strings so the example is self-contained.

```python
policy_documents = {
    "electronics_returns.md": """
Electronics Return Policy

GlobalMart accepts returns of electronics within 30 days of delivery. The item
must be unopened, or opened only to verify it works, with all original packaging.

Laptops and phones follow a shorter 15-day window if the box was opened, because
of resale value loss. Unopened laptops and phones keep the full 30-day window.

Refunds for electronics are issued to the original payment method within 5 to 7
business days after GlobalMart receives the returned item.
""",
    "clothing_returns.md": """
Clothing and Apparel Return Policy

Clothing may be returned within 60 days of delivery, as long as tags are still
attached and the item has not been worn, except for trying it on.

Swimwear and underwear cannot be returned once the hygiene seal is removed.

Refunds for clothing are issued as store credit by default. Customers may
request a refund to the original payment method instead, at checkout of the
return.
""",
    "shipping_policy.md": """
Shipping Policy

Standard shipping within the country takes 3 to 5 business days. Express
shipping takes 1 to 2 business days and costs an extra fee shown at checkout.

International shipping to Canada and Mexico takes 7 to 10 business days.
GlobalMart does not currently ship electronics internationally, due to
customs restrictions on lithium batteries.
""",
}
```

Now the offline indexing pipeline: chunk each document, embed the chunks, and store them in
Chroma.

```python
import chromadb
import voyageai

voyage = voyageai.Client()

# A persistent Chroma client writes to disk, so the index survives a restart.
chroma_client = chromadb.PersistentClient(path="./globalmart_chroma_db")
collection = chroma_client.get_or_create_collection(name="globalmart_policies")


def index_documents(documents: dict[str, str]) -> None:
    ids, texts, metadatas = [], [], []

    for source_name, full_text in documents.items():
        for i, chunk in enumerate(chunk_text(full_text)):
            ids.append(f"{source_name}-chunk-{i}")
            texts.append(chunk)
            metadatas.append({"source": source_name, "chunk_index": i})

    # Embed every chunk in one batch call — cheaper and faster than one call per chunk.
    embeddings = voyage.embed(texts, model="voyage-3", input_type="document").embeddings

    collection.add(ids=ids, embeddings=embeddings, documents=texts, metadatas=metadatas)
    print(f"Indexed {len(texts)} chunks from {len(documents)} documents.")


index_documents(policy_documents)
```

Run this once. It builds the index. Now the query-time pipeline: embed the question,
retrieve the closest chunks, build a prompt, and call Claude.

```python
import anthropic

claude = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment


def retrieve(question: str, top_k: int = 3) -> list[dict]:
    query_embedding = voyage.embed(
        [question], model="voyage-3", input_type="query"
    ).embeddings[0]

    results = collection.query(query_embeddings=[query_embedding], n_results=top_k)

    # Chroma returns parallel lists, one entry per matched chunk.
    return [
        {"text": doc, "source": meta["source"], "chunk_index": meta["chunk_index"]}
        for doc, meta in zip(results["documents"][0], results["metadatas"][0])
    ]


def answer_question(question: str) -> str:
    chunks = retrieve(question)

    context = "\n\n".join(
        f"[Source: {c['source']}, chunk {c['chunk_index']}]\n{c['text']}" for c in chunks
    )

    system_prompt = (
        "You are the GlobalMart support assistant. Answer the question using ONLY "
        "the policy text given below. If the text does not contain the answer, say "
        "you do not know. Do not guess.\n\n"
        f"POLICY TEXT:\n{context}"
    )

    response = claude.messages.create(
        model="claude-opus-4-8",
        max_tokens=500,
        system=system_prompt,
        messages=[{"role": "user", "content": question}],
    )

    return "".join(block.text for block in response.content if block.type == "text")


print(answer_question("What is the return policy for electronics?"))
```

Notice two details in `retrieve`. First, we set `input_type="query"` for the question, but
`input_type="document"` when we indexed the chunks. Voyage's models use this hint to embed
questions and passages slightly differently, which improves matching — this is specific to
how Voyage's models were trained, so always check the embedding provider's docs for a
similar setting. Second, notice there is no `temperature` on the Claude call. Current Claude
models do not accept `temperature`, `top_p`, or `top_k` — steer behavior with the prompt
instead, as we did with "Answer using ONLY the policy text."

## 4.8 Adding simple citations

A support assistant that states a policy without saying where it came from is hard to
trust, and hard to audit later. We already have the source and chunk number in metadata
from indexing. We just need to (1) label each chunk when we build the prompt, and (2) ask
Claude to reference the labels in its answer.

```python
def answer_with_citations(question: str) -> dict:
    chunks = retrieve(question)

    # Give each retrieved chunk a short label like [1], [2], [3].
    labeled_context = "\n\n".join(
        f"[{i + 1}] (source: {c['source']})\n{c['text']}" for i, c in enumerate(chunks)
    )

    system_prompt = (
        "You are the GlobalMart support assistant. Answer using ONLY the numbered "
        "policy excerpts below. After each claim, add the excerpt number in brackets, "
        "like this: 'Electronics can be returned within 30 days [1].' If the excerpts "
        "do not answer the question, say you do not know.\n\n"
        f"POLICY EXCERPTS:\n{labeled_context}"
    )

    response = claude.messages.create(
        model="claude-opus-4-8",
        max_tokens=500,
        system=system_prompt,
        messages=[{"role": "user", "content": question}],
    )

    answer_text = "".join(block.text for block in response.content if block.type == "text")

    # Map [1], [2], [3] back to the real source document, so the UI can show real links.
    sources = {i + 1: c["source"] for i, c in enumerate(chunks)}

    return {"answer": answer_text, "sources": sources}


result = answer_with_citations("Can I get a refund to my card for returned clothing?")
print(result["answer"])
print(result["sources"])  # e.g. {1: "clothing_returns.md", 2: "shipping_policy.md"}
```

This is a simple citation style, and it is not perfect — Claude might attach the wrong
bracket number to a claim, especially with longer excerpts. Chapter 6 (Evaluating RAG)
covers how to measure whether citations are actually accurate, not just present.

## 4.9 Why this naive setup has weaknesses

The system above works, and it is a completely valid starting point. But it has real
weaknesses that will show up as GlobalMart's document set grows:

- **Keyword mismatches.** Embedding similarity is not perfect. A question using an exact
  product code or an exact legal term sometimes matches worse than plain keyword search
  would. Naive RAG uses only semantic search, with no keyword fallback.
- **No re-ranking.** We treat the top-k results from the vector database as final. A
  smarter second pass could re-score them for the specific question, and reorder or drop
  weak matches.
- **No query rewriting.** A short, vague question ("what about returns?") embeds poorly.
  A better system rewrites it first ("what is GlobalMart's return policy?") before
  searching.
- **No metadata filtering.** If a customer clearly asks about electronics, we still search
  across all documents, including clothing and shipping. Filtering by metadata
  (`source == "electronics_returns.md"`) before or during search would narrow this.
- **Fixed top-k.** We always retrieve exactly 3 chunks. Some questions need more context,
  some need less.

**Chapter 5 (Advanced RAG & Re-ranking)** fixes these one at a time: combining semantic
search with keyword search (hybrid search), adding a re-ranking step, rewriting queries
before retrieval, and filtering by metadata. **Chapter 6 (Evaluating RAG)** then shows how
to measure whether any of these changes actually made answers better, instead of guessing.

## What You Built / Learned

- Why stuffing whole documents into a prompt fails: size limits, cost, and noise that hurts
  answer quality.
- What RAG is: retrieve relevant passages, augment the prompt with them, then generate the
  answer — a three-step pattern, not a new model.
- **Embeddings**: numbers that capture meaning, so text with similar meaning gets similar
  number-lists, measured with cosine similarity.
- Why the Claude API has no embeddings endpoint, and how to call a real embeddings model
  (Voyage AI's `voyage-3`, or the free local `sentence-transformers`) instead.
- What a **vector database** stores (embedding, text, metadata) and why it exists: fast
  nearest-neighbor search over many vectors, using Chroma here and pgvector as the
  production alternative.
- **Chunking**: why we split documents, how chunk size and overlap trade off context against
  precision, and common mistakes (splitting mid-sentence, no overlap, one chunking strategy
  for every document type).
- The two phases of any RAG system: offline indexing (load → chunk → embed → store) and
  query time (embed question → retrieve → augment prompt → generate).
- Built a complete, runnable naive RAG pipeline for GlobalMart policy documents, with Chroma
  and Voyage embeddings, plus simple bracket-style citations back to source documents.
- Where naive RAG breaks down, and what Chapters 5 and 6 do about it.

## Production Notes & Pitfalls

- **Embedding model consistency is not optional.** You must use the same embedding model
  (and the same `input_type` conventions) for indexing and for querying. Switching models
  without re-indexing every stored chunk silently produces meaningless similarity scores —
  no error, just bad retrieval.
- **Re-index on every document change.** Naive RAG has no way to know a policy document
  changed. Build a pipeline step (even a simple cron job or file-watcher) that re-chunks and
  re-embeds a document whenever it is edited, and deletes stale chunks by their `source`
  metadata.
- **Chunk size is a tuning parameter, not a constant.** Different document types need
  different chunk sizes — FAQs, legal policies, and product tables behave differently.
  Measure retrieval quality (Chapter 6) before locking in one size for an entire system.
  Getting this wrong causes silent, hard-to-notice quality loss, not a crash.
- **Watch your token budget in the prompt.** More retrieved chunks is not always better.
  Large `top_k` values raise cost and can dilute Claude's attention across irrelevant
  chunks. Start small (`top_k=3` to `5`) and increase only if evaluation shows it helps.
- **"I do not know" must be a real, encouraged answer.** Without an explicit instruction, a
  model will often guess an answer from general knowledge when retrieval fails, instead of
  admitting the documents did not cover it. This is a common source of confident-sounding,
  wrong support answers. Always instruct the model to say so when the context does not
  contain the answer.
- **Citations need periodic auditing.** A model can attach a citation number to the wrong
  claim, especially as excerpts get longer or more numerous. Treat citations as a
  quality feature to measure, not a solved problem — see the LLM-as-judge techniques in
  Chapter 6.
- **Chroma's in-process mode is for development.** For production traffic, run Chroma as a
  hosted server, or move to pgvector inside your existing Postgres database, so the vector
  store scales and is backed up along with the rest of your data.
- **This chapter's retrieval has no access control.** In a real GlobalMart system, some
  documents (internal seller agreements, refund fraud rules) should not be retrievable by a
  public customer-facing assistant. Filter retrieval by the caller's permissions, at the
  vector database query level — not by trusting Claude to avoid discussing what it read.
- **Batch your embedding calls.** Embedding one chunk at a time, in a loop, is slow and
  usually more expensive than sending a batch. Both Voyage and `sentence-transformers`
  accept a list of texts in one call — always index in batches of tens or hundreds, not
  one at a time.
- **Keep the embedding call in the critical path small.** At query time, the user is
  waiting. Embedding one short question is fast, but a slow vector database, a cold
  connection, or a large `top_k` can add noticeable latency. Measure the retrieval step on
  its own, separately from the Claude call, so you know which part to optimize first.
