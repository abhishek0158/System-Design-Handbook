# Chapter 12 — Reliability, Cost & Latency

In the last two chapters, you gave the GlobalMart Assistant a graph-based brain (LangGraph) and
learned how to compose it from small parts (LangChain). That brain works well in a demo. But a
demo runs once, for one friendly user, on a fast network. Production is different. Networks
drop packets. Models time out under load. A support-ticket tool might get called twice by
accident. And every token you send or receive costs money and adds delay.

This chapter is about making the assistant **reliable** (it keeps working when things go
wrong), **cheap** (it does not waste money), and **fast** (the user does not wait too long).
These three goals often pull in different directions. A safer retry policy can add latency. A
cheaper model can be less reliable at hard tasks. Part of system design is picking the right
trade-off for each part of the system, not applying one rule everywhere.

We will harden the real GlobalMart Assistant code from earlier chapters: add retries with
fallback, add a circuit breaker, add a cheap-to-expensive model router, add prompt caching, and
add a simple semantic cache. At the end we estimate the cost and latency drop with real numbers.

---

## 12.1 The three failure modes, in one picture

```
User question
     |
     v
[ Reliability ]  -- does the call even complete? (timeouts, retries, fallback, circuit breaker)
     |
     v
[ Cost ]         -- how many tokens did that cost? (routing, caching, trimming, batching)
     |
     v
[ Latency ]      -- how long did the user wait? (streaming, parallel calls, semantic cache)
     |
     v
Answer to user
```

A production agent needs an answer for each layer. Skipping one layer does not remove the
problem — it just means the problem shows up later, in an incident review.

---

## 12.2 Reliability

**Reliability** here means: the system keeps giving correct answers even when a network call
fails, a model is slow, or a downstream service is down. Reliability is not about being clever.
It is about being predictable when something breaks.

### 12.2.1 Timeouts

A **timeout** is a limit on how long you wait for a reply before giving up. Without a timeout, a
single slow model call can block a request thread forever. That is worse than an error, because
an error at least lets the caller move on.

The Claude Python SDK lets you set a timeout per client or per call:

```python
import anthropic

# Client-wide default: 30 seconds for the whole request
client = anthropic.Anthropic(timeout=30.0)

# Override for one call that you know is short
resp = client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=200,
    system="Classify the user request in one word: order, product, or other.",
    messages=[{"role": "user", "content": "Where is my package?"}],
    timeout=8.0,
)
```

Pick the timeout from the job, not from a guess. A quick classification call (Haiku, short
answer) should time out fast — 5 to 10 seconds is enough. A long, streamed answer with tool use
can reasonably take 30 to 60 seconds. If you set one global timeout for everything, you will
either cut off slow-but-fine calls, or wait too long on calls that should have failed fast.

### 12.2.2 Retries: the SDK already does the easy part

A **retry** means: if a call fails, try it again. Not every failure deserves a retry. A `429`
(rate limited) or a `5xx` (server error) is often temporary — retrying can succeed. A `400`
(bad request, like a malformed tool schema) will fail again every time — retrying wastes time
and money.

Good news: **the Claude Python SDK already retries `429` and `5xx` errors for you**, using
exponential backoff, with a default of 2 retries. You do not need to hand-write this for the
common case. You can raise the count if your traffic is bursty:

```python
client = anthropic.Anthropic(max_retries=4)
```

**Exponential backoff** means each retry waits longer than the last one — for example 1s, 2s,
4s, 8s — instead of retrying instantly. This gives an overloaded server time to recover. **Jitter**
means adding a small random amount to each wait time, so that many clients retrying at once do
not all hit the server at the exact same moment again (which would just recreate the overload).
The SDK's built-in retry already includes both. If you ever need to write your own retry loop —
for example, around a non-Claude call, or a tool call to an internal service — do it like this:

```python
import random
import time

def call_with_backoff(fn, max_attempts=4, base_delay=0.5):
    """Retry fn() with exponential backoff + jitter. Raises on the last failure."""
    for attempt in range(max_attempts):
        try:
            return fn()
        except (TimeoutError, ConnectionError) as exc:
            if attempt == max_attempts - 1:
                raise
            delay = base_delay * (2 ** attempt)       # 0.5s, 1s, 2s, 4s...
            delay += random.uniform(0, delay * 0.3)     # jitter: up to +30%
            time.sleep(delay)
```

Only wrap calls that are **safe to repeat** (see idempotency, §12.2.5) or that are read-only,
such as `get_order_status`. Never blindly retry a tool call that charges a card or sends an
email, unless you know the retry cannot double the effect.

### 12.2.3 Fallbacks: have a second option ready

A **fallback** means: if your first choice fails (or times out, or returns something unusable),
switch to a second choice instead of failing the whole request. For an LLM app, that second
choice is usually a different model, and sometimes a different provider.

```python
import anthropic

client = anthropic.Anthropic(timeout=20.0, max_retries=2)

FALLBACK_CHAIN = ["claude-opus-4-8", "claude-sonnet-5", "claude-haiku-4-5"]

def call_with_fallback(system: str, messages: list, models=FALLBACK_CHAIN):
    """Try each model in order. Return the first one that succeeds."""
    last_error = None
    for model in models:
        try:
            return client.messages.create(
                model=model,
                max_tokens=1024,
                system=system,
                messages=messages,
            )
        except anthropic.RateLimitError as exc:
            last_error = exc
            continue  # this model's capacity is busy right now, try the next one
        except anthropic.APIStatusError as exc:
            last_error = exc
            continue  # server-side error, try the next model
        except anthropic.APIConnectionError as exc:
            last_error = exc
            continue  # network issue, try the next model
    raise RuntimeError(f"All models failed. Last error: {last_error}")
```

Notice we catch the SDK's **typed exceptions** — `anthropic.RateLimitError`,
`anthropic.APIStatusError`, `anthropic.APIConnectionError` — instead of a bare `except Exception`.
Typed exceptions let you react differently to different failures. For example, you might log a
`RateLimitError` as "expected, moving on" but page an on-call engineer for repeated
`APIConnectionError`s, since that could mean your network setup is broken.

Falling back from Opus to Sonnet to Haiku is a same-provider fallback. If you need protection
against an outage of the whole provider, the same pattern works across providers: try Claude
first, and if every Claude attempt fails, call a backup provider's API with an equivalent prompt.
Keep provider-specific code behind one function (like `call_with_fallback` above) so the rest of
the app does not need to know which provider actually answered.

### 12.2.4 Circuit breakers: stop calling a service that is already down

Retries and fallbacks handle a single failed call. But what if a model or a tool's backend is
having a real outage that will last five minutes? Retrying every single request during that
window wastes time on calls that are very likely to fail anyway, and it can add extra load to a
service that is already struggling.

A **circuit breaker** tracks recent failures for a dependency (a model, or a tool's backend
service). When failures cross a threshold, it "opens" — for a cooldown period, it skips the real
call completely and fails fast (or returns a fallback) instead. After the cooldown, it lets a
few test calls through to check if the dependency has recovered. This chapter's Java Coding
Problems book has a full write-up of the pattern, including the state machine and a thread-safe
implementation: see
[`Java-Coding-Problems/concurrency/20-Circuit-Breaker.md`](../Java-Coding-Problems/concurrency/20-Circuit-Breaker.md).
The state machine is identical in Python — only the syntax changes:

```
CLOSED  --(failures >= threshold)--> OPEN
OPEN    --(cooldown time passed)--> HALF_OPEN
HALF_OPEN --(trial calls succeed)--> CLOSED
HALF_OPEN --(a trial call fails)--> OPEN
```

A minimal Python version, single-threaded (fine for one process; see Chapter 13 for
multi-instance state):

```python
import time


class CircuitBreaker:
    def __init__(self, failure_threshold=5, cooldown_seconds=30, half_open_trials=2):
        self.failure_threshold = failure_threshold
        self.cooldown_seconds = cooldown_seconds
        self.half_open_trials = half_open_trials
        self.state = "CLOSED"
        self.failure_count = 0
        self.opened_at = None
        self.trial_successes = 0

    def _maybe_recover(self):
        if self.state == "OPEN" and time.time() - self.opened_at >= self.cooldown_seconds:
            self.state = "HALF_OPEN"
            self.trial_successes = 0

    def call(self, fn, fallback=None):
        self._maybe_recover()
        if self.state == "OPEN":
            if fallback is not None:
                return fallback()
            raise RuntimeError("Circuit is OPEN — call skipped")

        try:
            result = fn()
        except Exception:
            self.failure_count += 1
            if self.state == "HALF_OPEN" or self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
                self.opened_at = time.time()
            if fallback is not None:
                return fallback()
            raise
        else:
            if self.state == "HALF_OPEN":
                self.trial_successes += 1
                if self.trial_successes >= self.half_open_trials:
                    self.state = "CLOSED"
                    self.failure_count = 0
            else:
                self.failure_count = 0
            return result
```

Use one `CircuitBreaker` instance **per dependency**, not one shared instance for everything. The
GlobalMart Assistant should have a separate breaker for the model API and a separate breaker for
each tool's backend (order service, ticketing service). If the ticketing service is down, you
still want the model breaker CLOSED, so the assistant can keep answering questions — it just
cannot open new tickets right now.

### 12.2.5 Idempotency: make retries safe for actions

**Idempotency** means: doing the same action twice has the same effect as doing it once. This
matters because retries and fallbacks both mean "maybe try this again." If `create_support_ticket`
is not idempotent, a retry after a network blip can create two tickets for one complaint.

The standard fix is an **idempotency key**: a unique ID generated once per logical action, sent
with every attempt (including retries). The backend remembers keys it already processed and
returns the original result instead of doing the action again.

```python
import uuid

def create_support_ticket(user_id: str, issue: str, idempotency_key: str | None = None) -> dict:
    """Create a ticket. Safe to call more than once with the same idempotency_key."""
    key = idempotency_key or str(uuid.uuid4())
    # The ticketing service stores `key` and returns the same ticket_id
    # if it has already seen this key — even if this exact call is retried.
    return ticketing_client.post(
        "/tickets",
        json={"user_id": user_id, "issue": issue},
        headers={"Idempotency-Key": key},
    )
```

Generate the key **once**, before the first attempt, and reuse it on every retry of that same
logical action. If you generate a new key on each retry, you defeat the whole point. Read-only
tools like `get_order_status` do not need this — reading data twice causes no harm. Write tools
(create, cancel, refund, send) always should.

| Tool type | Example | Needs idempotency key? |
|---|---|---|
| Read-only | `get_order_status`, `search_products` | No |
| Write, side effect | `create_support_ticket`, `issue_refund` | Yes |
| Write, no real cost if repeated | `log_event` (append-only, dedup downstream) | Usually no |

---

## 12.3 Cost

Every input token and output token costs money. At low volume this barely matters. At production
scale — thousands of users, or an agent that loops several times per request — it adds up fast
enough to change your architecture.

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Use it for |
|---|---|---|---|
| Claude Haiku 4.5 | $1 | $5 | Routing, classification, simple lookups, easy questions |
| Claude Sonnet 5 | $3 | $15 | Most GlobalMart questions: policy Q&A, order help |
| Claude Opus 4.8 | $5 | $25 | Hard reasoning, multi-step tool plans, escalations |

(Check current pricing before relying on exact numbers — providers update prices. The relative
shape, cheap-fast-simple to expensive-slow-capable, tends to stay true.)

### 12.3.1 Token budgets

A **token budget** is a limit on how many tokens a request (or a whole conversation) is allowed
to use. Without one, a single confused agent loop, or a user pasting a huge document, can burn
through your monthly budget in one conversation.

Check the size of a request **before** sending it, with the SDK's token counter — never with
`tiktoken`, which is OpenAI's tokenizer and gives wrong numbers for Claude:

```python
count = client.messages.count_tokens(
    model="claude-sonnet-5",
    system=system_prompt,
    messages=messages,
)
print(count.input_tokens)

MAX_INPUT_TOKENS = 6000
if count.input_tokens > MAX_INPUT_TOKENS:
    messages = trim_history(messages)  # see 12.3.4
```

Set budgets at more than one level: per request (stop one runaway call), per conversation (stop
an endless back-and-forth), and per user per day (stop abuse or a stuck retry loop from silently
draining money).

### 12.3.2 Model routing: try cheap first

**Model routing** means picking which model to call based on how hard the task looks, instead of
always calling your biggest model. Most real questions to the GlobalMart Assistant are simple:
"Where is my order?", "What's your return window?" A small, cheap, fast model can answer these
correctly. Save the expensive model for the questions that actually need deep reasoning or a long
tool-use plan.

```python
def classify_difficulty(question: str) -> str:
    """Ask Haiku to label the question. Cheap and fast."""
    resp = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=10,
        system=(
            "Label the user question as exactly one word: "
            "'simple' (a direct lookup or policy fact) or "
            "'complex' (needs multi-step reasoning or planning)."
        ),
        messages=[{"role": "user", "content": question}],
    )
    return resp.content[0].text.strip().lower()


def route_and_answer(question: str) -> str:
    label = classify_difficulty(question)
    model = "claude-haiku-4-5" if label == "simple" else "claude-sonnet-5"
    resp = client.messages.create(
        model=model,
        max_tokens=1024,
        system=GLOBALMART_SYSTEM_PROMPT,
        messages=[{"role": "user", "content": question}],
    )
    return resp.content[0].text
```

You can extend this to three tiers: Haiku classifies and answers simple questions directly;
Sonnet handles normal questions and most tool use; Opus is called only when Sonnet's own answer
looks uncertain, or when the task is flagged as needing a long plan (Chapter 9's planner pattern).
Escalating up is cheap — the extra Haiku call to classify costs a fraction of a cent. Not
escalating when needed is what costs you in wrong answers, so keep a way to detect low-confidence
Sonnet answers and retry with Opus rather than routing on guesswork alone.

### 12.3.3 Prompt caching: pay once for the big fixed part

Every call to the GlobalMart Assistant sends the same **system prompt** — instructions, tool
definitions, maybe policy documents — plus a new, small user question. Resending that big fixed
part on every single call is wasteful. **Prompt caching** lets the API remember a large stable
block and charge you a much lower rate for it on later calls that reuse it, instead of full price
every time.

Mark the big, unchanging part with `cache_control`:

```python
resp = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": GLOBALMART_SYSTEM_PROMPT,   # long: policies, tone rules, tool docs
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=[{"role": "user", "content": question}],
)

print(resp.usage.cache_read_input_tokens)  # tokens served from cache, at a lower rate
print(resp.usage.input_tokens)             # tokens NOT served from cache
```

Check `resp.usage.cache_read_input_tokens` to confirm caching is actually working — a nonzero
number here means the cache hit. Cache only the part that stays the same across calls. Do not put
the user's changing question inside the cached block, or you will get cache misses on every
request and gain nothing.

### 12.3.4 Trimming and compacting history

A long-running conversation keeps growing. Every reply adds to a history you resend on the next
turn. Left alone, cost per turn keeps rising, and you can even hit the model's context window
limit. **Trimming** means dropping or shortening old turns before sending the next request.

A simple trimming policy: keep the system prompt (cached, see above), keep the last N turns in
full, and replace anything older with a short summary.

```python
def trim_history(messages: list, keep_last: int = 6) -> list:
    """Keep the most recent turns; summarize everything before that."""
    if len(messages) <= keep_last:
        return messages
    old, recent = messages[:-keep_last], messages[-keep_last:]
    summary_text = summarize(old)  # one cheap Haiku call: "summarize this in 3 sentences"
    summary_msg = {"role": "user", "content": f"[Earlier conversation summary]: {summary_text}"}
    return [summary_msg] + recent
```

This is the same idea as **compaction**, which the API also supports natively as a beta feature
for long agent sessions (Chapter 13 covers this in depth, for agents that run for a long time).
The manual version above is enough for most chat-style assistants.

### 12.3.5 The Batch API: half price for work that is not urgent

Some work does not need an answer right now. Examples: re-summarizing all support tickets from
last week, re-checking a whole product catalog against a new policy, or running nightly
evaluation prompts (Chapter 6, Chapter 15). For this kind of **bulk, non-urgent** work, the
**Message Batches API** processes many requests together and costs about **50% less** than
normal calls, with results ready within hours instead of instantly.

```python
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"ticket-{ticket_id}",
            "params": {
                "model": "claude-haiku-4-5",
                "max_tokens": 200,
                "system": "Summarize this support ticket in one sentence.",
                "messages": [{"role": "user", "content": ticket_text}],
            },
        }
        for ticket_id, ticket_text in tickets_to_summarize
    ]
)

# Poll (or use a webhook) until the batch finishes, then read results:
status = client.messages.batches.retrieve(batch.id)
if status.processing_status == "ended":
    for result in client.messages.batches.results(batch.id):
        print(result.custom_id, result.result.type)  # "succeeded", "errored", ...
```

The rule of thumb: if a human is waiting on the screen for the answer, use the normal Messages
API. If a script is processing a large list overnight, use the Batch API and save half the money.

---

## 12.4 Latency

**Latency** is how long the user waits. Two requests can use the same number of tokens and cost
the same, but feel completely different if one streams the first word in 400ms and the other
shows nothing for 6 seconds.

### 12.4.1 Streaming to first token

Without streaming, the user sees nothing until the entire reply is generated. With **streaming**,
you get pieces of the reply as they are generated, so you can show the first words almost
immediately — this is often called **time to first token (TTFT)**. Even if the total generation
time is the same, streaming makes the assistant feel much faster, because the user has something
to read right away.

```python
with client.messages.stream(
    model="claude-sonnet-5",
    max_tokens=1024,
    system=GLOBALMART_SYSTEM_PROMPT,
    messages=[{"role": "user", "content": question}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)   # send this chunk to the UI right away

    final_message = stream.get_final_message()  # full message + usage, once done
```

Use streaming for any reply that goes straight to a user-facing UI. Skip it for calls whose
output goes to another function, not a screen — for example, the Haiku classification call in
§12.3.2 has a 10-token answer; streaming adds overhead there for no benefit.

### 12.4.2 Run independent tool calls in parallel

If the model asks for two tools in one turn — say, `get_order_status` and `search_products` —
and neither depends on the other's result, running them one after another wastes time. Running
them **in parallel** takes roughly as long as the slower of the two, instead of the sum of both.

```python
import concurrent.futures

def execute_tool_calls(tool_use_blocks: list) -> list:
    """Run independent tool calls in parallel; return tool_result blocks in matching order."""
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as pool:
        futures = {pool.submit(run_tool, block): block for block in tool_use_blocks}
        results = {}
        for future in concurrent.futures.as_completed(futures):
            block = futures[future]
            results[block.id] = future.result()

    return [
        {"type": "tool_result", "tool_use_id": block.id, "content": str(results[block.id])}
        for block in tool_use_blocks
    ]
```

This only helps when the calls truly do not depend on each other. If tool B needs tool A's
output, running them in parallel is not just useless — it is wrong, since B would run before A's
result exists. Check the plan for real dependencies before parallelizing (Chapter 8 covers tool
call design in more depth).

### 12.4.3 Semantic caching: reuse the answer to an equivalent question

A normal cache matches on **exact** input. **Semantic caching** matches on **meaning**: "What's
your return policy for electronics?" and "Can I return a laptop I bought?" are worded
differently but ask almost the same thing. A semantic cache can answer the second one instantly
from the first one's stored answer, using an **embedding** — a list of numbers that captures the
meaning of text — to measure how similar two questions are.

```python
import numpy as np
import voyageai

embed_client = voyageai.Client()  # reads VOYAGE_API_KEY from env


def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


class SemanticCache:
    def __init__(self, similarity_threshold: float = 0.92):
        self.threshold = similarity_threshold
        self.entries = []  # list of (embedding, question, answer)

    def _embed(self, text: str) -> np.ndarray:
        result = embed_client.embed([text], model="voyage-3", input_type="query")
        return np.array(result.embeddings[0])

    def get(self, question: str) -> str | None:
        query_vec = self._embed(question)
        best_score, best_answer = 0.0, None
        for vec, _, answer in self.entries:
            score = cosine_similarity(query_vec, vec)
            if score > best_score:
                best_score, best_answer = score, answer
        if best_score >= self.threshold:
            return best_answer
        return None

    def set(self, question: str, answer: str) -> None:
        self.entries.append((self._embed(question), question, answer))
```

`voyageai` is the recommended embeddings provider for Claude apps — the Claude API itself does
not generate embeddings. `sentence-transformers` is a free, local alternative if you cannot call
an external embeddings API. A similarity threshold around 0.90–0.95 is a reasonable starting
point; tune it against real question pairs, since too low a threshold returns wrong cached
answers, and too high a threshold barely saves anything. Only cache answers you know are still
correct if reused — for example, policy answers are safe to cache; "where is my order right now"
is not, because the real answer changes.

---

## 12.5 Hands-on: hardening the GlobalMart Assistant

Now we combine all three areas into one small, realistic wrapper for the assistant's model
calls. This is close to what you would actually deploy: cheap-first routing, retry with
fallback, a circuit breaker per model, and a semantic cache in front of everything.

```python
import time
import random
import anthropic

client = anthropic.Anthropic(timeout=20.0, max_retries=2)

model_breaker = {
    "claude-haiku-4-5": CircuitBreaker(failure_threshold=5, cooldown_seconds=30),
    "claude-sonnet-5": CircuitBreaker(failure_threshold=5, cooldown_seconds=30),
    "claude-opus-4-8": CircuitBreaker(failure_threshold=5, cooldown_seconds=30),
}
semantic_cache = SemanticCache(similarity_threshold=0.92)

FALLBACK_ORDER = ["claude-haiku-4-5", "claude-sonnet-5", "claude-opus-4-8"]


def reliable_answer(question: str, system_prompt: str) -> str:
    # 1. Latency + cost: semantic cache first — cheapest and fastest possible path
    cached = semantic_cache.get(question)
    if cached is not None:
        return cached

    # 2. Cost: route to the cheapest model that should handle this question
    label = classify_difficulty(question)
    start_index = 0 if label == "simple" else 1  # skip Haiku for complex questions

    # 3. Reliability: walk up the fallback chain, respecting each model's circuit breaker
    last_error = None
    for model in FALLBACK_ORDER[start_index:]:
        breaker = model_breaker[model]
        try:
            resp = breaker.call(lambda m=model: client.messages.create(
                model=m,
                max_tokens=1024,
                system=[{"type": "text", "text": system_prompt,
                         "cache_control": {"type": "ephemeral"}}],
                messages=[{"role": "user", "content": question}],
            ))
            answer = resp.content[0].text
            semantic_cache.set(question, answer)
            return answer
        except (anthropic.RateLimitError, anthropic.APIStatusError,
                 anthropic.APIConnectionError, RuntimeError) as exc:
            last_error = exc
            continue

    raise RuntimeError(f"All models unavailable. Last error: {last_error}")
```

### What this buys you, in numbers

Assume 10,000 GlobalMart questions a day, 70% "simple" and 30% "complex," average 500 input
tokens and 200 output tokens per call, and a 15% semantic-cache hit rate after a week of traffic
(policy questions repeat a lot).

| Setup | Model used | Requests hitting the model | Rough daily cost |
|---|---|---|---|
| Before: always Opus, no cache | Opus 4.8 for everything | 10,000 | ~$32.50 |
| After: routing + semantic cache | Haiku for simple, Sonnet for complex | 8,500 (1,500 cache hits) | ~$9.10 |

*(500 in / 200 out tokens per call; Opus $5/$25 per 1M; Sonnet $3/$15 per 1M; Haiku $1/$5 per 1M.
Cache-hit requests cost nothing beyond the one-time embedding call. Numbers are illustrative —
measure your own traffic mix and token sizes.)*

That is roughly a **70% cost drop**, from routing and caching alone, before even counting prompt
caching on the shared system prompt (which further cuts input-token cost on every non-cached
call). On latency: the cache hits return in well under 100ms (one embedding call plus a
similarity check, no model call at all), and simple questions routed to Haiku typically finish in
a fraction of the time Opus would take. The 30% of questions that truly need Opus still get it —
routing does not make the assistant worse at hard questions, it just stops using a sledgehammer
on easy ones.

---

## What You Built / Learned

- The three production concerns for any LLM system: **reliability** (does the call complete),
  **cost** (how many tokens does it use), **latency** (how long does the user wait) — and that
  they trade off against each other.
- Timeouts sized to the job, and why the Claude SDK already retries `429`/`5xx` with backoff —
  so you only need custom retry logic for special cases.
- Exponential backoff with jitter, written from scratch, for calls the SDK does not auto-retry.
- A model-and-provider fallback chain, using the SDK's typed exceptions
  (`RateLimitError`, `APIStatusError`, `APIConnectionError`) to decide when to move to the next
  option.
- A circuit breaker for model calls and tool calls, adapted from the Java concurrency version, so
  a known-down dependency is skipped instead of retried into the ground.
- Idempotency keys, so a retried write action (like `create_support_ticket`) cannot double its
  effect.
- Token budgets checked with `client.messages.count_tokens` (never `tiktoken`), model routing
  from Haiku up to Opus, prompt caching with `cache_control`, history trimming, and the Batch API
  for non-urgent bulk work at half price.
- Streaming for fast time-to-first-token, parallel execution of independent tool calls, and a
  semantic cache built on Voyage embeddings for reusing answers to equivalent questions.
- A combined `reliable_answer()` function for the GlobalMart Assistant that applies all of the
  above in one request path, with an estimate of the resulting cost and latency drop.

## Production Notes & Pitfalls

- **Do not retry non-idempotent actions blindly.** A retry that resends a refund request without
  an idempotency key is a real production incident, not a hypothetical one. Fix this at the tool
  layer, not by telling the model to "be careful."
- **A circuit breaker's state is per-process by default.** If you run many instances of the
  assistant behind a load balancer, each instance opens its breaker independently — this is
  normal, but it means one instance can still be hammering a dead dependency while another has
  already given up. Chapter 13 covers sharing this state when you truly need a global view.
- **Fallback models can give different quality answers.** If Opus fails and you fall back to
  Haiku, the user gets a weaker answer, not an error — which is usually the right trade, but
  log which model actually answered so you can measure how often this happens and catch quality
  regressions.
- **Model routing needs its own evaluation.** A cheap classifier that mislabels "complex"
  questions as "simple" quietly degrades answer quality without ever raising an error. Track
  accuracy on the labels themselves (Chapter 6, Chapter 15), not just on final answers.
- **Prompt caching has a short time-to-live and exact-match requirements on the cached block.**
  Small edits to a "static" system prompt — even whitespace — break the cache. Keep the cached
  block truly frozen, and version it explicitly when it does change.
- **Semantic cache thresholds drift.** A threshold tuned today can start returning wrong answers
  as your product changes (a policy update means an old cached answer is now false). Expire
  cache entries when the underlying source document changes, not just by time.
- **The Batch API is not for anything time-sensitive.** Results can take hours. Never route a
  live user-facing question through it, even if the discount is tempting.
- **Measure before you optimize.** Add tracing (Chapter 15) before adding routing, caching, and
  circuit breakers. Otherwise you are guessing at thresholds and hit rates instead of setting
  them from real traffic — and you cannot prove the change helped.
