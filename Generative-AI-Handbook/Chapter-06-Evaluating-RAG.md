# Chapter 6 — Evaluating RAG

Chapters 4 and 5 built RAG (Retrieval-Augmented Generation) for the GlobalMart Assistant.
*RAG* means: find relevant text chunks in a knowledge base, then ask Claude to answer using
those chunks. You now have a naive version and an advanced version (hybrid search,
re-ranking, query rewriting). But which one is actually better? You cannot answer that by
reading a few sample outputs. This chapter builds a real way to measure RAG quality, and
turns that measurement into a test you run on every code change.

## 1. Why evaluating RAG is hard

A normal unit test checks one exact output. RAG cannot work that way, for two reasons.

**The output is not deterministic.** Ask Claude the same question twice, and you can get two
answers that are worded differently but both correct. An exact-match test (`assert answer ==
expected`) will fail even when the system works fine. You need to check *meaning*, not exact
text.

**"Looks right" is not enough.** A wrong answer can still sound confident and well-written.
This is called *hallucination*: the model states something that is not true, or not
supported by the retrieved text. A human skimming five examples will often miss this. You
need a repeatable, written way to check answers, at a scale a human cannot do by hand every
time you change a prompt or a chunking rule.

The fix is to split "is RAG good?" into smaller, measurable questions, then automate the
checking.

## 2. Two things to measure

A RAG answer can be wrong in two different places. You must measure both separately, or you
will not know what to fix.

```
question → [RETRIEVAL: find chunks] → [GENERATION: write answer from chunks] → answer
              ↑ measure this                  ↑ measure this
           "did we find the                "is the answer grounded
            right text?"                     in that text, and does
                                              it answer the question?"
```

- **Retrieval quality**: did the search step fetch the chunks that actually contain the
  answer? If not, generation cannot succeed — Claude cannot answer from text it never saw.
- **Generation quality**: given the chunks we *did* fetch, did Claude write an answer that is
  faithful to them (no hallucination) and relevant (it actually answers the question)?

Good retrieval with bad generation, and bad retrieval with good-sounding-but-wrong generation
are both failure modes. They need different fixes: retrieval problems are fixed with better
chunking, embeddings, or search (Chapter 5); generation problems are fixed with better
prompts (Chapter 2).

### 2.1 Retrieval quality metrics

An *embedding* is a list of numbers that captures the meaning of text. Retrieval uses
embeddings to find chunks whose meaning is close to the question's meaning. To score
retrieval, you need to know, for each test question, which chunk(s) actually contain the
answer. Call that the **expected chunk(s)**.

| Metric | Plain meaning | Formula (top-k results) |
|---|---|---|
| **Hit rate** | Did at least one expected chunk show up anywhere in the top k results? | 1 if yes, 0 if no, averaged over all questions |
| **Recall@k** | Of all the expected chunks, what fraction did we find in the top k? | `(expected chunks found in top k) / (total expected chunks)` |
| **Precision@k** | Of the k chunks we returned, what fraction were actually relevant? | `(expected chunks found in top k) / k` |

Hit rate is the simplest: "did retrieval work at all, yes or no?" Recall@k matters most when
a question needs several chunks to answer fully (for example, a policy with three
conditions). Precision@k matters because noisy, irrelevant chunks waste context and can
confuse the model, even if the right chunk is also there.

```python
def hit(retrieved_ids: list[str], expected_ids: set[str]) -> int:
    """1 if any expected chunk appears in the retrieved list, else 0."""
    return int(any(cid in expected_ids for cid in retrieved_ids))


def recall_at_k(retrieved_ids: list[str], expected_ids: set[str], k: int) -> float:
    """Fraction of expected chunks that appear in the top k retrieved chunks."""
    top_k = set(retrieved_ids[:k])
    if not expected_ids:
        return 1.0
    found = len(top_k & expected_ids)
    return found / len(expected_ids)


def precision_at_k(retrieved_ids: list[str], expected_ids: set[str], k: int) -> float:
    """Fraction of the top k retrieved chunks that were actually expected."""
    top_k = retrieved_ids[:k]
    if not top_k:
        return 0.0
    found = sum(1 for cid in top_k if cid in expected_ids)
    return found / len(top_k)
```

These functions take plain lists of chunk IDs. They do not call any AI model — that is the
point. Retrieval metrics are cheap, fast, and fully deterministic, so run them often.

### 2.2 Generation quality: faithfulness and relevance

Once we know retrieval fetched the right chunks, we still must check what Claude *wrote*
from them.

- **Faithfulness** (also called *groundedness*): every claim in the answer must be supported
  by the retrieved chunks. If the answer adds a fact that is not in the chunks, that is a
  hallucination — even if the fact happens to be true in the real world. In a production
  system, "true but not in our documents" is still a failure, because you cannot trust it
  came from your source of truth.
- **Relevance**: does the answer actually address the question that was asked? An answer can
  be 100% faithful to the chunks and still fail here, for example by quoting the correct
  policy but the wrong section.

Both of these need *reading comprehension*, not string matching. That is a job for a
language model. This is where **LLM-as-judge** comes in (Section 4).

## 3. Building a golden set

A **golden set** is a fixed list of test questions with known-correct answers. It is the
backbone of RAG evaluation — without it, you have no ground truth to compare against. Each
item needs three things:

1. **question** — a realistic user question.
2. **expected_source_ids** — the chunk ID(s) that should be retrieved to answer it.
3. **expected_answer** — a short, correct answer a human would accept.

Build this by hand, from real support tickets, FAQs, or by asking a few colleagues to write
questions against your documents. Aim for 30–100 items to start. Include easy questions,
hard multi-fact questions, and a few "should refuse" questions (for example, asking about a
policy that does not exist) — a good RAG system should say "I don't know" rather than invent
an answer.

```python
from pydantic import BaseModel


class GoldenItem(BaseModel):
    question: str
    expected_source_ids: list[str]   # chunk IDs that contain the answer
    expected_answer: str             # short reference answer, for the judge to compare against


golden_set = [
    GoldenItem(
        question="What is the return window for electronics on GlobalMart?",
        expected_source_ids=["policy_returns_electronics_02"],
        expected_answer=(
            "Electronics can be returned within 30 days of delivery, "
            "if they are unopened or defective."
        ),
    ),
    GoldenItem(
        question="Can I return a customized product?",
        expected_source_ids=["policy_returns_general_01", "policy_returns_custom_05"],
        expected_answer=(
            "No. Customized or personalized products cannot be returned, "
            "except if they arrive damaged."
        ),
    ),
    GoldenItem(
        question="Does GlobalMart offer a lifetime warranty on furniture?",
        expected_source_ids=[],  # no such policy exists — the model should say so
        expected_answer="There is no lifetime warranty on furniture at GlobalMart.",
    ),
]
```

Keep the golden set in version control, next to the code. Update it when policies change.
Treat it like test data, because that is exactly what it is.

## 4. LLM-as-judge

Checking "is this answer faithful and relevant?" by reading meaning, not exact words, needs
a language model. This pattern is called **LLM-as-judge**: you send the question, the
retrieved chunks, and the generated answer to Claude, along with a clear scoring rubric, and
ask it to score the answer.

The judge must return a fixed, structured shape — not free-form text — so your test harness
can check thresholds automatically. Use `client.messages.parse` with a Pydantic model to get
that.

```python
import anthropic
from pydantic import BaseModel, Field

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env


class JudgeScore(BaseModel):
    faithfulness: int = Field(ge=1, le=5, description="1=invents facts, 5=fully grounded")
    relevance: int = Field(ge=1, le=5, description="1=off-topic, 5=directly answers")
    reasoning: str = Field(description="One or two sentences explaining the scores.")


JUDGE_RUBRIC = """\
You are grading one answer from a customer-support RAG system.

Score FAITHFULNESS (1-5): does every claim in the answer come from the retrieved context?
- 5: every claim is directly supported by the context.
- 3: mostly supported, with one minor unsupported detail.
- 1: the answer states facts that are not in the context at all.
If the context is empty and the answer correctly says "I don't know", score faithfulness 5.

Score RELEVANCE (1-5): does the answer address the user's actual question?
- 5: directly and completely answers the question.
- 3: partially answers, or answers a related but different question.
- 1: does not answer the question at all.

Be strict. An answer that "sounds right" but adds unsupported details must lose
faithfulness points, even if the added detail happens to be true in general.
"""


def judge_answer(question: str, context: str, answer: str) -> JudgeScore:
    prompt = (
        f"QUESTION:\n{question}\n\n"
        f"RETRIEVED CONTEXT:\n{context or '(no context retrieved)'}\n\n"
        f"ANSWER TO GRADE:\n{answer}\n"
    )
    resp = client.messages.parse(
        model="claude-opus-4-8",
        max_tokens=1024,
        system=JUDGE_RUBRIC,
        messages=[{"role": "user", "content": prompt}],
        output_format=JudgeScore,
    )
    return resp.parsed_output
```

### 4.1 Limits of LLM-as-judge, and how to keep it fair

An LLM judge is a model judging another model's output. It is useful, but it has known
weaknesses. Design around them:

- **Write the rubric down, and be specific.** "Rate this answer 1–5" is too vague — different
  runs will drift. Give concrete anchors for each score, like the rubric above does.
- **The judge can be fooled by confident, well-formatted text**, the same way a human skimming
  can be. Explicitly tell it to be strict about unsupported details (see the rubric).
- **Judge one thing at a time.** Do not ask for faithfulness, relevance, tone, and grammar all
  in one vague score. Separate scores are easier to act on and easier to sanity-check.
- **Calibrate against humans.** Periodically have a person score 20–30 of the same answers,
  and compare to the judge's scores. If they disagree a lot, fix the rubric, not just the
  prompt style.
- **Use structured output, not free text**, so scores are consistent to parse and threshold —
  this is why `messages.parse` with a Pydantic model matters here, not just as a coding
  nicety.
- **Cost adds up.** The judge runs once per test case, per test run. For a large golden set
  run on every commit, consider `claude-sonnet-5` for the judge to control cost (Chapter 12
  covers this trade-off); keep `claude-opus-4-8` for spot-checks where judgment quality
  matters most.
- **The judge is not free of bias.** It can favor longer answers, or answers that reuse the
  same wording as the context. Reviewing a sample of judge reasoning (the `reasoning` field
  above) regularly catches this early.

## 5. Catching hallucination specifically

Faithfulness scoring (Section 4) already targets hallucination, but you can make the check
sharper by asking the judge for a *supported / unsupported* verdict per claim, instead of one
number. This gives you a debuggable signal, not just a score.

```python
class ClaimCheck(BaseModel):
    supported: bool
    unsupported_claims: list[str] = Field(
        description="Direct quotes from the answer that are NOT backed by the context. Empty if none."
    )


def check_hallucination(context: str, answer: str) -> ClaimCheck:
    prompt = (
        f"CONTEXT:\n{context or '(no context)'}\n\nANSWER:\n{answer}\n\n"
        "List any sentence or claim in the ANSWER that is not directly supported "
        "by the CONTEXT. If the answer correctly declines to answer due to missing "
        "context, that counts as fully supported."
    )
    resp = client.messages.parse(
        model="claude-opus-4-8",
        max_tokens=1024,
        system="You are a strict fact-checker comparing an answer against its source context.",
        messages=[{"role": "user", "content": prompt}],
        output_format=ClaimCheck,
    )
    return resp.parsed_output
```

This is more expensive than a single 1–5 score, so a common pattern is: run the cheap
faithfulness score on every test case, and only run this detailed claim check when the
faithfulness score is 3 or below — you get a clear explanation exactly when you need one.

## 6. Turning eval into a regression test

A number you compute once is a data point. A number you compute automatically on every code
change is a **regression test** — it stops quality from silently getting worse as you edit
prompts, chunking, or retrieval settings. Wire the golden set and the metrics from Sections
2 and 4 into `pytest`.

```python
import pytest

# rag_pipeline is your own module: retrieve(question, k) -> (chunk_ids, chunk_text)
# and answer(question, chunk_text) -> str, built in Chapters 4-5.
from rag_pipeline import retrieve, answer


@pytest.mark.parametrize("item", golden_set, ids=lambda i: i.question[:40])
def test_retrieval_recall(item):
    chunk_ids, _ = retrieve(item.question, k=5)
    score = recall_at_k(chunk_ids, set(item.expected_source_ids), k=5)
    assert score >= 0.8, f"Recall too low for: {item.question!r} (got {score:.2f})"


@pytest.mark.parametrize("item", golden_set, ids=lambda i: i.question[:40])
def test_generation_quality(item):
    chunk_ids, chunk_text = retrieve(item.question, k=5)
    generated = answer(item.question, chunk_text)
    scores = judge_answer(item.question, chunk_text, generated)
    assert scores.faithfulness >= 4, f"Hallucination risk: {scores.reasoning}"
    assert scores.relevance >= 4, f"Off-topic answer: {scores.reasoning}"
```

Run this with `pytest -q` locally, and in CI on every pull request that touches the RAG
pipeline. Two practical notes:

- **Set thresholds, not exact scores.** Requiring `faithfulness == 5` is too strict — the
  judge itself has some noise. `>= 4` is a reasonable bar for most support-style answers.
- **Track the average score over time**, not just pass/fail. A pipeline that drops from
  average faithfulness 4.8 to 4.3 might still pass a `>= 4` threshold, but that drop is a
  warning sign worth investigating before it gets worse.

## 7. Frameworks worth knowing

You do not have to hand-roll every metric. Open-source eval libraries in the **ragas** style
provide ready-made implementations of faithfulness, answer relevance, context precision, and
context recall, plus dataset helpers. They are a good fit once your golden set grows past
what you want to maintain by hand, or when you want metrics that other teams also use, so
scores are comparable across projects. This chapter builds the harness from scratch on
purpose — once you understand what recall@k and faithfulness actually measure, adopting a
framework is just swapping the implementation, not the concept. Chapter 15 revisits this in
a full production observability setup.

## 8. Hands-on: scoring naive vs advanced RAG

Now put it together. GlobalMart has two RAG pipelines from Chapters 4 and 5: `naive_rag`
(plain vector search, top-5 chunks) and `advanced_rag` (hybrid search + re-ranking, Chapter
5). We will run both through the same golden set and the same metrics, and check that
"advanced" really is better — not just assumed to be.

```python
from dataclasses import dataclass
from statistics import mean


@dataclass
class EvalResult:
    name: str
    avg_recall: float
    avg_precision: float
    avg_faithfulness: float
    avg_relevance: float


def evaluate_pipeline(name: str, retrieve_fn, answer_fn, golden_set, k: int = 5) -> EvalResult:
    recalls, precisions, faiths, relevances = [], [], [], []

    for item in golden_set:
        chunk_ids, chunk_text = retrieve_fn(item.question, k=k)
        expected = set(item.expected_source_ids)

        recalls.append(recall_at_k(chunk_ids, expected, k))
        precisions.append(precision_at_k(chunk_ids, expected, k))

        generated = answer_fn(item.question, chunk_text)
        scores = judge_answer(item.question, chunk_text, generated)
        faiths.append(scores.faithfulness)
        relevances.append(scores.relevance)

    return EvalResult(
        name=name,
        avg_recall=mean(recalls),
        avg_precision=mean(precisions),
        avg_faithfulness=mean(faiths),
        avg_relevance=mean(relevances),
    )


results = [
    evaluate_pipeline("naive_rag", naive_rag.retrieve, naive_rag.answer, golden_set),
    evaluate_pipeline("advanced_rag", advanced_rag.retrieve, advanced_rag.answer, golden_set),
]

print(f"{'pipeline':<14}{'recall@5':>10}{'precision@5':>14}{'faithful':>10}{'relevant':>10}")
for r in results:
    print(
        f"{r.name:<14}{r.avg_recall:>10.2f}{r.avg_precision:>14.2f}"
        f"{r.avg_faithfulness:>10.2f}{r.avg_relevance:>10.2f}"
    )
```

A typical run on a 30-question GlobalMart golden set looks like this:

| pipeline | recall@5 | precision@5 | faithfulness (1–5) | relevance (1–5) |
|---|---|---|---|---|
| naive_rag | 0.71 | 0.34 | 3.9 | 4.1 |
| advanced_rag | 0.93 | 0.52 | 4.6 | 4.7 |

This table tells a clear story. Naive RAG's biggest problem is **recall** — it misses almost
30% of the chunks it needs, mostly on multi-fact questions where the answer spans two policy
sections. Re-ranking and query rewriting in the advanced pipeline fix most of that. Because
retrieval improved, generation quality improved too: faithfulness rose from 3.9 to 4.6,
because Claude had the right text in front of it and did not need to guess or fill gaps.
This is the payoff of separating the two measurements (Section 2) — without it, you would
only see "answers got better" and not know *why*.

Save this comparison script and re-run it whenever you change chunking, the embedding model,
or the number of retrieved chunks (`k`). It turns "I think this change helped" into a number
you can point to.

## What You Built / Learned

- Why RAG cannot be tested with exact-match assertions: output is not deterministic, and
  wrong answers can still look confident.
- The two things to measure separately: **retrieval quality** (hit rate, recall@k,
  precision@k) and **generation quality** (faithfulness/groundedness, relevance).
- How to build a **golden set**: question, expected source chunk IDs, and a reference
  answer, kept in version control like test data.
- **LLM-as-judge**: scoring faithfulness and relevance with a written rubric and structured
  output (`client.messages.parse` + a Pydantic model), plus its limits and how to keep it
  fair (specific rubrics, calibration against humans, separate scores per dimension).
- A focused hallucination check that lists unsupported claims, not just a score.
- How to turn all of this into a `pytest` regression suite that runs on every change.
- A hands-on comparison showing that GlobalMart's advanced RAG pipeline (Chapter 5) measurably
  beats the naive one (Chapter 4), and *why* — better recall drives better faithfulness.

## Production Notes & Pitfalls

- **Retrieval metrics need labeled data you must maintain.** Expected source IDs go stale
  when documents are edited, split, or removed. Re-check the golden set whenever the
  underlying documents change, or your recall numbers become meaningless.
- **Do not let the judge grade itself unchecked.** If the same model generates the answer and
  judges it, biases can compound. Periodically use a different model, or a human spot-check,
  as a sanity check on the judge's scores.
- **LLM-as-judge costs real money and time at scale.** A 200-question golden set, run on every
  pull request, means 200+ extra API calls per run. Route judging to `claude-sonnet-5` or
  `claude-haiku-4-5` for routine CI runs, and reserve `claude-opus-4-8` for periodic deeper
  audits (Chapter 12 has the full cost-routing pattern).
- **A single overall score hides where the problem is.** "RAG quality: 82%" tells you nothing
  actionable. Always report retrieval and generation metrics separately, as this chapter
  does, so you know whether to fix chunking/search or the generation prompt.
- **Small golden sets give noisy averages.** With fewer than ~20 questions, one bad question
  can swing the average score a lot. Grow the set over time, and add real production
  questions (with PII removed) as they come in.
- **Thresholds need periodic review.** A `faithfulness >= 4` bar that was reasonable at launch
  may need to tighten as the product matures, or loosen if the judge rubric changes. Treat
  eval thresholds as a living part of the system, not a one-time setting.
- **Evaluation alone is not observability.** Passing tests on a fixed golden set does not
  guarantee real users get good answers — production traffic has questions your golden set
  never anticipated. Chapter 15 adds live tracing and sampling-based production evaluation on
  top of the offline tests built here.
