# Chapter 7 — From Calls to Agents

You now know how to make one Claude call (Chapter 1), write good prompts (Chapter 2), get
structured output and call tools (Chapter 3), and build RAG (Chapters 4–6). That covers most of
what a GenAI feature needs. But some tasks need more than one call. This chapter shows you three
ways to chain calls together, and how to pick the simplest one that works.

An **agent** is a system where the LLM itself decides which steps to run, in what order, using a
loop. That sounds powerful. It is also the most expensive, slowest, and hardest-to-debug option
on the table. Most of this chapter is about *not* reaching for it until you need it. Then we
build one from scratch, so you understand exactly what a framework does later when it hides the
loop from you (Chapters 10–11).

---

## 7.1 Three levels of "LLM-powered system"

Think of these as three levels. Each level gives the LLM more control, and each level costs you
more predictability.

**Level 1 — A single call.**
You send one prompt. Claude sends back one answer. No loop, no branching. This is what Chapters
1–3 covered. Example: "Summarize this support ticket," or "Extract the order ID from this text
as JSON."

**Level 2 — A workflow.**
You write the steps in your own code (Python `if`/`for`/function calls). The LLM fills in one or
more of those steps, but it does not choose which steps run. You, the engineer, control the
control flow. Example: "Step 1: classify the ticket type with Claude. Step 2: if it is a refund
request, run a refund-policy RAG lookup. Step 3: draft a reply with Claude, using the retrieved
policy text."

**Level 3 — An agent.**
The LLM decides which tool to call, how many times, and when to stop. Your code just runs the
loop and executes whatever the LLM asks for. Example: "Handle this customer message." The LLM
might check order status, then search a knowledge base, then decide it needs a human, all
without you coding that exact sequence in advance.

```
Level 1: Single call
  input ──► [ Claude ] ──► output

Level 2: Workflow (you control the steps)
  input ──► [ Claude: classify ] ──► [ your code: branch ] ──► [ Claude: draft reply ] ──► output

Level 3: Agent (Claude controls the steps)
  input ──► [ Claude decides + acts, in a loop, until it decides it's done ] ──► output
```

A simple table to compare them:

| Level | Who decides the steps | Predictability | Cost & latency | Debugging |
|---|---|---|---|---|
| 1. Single call | N/A — one step | Highest | Lowest | Easiest |
| 2. Workflow | You (in code) | High | Low–medium | Easy — steps are fixed |
| 3. Agent | The LLM (in a loop) | Lower | Higher — multiple calls | Harder — path varies per run |

### Why this matters

A workflow runs the same steps every time (the *inputs* to each step vary, not the *sequence* of
steps). That makes it easy to test, log, and reason about. An agent can take a different path
every time, even for the same question, because the LLM is choosing the path. That flexibility is
useful when the task is genuinely open-ended. It is a liability when the task is not.

**The rule for this whole book: use the simplest level that works. Do not reach for an agent when
a workflow will do.** An agent adds real cost: more API calls (more money, more latency), and a
less predictable path (harder to test, harder to debug when it goes wrong). Only pay that cost
when the task actually needs it.

### A quick check before you build an agent

Ask these four questions. If you answer "no" to any of them, stay at Level 1 or 2.

1. **Is the task hard to fully specify in advance?** If you can write the steps as a fixed list,
   it is a workflow, not an agent.
2. **Does the outcome justify the extra cost and latency?** An agent might take 3–8 calls to
   answer one question. Is that worth it here?
3. **Is Claude actually good at this kind of task?** Some tasks (rigid, mechanical ones) are
   better served by plain code, not an LLM decision at every step.
4. **Can you recover from a mistake?** Agents sometimes pick the wrong tool or the wrong order.
   Is there a way to catch that (tests, a review step, a rollback) before it causes real damage?

For GlobalMart, "What is your return policy?" is a single RAG call (Level 1: retrieve, then
answer — the retrieval step is fixed, so it is really a tiny workflow, not an agent). But "Help
me with my order" could mean many things: check status, request a refund, open a ticket, ask a
policy question. The right tool depends on what the customer actually says. That is a good fit
for an agent — which is exactly the feature we build in this chapter.

---

## 7.2 What is an agent, exactly?

Here is the simple definition this book uses:

> An **agent** is an LLM, plus a set of **tools** it can call, running in a **loop**, optionally
> with **memory** of what happened earlier.

That is the whole idea. There is no magic. The LLM does not "run" anything by itself — it only
outputs *text describing which tool to call and with what input*. Your Python code is the one
that actually runs the tool, and hands the result back. The "intelligence" is in the LLM deciding
*when* to call a tool and *how* to use the result. The "agent" behavior comes from repeating this
back-and-forth until the LLM decides it has enough information to answer.

### The ReAct idea

The pattern we use is called **ReAct**, short for **Reason + Act**. It is simple:

1. **Reason.** The LLM looks at the conversation so far and thinks about what to do next.
2. **Act.** If it needs more information, it calls a tool (a function you gave it).
3. **Observe.** Your code runs that tool and sends the result back to the LLM.
4. Repeat from step 1, using the new information, until the LLM decides it is done and gives a
   final answer.

```
      ┌─────────────────────────────────────────────────┐
      │                                                   │
      ▼                                                   │
  [ REASON ]  Claude reads the conversation so far.       │
      │       Decides: "answer now" or "call a tool".     │
      ▼                                                   │
  stop_reason == "tool_use"?                               │
      │                                                   │
     yes                                                  no
      │                                                    │
      ▼                                                    ▼
  [ ACT ]  Your Python code runs the tool.            [ DONE ]
      │                                              Final answer
      ▼                                              returned to
  [ OBSERVE ]  Send the tool's result                the user.
      │        back to Claude as a
      │        "tool_result" message.
      └──────────────► back to REASON
```

This loop is exactly the tool-calling pattern from Chapter 3, just repeated instead of run once.
In Chapter 3, you called a tool, got the result, and were probably done. In an agent, you keep
going: feed the result back, let the LLM reason again, and let it decide whether it needs another
tool call or is ready to answer.

---

## 7.3 Building a minimal agent loop, from scratch

Let's build this in plain Python, using only the `anthropic` SDK — no framework. This is the same
tool-definition style from Chapter 3: each tool has a `name`, a `description`, and an
`input_schema` (a JSON schema describing its arguments).

### Step 1 — Define the tools

We give the agent two tools, matching the GlobalMart Assistant's action list from the
introduction: `get_order_status` and `search_products`.

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment

tools = [
    {
        "name": "get_order_status",
        "description": (
            "Look up the current status of a GlobalMart order by its order ID. "
            "Use this when the customer asks about a specific order they already placed."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "The GlobalMart order ID, e.g. 'ORD-5001'.",
                }
            },
            "required": ["order_id"],
        },
    },
    {
        "name": "search_products",
        "description": (
            "Search the GlobalMart product catalog by keyword. "
            "Use this when the customer is looking for a product, not an existing order."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search keywords."}
            },
            "required": ["query"],
        },
    },
]
```

### Step 2 — Write the real functions behind the tools

These are plain Python functions. In a real system, `get_order_status` would call GlobalMart's
order database. Here we use a small mock so the example runs on its own.

```python
_ORDERS = {
    "ORD-5001": {"status": "shipped", "eta_days": 2},
    "ORD-5002": {"status": "processing", "eta_days": 5},
}

_PRODUCTS = {
    "headphones": ["Wireless Headphones Pro", "Kids Headphones Mini"],
    "kettle": ["Electric Kettle 1.5L"],
}


def get_order_status(order_id: str) -> dict:
    if order_id not in _ORDERS:
        # A real, expected error: the order simply does not exist.
        raise ValueError(f"No order found with ID '{order_id}'.")
    return _ORDERS[order_id]


def search_products(query: str) -> dict:
    matches = [
        item
        for keyword, items in _PRODUCTS.items()
        if keyword in query.lower()
        for item in items
    ]
    return {"matches": matches}


# A simple dispatch table: tool name -> Python function.
TOOL_FUNCTIONS = {
    "get_order_status": get_order_status,
    "search_products": search_products,
}
```

Note the `raise ValueError(...)` inside `get_order_status`. We do this on purpose. Real tools
fail — a bad order ID, a timeout calling a database, a permission error. The agent loop must
handle that without crashing. We will catch it in Step 3.

### Step 3 — The loop itself

This is the core of the chapter. Read it slowly — every line matters.

```python
import json

SYSTEM_PROMPT = (
    "You are the GlobalMart Assistant. Help customers with orders and products. "
    "Use the tools when you need real data. If you already know the answer "
    "(for example, a general question that needs no lookup), answer directly."
)


def run_tool(name: str, tool_input: dict) -> str:
    """Run one tool and return a JSON string result, or raise on failure."""
    fn = TOOL_FUNCTIONS[name]
    result = fn(**tool_input)
    return json.dumps(result)


def run_agent(user_message: str, max_steps: int = 5) -> str:
    messages = [{"role": "user", "content": user_message}]

    for step in range(max_steps):
        resp = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            tools=tools,
            messages=messages,
        )

        # Always append what the model said, so it stays in the conversation history.
        messages.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason != "tool_use":
            # The model is done reasoning and has given a final answer.
            return "".join(
                block.text for block in resp.content if block.type == "text"
            )

        # stop_reason == "tool_use": run every requested tool, collect the results.
        tool_results = []
        for block in resp.content:
            if block.type != "tool_use":
                continue  # a text block can appear alongside a tool_use block

            print(f"  [step {step + 1}] calling {block.name}({block.input})")
            try:
                output = run_tool(block.name, block.input)
                tool_results.append(
                    {
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": output,
                    }
                )
            except Exception as exc:
                # Tool failed. Tell the model, so it can recover — retry with a
                # different input, apologize, or try a different tool.
                tool_results.append(
                    {
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": f"Error: {exc}",
                        "is_error": True,
                    }
                )

        # Send all results back in ONE user message, matching each tool_use_id.
        messages.append({"role": "user", "content": tool_results})

    return "I could not finish this request in time. Please rephrase or try again."
```

Walk through the pieces:

- **The loop condition is `resp.stop_reason == "tool_use"`.** This is the exact rule from
  Chapter 3, now driving a `for` loop instead of an `if`. As long as Claude keeps asking for
  tools, we keep running them and calling the API again.
- **Every `tool_use` block gets a matching `tool_result` block**, linked by `tool_use_id`. If
  Claude asks for two tools in one turn (it can — this is called *parallel tool use*), you must
  return both results in a single user message. Never split them across separate messages; doing
  so trains the model to stop batching tool calls.
- **We append `resp.content` itself**, not just the text, as the assistant's turn. The
  `tool_use` blocks need to stay in history so the API can match them to your `tool_result`
  blocks on the next call.
- **The `try`/`except` around `run_tool`** is what makes the agent robust. If
  `get_order_status("ORD-9999")` raises `ValueError`, we do not crash the loop. We send back a
  `tool_result` with `is_error=True` and the error text. Claude sees this like a person would:
  "that lookup failed, the ID may be wrong" — and it can apologize, ask the customer to check the
  ID, or try something else. Dropping a failed tool call silently, instead of returning an error
  result, is a common bug: the model is left waiting for a result that never arrives.
- **`max_steps` stops an infinite loop.** Nothing in the API stops Claude from asking for tools
  forever, in theory (in practice this is rare, but it can happen — a confused model retrying a
  broken tool). The `for step in range(max_steps)` loop guarantees your program always returns,
  even in the worst case. Five to ten steps is a reasonable ceiling for most agents; pick a
  number based on how many tool calls a real task should ever need.

### Stopping conditions, in full

`stop_reason` can be a few different values. Your loop should know what each one means:

| `stop_reason` | Meaning | What to do |
|---|---|---|
| `"tool_use"` | Claude wants to call one or more tools. | Run them, send back `tool_result`s, loop again. |
| `"end_turn"` | Claude has a final answer, no more tools needed. | Read the text blocks, return to the caller. |
| `"max_tokens"` | The response was cut off — `max_tokens` was too small. | Increase `max_tokens`, or treat as an error. |
| `"stop_sequence"` | Claude hit a custom stop string you configured. | Rare in agent loops; handle if you set one. |

The loop above only branches on `"tool_use"` vs. everything else, which is fine for a simple
agent. A production system should check for `"max_tokens"` explicitly and retry with more room,
rather than silently returning a truncated answer as if it were complete.

---

## 7.4 Hands-on: turning the GlobalMart Assistant into an agent

Let's run the loop we just built, with two different questions. Watch how the agent chooses what
to do — we never tell it which tool to use.

```python
if __name__ == "__main__":
    print("Q1: Where is my order ORD-5001?")
    print(run_agent("Where is my order ORD-5001?"))
    print()

    print("Q2: Do you sell kettles?")
    print(run_agent("Do you sell kettles?"))
    print()

    print("Q3: What's the status of order ORD-9999?")
    print(run_agent("What's the status of order ORD-9999?"))
```

What happens for each question:

- **Q1** — Claude sees "order ORD-5001," recognizes it needs real data, and calls
  `get_order_status(order_id="ORD-5001")`. The tool returns `{"status": "shipped", "eta_days": 2}`.
  Claude then writes a final answer like "Your order has shipped and should arrive in about 2
  days." That is two API calls total: one that asks for the tool, one that reads the result and
  answers.
- **Q2** — Claude recognizes this is a product search, not an order lookup, and calls
  `search_products(query="kettles")` instead. Same loop, different tool — the agent chose it based
  on the question, not on code you wrote to branch on keywords.
- **Q3** — Claude calls `get_order_status(order_id="ORD-9999")`. Our mock function raises
  `ValueError` because that order does not exist. The loop catches it, sends back
  `{"is_error": true, "content": "Error: No order found with ID 'ORD-9999'."}`. Claude reads that
  and answers something like "I could not find an order with that ID. Could you double check the
  order number?" — a graceful recovery, not a crash.

This is the whole point of Level 3. You did not write an `if "order" in message` branch anywhere.
You gave the LLM tools and a description of when to use each one, and it picked correctly for
three different kinds of questions, including one that failed cleanly.

Compare this to a **workflow** version of the same feature: you would write
`if "order" in message: call get_order_status(...) elif "product" in message or "sell" in
message: call search_products(...) else: answer directly`. That workflow is faster, cheaper (one
call, not two), and 100% predictable — but it breaks the moment a customer phrases things in a
way your keyword rules did not expect ("has my package shipped yet?"). The agent handles that
case for free, because Claude is doing the routing, not a keyword match. That flexibility is
exactly what you are paying the extra latency and cost for.

---

## 7.5 What frameworks add on top of this

You just wrote the entire agent loop by hand: about 40 lines of Python. That is deliberate — you
should know exactly what is happening before you let a framework hide it. Once you understand
this loop, two things become useful:

- **The Anthropic SDK's Tool Runner** (a beta helper) removes the loop boilerplate for you. You
  decorate your Python functions, hand them to `client.beta.messages.tool_runner(...)`, and call
  `.until_done()`. It runs the same reason → act → observe cycle internally, with hooks for
  things like approval gates or logging on each turn. It is still *your* tools and *your* hosting
  — just less code to write.
- **LangChain and LangGraph** (Chapters 10–11) go further: they add typed message objects, a graph
  structure for more complex control flow (branches, cycles, human-in-the-loop pauses), and
  built-in state persistence. LangGraph in particular models an agent as a graph with a loop edge
  — the exact same idea as our `for` loop, just drawn as nodes and edges so you can add branches
  without rewriting the loop by hand.

Every one of these still does what our loop does underneath: call the model, check if it wants a
tool, run the tool, send the result back, repeat. Knowing that means you can debug a LangGraph
agent by asking the same three questions you'd ask about our raw loop: what did the model decide
to do, what did the tool return, and did that get back into the next call correctly?

---

## What You Built / Learned

- The three levels of LLM-powered systems: a **single call** (Level 1), a **workflow** where you
  control the steps and the LLM fills some in (Level 2), and an **agent** where the LLM chooses
  the steps itself, in a loop (Level 3).
- The core rule: **use the simplest level that works.** Do not reach for an agent when a workflow
  will do — an agent costs more in latency, money, and predictability.
- A simple definition of an agent: **an LLM, plus tools, running in a loop, optionally with
  memory.**
- The **ReAct** pattern: reason (the model decides what to do), act (your code runs a tool),
  observe (the result goes back to the model), repeat until the model is done.
- A minimal agent loop built from scratch in Python: a `for` loop over `client.messages.create`,
  branching on `stop_reason == "tool_use"`, matching every `tool_use` block to a `tool_result`
  block by `tool_use_id`.
- A **`max_steps`** limit, so the loop always terminates even if the model keeps requesting
  tools.
- **Tool error handling**: catch exceptions from your tool functions, send back
  `{"is_error": true, ...}` instead of crashing, so the model can recover.
- A working example: the GlobalMart Assistant choosing between `get_order_status` and
  `search_products` on its own, based on the question — no keyword-matching code required.
- Frameworks (Chapters 10–11) and the SDK's beta Tool Runner implement this exact loop for you,
  with less boilerplate — but the underlying mechanics are what you just built.

## Production Notes & Pitfalls

- **Agents are not always the answer.** Teams often reach for an agent because it feels more
  "AI-native," then discover a fixed workflow would have been cheaper, faster, and easier to test.
  Re-check the four questions in §7.1 before adding a loop to any feature.
- **Every extra step is a full API call.** A three-step agent run costs roughly three times what
  a single call costs, plus the added latency of three round trips. Budget for this before
  shipping — Chapter 12 covers cost and latency controls (caching, model routing) in depth.
  Chapter 13 covers what happens when an agent needs to run far longer than a few steps.
- **Unbounded loops are a real production risk.** Without a `max_steps` limit (or a token budget,
  or a wall-clock timeout), a confused agent can call tools dozens of times, burning money and
  time before anyone notices. Always cap it, and log when the cap is hit — that log line usually
  means a tool description was unclear or a tool is returning something the model cannot use.
- **Silently swallowing tool errors is worse than surfacing them.** If you catch an exception and
  return nothing, or an empty string, the model may hallucinate a plausible-looking answer instead
  of telling the user something went wrong. Always send a clear `is_error` result back, and write
  tool descriptions that tell the model what a failure might mean (bad ID vs. system down).
- **Non-determinism makes testing harder.** The same input can take a different path through the
  loop on different runs — a different tool, a different order of tool calls, sometimes no tool
  call at all. Test at the level of *outcomes* ("did it correctly report the order status") rather
  than *exact call sequences*, and build a small eval set (Chapter 6's approach extends naturally
  to agents) rather than relying only on manual spot checks.
- **Tool descriptions are part of your prompt engineering.** The model picks a tool based on its
  `name` and `description`, the same way it reads any other instruction. A vague description
  ("gets order info") leads to wrong tool choices far more often than a bad user question does.
  Chapter 8 goes deeper into designing tools well and adding memory across turns.
- **Watch parallel tool calls.** Claude may return more than one `tool_use` block in a single
  response. If your code only handles one and ignores the rest, the conversation history becomes
  inconsistent (a `tool_use` block with no matching `tool_result`), and the next API call will
  error. Always loop over *all* blocks in `resp.content`.
- **Memory is a separate concern from the loop.** This chapter's agent only remembers what
  happened inside one call to `run_agent`. A real assistant needs to remember earlier turns from
  the same conversation, and sometimes facts from past sessions. Chapter 8 covers short-term and
  long-term memory design.
