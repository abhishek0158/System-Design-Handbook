# Chapter 8 — Tools & Memory

Chapter 7 built an agent loop: the model thinks, calls a tool, reads the result, and thinks
again, until it is done. That loop is only as good as two things: the **tools** it can call, and
the **memory** it can draw on. This chapter covers both.

First, we look at what makes a tool good, how to run tools safely, and how to run several tools
at once. Then we look at memory: the short-term kind that lives in the conversation, and the
long-term kind that lives outside the model. We finish by giving the GlobalMart Assistant real
memory — it will remember a user's name and past support tickets across separate chat sessions,
and it will compact a long conversation so it does not overflow the context window.

---

## 1. What Makes a Good Tool

A **tool** is a function the model can ask your code to run. The model does not run the function
itself. It only outputs a request — a name and some arguments — and your code runs the real
function and sends the result back. Chapter 3 showed the basic mechanics. Here we focus on
**design**: how to write tools that a model can use well.

A good tool has three parts:

| Part | What it is | Why it matters |
|---|---|---|
| **Name** | A short, clear verb phrase, e.g. `get_order_status` | The model picks tools by name and description. A vague name causes wrong picks. |
| **Description** | A sentence telling the model exactly when and how to use it | This is the only "documentation" the model reads. Write it like an API doc, not a comment. |
| **Input schema** | A small JSON schema listing the exact fields the tool needs | Small and strict schemas reduce malformed calls. |

### 1.1 Keep tools few, focused, and non-overlapping

Each tool should do **one thing**. Do not build one giant `manage_order` tool that can look up,
cancel, or refund an order based on a `mode` field. That forces the model to guess which mode to
use, and it hides the real inputs from the schema. Instead, write three small tools:
`get_order_status`, `cancel_order`, `refund_order`.

Also keep the **total number of tools small**. Every tool definition (name, description, schema)
is sent to the model on every call. More tools means more tokens spent, and — more importantly —
more chances for the model to pick the wrong one. Five to ten well-named tools work better than
thirty overlapping ones. If you truly need many tools, group them behind a router (Chapter 9
covers this pattern) or expose them through MCP (Model Context Protocol), the standard way to
plug a large, shared set of tools into an agent without hardcoding each one.

Here is a small, focused tool set for the GlobalMart Assistant, matching the tools introduced in
Chapter 7:

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment

tools = [
    {
        "name": "get_order_status",
        "description": (
            "Look up the current status of one order by its order ID. "
            "Use this when the user asks 'where is my order' or 'is my order shipped'. "
            "Returns the order status, carrier, and expected delivery date, "
            "or an error if the order ID does not exist."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "GlobalMart order ID, format ORD-XXXXXX (6 digits).",
                }
            },
            "required": ["order_id"],
        },
    },
    {
        "name": "search_products",
        "description": (
            "Search the GlobalMart catalog by keyword. "
            "Use this when the user is looking for a product, not tracking an order."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search keywords, e.g. 'wireless mouse'."},
                "max_results": {"type": "integer", "description": "Max items to return.", "default": 5},
            },
            "required": ["query"],
        },
    },
    {
        "name": "create_support_ticket",
        "description": (
            "Open a new support ticket for a user issue that you cannot resolve directly. "
            "Use this only after you have gathered the order ID (if relevant) and a clear "
            "one-line description of the problem."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "user_id": {"type": "string", "description": "The GlobalMart account ID of the user."},
                "issue": {"type": "string", "description": "A short, clear summary of the problem."},
                "order_id": {"type": "string", "description": "Related order ID, if any."},
            },
            "required": ["user_id", "issue"],
        },
    },
]
```

Notice each `description` tells the model **when** to use the tool, not just what it does. That
line is doing real work: it stops the model from calling `create_support_ticket` when the user
just wants an order status, and it tells the model what to collect before calling it.

### 1.2 Return useful results, not raw dumps

A tool result becomes part of the conversation. If you return a huge raw database row or a wall
of JSON, you waste tokens and confuse the model. Shape the result the way you would shape an API
response for a human client: short, relevant fields only.

```python
def get_order_status(order_id: str) -> dict:
    order = _lookup_order(order_id)  # your real backend call
    if order is None:
        return {"error": f"No order found with ID {order_id}."}
    return {
        "order_id": order_id,
        "status": order.status,          # e.g. "shipped"
        "carrier": order.carrier,
        "expected_delivery": order.eta.isoformat(),
    }
```

### 1.3 Return errors as tool results, not exceptions

This is the most common mistake in early agent code: letting a tool raise an exception that
crashes the whole loop. If `get_order_status` throws because the order ID does not exist, your
Python process stops — and the user gets nothing. Instead, **catch the error inside the tool
function and return it as a normal tool result**, with an `"error"` field. The model reads that
result like any other, and it can recover: apologize, ask the user to check the order ID, or try
a different tool.

```python
def run_tool(name: str, tool_input: dict) -> dict:
    try:
        if name == "get_order_status":
            return get_order_status(**tool_input)
        if name == "search_products":
            return search_products(**tool_input)
        if name == "create_support_ticket":
            return create_support_ticket(**tool_input)
        return {"error": f"Unknown tool: {name}"}
    except Exception as exc:
        # Never let a tool crash the agent loop. Report the failure instead.
        return {"error": f"Tool '{name}' failed: {exc}"}
```

The agent loop then sends this dict back as a `tool_result` block, whether it succeeded or
failed:

```python
tool_result_block = {
    "type": "tool_result",
    "tool_use_id": tool_use_id,   # must match the id the model sent
    "content": json.dumps(result),
}
```

A model that sees `{"error": "No order found with ID ORD-99999."}` can say "I could not find
that order — can you double-check the ID?" A model that never gets a result, because your code
crashed, cannot say anything useful at all.

### 1.4 Validate tool inputs before running them

Never trust the arguments a model sends, even though they follow your JSON schema. The schema
only checks *shape* (is `order_id` a string?), not *safety* (is this a real order ID? does this
user own it?). Always validate before you touch a real system:

```python
import re

def validate_order_id(order_id: str, requesting_user_id: str) -> str | None:
    """Returns an error message, or None if the input is safe to use."""
    if not re.fullmatch(r"ORD-\d{6}", order_id):
        return "order_id must look like ORD-123456."
    if not order_belongs_to_user(order_id, requesting_user_id):
        return "This order does not belong to the current user."
    return None
```

Run this check first, and return the error as a normal tool result if it fails — the same
pattern as any other tool error. This matters most for tools that write data (create a ticket,
cancel an order, send an email) or that touch another user's data. Chapter 14 (Guardrails &
Security) goes deeper on this: validating inputs against injected instructions, limiting which
tools can run without a human check, and sandboxing tools that shell out or write files. Treat
every tool as a small API endpoint that an untrusted client (the model, which can be steered by
untrusted text it read) is calling. Design it with the same care.

### 1.5 Running tools in parallel

A model can ask for **more than one tool call in a single turn**. When `resp.stop_reason ==
"tool_use"`, look through `resp.content` — it may contain several `tool_use` blocks, for example
one call to `get_order_status` and one to `search_products`, because the user asked two things at
once ("where's my order, and do you sell phone cases?"). Running these one after another wastes
time if they are independent. Run them at the same time with a thread pool, then send all the
results back together, in one `user` message, each tagged with its own `tool_use_id`:

```python
import json
from concurrent.futures import ThreadPoolExecutor

def handle_tool_use_turn(resp) -> dict:
    """Run every tool_use block in resp.content in parallel, return one user message."""
    tool_calls = [block for block in resp.content if block.type == "tool_use"]

    with ThreadPoolExecutor(max_workers=len(tool_calls)) as pool:
        futures = {
            pool.submit(run_tool, call.name, call.input): call
            for call in tool_calls
        }
        results = {}
        for future, call in futures.items():
            results[call.id] = future.result()  # run_tool never raises, see 1.3

    return {
        "role": "user",
        "content": [
            {
                "type": "tool_result",
                "tool_use_id": call.id,
                "content": json.dumps(results[call.id]),
            }
            for call in tool_calls
        ],
    }
```

Because `run_tool` always returns a dict (errors included, see 1.3) instead of raising, one
failing tool call cannot stop the others from finishing. This is safe to parallelize precisely
*because* we already removed exceptions from the tool layer. If your tools call external APIs
with rate limits, cap `max_workers` instead of using one worker per call. For very large tool
sets or slow I/O-bound tools, `asyncio` with an async client is another option — the shape of the
result is the same.

---

## 2. Memory: Two Kinds, Different Jobs

An agent that forgets everything after each message is not very useful. "Memory" in an LLM
system is not one thing — it is two different mechanisms, solving two different problems.

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│      SHORT-TERM MEMORY       │        │       LONG-TERM MEMORY        │
│  the message list you send   │        │   facts stored outside the    │
│  to the model every call     │        │   model, fetched when needed  │
│                               │        │                                │
│  • lives inside the request  │        │  • lives in a database or     │
│  • grows every turn          │        │    a vector store              │
│  • limited by context window │        │  • persists across sessions   │
│  • gone when the chat ends   │        │  • fetched by lookup or search │
└───────────────────────────────┘        └──────────────────────────────┘
```

**Short-term memory** is simply the conversation history: the list of `messages` you pass to
`client.messages.create()` on every call. Each turn adds to it. A **context window** is the
maximum number of tokens (pieces of text) a model can read in one call. Claude's context window
is large, but not infinite, and every token you send costs money and time. Left unmanaged, a
long support chat, or an agent that loops through many tool calls, will eventually fill it.

**Long-term memory** is information stored *outside* the model, in a normal database or a
**vector store** (a database built for similarity search over embeddings — see Chapter 4). You
fetch from it only when needed, and inject the result into the prompt for that one call. It
survives after the conversation ends, because it lives in storage, not in the message list.

| | Short-term memory | Long-term memory |
|---|---|---|
| Where it lives | The `messages` list, in the request | A database or vector store, outside the model |
| Survives past the chat? | No — cleared when the session ends | Yes — read again in a future session |
| How you access it | Automatically — it's just sent every call | Deliberately — you write a lookup or search step |
| Typical use | The current back-and-forth of one conversation | User profile, past tickets, product catalog, prior decisions |
| Limit | The context window (tokens) | Storage size — much larger, effectively unbounded |

A simple rule of thumb: **if the agent needs it right now, in this exchange, keep it in
short-term memory. If the agent might need it again next week, or if it is too large to keep
resending every turn, put it in long-term memory and fetch it on demand.**

### 2.1 Managing short-term memory: summarization and compaction

Every message you keep costs tokens on every future call, because the whole history is resent
each time. You can check how many tokens a message list currently costs with
`client.messages.count_tokens()`:

```python
count = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=conversation_history,
)
print(count.input_tokens)
```

When that number gets large — say, past a threshold you pick, like 6,000 tokens for a support
chat — **summarize and compact**: replace the older turns with a short summary, and keep only the
summary plus the most recent few turns. The summary is produced by the model itself, in a small,
separate call:

```python
def summarize_conversation(history: list[dict]) -> str:
    """Ask the model to compress older turns into a short summary."""
    transcript = "\n".join(
        f"{m['role']}: {m['content']}" for m in history if isinstance(m["content"], str)
    )
    resp = client.messages.create(
        model="claude-haiku-4-5",  # cheap model: this is a simple summarization task
        max_tokens=300,
        system=(
            "Summarize this support conversation in under 100 words. "
            "Keep names, order IDs, and any decisions made. Drop small talk."
        ),
        messages=[{"role": "user", "content": transcript}],
    )
    return "".join(b.text for b in resp.content if b.type == "text")


def compact_history(history: list[dict], keep_last: int = 4) -> list[dict]:
    """Replace all but the last few turns with one summary message."""
    if len(history) <= keep_last:
        return history
    older, recent = history[:-keep_last], history[-keep_last:]
    summary = summarize_conversation(older)
    summary_message = {"role": "user", "content": f"[Earlier conversation summary]: {summary}"}
    return [summary_message] + recent
```

Call `compact_history` right before you build the next request, once `count_tokens` crosses your
threshold. This is the same idea Anthropic's beta **context editing** and **compaction** features
automate on the API side for long-running agents — worth knowing they exist (Chapter 13 covers
long-running agents in more depth), but the manual version above shows exactly what they are
doing under the hood, and works today without any beta flag.

Note that summarizing loses detail. That is the trade-off: you keep the conversation usable
inside the context window, at the cost of exact wording from early turns. This is exactly why
important facts — the user's name, a past ticket number, a policy decision — should not depend on
surviving a summary. They belong in long-term memory instead, stored once, and fetched again
whenever needed, in full detail.

### 2.2 Managing long-term memory: lookup and similarity search

Long-term memory has two common access patterns:

1. **Lookup by key** — "get the profile for user `U-4471`." A normal database call: exact match,
   fast, cheap. Good for structured facts like a name, an account tier, or a list of order IDs.
2. **Similarity search** — "find past tickets like *this new one*, even if the wording is
   different." This needs **embeddings**: a model turns text into a list of numbers that captures
   its meaning, and a **vector store** finds the stored items whose numbers are closest to the
   query's numbers. The Claude API does **not** produce embeddings — there is no
   `client.embeddings` method. Use a separate embeddings model such as **Voyage AI**
   (`voyageai`, model `voyage-3`), or the free, local `sentence-transformers` library, and store
   the vectors in a vector store such as **Chroma**.

Use lookup for facts you will retrieve the same way every time. Use similarity search for facts
you want to retrieve by *meaning* — "has this user had a similar problem before?" — where you
cannot predict the exact wording in advance.

---

## 3. Hands-On: Giving the GlobalMart Assistant Long-Term Memory

We will add two small stores to the assistant:

1. A **profile store** (SQLite) — remembers the user's name across sessions, looked up by
   `user_id`.
2. A **ticket memory** (Chroma + Voyage embeddings) — remembers past support tickets, and can
   find tickets similar to a new issue, even across separate chat sessions.

Then we wire both into the agent loop, and add the compaction step from Section 2.1 so a long
session never overflows the context window.

### 3.1 The profile store (lookup memory)

```python
import sqlite3

conn = sqlite3.connect("globalmart_memory.db")
conn.execute(
    "CREATE TABLE IF NOT EXISTS profiles (user_id TEXT PRIMARY KEY, name TEXT)"
)
conn.commit()


def remember_name(user_id: str, name: str) -> None:
    conn.execute(
        "INSERT INTO profiles (user_id, name) VALUES (?, ?) "
        "ON CONFLICT(user_id) DO UPDATE SET name=excluded.name",
        (user_id, name),
    )
    conn.commit()


def recall_name(user_id: str) -> str | None:
    row = conn.execute("SELECT name FROM profiles WHERE user_id = ?", (user_id,)).fetchone()
    return row[0] if row else None
```

This is plain SQL. No embeddings, no model call. A name is an exact fact, so exact lookup is the
right tool — do not reach for a vector store when a key-value lookup solves the problem.

### 3.2 The ticket memory (similarity-search memory)

Past tickets need similarity search, because a user will not repeat the exact words of an old
ticket. "My package never arrived" and "order still hasn't shown up" should match. We embed each
ticket with Voyage AI and store the vector in Chroma.

```python
import chromadb
import voyageai

voyage = voyageai.Client()  # reads VOYAGE_API_KEY from the environment
chroma_client = chromadb.PersistentClient(path="./globalmart_chroma")
tickets = chroma_client.get_or_create_collection("support_tickets")


def embed(texts: list[str]) -> list[list[float]]:
    result = voyage.embed(texts, model="voyage-3", input_type="document")
    return result.embeddings


def store_ticket(user_id: str, ticket_id: str, issue: str) -> None:
    tickets.add(
        ids=[ticket_id],
        embeddings=embed([issue]),
        documents=[issue],
        metadatas=[{"user_id": user_id}],
    )


def find_similar_tickets(user_id: str, new_issue: str, k: int = 3) -> list[str]:
    query_vector = embed([new_issue])[0]
    result = tickets.query(
        query_embeddings=[query_vector],
        n_results=k,
        where={"user_id": user_id},  # only this user's own tickets
    )
    return result["documents"][0] if result["documents"] else []
```

`store_ticket` runs every time `create_support_ticket` is called, so the memory grows over time.
`find_similar_tickets` runs the search: it turns the new issue into a vector, and asks Chroma for
the closest stored vectors that belong to the same user. The `where` filter matters for privacy —
without it, one user's search could return another user's ticket text.

### 3.3 Wiring memory into the agent loop

At the start of a session, look up the user's name. Before creating a new ticket, search past
tickets and give the model that context, so it can say "I see you had a similar delivery issue
last month — is this the same order?" instead of starting cold.

```python
def build_system_prompt(user_id: str, current_issue: str | None) -> str:
    name = recall_name(user_id)
    parts = ["You are the GlobalMart support assistant."]

    if name:
        parts.append(f"The user's name is {name}. Greet them by name.")

    if current_issue:
        similar = find_similar_tickets(user_id, current_issue)
        if similar:
            bullet_list = "\n".join(f"- {t}" for t in similar)
            parts.append(
                "This user has had these related past tickets:\n" + bullet_list +
                "\nMention them if relevant, but do not assume this is the same problem."
            )

    return "\n".join(parts)
```

This function builds the `system` prompt fresh, every call, from long-term storage — the model
itself holds nothing between sessions. That is the core idea of long-term memory: the model stays
stateless, and *your code* is what remembers, by reading from storage and writing it into the
next prompt.

### 3.4 Putting it together: one session

```python
def run_turn(user_id: str, history: list[dict], user_message: str) -> list[dict]:
    history.append({"role": "user", "content": user_message})

    # Compact short-term memory if it has grown large.
    token_count = client.messages.count_tokens(
        model="claude-opus-4-8", messages=history
    ).input_tokens
    if token_count > 6000:
        history = compact_history(history)

    system_prompt = build_system_prompt(user_id, current_issue=user_message)

    resp = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        system=system_prompt,
        tools=tools,
        messages=history,
    )

    while resp.stop_reason == "tool_use":
        history.append({"role": "assistant", "content": resp.content})
        tool_result_message = handle_tool_use_turn(resp)  # from Section 1.5
        history.append(tool_result_message)

        # If a ticket was just created, save it to long-term memory.
        for block in resp.content:
            if block.type == "tool_use" and block.name == "create_support_ticket":
                store_ticket(
                    user_id=block.input["user_id"],
                    ticket_id=f"T-{block.id[:8]}",
                    issue=block.input["issue"],
                )

        resp = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=system_prompt,
            tools=tools,
            messages=history,
        )

    history.append({"role": "assistant", "content": resp.content})
    return history
```

Run this once, end the process, and start a fresh Python session tomorrow with a new, empty
`history` list. The user's name still greets them, and if they raise a similar issue again, the
assistant remembers the earlier ticket — because that information never lived in `history` at
all. It lived in `globalmart_memory.db` and `globalmart_chroma`, read back in on demand.

---

## What You Built / Learned

- What makes a tool easy for a model to use well: a clear name, a description that says *when* to
  use it, and a small, focused input schema.
- Why to keep the tool set small and each tool doing one job, instead of one big tool with modes.
- Why tool errors should come back as normal tool results (a dict with an `"error"` field), never
  as a raised exception that crashes the agent loop.
- How to validate tool inputs (shape *and* safety) before running them, as a first line of
  defense — with full guardrail coverage coming in Chapter 14.
- How to run several tool calls from one model turn in parallel with a thread pool, and send all
  results back together, matched by `tool_use_id`.
- The difference between **short-term memory** (the message list, inside the context window,
  gone when the chat ends) and **long-term memory** (a database or vector store outside the
  model, fetched on demand, persists across sessions).
- How to measure conversation size with `client.messages.count_tokens()`, and how to compact a
  long conversation by summarizing older turns into one short message.
- How to build a small long-term memory system for the GlobalMart Assistant: a SQLite lookup for
  the user's name, and a Chroma + Voyage AI similarity search over past support tickets — both
  read fresh into the prompt at the start of every new session.

## Production Notes & Pitfalls

- **Tool descriptions are prompts, not comments.** Treat them with the same care as your system
  prompt. A vague description ("looks up orders") causes wrong tool picks far more often than a
  precise one ("use this when the user asks where their order is; do not use it for product
  questions").
- **Never let a tool raise into the agent loop.** One unhandled exception in one tool, deep in a
  multi-step agent run, should not take down the whole run. Catch it, log it, and return an error
  result the model can react to.
- **Don't trust the schema as your only safety net.** A JSON schema checks shape, not
  permissions. Always check that the requesting user actually owns the resource (order, ticket,
  account) they are asking about — this is a common real-world security gap.
- **Parallel tool calls need a concurrency limit.** If tools call rate-limited external APIs,
  running ten of them at once can trip the rate limit and turn a fast path into a slow, retried
  one. Cap `max_workers` to match what the downstream systems can take.
- **Summarization is lossy — do not rely on it for facts that matter.** A summary can drop a
  detail the user mentioned three turns ago. Anything that must survive exactly (an order ID, a
  legal commitment, a refund amount) belongs in long-term memory, written once as a structured
  fact, not left to survive inside a shrinking conversation summary.
- **Vector search needs the same access control as your database.** Without a `where` filter on
  `user_id` (or an equivalent), a similarity search over "all tickets" can leak one customer's
  ticket text into another customer's session. Filter by ownership, not just by similarity score.
- **Long-term memory can go stale.** A stored fact ("user's tier: Gold") can become wrong after
  the source system changes. Prefer looking up frequently-changing facts from the system of
  record at request time, and reserve your own long-term store for things that do not have a
  better source — history, past interactions, and preferences.
- **Compaction and context editing are becoming platform features.** Anthropic's beta context
  editing and compaction do a version of Section 2.1 automatically for long-running agents
  (Chapter 13). It is still worth knowing the manual version — it is what you fall back to today,
  and it explains what those beta features are doing for you later.
