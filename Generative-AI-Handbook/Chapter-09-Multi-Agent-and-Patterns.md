# Chapter 9 — Multi-Agent & Patterns

Chapter 7 built one agent: one loop, one system prompt, a set of tools, and memory. That
covers most tasks. This chapter asks a harder question: when is one agent not enough, and
when does adding more agents make things *worse*? Then it walks through five patterns for
splitting work across steps or agents, with small Python examples for each. It ends with a
hands-on build: a router for the GlobalMart Assistant that sends policy questions to the RAG
path and order problems to the tool/action path.

The theme repeats through this whole book: **use the simplest design that works.** Single
call < workflow < single agent < multi-agent. Multi-agent is the most expensive and hardest
to debug option on that list. Reach for it only when you have a clear reason.

## 1. When one agent is not enough

A single agent starts to strain in a few concrete situations:

- **Its system prompt is trying to do too many jobs.** If you find yourself writing "if the
  user asks about X, do this; if they ask about Y, do that entirely different thing," the
  prompt is really two prompts glued together.
- **It has too many tools.** Every tool you add to one agent adds to every prompt it sees.
  Past 15–20 tools, the model starts picking the wrong one more often, and your token bill
  grows on every turn.
- **The task naturally splits into independent chunks.** For example, "summarize these 20
  support tickets" is 20 independent small jobs, not one long one.
- **You need a real separation between "do the work" and "check the work."** One prompt that
  both writes an answer and grades it tends to be lenient on itself.

A *workflow* is fixed code that calls the LLM at one or more steps, in an order you control.
An *agent* is an LLM in a loop, deciding for itself what to do next (calling tools, checking
results) until the task is done, as built in Chapter 7. A *multi-agent system* is more than
one such loop — or more than one LLM call making decisions — with a way to hand work between
them.

## 2. When multi-agent makes things worse

This is just as important as knowing when to use it. Adding agents adds four costs, and they
are easy to underestimate:

| Cost | What happens |
|---|---|
| **Money** | Every extra agent is extra LLM calls. A 3-agent pipeline can easily cost 3–5x a single well-designed agent, for the same user request. |
| **Latency** | Agents that run one after another (not in parallel) add up their response times. A user waiting 20 seconds for an answer that a single call could give in 4 seconds is a real product problem. |
| **Failure points** | Each agent can misread its input, call the wrong tool, or return a badly formatted result. With N agents, you have N places a chain can break, and errors from an early agent quietly poison every agent after it. |
| **Debuggability** | When the final answer is wrong, which agent caused it? You now need to trace a hand-off chain, not read one transcript. |

A good rule: before adding a second agent, ask "could one agent with a better prompt and the
right tools do this?" Most of the time, the answer is yes. Multi-agent earns its cost when the
sub-tasks are genuinely different (different tools, different context, different "voice"), or
when they are independent and can run in parallel to save real time.

## 3. Five common patterns

These patterns are not framework features — they are shapes you build with plain function
calls and `if` statements. Chapters 10–11 show the same shapes again using LangChain and
LangGraph, once you have seen them built from scratch.

### 3.1 Router

**Idea:** one cheap step looks at the request and decides which path handles it. The router
itself does not do the work — it just picks a lane.

```
request --> [router: classify] --> path A (e.g. RAG)
                               --> path B (e.g. tool agent)
                               --> path C (e.g. "I can't help with that")
```

Routing is a **classification** problem: sort the input into one of a few known categories.
This is a great job for a small, cheap model like Claude Haiku 4.5, because the decision is
simple and you may make it on every single request.

```python
import json
import anthropic

client = anthropic.Anthropic()

ROUTE_SCHEMA = {
    "type": "object",
    "properties": {
        "route": {"type": "string", "enum": ["billing", "technical", "general"]},
    },
    "required": ["route"],
    "additionalProperties": False,
}

def route(user_message: str) -> str:
    resp = client.messages.create(
        model="claude-haiku-4-5",  # routing is simple; a cheap model is enough
        max_tokens=100,
        system="Classify the support message into one category.",
        messages=[{"role": "user", "content": user_message}],
        output_config={"format": {"type": "json_schema", "schema": ROUTE_SCHEMA}},
    )
    return json.loads(resp.content[0].text)["route"]
```

**Use it when:** requests fall into a small number of clearly different categories, and each
category needs a different system prompt, tool set, or downstream system.
**Skip it when:** you only have two or three request types and giving one agent all the tools
for all of them is not expensive. A router adds one extra LLM call to *every* request — make
sure that call earns its cost.

### 3.2 Orchestrator–workers

**Idea:** a lead agent (the *orchestrator*) breaks a job into smaller pieces, hands each piece
to a *worker*, and combines the worker outputs into one final result. Workers do not talk to
each other — only to the orchestrator.

```
                 +---------> worker 1 --------+
request -> orchestrator -> worker 2 --------> combine -> final answer
                 +---------> worker 3 --------+
```

```python
def orchestrate(topic: str) -> str:
    # Step 1: orchestrator plans the subtasks
    plan = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=500,
        system="Break the research topic into 3 independent sub-questions. "
               "Return one sub-question per line, nothing else.",
        messages=[{"role": "user", "content": topic}],
    )
    sub_questions = plan.content[0].text.strip().split("\n")

    # Step 2: each worker answers its own sub-question (cheaper model is fine here)
    worker_answers = []
    for q in sub_questions:
        w = client.messages.create(
            model="claude-sonnet-5",
            max_tokens=400,
            system="Answer this sub-question in 2-3 sentences.",
            messages=[{"role": "user", "content": q}],
        )
        worker_answers.append(w.content[0].text)

    # Step 3: orchestrator combines the pieces
    combined = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=800,
        system="Combine these findings into one short, coherent answer.",
        messages=[{"role": "user", "content": "\n\n".join(worker_answers)}],
    )
    return combined.content[0].text
```

In real systems, "worker" often means a whole sub-agent with its own tools — for example, one
worker per data source in a research task. Run independent workers concurrently (with
threads, `asyncio`, or a task queue) to get real latency savings; running them one after
another gives you none of the speed benefit and all of the extra cost.

**Use it when:** the task splits into pieces that do not depend on each other's output.
**Skip it when:** the pieces must happen in a specific order — that is the next pattern.

### 3.3 Planner–executor

**Idea:** one step writes an ordered plan. A second step (often a loop) carries the plan out,
step by step, possibly using tools. This separates "decide what to do" from "go do it."

```
request -> [planner: write steps 1..N] -> [executor: run step 1, step 2, ... step N]
```

```python
PLAN_SCHEMA = {
    "type": "object",
    "properties": {"steps": {"type": "array", "items": {"type": "string"}}},
    "required": ["steps"],
    "additionalProperties": False,
}

def make_plan(goal: str) -> list[str]:
    resp = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=500,
        system="Write an ordered list of steps to reach this goal. Keep it short.",
        messages=[{"role": "user", "content": goal}],
        output_config={"format": {"type": "json_schema", "schema": PLAN_SCHEMA}},
    )
    return json.loads(resp.content[0].text)["steps"]

def execute_plan(steps: list[str], run_step) -> list[str]:
    """run_step(step_text) -> result string. Could itself be a tool-using agent."""
    results = []
    for step in steps:
        results.append(run_step(step))
    return results
```

Planning first is worth it when the order of actions matters and mistakes are costly to
undo — for example, "cancel the order, then refund, then notify the customer" must happen in
that order. A plain agent loop (Chapter 7's ReAct-style loop) already handles most tasks fine
by deciding the next step as it goes; only split out a separate planning step when you have
seen the agent take a bad first action because it did not think ahead.

**Use it when:** step order matters, or you want a plan you can show the user or a human
reviewer before anything runs.
**Skip it when:** the agent loop already picks reasonable next steps on its own — planning
ahead adds an extra call and extra complexity for no measurable gain.

### 3.4 Reflection

**Idea:** the agent writes a first draft, then a second call critiques that draft against
some criteria, then the agent revises. This trades latency and cost for quality.

```
draft -> [critique] -> "good enough"? --no--> revise -> [critique] -> ...
                              |
                             yes
                              v
                        final answer
```

```python
def generate_with_reflection(prompt: str, max_rounds: int = 2) -> str:
    draft = client.messages.create(
        model="claude-opus-4-8", max_tokens=800,
        messages=[{"role": "user", "content": prompt}],
    ).content[0].text

    for _ in range(max_rounds):
        critique = client.messages.create(
            model="claude-opus-4-8", max_tokens=300,
            system="Critique this draft. If it is good, reply exactly 'GOOD'. "
                   "Otherwise list what to fix.",
            messages=[{"role": "user", "content": draft}],
        ).content[0].text

        if critique.strip() == "GOOD":
            break

        draft = client.messages.create(
            model="claude-opus-4-8", max_tokens=800,
            system="Revise the draft using this feedback.",
            messages=[{"role": "user", "content": f"Draft:\n{draft}\n\nFeedback:\n{critique}"}],
        ).content[0].text

    return draft
```

**Always cap the number of rounds** (`max_rounds` above). Without a cap, reflection can loop
for a long time on tasks where "good" is fuzzy, burning tokens with no guarantee of a better
answer.

**Use it when:** output quality matters more than speed or cost — long-form writing, code,
or answers with strict correctness needs.
**Skip it when:** the task is simple lookup or short factual Q&A. Reflecting on "what is the
order status" doubles the cost for no benefit.

### 3.5 Evaluator–optimizer

**Idea:** this generalizes reflection. A *generator* proposes an answer. A separate
*evaluator* scores it against a stated **rubric** (a written scoring guide) and returns
structured feedback. The loop repeats until the score passes a threshold or a max-tries limit
is hit. The evaluator can be a different, stricter model — or even a non-LLM check (a unit
test, a regex, a database lookup).

```python
EVAL_SCHEMA = {
    "type": "object",
    "properties": {
        "score": {"type": "integer", "minimum": 1, "maximum": 5},
        "feedback": {"type": "string"},
    },
    "required": ["score", "feedback"],
    "additionalProperties": False,
}

def generate_and_evaluate(task: str, rubric: str, threshold: int = 4, max_tries: int = 3):
    feedback = ""
    for attempt in range(max_tries):
        gen_prompt = task if not feedback else f"{task}\n\nPrevious feedback:\n{feedback}"
        candidate = client.messages.create(
            model="claude-opus-4-8", max_tokens=600,
            messages=[{"role": "user", "content": gen_prompt}],
        ).content[0].text

        result = client.messages.create(
            model="claude-sonnet-5", max_tokens=300,
            system=f"Score the answer 1-5 against this rubric:\n{rubric}",
            messages=[{"role": "user", "content": candidate}],
            output_config={"format": {"type": "json_schema", "schema": EVAL_SCHEMA}},
        )
        verdict = json.loads(result.content[0].text)

        if verdict["score"] >= threshold:
            return candidate
        feedback = verdict["feedback"]

    return candidate  # best effort after max_tries
```

The difference from reflection: reflection is usually the *same* agent grading its own work in
free text. Evaluator–optimizer uses an explicit, often-different scorer and a numeric or
pass/fail rubric, which makes it easier to log, test, and tune — and it is the same shape you
will reuse for LLM-as-judge evaluation in Chapter 15.

**Use it when:** you can write down what "good" means as a rubric, and you want an automatic
quality gate before a user sees the output.
**Skip it when:** you cannot define the rubric clearly. A vague evaluator gives vague scores,
and the loop just burns tokens without converging.

### 3.6 Picking a pattern

The five patterns answer different questions. Use this table to pick the right one instead of
starting from "let's add an agent":

| Pattern | Question it answers | Extra LLM calls per request | Good fit |
|---|---|---|---|
| Router | "Which path handles this?" | +1 (cheap) | A few clearly different request types |
| Orchestrator–workers | "How do I split this into independent pieces?" | +1 per piece, +1 to combine | Fan-out work with no ordering between pieces |
| Planner–executor | "What order should steps happen in?" | +1 for the plan | Multi-step actions where order matters |
| Reflection | "Is my own answer good, and can I improve it?" | +2 per round | Long-form or code output, quality over speed |
| Evaluator–optimizer | "Does this pass a rubric, checked by someone else?" | +1 per round (evaluator) | You can write a clear pass/fail or scored rubric |

Most production systems use **one, maybe two** of these, not all five at once. A router in
front of a single well-tooled agent already covers a large share of real assistants. Stack in
a second pattern (say, evaluator–optimizer behind the tool path, to check a refund amount
before it is issued) only where a mistake would be expensive.

## 4. How agents pass work to each other

Whatever the pattern, agents need a way to hand off work. There are two basic mechanisms, and
most real systems mix them:

- **Shared state:** a single object (a dict, a database row, a LangGraph `State`) that every
  step reads from and writes to. Simple to reason about — you can print the state at any
  point and see exactly what every step knows. The risk: any step can overwrite a field
  another step needed, so define the state's shape (a schema or a `TypedDict`) up front.
- **Messages:** steps exchange discrete messages, the same shape as the tool-use loop from
  Chapter 7 — one side sends a request, the other sends back a result. This is more flexible
  for open-ended hand-offs (an orchestrator can send a different message to each worker), but
  harder to inspect — you must log every message to see what happened.

With the Claude API, both mechanisms end up as the same thing on the wire: whatever you put
into the next `messages` list you send. A "shared state" design just means you serialize a
dict into a message or a system prompt before the next call; a "messages" design passes
conversation turns directly. Pick shared state when steps mostly need to see the same
information; pick messages when each step needs a different, tailored view of the work so
far. Either way, keep the state or message format simple and typed — this is what Chapter 11
(LangGraph) builds structure around.

## 5. Hands-on: a router for the GlobalMart Assistant

The GlobalMart Assistant (Chapters 4–8) does two different jobs:

1. Answer policy questions from company documents (**RAG path** — Chapters 4–6).
2. Take real actions through tools, like checking an order (**tool/action path** —
   Chapters 7–8).

These two jobs need different things: the RAG path needs a retriever and grounded-answer
prompting; the tool path needs `get_order_status`, `create_support_ticket`, and a loop. Giving
one agent both retrieval instructions *and* every tool, in one prompt, works but grows messy
fast. A router is the simplest fix: one small call decides which path to use, then plain code
sends the request there.

```python
import json
import anthropic

client = anthropic.Anthropic()

ROUTE_SCHEMA = {
    "type": "object",
    "properties": {
        "route": {"type": "string", "enum": ["policy_question", "order_problem"]},
        "reason": {"type": "string"},
    },
    "required": ["route", "reason"],
    "additionalProperties": False,
}

def classify_request(user_message: str) -> dict:
    """One cheap call decides which path handles the request."""
    resp = client.messages.create(
        model="claude-haiku-4-5",  # routing is a simple decision; no need for Opus
        max_tokens=200,
        system=(
            "Classify the user message into exactly one route.\n"
            "- policy_question: a general question about GlobalMart policy — "
            "shipping rules, returns, warranties. No specific order is involved.\n"
            "- order_problem: the user has a specific order that needs checking "
            "or fixing (status, refund, missing item, ticket)."
        ),
        messages=[{"role": "user", "content": user_message}],
        output_config={"format": {"type": "json_schema", "schema": ROUTE_SCHEMA}},
    )
    return json.loads(resp.content[0].text)


def answer_policy_question(question: str) -> str:
    """RAG path. Chapters 4-5 build the real retriever; this is the shape."""
    # In the full assistant: embed `question`, search Chroma for matching
    # policy chunks, then pass those chunks to Claude as context. Here we
    # call Claude directly to keep the router example self-contained.
    resp = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        system="Answer GlobalMart policy questions clearly and briefly.",
        messages=[{"role": "user", "content": question}],
    )
    return resp.content[0].text


ORDER_TOOLS = [
    {
        "name": "get_order_status",
        "description": "Look up the current status of a GlobalMart order.",
        "input_schema": {
            "type": "object",
            "properties": {"order_id": {"type": "string"}},
            "required": ["order_id"],
        },
    },
    {
        "name": "create_support_ticket",
        "description": "Open a support ticket for a problem the agent cannot fix itself.",
        "input_schema": {
            "type": "object",
            "properties": {
                "user_id": {"type": "string"},
                "issue": {"type": "string"},
            },
            "required": ["user_id", "issue"],
        },
    },
]

def run_tool(name: str, tool_input: dict) -> str:
    """Fake tool execution for this example — see Chapter 8 for real handlers."""
    if name == "get_order_status":
        return f"Order {tool_input['order_id']} is 'out for delivery', arriving tomorrow."
    if name == "create_support_ticket":
        return f"Ticket created for user {tool_input['user_id']}: {tool_input['issue']}"
    return "Unknown tool."

def handle_order_problem(user_message: str) -> str:
    """Tool/action path — the agent loop from Chapter 7."""
    messages = [{"role": "user", "content": user_message}]
    system = "You help GlobalMart customers with order problems. Use tools when needed."

    while True:
        resp = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=system,
            messages=messages,
            tools=ORDER_TOOLS,
        )
        messages.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason != "tool_use":
            return "".join(b.text for b in resp.content if b.type == "text")

        tool_results = []
        for block in resp.content:
            if block.type == "tool_use":
                result = run_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })
        messages.append({"role": "user", "content": tool_results})


def globalmart_assistant(user_message: str) -> str:
    route = classify_request(user_message)
    print(f"[router] -> {route['route']}  ({route['reason']})")

    if route["route"] == "policy_question":
        return answer_policy_question(user_message)
    return handle_order_problem(user_message)


if __name__ == "__main__":
    print(globalmart_assistant("What is the return policy for electronics?"))
    print(globalmart_assistant("My order A1029 has not arrived, where is it?"))
```

This is the router pattern, end to end: one Haiku call classifies, plain Python `if/else`
sends the request down the right path, and each path is built with the tools from earlier
chapters. Notice what did *not* happen: there is no orchestrator, no reflection, no separate
"combine" step. Two paths, one cheap classifier — that is the whole system. Add a third route
(for example, `product_search`) only when a real request type shows up that neither existing
path handles well.

A note on router mistakes: `classify_request` can be wrong. For a two-way router like this,
a cheap safety net is to also give the `order_problem` path a way to say "this looks like a
policy question instead" and hand back to the other path, rather than trusting the router's
one guess to always be right. For higher-stakes routing (for example, routing that decides
whether to refund money), add a rule-based check after the classifier — never let a single LLM
call be the only gate in front of an action that costs real money.

## What You Built / Learned

- The difference between a workflow (fixed code path), an agent (one LLM loop deciding its
  own next step), and a multi-agent system (more than one such decision-maker, handing off
  work).
- Why multi-agent costs more money, adds latency, multiplies failure points, and is harder to
  debug — and why "one agent, better prompt" beats multi-agent for most tasks.
- Five patterns: **router** (classify then dispatch), **orchestrator–workers** (split, run,
  combine), **planner–executor** (plan first, then run the plan), **reflection** (self-critique
  and revise, with a hard round cap), and **evaluator–optimizer** (generate, score against a
  rubric, loop to a threshold).
- Two ways agents hand off work: **shared state** (one object everyone reads/writes) and
  **messages** (discrete request/response turns) — and that both reduce to "what goes in the
  next `messages` list" when you use the Claude API directly.
- A working router for the GlobalMart Assistant that sends policy questions to the RAG path
  and order problems to the tool/action path, using Claude Haiku 4.5 for the cheap
  classification step and Claude Opus 4.8 for the real work on each path.

## Production Notes & Pitfalls

- **Measure before you split.** Do not add a router, an orchestrator, or a reflection loop
  because a pattern looks elegant. Add it because you measured a real problem — wrong tool
  picks, a prompt that keeps growing, or a quality bar the model cannot hit in one pass — that
  the pattern actually fixes.
- **Every extra agent hop is a place to add logging.** In a multi-agent chain, log the input
  and output of every step, not just the final answer. Chapter 15 covers tracing; without it,
  a wrong final answer in a 4-step chain is very hard to debug.
- **Cap every loop.** Reflection and evaluator–optimizer both loop until "good enough." Always
  set a `max_rounds` or `max_tries`, and always have a defined fallback (return the best
  attempt, or hand off to a human) when the cap is hit. An uncapped quality loop is a silent
  cost leak.
- **Routers need a fallback route.** A classifier will sometimes be unsure or wrong. Add a
  low-confidence or "none of the above" branch, and consider a second, cheap check before any
  branch that takes a real-world action (refunds, deletions, sending email).
- **Parallel workers need real concurrency to pay off.** Calling workers one after another in
  a `for` loop gives you the cost of orchestrator–workers without the latency benefit. Use
  threads, `asyncio`, or a task queue so independent workers actually run at the same time.
- **Cost adds up per completed task, not per call.** A cheap router call followed by one
  focused agent call is often cheaper *and* faster than one large agent that has to consider
  every tool on every turn — routing is one of the few multi-step patterns that usually saves
  money rather than spending more.
- **Multi-agent output is harder to test.** Chapter 6 and Chapter 15 build evaluation
  harnesses; for multi-agent systems, write tests for each step in isolation (does the router
  pick the right path? does the worker return the expected shape?) in addition to end-to-end
  tests. An end-to-end failure in a 5-step system tells you almost nothing about which step
  broke.
