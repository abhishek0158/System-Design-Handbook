# Context Distiller — Project Playbook

*A plan you can lead from. Simple English. Every stage has: what it is, why we do it, how to
do it, who owns it, and when it is done.*

---

## Part A — Understand it first (in plain words)

### What are we building?
A tool that sits **between** a code repository and a coding assistant (Claude Code, Copilot,
Cursor, or similar). It has two jobs:

1. **Preprocess the repo once** (and then a little bit on every commit) into a compact,
   searchable form. We call this the **index**.
2. **Serve minimal, ranked, just-in-time context** to the coding assistant for each task,
   pulled from that index — through an **MCP server**. MCP (Model Context Protocol) is a
   standard way for a coding assistant to call an outside tool and get data back. Building
   this as an MCP server means Claude Code, Copilot, and Cursor can all use it the same way,
   without custom glue code for each one.

### What goes in, what comes out?
- **Input:** a code repository, and a task or question from a developer (for example, *"add
  rate limiting to the login endpoint"*).
- **Output:** a small, focused bundle of context — file signatures, short summaries, and a
  few real code snippets — handed to the coding assistant so it can answer the task. The
  assistant then writes the actual code change; we do not write the code change ourselves.

### Why does this matter? (the real problem)
Today, most coding assistants get context in a wasteful way. They either:
- read the **whole file** (or several whole files) into the prompt, even when only one
  function matters, or
- rely on a plain keyword search that misses related code, so the developer pastes in more
  files "just in case".

Both waste **tokens**. Tokens cost money (the LLM provider bills per token) and they cost
time (a bigger prompt takes longer to process). A prompt stuffed with 10 whole files when
only 3 functions matter is slow and expensive, and it does not make the answer better — often
it makes it worse, because the LLM has to find the needle in the haystack itself.

Our tool fixes this: it does the "find the needle" work **once, offline, cheaply**, and then
hands the LLM only the needle, every time a task comes in.

### How is this different from a plain "repo map" (be honest with the team)
Tools like **Aider** already build a repo map: a compact view of function and class signatures
across the repo, ranked by importance. If we only built that, we would not be building
anything new. So we must be honest about this with the team on day 1.

Our two real differentiators are:
1. **The agentic, just-in-time retrieval loop.** A static repo map is fixed and given to the
   LLM once. Ours is different: the LLM is the **"brain in a loop"** — it looks at the task,
   asks for a first small slice of context, checks for itself "is this enough?", and if not,
   asks for more (a function's neighbors in the call graph, or the real code behind a
   summary). It decides when to stop. This loop is what makes the context *minimal* instead
   of just *compact*.
2. **A real evaluation harness.** We do not just claim "this saves tokens." We run the same
   set of tasks two ways — once with the naive "stuff whole files in" approach, and once with
   our tool — and we measure and report the token count and the answer quality for both. The
   proof is a number, not a slide.

Say this clearly to the team: *"We are not reinventing the repo map. We are building the loop
that decides, live, what slice of the map and the code is actually needed — and we are proving
it saves tokens with real measurements."*

---

## Part B — The big picture (two phases)

The system has two phases. **Phase 1 runs once** (then a little on every commit) and builds
an index. **Phase 2 runs on every task** and is the "brain in a loop" that reads the index and
decides what to fetch.

```
PHASE 1 — OFFLINE PREPROCESS (once, then incremental on each commit)
┌─────────────────────────────────────────────────────────────────────┐
│ [1] Parse       -> tree-sitter turns each file into a tree;         │
│                     pull out functions/classes with file+line       │
│ [2] Build graphs-> call graph ("who calls who") + import graph      │
│                     stored in networkx                              │
│ [3] Rank        -> PageRank over the call graph scores each symbol  │
│ [4] Repo map    -> compact map: signatures + docstrings + relations,│
│                     NO function bodies ("skeletonization")          │
│ [5] Summaries   -> Claude writes a short summary per function,      │
│                     then per file, then per module; cached          │
│ [6] Embeddings  -> chunks of code/summaries -> vectors (Voyage AI   │
│                     or sentence-transformers) -> Chroma/FAISS index │
│ [7] Incremental -> on each commit, redo ONLY the changed files      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                     [ THE INDEX ]  (repo map + summaries + graph + vectors)
                              │
                              ▼
PHASE 2 — ONLINE AGENTIC RETRIEVAL (per task, the "brain in a loop", LangGraph)
┌─────────────────────────────────────────────────────────────────────┐
│ [1] Receive task -> "add rate limiting to the login endpoint"       │
│ [2] Plan          -> LLM decides what context it needs              │
│ [3] Retrieve      -> pull smallest useful set: repo map slice +     │
│                       graph neighbors + top-k semantic matches      │
│ [4] Check         -> LLM asks itself "is this enough?"              │
│         │ no  ──────────────► loop back to [3], expand (neighbors,  │
│         │                     or drill from summary into real code) │
│         ▼ yes                                                       │
│ [5] Assemble      -> build a minimal, token-budgeted bundle         │
│ [6] Serve         -> return it via the MCP server to the coding     │
│                       assistant (Claude Code / Copilot / Cursor)     │
└─────────────────────────────────────────────────────────────────────┘
```

If your team understands only this diagram on day 1, that is enough to start. Phase 1 builds
the index. Phase 2 is the loop that reads it, and it is the loop that makes this project worth
building.

---

## Part C — Each stage in detail

For each stage below: **What it is · Why we do it · How to do it · Who owns it · Done when.**

### PHASE 1 — Offline preprocess

#### Stage 1 — Parse
- **What it is:** Turn each source file into an **AST** (Abstract Syntax Tree — a tree that
  knows what is a function, a class, a call), and pull out every function/class with its file
  and line numbers. This reuses the same parsing step as the Code Explorer project (see
  `Pipeline-Deep-Dive-Steps-1-5.md` in that project's folder for real tree-sitter code).
- **Why we do it:** Everything later (the graph, the map, the summaries) needs to know exactly
  where each function starts and ends. Plain text does not give us that.
- **How to do it:**
    - Clone the repo. Walk the files, skip tests/build output/binaries.
    - Parse each file with **tree-sitter** (one parser setup covers many languages).
    - Record, per function/class: name, file, start line, end line, and the calls inside it.
- **Who owns it:** Developer.
- **Done when:** For any file in the test repo, we can print the list of functions/classes it
  defines, with file and line numbers.

#### Stage 2 — Build graphs
- **What it is:** Two graphs. A **call graph** (function A calls function B) and an **import
  graph** (file A imports file B). A graph is dots (nodes) joined by lines (edges).
- **Why we do it:** A function's importance and its relevant neighbors cannot be seen from one
  file alone. The graph lets us ask "what does this touch, and what touches this?" across the
  whole repo. This is also what makes step [3] of the online loop (graph neighbors) possible.
- **How to do it:**
    - Build a symbol table (name → node) from Stage 1's output.
    - Walk the calls again and add an edge for each resolved call. Store this in **networkx** (a
      Python graph library).
    - Build the import graph the same way, using each file's `import` statements.
- **Who owns it:** Developer.
- **Done when:** For any function, we can list what it calls, what calls it, and which files
  import which.

#### Stage 3 — Rank
- **What it is:** Run **PageRank** (an algorithm that scores each node in a graph by how
  central it is — originally built by Google to rank web pages) over the call graph. Every
  symbol gets an importance score.
- **Why we do it:** Not every function matters equally. A small helper called once is not as
  important as a function called from twenty places. Ranking lets us put the important
  symbols first when space (our token budget) is tight.
- **How to do it:**
    - `networkx.pagerank(call_graph)` gives a score per node in a few lines of code.
    - Store the score on each symbol's record.
- **Who owns it:** Developer.
- **Done when:** We can list the top 10 most central functions in the test repo, and they make
  sense to a person who knows that repo (for example, the main request handler ranks high).

#### Stage 4 — Repo map (skeletonization)
- **What it is:** A compact "map" of the whole repo: every function/class signature (its name
  and parameters) plus its docstring (the short comment describing it) and its relations
  (calls, imports) — but **without the function body** (the actual code inside). Removing the
  body is called **skeletonization**.
- **Why we do it:** A signature plus a one-line docstring tells the LLM almost everything it
  needs to decide "is this function relevant to my task?" — without the cost of the full body.
  This map is typically **10 to 30 times smaller** than the raw code, so we can show the LLM
  the shape of a large part of the repo for very few tokens.
- **How to do it:**
    - For each function/class from Stage 1, keep: name, file:line, signature, docstring (if
      any), and its PageRank score from Stage 3.
    - Drop the body. Store this as a lightweight, queryable structure (a JSON file or a small
      database is enough for v1).
- **Who owns it:** Developer.
- **Done when:** We can print the repo map for the test repo, and it is clearly much smaller
  (in tokens) than the raw source files.

#### Stage 5 — Hierarchical summaries
- **What it is:** Short, plain-English summaries, written by the LLM (Claude), at three
  levels: one **per function** (what it does), rolled up into one **per file** (what the file
  is for), rolled up into one **per module** (what the module/folder is for). These are
  cached — written once, read many times.
- **Why we do it:** A cached summary lets the online loop understand what a big piece of code
  does **without reading the code at all**. Reading a 3-line summary instead of a 200-line
  function is most of where the token savings come from.
- **How to do it:**
    - For each function: send its signature + body to Claude with a prompt like *"Summarise
      what this function does in one or two plain sentences."* Use the current model
      (`claude-sonnet-5` or `claude-opus-4-8`; do not set `temperature` or `top_p` — leave them
      at the model's defaults).
    - For each file: send Claude the list of its function summaries and ask for a one-paragraph
      file summary.
    - For each module: send Claude the list of file summaries and ask for a short module
      summary.
    - Cache every summary (keyed by a hash of the source, so we know when it is stale) so we
      never pay to regenerate an unchanged summary.
- **Who owns it:** Developer builds the pipeline; Product reviews summary quality on a sample.
- **Done when:** Every function, file, and module in the test repo has a cached summary, and a
  person reading the module summary understands what the module is for.

#### Stage 6 — Embeddings
- **What it is:** Split the code and summaries into small chunks, and turn each chunk into an
  **embedding** — a list of numbers that captures the *meaning* of the chunk, so that similar
  meanings end up as similar numbers. Store all embeddings in a **vector index** (a database
  built to search by "how similar is this meaning" instead of by keyword).
- **Why we do it:** Not every relevant piece of code is reachable by following the call graph.
  Sometimes the right code is related by *meaning*, not by a direct call (for example, another
  place that already does rate limiting, even if nothing calls it directly). Embeddings find
  that kind of match.
- **How to do it:**
    - Chunk each function body and each cached summary into pieces small enough to embed well
      (roughly one function or one summary per chunk).
    - Create embeddings with a **non-Anthropic embedder** — **Voyage AI** (a hosted embedding
      API) or **sentence-transformers** (a free, local embedding library). Anthropic's Claude
      API does not provide an embeddings endpoint, so this step always uses one of these two.
    - Store the vectors in **Chroma** or **FAISS** (both are vector index libraries; Chroma is
      simpler to start with, FAISS scales further).
- **Who owns it:** Developer.
- **Done when:** Given a plain-English query like "where do we check login attempts", the
  vector index returns the actually-relevant functions in its top results.

#### Stage 7 — Incremental update
- **What it is:** On every new commit, re-run Stages 1–6 **only for the files that changed**,
  instead of reprocessing the whole repo.
- **Why we do it:** Reprocessing a large repo on every commit is slow and wastes LLM calls on
  unchanged summaries. Incremental update keeps the index fresh at a fraction of the cost, and
  keeps our biggest risk (a stale index) under control.
- **How to do it:**
    - On each commit, get the list of changed files (`git diff --name-only`).
    - Re-parse only those files, update their nodes/edges in the graphs, regenerate only their
      summaries and embeddings, and mark their entries with a new content hash.
    - Anything that references a changed function (for example, a caller) does **not** need a
      new summary, but its cached edges may need a light refresh.
- **Who owns it:** Developer.
- **Done when:** Changing one function in the test repo and re-running the update only
  reprocesses that file (we can prove this by timing it or logging what got touched).

### PHASE 2 — Online agentic retrieval (the loop)

#### Stage 1 — Receive the task
- **What it is:** The entry point. A developer's task or question comes in, for example
  *"add rate limiting to the login endpoint."*
- **Why we do it:** This is the only input the loop gets. Everything else is the loop deciding,
  on its own, what context that one sentence requires.
- **How to do it:** The MCP server exposes a tool (for example `get_context(task: str)`) that
  the coding assistant calls with the developer's task text.
- **Who owns it:** Developer (the MCP server plumbing); Product (defines what a good task
  description looks like, for the eval set).
- **Done when:** A task string reaches the LangGraph loop as its starting input.

#### Stage 2 — Plan
- **What it is:** The LLM reads the task and decides, in plain terms, what kind of context it
  probably needs — which modules, which symbols, which summaries look relevant.
- **Why we do it:** Without a plan, the loop would just grab everything "to be safe," which is
  the exact waste we are trying to remove. A plan keeps the first fetch small and targeted.
- **How to do it:**
    - Give the LLM the **top-level repo map** (module and file names with one-line summaries —
      small enough to always include) plus the task text.
    - Ask it: *"Which modules or symbols are most likely relevant? List them, and say what you
      are looking for in each."*
    - This is a LangGraph node (see Part E) that calls Claude once with this small prompt.
- **Who owns it:** Developer.
- **Done when:** For the test task, the plan names the right module (for example, the login
  or auth module) without having read any function bodies yet.

#### Stage 3 — Retrieve
- **What it is:** Pull the smallest useful set of context based on the plan: the relevant
  slice of the repo map (signatures only), plus the top few real code snippets found two ways —
  through the **call graph** (neighbors of the target symbol: who calls it, what it calls) and
  through **embeddings** (semantically similar chunks).
- **Why we do it:** Two different retrieval methods catch two different kinds of relevance:
  the graph catches "structurally connected", embeddings catch "similar in meaning but not
  connected". Using both, but only a few results from each, keeps the bundle small.
- **How to do it:**
    - Look up the planned symbols in the repo map; pull their signatures + summaries.
    - Walk 1–2 hops out in the call graph from those symbols (networkx `successors`/
      `predecessors`) for structural neighbors.
    - Query the vector index (Chroma/FAISS) with the task text for the top-k semantic matches.
    - Merge and de-duplicate. This is a LangGraph node.
- **Who owns it:** Developer.
- **Done when:** For the test task, this stage returns a short list (not the whole repo) that
  a human reviewer agrees is on-topic.

#### Stage 4 — Check
- **What it is:** The LLM looks at what it has gathered so far and asks itself: *"Is this
  enough to answer the task, or do I need more?"* This is the **agentic core** of the whole
  project — the decision is made live, by the LLM, not by a fixed rule.
- **Why we do it:** This is what makes retrieval "just-in-time" instead of "grab a fixed
  amount and hope." If the first small bundle is enough, we stop there and save tokens. If it
  is not enough, we expand — but only exactly as much as needed.
- **How to do it:**
    - Prompt: *"Given the task and this context, can you answer confidently? If yes, say ENOUGH.
      If no, say what specific extra information you need (a function's body, a symbol's
      callers, another module)."*
    - This is a LangGraph **conditional edge**: if the LLM says "not enough," route back to
      Retrieve (Stage 3) with the new, more specific request. If it says "enough," or a token
      budget limit is hit, move on to Assemble (Stage 5).
- **Who owns it:** Developer builds the loop; QA writes test tasks that require at least one
  expand-and-retry, to prove the loop actually loops.
- **Done when:** We can show, on a real task, at least one case where Check said "not enough,"
  the loop expanded, and the second pass answered correctly.

#### Stage 5 — Assemble
- **What it is:** Take everything gathered across all Retrieve passes and build one **minimal,
  token-budgeted context bundle** — the final thing the coding assistant will read.
- **Why we do it:** The loop may have gathered things across several passes; some may now be
  redundant (a summary superseded by the real code it was standing in for). Assemble cleans
  this into one tight, non-redundant bundle that respects a token budget.
- **How to do it:**
    - Deduplicate: if we have both a summary and the real body for the same function, keep the
      body only.
    - Sort by PageRank score / relevance so the most important pieces come first.
    - Trim to the token budget (count tokens; if over budget, drop the lowest-ranked pieces
      first).
- **Who owns it:** Developer.
- **Done when:** The bundle for the test task is well under the token budget and still
  contains everything the Check stage said was needed.

#### Stage 6 — Serve
- **What it is:** Return the assembled bundle to the calling coding assistant, through the
  **MCP server**.
- **Why we do it:** This is the delivery mechanism. Building it as an MCP tool call (instead of
  a one-off script) means any MCP-compatible assistant (Claude Code, Copilot, Cursor) can use
  Context Distiller the same way, today or in the future.
- **How to do it:**
    - Wrap the whole Phase 2 loop as one MCP tool, for example `get_context(task) -> bundle`.
    - Return the bundle as structured text (signatures, summaries, and the few real snippets),
      clearly labelled per piece, so the assistant's own model can read it easily.
- **Who owns it:** Developer.
- **Done when:** A real coding assistant (Claude Code with the MCP server registered) can call
  the tool with a task and receive the bundle, then correctly complete the coding task using
  only that bundle.

---

## Part D — Where the token savings come from

This table is the "why it works" summary. Use it when someone asks *"why would this actually
use fewer tokens?"*

| Technique | What it removes or avoids | Where it happens |
|---|---|---|
| Skeletonization | Function bodies (the biggest part of source code) | Offline Stage 4 (repo map) |
| Cached summaries | Re-reading raw code — read a 2-line summary instead | Offline Stage 5 |
| Graph-ranked selection | Irrelevant symbols — only central, connected ones are pulled | Offline Stage 3, Online Stage 3 |
| Semantic retrieval (embeddings) | A whole-repo keyword dump — only meaning-relevant chunks come back | Offline Stage 6, Online Stage 3 |
| Just-in-time expansion | Fetching "just in case" — the loop stops as soon as it has enough | Online Stage 4 |
| Incremental indexing | Reprocessing unchanged files/summaries on every commit | Offline Stage 7 |

---

## Part E — The 5-week plan (who does what, when)

### The three roles
- **Developer:** builds Phase 1 (offline preprocess) and Phase 2 (the LangGraph loop and the
  MCP server).
- **QA:** builds and runs the evaluation harness (Part G below), writes test tasks, and judges
  answer quality and correctness of the loop's "enough / not enough" decisions.
- **Product:** picks the test repo, defines what a good task set looks like (real tasks a
  developer would actually ask), and reviews summary and bundle quality for readability.

> **Scope rule (important):** For 5 weeks, pick **ONE language and ONE medium-sized repo**
> first (for example, a mid-size Python/Flask codebase). Get the **token number down** on that
> one repo before adding a second language, a bigger repo, or extra features. Say this to the
> team: *"One repo with a proven token reduction beats five repos with none."*

### Week 1 — Offline preprocess, part 1 (parse, graph, rank)
| Day | Goal |
|-----|------|
| 1 | Kickoff. Agree scope: one language, one test repo. Everyone reads this playbook. Product picks the repo and drafts 8–10 real test tasks. |
| 2 | Stage 1 (Parse) working: tree-sitter extracts functions/classes with file+line for the test repo. |
| 3 | Stage 2 (Build graphs): call graph + import graph in networkx. |
| 4 | Stage 3 (Rank): PageRank scores computed; sanity-check the top 10 with someone who knows the repo. |
| 5 | Buffer/catch-up day. QA sets up the skeleton of the eval harness (Part G): how we will measure tokens and quality later. |

**End of Week 1 target:** for the test repo, we have a call graph, an import graph, and
PageRank scores — the backbone Phase 1 needs.

### Week 2 — Offline preprocess, part 2 (repo map, summaries, embeddings)
| Day | Goal |
|-----|------|
| 6 | Stage 4 (Repo map / skeletonization): signatures + docstrings, no bodies. Measure and report its size in tokens vs. the raw repo. |
| 7 | Stage 5 (Summaries), function level: Claude writes and caches a summary per function. |
| 8 | Stage 5 continued: file-level and module-level summaries, rolled up from function summaries. |
| 9 | Stage 6 (Embeddings): chunk code + summaries, embed with Voyage AI or sentence-transformers, load into Chroma/FAISS. |
| 10 | Stage 7 (Incremental update): change one file, re-run, confirm only that file's index entries update. Demo Phase 1 end to end. |

**End of Week 2 target:** point the tool at the test repo and get a full index — repo map,
cached summaries at three levels, and a searchable vector index. This is reused, not rebuilt,
for every task from here on.

### Week 3 — Online loop, part 1 (plan, retrieve, MCP skeleton)
| Day | Goal |
|-----|------|
| 11 | Set up the MCP server skeleton: one tool, `get_context(task)`, that returns a placeholder bundle. |
| 12 | Online Stage 1 + 2 (Receive + Plan): LLM reads the task and the top-level repo map, and names likely-relevant modules/symbols. |
| 13 | Online Stage 3 (Retrieve), graph side: pull signatures + call-graph neighbors for the planned symbols. |
| 14 | Online Stage 3 (Retrieve), embedding side: add the top-k semantic matches from the vector index; merge and de-duplicate. |
| 15 | Wire Plan → Retrieve as a real LangGraph graph (not yet looping). Run one test task end to end. |

**End of Week 3 target:** one test task goes in, and a first-pass context bundle comes out —
but the loop does not yet check itself or expand.

### Week 4 — Online loop, part 2 (the agentic core: check + expand + assemble)
| Day | Goal |
|-----|------|
| 16 | Online Stage 4 (Check): LLM self-assesses "enough / not enough" on the first-pass bundle. |
| 17 | Wire the conditional edge: "not enough" routes back to Retrieve with a specific new request; add a max-loop / token-budget guard so it cannot loop forever. |
| 18 | Online Stage 5 (Assemble): de-duplicate, rank, and trim to the token budget. |
| 19 | Online Stage 6 (Serve): return the real bundle through the MCP tool; connect a real coding assistant (Claude Code) to it. |
| 20 | Run all 8–10 test tasks through the full loop. QA logs, for each: did it loop, how many passes, final token count. |

**End of Week 4 target:** the full agentic loop works end to end on real tasks, through the
MCP server, into a real coding assistant.

### Week 5 — Evaluation and polish
| Day | Goal |
|-----|------|
| 21 | Build the "naive baseline" comparison: same tasks, but stuff whole relevant files into the prompt instead of using our tool. |
| 22 | Run the full eval harness (Part G): tokens-per-task and answer quality, naive vs. distilled, for all test tasks. |
| 23 | Fix the worst gaps found (a task where distilled quality dropped, or savings were small). Re-run only the affected tasks. |
| 24 | Product reviews the final numbers and the demo story. Write up the headline result (for example, "60% fewer tokens, same task success"). |
| 25 | Polish, write a short README for the MCP server, and demo live: same task, two ways, side by side. |

**End of Week 5 target:** a working MCP server, a proven token-reduction number backed by the
eval harness, and a live demo showing the same task answered two ways.

### What "good enough" means (set this expectation now)
It will **not** cover every task or every edge case in the repo. Aim for: a clear, provable
token reduction (even 40–50% is a strong result) on a realistic set of tasks, with equal or
better answer quality, on **one** repo. A proven number on one repo beats a vague claim about
many repos.

---

## Part F — The agentic retrieval loop (code skeleton)

This is a short, realistic skeleton of the online loop from Part C, using **LangGraph** (a
library for building an LLM app as a graph of steps, with the ability to loop). Keep it this
simple to start; do not over-build it in week 3.

```python
from typing import TypedDict, List
from langgraph.graph import StateGraph, END
import anthropic

client = anthropic.Anthropic()
MODEL = "claude-sonnet-5"          # or "claude-opus-4-8" for harder tasks
MAX_LOOPS = 3                      # safety guard: never expand forever
TOKEN_BUDGET = 4000                # rough budget for the final bundle


class LoopState(TypedDict):
    task: str                # the developer's task, e.g. "add rate limiting to /login"
    plan: str                # what the LLM thinks it needs
    bundle: List[dict]       # pieces of context gathered so far (map slices, snippets)
    verdict: str             # "ENOUGH" or "MORE:<what is missing>"
    loops: int               # how many retrieve/check cycles have run


def plan_node(state: LoopState) -> LoopState:
    resp = client.messages.create(
        model=MODEL, max_tokens=300,
        messages=[{"role": "user", "content":
            f"Task: {state['task']}\n"
            f"Given the top-level repo map, which modules/symbols are relevant, "
            f"and what should we look for in each?"}],
    )
    state["plan"] = resp.content[0].text
    return state


def retrieve_node(state: LoopState) -> LoopState:
    # pulls signatures + summaries from the repo map, call-graph neighbours,
    # and top-k semantic matches from the vector index (Chroma/FAISS) —
    # guided by state["plan"] on pass 1, and by state["verdict"] on later passes
    new_pieces = fetch_context(state["plan"], state.get("verdict"))
    state["bundle"] = dedupe(state["bundle"] + new_pieces)
    state["loops"] += 1
    return state


def check_node(state: LoopState) -> LoopState:
    resp = client.messages.create(
        model=MODEL, max_tokens=200,
        messages=[{"role": "user", "content":
            f"Task: {state['task']}\n"
            f"Context gathered so far:\n{render(state['bundle'])}\n\n"
            f"Can you answer this task confidently with only this context? "
            f"Reply 'ENOUGH' or 'MORE: <exactly what is missing>'."}],
    )
    state["verdict"] = resp.content[0].text.strip()
    return state


def should_expand(state: LoopState) -> str:
    if state["verdict"].startswith("ENOUGH"):
        return "assemble"
    if state["loops"] >= MAX_LOOPS or token_count(state["bundle"]) >= TOKEN_BUDGET:
        return "assemble"          # budget/loop guard wins even if not "ENOUGH"
    return "retrieve"              # loop back and expand


def assemble_node(state: LoopState) -> LoopState:
    state["bundle"] = trim_to_budget(rank(dedupe(state["bundle"])), TOKEN_BUDGET)
    return state


graph = StateGraph(LoopState)
graph.add_node("plan", plan_node)
graph.add_node("retrieve", retrieve_node)
graph.add_node("check", check_node)
graph.add_node("assemble", assemble_node)

graph.set_entry_point("plan")
graph.add_edge("plan", "retrieve")
graph.add_edge("retrieve", "check")
graph.add_conditional_edges("check", should_expand, {
    "retrieve": "retrieve",   # not enough yet -> go fetch more
    "assemble": "assemble",   # enough, or budget hit -> finish
})
graph.add_edge("assemble", END)

app = graph.compile()

# Called by the MCP server's get_context(task) tool:
# result = app.invoke({"task": task, "plan": "", "bundle": [], "verdict": "", "loops": 0})
# return result["bundle"]
```

The important part is the **conditional edge** on `check`: it is what turns a fixed pipeline
into a loop that stops itself as soon as it has enough — the "brain in a loop" the whole
project is built around. `fetch_context`, `dedupe`, `render`, `token_count`, `rank`, and
`trim_to_budget` are the small helper functions built in Phase 1 and Online Stages 3/5.

---

## Part G — Evaluation (this is how we prove the project worked)

The whole point of Context Distiller is a **measurable number**, not a feeling. Every test
task must be run **two ways** and compared:
- **Naive baseline:** stuff the whole relevant file(s) into the prompt, the way most people
  use a coding assistant today.
- **Distilled:** run the same task through our MCP server and the agentic loop.

| Metric | What it tells us | How to measure |
|---|---|---|
| **Tokens per task (main metric)** | How much cheaper and faster the same task is | Count input tokens sent to the LLM, naive vs. distilled, for the same task |
| Answer quality | Whether the savings cost us correctness | LLM-as-judge (ask Claude to score the two answers blind) and/or task success (does the generated code actually work / pass a test) |
| Latency | Whether the extra loop passes slow things down too much | Wall-clock time from task-in to bundle-out, naive vs. distilled |

**The headline result we want** looks like this: *"Same task, same or better answer quality,
60% fewer tokens."* Report it per task and as an average across the full test-task set (built
in Week 1, run in Week 5). If the number does not exist, the project is not done — do not skip
this in favour of more features.

---

## Part H — Risks and how to handle them

| Risk | What happens | How you handle it |
|------|--------------|-------------------|
| Stale index | Repo changes but the index does not, so summaries/graph describe old code | Stage 7 (incremental update) on every commit; hash-check before trusting a cached summary |
| Wrong or missing context hurts the answer | The loop stops too early, or Retrieve picks irrelevant pieces, and the assistant gives a wrong answer | QA builds test tasks that specifically require an expand step; track task success, not just token count, in the eval harness |
| Summary drift | A cached summary no longer matches the code after small edits that did not trigger reprocessing | Key every cached summary by a content hash of the exact code it describes; any mismatch forces regeneration |
| Cost of building summaries | Summarising every function/file/module with an LLM call adds real dollar cost and time, especially on a big repo | Cache aggressively (Stage 5); only regenerate on real changes (Stage 7); start with one medium repo, not the whole company's codebase |
| "Isn't this just Aider's repo map?" | Team or stakeholders doubt the value | Use Part A: our differentiator is the agentic just-in-time loop plus the proven token/quality numbers, not the map itself |
| Loop runs forever or blows the budget | The Check stage keeps saying "not enough," costing more tokens than the naive approach | Hard `MAX_LOOPS` and `TOKEN_BUDGET` guards in the conditional edge (see Part F); always fall through to Assemble |

---

## Part I — How you lead it day to day
1. **Every morning, 10 minutes:** each person says yesterday's result, today's goal, any
   blocker.
2. **Track by the two phases:** the wall shows Phase 1's 7 stages and Phase 2's 6 stages. Move
   each to Done as it works end-to-end on the test repo. Progress is visible.
3. **Protect the scope:** when someone wants to add a language, a second repo, or a fancier
   ranking method, write it on a "later" list. Do not let it into the 5 weeks.
4. **Demo on the real number, not slides.** The proof is: same task, naive vs. distilled,
   token counts side by side, live.

*You do not need to be the deepest coder to lead this. Your job is to keep the scope tight,
keep the two phases moving, and keep asking "do we have a real token number yet, on a real
task?"*
