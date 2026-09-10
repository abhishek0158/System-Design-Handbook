# Chapter 13 — Scaling & Long-Running Agents

So far, every agent in this book ran once, in one process, and finished in a few seconds.
Chapter 12 made a single agent call more reliable. This chapter asks a different question:
what happens when GlobalMart needs to run **thousands of agent jobs a day**, and some jobs
take minutes or hours to finish?

Two problems show up together:
1. **Scale** — many requests at once, hitting the same Claude API rate limit.
2. **Duration** — a single agent run that is too long to hold open in one request, or that
   must survive a server restart.

We solve both with the same idea: **do not keep agent state only in memory. Put it
somewhere durable, and let a pool of workers pick up jobs from a queue.**

## 1. From one process to many: queues and worker pools

Imagine a support team lead who takes calls one at a time, then writes a to-do list. If ten
customers call at once, the lead cannot serve them all in parallel — but the lead *can* write
down all ten requests, then work through the list, and ask coworkers to help.

That is the queue-and-worker pattern:

```mermaid
flowchart LR
    A[GlobalMart app] -->|enqueue job| Q[(Task Queue)]
    Q --> W1[Worker 1]
    Q --> W2[Worker 2]
    Q --> W3[Worker 3]
    W1 --> C[Claude API]
    W2 --> C
    W3 --> C
    W1 --> D[(Job Store / DB)]
    W2 --> D
    W3 --> D
```

- A **task queue** is a list of jobs waiting to be processed. Producers add jobs; workers
  remove and run them.
- A **worker pool** is a set of processes that each pull one job at a time from the queue,
  run it, and go back for the next one.
- The **job store** (a database) holds the result and status of each job (`queued`,
  `running`, `done`, `failed`) so the app can check progress without talking to the worker
  directly.

Why not just run every agent call in the web request that triggered it? Because an agent
call to Claude can take seconds to minutes, especially with tool loops (Chapter 7–8). If your
web server thread blocks on that call, a burst of users freezes the whole app. A queue
decouples "accept the job" from "do the job." The user gets an immediate `job_id`; the
worker does the slow part in the background.

### A minimal queue in Python

For learning, we can build a queue with Python's standard library. In production, swap this
for **Redis Queue (RQ)**, **Celery**, or a managed queue (SQS, Cloud Tasks) — the pattern is
the same, only the transport changes.

```python
import queue
import threading
import time
import uuid
from dataclasses import dataclass, field

import anthropic

client = anthropic.Anthropic()

job_queue: "queue.Queue[dict]" = queue.Queue()
job_results: dict[str, dict] = {}
results_lock = threading.Lock()


@dataclass
class Job:
    job_id: str
    question: str
    status: str = "queued"


def submit_job(question: str) -> str:
    """Called by the app. Puts a job on the queue and returns right away."""
    job_id = str(uuid.uuid4())
    with results_lock:
        job_results[job_id] = {"status": "queued", "answer": None}
    job_queue.put({"job_id": job_id, "question": question})
    return job_id


def worker_loop(worker_name: str) -> None:
    """Runs forever in its own thread. Pulls one job at a time."""
    while True:
        job = job_queue.get()
        job_id, question = job["job_id"], job["question"]
        with results_lock:
            job_results[job_id]["status"] = "running"
        try:
            answer = answer_globalmart_question(question)
            with results_lock:
                job_results[job_id] = {"status": "done", "answer": answer}
        except Exception as exc:
            with results_lock:
                job_results[job_id] = {"status": "failed", "answer": str(exc)}
        finally:
            job_queue.task_done()


def answer_globalmart_question(question: str) -> str:
    resp = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        system="You are the GlobalMart support assistant. Be concise.",
        messages=[{"role": "user", "content": question}],
    )
    return "".join(b.text for b in resp.content if b.type == "text")


# Start a small pool of 3 workers.
for i in range(3):
    threading.Thread(target=worker_loop, args=(f"worker-{i}",), daemon=True).start()

job_id = submit_job("What is the return window for electronics?")
time.sleep(2)  # in a real app, poll job_results[job_id]["status"] instead
print(job_results[job_id])
```

This runs three workers in threads inside one process. In production you would run this as a
**separate worker service** — a process (or many machines) whose only job is to pop tasks
from a real queue and call Claude. The web app and the worker pool scale independently: add
more workers when the queue backs up, without touching the app servers.

## 2. Async vs. sync: two ways to get concurrency

There are two common ways to run many Claude calls at once inside one worker process:

| Approach | How it works | Best for |
|---|---|---|
| **Sync + threads/processes** | Each worker thread makes a normal blocking `client.messages.create()` call | Simple code, CPU-bound tool logic, small worker counts |
| **Async (`asyncio`)** | One event loop runs many `await client.messages.create()` calls concurrently, using `anthropic.AsyncAnthropic` | High-volume I/O-bound work — waiting on network calls, not CPU |

Since a Claude API call is mostly *waiting for the network*, async is a natural fit when you
need hundreds of concurrent calls from one process:

```python
import asyncio
import anthropic

async_client = anthropic.AsyncAnthropic()


async def answer_one(question: str) -> str:
    resp = await async_client.messages.create(
        model="claude-opus-4-8",
        max_tokens=512,
        messages=[{"role": "user", "content": question}],
    )
    return "".join(b.text for b in resp.content if b.type == "text")


async def answer_many(questions: list[str]) -> list[str]:
    tasks = [answer_one(q) for q in questions]
    return await asyncio.gather(*tasks)


# asyncio.run(answer_many(["Q1...", "Q2...", "Q3..."]))
```

A simple rule: **use sync workers when each job also does heavy non-network Python work**
(complex tool logic, file processing). **Use async when the job is mostly "send a request,
wait, get a reply,"** and you want one process to juggle hundreds of those at once. Many
real systems mix both: async workers for the light Q&A jobs, and a separate sync worker pool
for jobs that also run heavier tools.

## 3. Rate limits and backpressure

Claude's API — like every provider's API — enforces **rate limits**: a maximum number of
requests and tokens per minute for your account. **Backpressure** means slowing down the
producer (your app) when the system downstream (the API, or your own queue) cannot keep up.
Without backpressure, a traffic spike either crashes your workers or wastes money on failed
retries.

Three layers of defense:

**Layer 1 — let the SDK retry.** The `anthropic` Python SDK automatically retries `429`
(rate limit) and `5xx` (server error) responses with exponential backoff. This handles
short, occasional bursts for free:

```python
client = anthropic.Anthropic(max_retries=5)  # default is 2; raise it for background workers
```

**Layer 2 — catch the typed exception and back off yourself.** For a worker pool, you often
want to pause the *whole worker*, not just retry one call, when you are being throttled:

```python
import time
import anthropic

def call_with_backoff(messages, attempts: int = 5):
    delay = 1.0
    for attempt in range(attempts):
        try:
            return client.messages.create(
                model="claude-opus-4-8",
                max_tokens=1024,
                messages=messages,
            )
        except anthropic.RateLimitError:
            print(f"Rate limited. Sleeping {delay:.1f}s before retry {attempt + 1}.")
            time.sleep(delay)
            delay *= 2  # exponential backoff
        except anthropic.APIConnectionError:
            time.sleep(delay)
            delay *= 2
    raise RuntimeError("Gave up after repeated rate limits.")
```

**Layer 3 — limit concurrency before you even hit the API.** The cheapest way to avoid rate
limits is to never send more requests than your account allows. A **semaphore** caps how
many calls run at once:

```python
import asyncio

MAX_CONCURRENT_CALLS = 10
semaphore = asyncio.Semaphore(MAX_CONCURRENT_CALLS)

async def answer_one_limited(question: str) -> str:
    async with semaphore:
        resp = await async_client.messages.create(
            model="claude-opus-4-8",
            max_tokens=512,
            messages=[{"role": "user", "content": question}],
        )
        return "".join(b.text for b in resp.content if b.type == "text")
```

For queue-based systems, backpressure also means: if the queue grows past a threshold, stop
accepting new jobs (return a "try later" response) or scale up workers — instead of piling
up requests that will just fail later. Watch queue depth as a core metric (Chapter 15).

## 4. Durable, long-running agents

A support-ticket triage agent that answers one question in five seconds does not need to
"remember" anything between calls. But some GlobalMart agent jobs are different: "research
this vendor dispute, check five internal systems, draft a resolution, and wait for a manager
to approve it." That can take minutes of tool calls, or hours if a human needs to review it.

If that agent's entire state lives only in a Python variable in one process, a crash, a
deploy, or a server restart **destroys the job**. That is not acceptable for anything
customers or staff depend on. The fix is **checkpointing**: saving the agent's state after
each step, to a place that survives process restarts.

### Why checkpointing matters

Think of it like a video game save file. Without saves, dying means starting the whole level
over. With saves, you resume from the last checkpoint. An agent's "level" is its
conversation history, its tool call results so far, and where it is in its plan.

A durable agent needs to answer three questions at any point in time:
- What has happened so far? (message history, tool results)
- Where in the plan am I? (which step, which node)
- Can I pick this back up on a *different* machine, later? (state is not in local memory)

### LangGraph checkpointers

Chapter 11 introduced LangGraph: a graph of nodes and edges, with state passed between them.
LangGraph ships **checkpointers** — pluggable savers that persist the graph's state after
every step. This is the simplest way to get durable agent runs in this book's stack.

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from typing import TypedDict


class TicketState(TypedDict):
    ticket_id: str
    findings: list[str]
    resolved: bool


def investigate(state: TicketState) -> TicketState:
    # In real code: call Claude with tools to check order/inventory systems.
    state["findings"].append("Checked order history: no delivery scan recorded.")
    return state


def draft_resolution(state: TicketState) -> TicketState:
    state["findings"].append("Drafted refund proposal for manager review.")
    state["resolved"] = True
    return state


graph = StateGraph(TicketState)
graph.add_node("investigate", investigate)
graph.add_node("draft_resolution", draft_resolution)
graph.add_edge(START, "investigate")
graph.add_edge("investigate", "draft_resolution")
graph.add_edge("draft_resolution", END)

# InMemorySaver is fine for local testing. Swap for a database-backed
# checkpointer (below) to survive a process restart.
app = graph.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "ticket-8821"}}
result = app.invoke(
    {"ticket_id": "8821", "findings": [], "resolved": False},
    config=config,
)
print(result)
```

The `thread_id` is the key idea: it names *this specific run*. If the process restarts, you
can call `app.invoke(None, config=config)` again with the same `thread_id`, and LangGraph
resumes from the last saved step instead of starting over — as long as the checkpointer
itself is durable.

`InMemorySaver` keeps state in RAM, so it does **not** survive a restart — use it only for
local development. For anything real, use a **database-backed checkpointer**, such as
LangGraph's Postgres or SQLite checkpoint savers, which write each step's state to a real
database:

```python
from langgraph.checkpoint.postgres import PostgresSaver

# Each step of the graph is written to Postgres. If the worker process
# crashes, a new process can open the same connection string and resume
# any thread_id exactly where it left off.
with PostgresSaver.from_conn_string("postgresql://user:pass@host/globalmart") as checkpointer:
    app = graph.compile(checkpointer=checkpointer)
    app.invoke({"ticket_id": "8821", "findings": [], "resolved": False}, config=config)
```

If you are not using LangGraph, the same idea works with your own database: after every
agent step, write a row containing the job ID, the current message history (as JSON), the
current step name, and a status. To resume, read that row back and continue the loop from
there. LangGraph's checkpointer is simply a well-tested version of this pattern.

### Human approval without losing progress

Chapter 11 showed LangGraph's `interrupt()` for human-in-the-loop approval: the graph pauses
at a node, waits for a human decision, then continues. Checkpointing is what makes this safe
across time, not just across one process's lifetime.

```python
from langgraph.types import interrupt

def request_manager_approval(state: TicketState) -> TicketState:
    decision = interrupt({"question": "Approve refund?", "findings": state["findings"]})
    state["resolved"] = decision == "approve"
    return state
```

When `interrupt()` fires, LangGraph saves a checkpoint and stops. That checkpoint can sit for
seconds — or for two days, if the manager is out of office — because it is on disk in the
database, not in a process's memory. When the manager approves it (perhaps through a small
internal tool that calls `app.invoke(Command(resume="approve"), config=config)`), the run
picks up exactly where it paused. No re-running the investigation, no re-spending tokens on
steps already done. This is the pattern that makes long-running agents with human oversight
practical: **pause and resume are just "read the checkpoint, keep going."**

## 5. Keeping long conversations inside the context window

A long-running agent can also accumulate a *very* long message history — every tool call and
result adds tokens. Chapter 1 explained the context window: the maximum tokens a model can
see at once. Two problems follow from a growing history:

1. **Cost** — you resend the whole history on every call, and Claude bills for input tokens
   every time.
2. **Limit** — eventually the history is too big for the context window, and the call fails.

There are three tools to manage this, from simplest to most automatic:

**Summarization (you write it).** Periodically, replace older turns with a short summary
written by Claude itself, and keep only recent turns in full:

```python
def summarize_history(messages: list[dict]) -> str:
    resp = client.messages.create(
        model="claude-haiku-4-5",  # cheap model for a mechanical task
        max_tokens=512,
        system="Summarize this conversation in 5 bullet points, keeping key facts and decisions.",
        messages=messages,
    )
    return "".join(b.text for b in resp.content if b.type == "text")


def compact_history(messages: list[dict], keep_last: int = 6) -> list[dict]:
    if len(messages) <= keep_last:
        return messages
    older, recent = messages[:-keep_last], messages[-keep_last:]
    summary = summarize_history(older)
    return [{"role": "user", "content": f"Earlier conversation summary: {summary}"}] + recent
```

**The API's beta compaction feature.** Instead of writing your own summarizer, the Claude
API offers **compaction**: a beta feature where the API itself automatically summarizes
older context once a conversation approaches a size threshold, and returns the summarized
history for you to keep resending. This removes the need to hand-write the logic above for
most cases — check current API docs for the exact beta flag, since beta features change.

**The API's context editing feature.** Related but different: **context editing** lets you
tell the API to clear out specific old content — for example, drop the full text of tool
results from three steps ago, once you no longer need them, while keeping the rest of the
conversation. This is useful for agents whose tool calls return large payloads (a big search
result, a long document) that are only needed for one or two steps.

Use summarization/compaction for "the conversation is getting long," and context editing for
"this one big tool result is no longer needed." Both are about the same underlying limit —
the context window — but they trim different things.

## 6. Build vs. buy: Managed Agents

Everything above — queue, workers, checkpointer, database — is infrastructure you build and
operate yourself. That is the right choice when you need full control over tools, data, and
cost. But it is real engineering work: a queue to run, a database schema for checkpoints, a
worker fleet to deploy and monitor.

Anthropic also offers **Managed Agents**: a hosted agent platform where Anthropic runs the
agent loop and provides a sandboxed environment for the agent to work in, instead of you
running that loop on your own servers. You still define the agent's tools and behavior; you
do not build and operate the loop, the retries, or the execution sandbox yourself.

This is a genuine **build-vs-buy** decision:

| | Build it yourself (this chapter) | Managed Agents |
|---|---|---|
| Who runs the loop | Your workers | Anthropic |
| Where state lives | Your database/checkpointer | Anthropic's platform |
| Control over tools/data | Full | High, within the platform's model |
| Ops burden | You run queue + workers + DB | Anthropic runs the infrastructure |
| Best for | Custom pipelines, strict data control, deep integration with your own systems | Getting a capable long-running agent live fast, without building the runtime |

For the GlobalMart Assistant in this book, we build the loop ourselves — it is the best way
to *learn* how agents work, and it matches most teams' need for control over tools and data.
But when scoping a real project, always ask: "does a hosted option cover this well enough?"
before committing to build and operate agent infrastructure. Chapter 16 revisits this
decision for the full production architecture.

## Hands-on: GlobalMart jobs through a queue with resumable state

Let's combine the pieces: a queue, a small worker pool, and checkpointed state that a
worker can resume after a simulated crash. We keep it dependency-light with `sqlite3`, so you
can see exactly what "durable state" means before reaching for LangGraph's checkpointer in a
real project.

```python
import json
import queue
import sqlite3
import threading
import time
import uuid

import anthropic

client = anthropic.Anthropic()
DB_PATH = "globalmart_jobs.db"
job_queue: "queue.Queue[str]" = queue.Queue()
db_lock = threading.Lock()


def init_db() -> None:
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            """CREATE TABLE IF NOT EXISTS jobs (
                job_id TEXT PRIMARY KEY,
                step INTEGER NOT NULL,
                state TEXT NOT NULL,
                status TEXT NOT NULL
            )"""
        )


def save_checkpoint(job_id: str, step: int, state: dict, status: str) -> None:
    """Write the job's current step and state to disk. Called after every step."""
    with db_lock, sqlite3.connect(DB_PATH) as conn:
        conn.execute(
            "INSERT OR REPLACE INTO jobs (job_id, step, state, status) VALUES (?, ?, ?, ?)",
            (job_id, step, json.dumps(state), status),
        )


def load_checkpoint(job_id: str) -> tuple[int, dict, str] | None:
    with db_lock, sqlite3.connect(DB_PATH) as conn:
        row = conn.execute(
            "SELECT step, state, status FROM jobs WHERE job_id = ?", (job_id,)
        ).fetchone()
    if row is None:
        return None
    step, state_json, status = row
    return step, json.loads(state_json), status


STEPS = ["search_orders", "check_policy", "draft_reply"]


def run_step(step_name: str, state: dict) -> dict:
    """Each step is one small piece of agent work. In real code, some of
    these would call Claude with tools (Chapter 7-8); here we keep the
    example short and simulate the work with one Claude call per step."""
    resp = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=256,
        system=f"You are performing the '{step_name}' step of a GlobalMart support job.",
        messages=[{"role": "user", "content": f"Order: {state['order_id']}. Do the {step_name} step and report one line."}],
    )
    note = "".join(b.text for b in resp.content if b.type == "text")
    state.setdefault("notes", []).append(note)
    return state


def run_job(job_id: str) -> None:
    """Runs (or RESUMES) a job step by step, checkpointing after each step."""
    checkpoint = load_checkpoint(job_id)
    if checkpoint is None:
        raise ValueError(f"Unknown job {job_id}")
    start_step, state, status = checkpoint
    if status == "done":
        print(f"[{job_id}] already done, skipping")
        return

    print(f"[{job_id}] resuming from step {start_step} ({STEPS[start_step] if start_step < len(STEPS) else 'finished'})")
    for i in range(start_step, len(STEPS)):
        state = run_step(STEPS[i], state)
        # Save progress immediately. If the process dies right after this
        # line, a resume will start at step i + 1, not from the beginning.
        save_checkpoint(job_id, i + 1, state, status="running")
        print(f"[{job_id}] completed step '{STEPS[i]}'")

    save_checkpoint(job_id, len(STEPS), state, status="done")
    print(f"[{job_id}] done. notes: {state['notes']}")


def submit_job(order_id: str) -> str:
    job_id = str(uuid.uuid4())[:8]
    save_checkpoint(job_id, step=0, state={"order_id": order_id, "notes": []}, status="queued")
    job_queue.put(job_id)
    return job_id


def worker_loop() -> None:
    while True:
        job_id = job_queue.get()
        try:
            run_job(job_id)
        except Exception as exc:
            print(f"[{job_id}] failed: {exc}")
        finally:
            job_queue.task_done()


if __name__ == "__main__":
    init_db()
    job_id = submit_job("ORD-9001")

    # Simulate a worker that crashes after the first step: run one step
    # manually, then stop, instead of calling run_job() to completion.
    checkpoint = load_checkpoint(job_id)
    step, state, status = checkpoint
    state = run_step(STEPS[step], state)
    save_checkpoint(job_id, step + 1, state, status="running")
    print(f"[{job_id}] simulated crash after step '{STEPS[step]}'")

    # "Restart": a fresh worker picks up the SAME job_id and resumes.
    # It reads the checkpoint from disk, sees step=1, and continues from
    # 'check_policy' onward — it never repeats 'search_orders'.
    run_job(job_id)
```

Run this once. You will see the job complete step 1, print a simulated crash, then resume
and finish steps 2 and 3 — without repeating step 1's Claude call. That last point is the
whole lesson: **the checkpoint, not the process, owns the job's progress.** Any worker,
on any machine, that can read `globalmart_jobs.db` can pick up where another worker left
off. Swap SQLite for Postgres and this file for the LangGraph checkpointer, and you have the
same guarantee at production scale.

## What You Built / Learned

- A **task queue + worker pool** pattern: the app enqueues jobs and returns immediately;
  workers pull jobs and run the (slow) agent logic in the background.
- The difference between **sync workers** (simple, good for CPU-bound tool logic) and
  **async workers** (`asyncio`, good for many concurrent network-bound Claude calls).
- Three layers of **rate limit and backpressure** handling: SDK auto-retry, your own
  exponential backoff on `RateLimitError`, and capping concurrency with a semaphore before
  you even call the API.
- Why **durable, long-running agents** need checkpointing: state saved to a database, keyed
  by a `thread_id`/`job_id`, so a run can pause, crash, and resume without losing progress.
- LangGraph's **checkpointers** (`InMemorySaver` for dev, Postgres/SQLite savers for
  production) and how `interrupt()` (Chapter 11) combines with checkpointing for human
  approval that survives long waits.
- Managing long conversations with **summarization/compaction** (shrink old turns to a
  summary) and the API's **context editing** (clear specific old content, like large tool
  results, you no longer need).
- **Managed Agents** as a build-vs-buy option: a hosted agent loop and sandbox from
  Anthropic, versus building the queue/worker/checkpoint stack yourself.
- A hands-on SQLite-backed job runner that checkpoints after every step and proves it can
  resume after a simulated crash.

## Production Notes & Pitfalls

- **A queue is infrastructure, not a library import.** In production, use a real broker
  (Redis, SQS, RabbitMQ, Cloud Tasks) with retry policies, dead-letter queues for jobs that
  fail repeatedly, and monitoring on queue depth. The `queue.Queue` example is for learning
  the pattern, not for production traffic.
- **Idempotency matters once you have retries.** If a worker crashes right after calling
  Claude but before saving the checkpoint, a resume will repeat that step. Design steps so
  running them twice is safe (for example, check "has a ticket already been created for this
  order?" before creating one), especially for tool calls with side effects (Chapter 8).
- **Checkpoint granularity is a real design choice.** Checkpointing after every tiny action
  is safest but adds latency and storage cost. Checkpointing only at big milestones is
  cheaper but repeats more work on a crash. Match granularity to how expensive each step is —
  cheap steps can be re-run; an expensive multi-tool investigation should be checkpointed
  right after it finishes.
- **Rate limits are per organization, not per worker.** Adding more workers does not add more
  API capacity — it just lets you hit the same limit faster. Size your worker pool and
  concurrency caps against your actual account limits, and request a limit increase before a
  big launch, not during it.
- **Watch for silent context growth.** A long-running agent that keeps appending tool results
  to its history without ever summarizing or clearing old content will eventually fail with a
  context-length error, usually during a demo or a real incident. Test long-running jobs with
  a large number of simulated steps before shipping, not just with the two- or three-step
  happy path.
- **Human-in-the-loop pauses need a timeout policy.** If a manager never responds to an
  approval request, the checkpoint can sit forever. Decide what "stale" means for your
  workflow (a day? a week?) and add a background sweep that escalates or cancels jobs stuck
  past that point.
- **Beta features change.** Compaction and context editing are beta at the time of writing.
  Check the current Claude API docs for the exact parameter names and behavior before
  building on them, and keep a fallback (your own summarization code) in case a beta feature
  is unavailable in your region or account tier.
- **Managed Agents does not remove the need for evaluation and observability.** Whether you
  build the loop or use a hosted one, you still need the practices from Chapter 15 —
  logging, tracing, and evals — to know whether your long-running agents are actually doing
  their job correctly.
