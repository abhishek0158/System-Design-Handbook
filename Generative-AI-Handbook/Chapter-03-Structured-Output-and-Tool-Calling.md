# Chapter 3 — Structured Output & Tool Calling

Chapters 1 and 2 got Claude to answer in plain text. A person can read plain text. Your code
cannot easily act on it. If Claude replies "Sure, the order shipped on March 2nd, I think," your
program cannot pull out a date with confidence — the wording changes every time.

This chapter fixes that problem in two ways. First, **structured output**: forcing Claude's reply
into a shape your code can parse every time, like a Python object or exact JSON. Second, **tool
calling** (also called function calling): letting Claude ask your code to run a real function —
look up an order, search a catalog — instead of guessing the answer from its training data.

Tool calling is the single most important idea in this book. Every agent in later chapters
(Chapter 7 onward) is really just this chapter's `tool_use` → `tool_result` loop, repeated and
made smarter. Read this chapter slowly. If the loop is clear here, everything after it is easier.

## 1. Why structured output matters

A normal `messages.create()` call returns text. Text is great for a chat UI. It is bad for a
program that needs to do something with the answer — save it to a database, show it in a table,
trigger a workflow.

Say you want to pull a customer's name, email, and requested plan out of a message, and store
them as a row in a table. Asking Claude to "reply in three lines: Name, Email, Plan" (Chapter 2's
approach) usually works. But "usually" is not good enough for code that runs unattended. One
reply might add an extra blank line. Another might write "N/A" instead of leaving a field out.
Your parser breaks on the edge case you did not test.

**Structured output** removes this guesswork. You give Claude a schema — a description of the
exact fields and types you want — and the API guarantees the reply matches that schema. Your code
never needs to guess at Claude's formatting choices again. There are two ways to get this on the
Claude API: a typed Python way (`messages.parse` with a Pydantic model) and a raw JSON Schema way
(`output_config`). We cover both, starting with the one you should default to.

## 2. Typed output with `messages.parse` and Pydantic

**Pydantic** is a Python library for defining data shapes as classes, with automatic validation.
You already may have used it if you have written a FastAPI app. Here, we use a Pydantic model to
describe exactly what we want back from Claude.

```python
import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()


class SupportTicket(BaseModel):
    customer_name: str
    order_id: str | None
    issue_summary: str
    urgency: str  # "low", "medium", or "high"


resp = client.messages.parse(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": (
            "Extract a support ticket from this message: "
            "'Hi, this is Priya. My order ORD-55210 arrived broken and I need "
            "a replacement fast, this is urgent.'"
        ),
    }],
    output_format=SupportTicket,
)

ticket = resp.parsed_output  # a real SupportTicket instance, already validated
print(ticket.customer_name)   # "Priya"
print(ticket.order_id)        # "ORD-55210"
print(ticket.urgency)         # "high"
```

Two things matter here. First, this is `client.messages.parse`, not `client.messages.create`.
`.parse()` is a dedicated method built for structured output. Second, the result is not text you
must parse yourself — `resp.parsed_output` is already a validated `SupportTicket` object, with
real Python attributes and real types. If Claude's raw reply somehow did not match your schema,
the SDK raises an error there, instead of your code crashing three steps later on a missing key.

This is the pattern to default to whenever you are writing Python and you know the shape of data
you want back. It is less code than a `create()` call plus manual JSON parsing, and it fails
loudly and early instead of quietly and late.

## 3. The raw way: `output_config` with a JSON Schema

Sometimes you are not in a place to define a Pydantic model — maybe the schema is built
dynamically, or you are working outside a typed Python data path. For that, use `output_config`
directly on `client.messages.create()`:

```python
resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": "Extract: John Smith (john@example.com) wants the Enterprise plan.",
    }],
    output_config={
        "format": {
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "email": {"type": "string"},
                    "plan": {"type": "string"},
                },
                "required": ["name", "email", "plan"],
                "additionalProperties": False,
            },
        }
    },
)

import json
text = next(block.text for block in resp.content if block.type == "text")
data = json.loads(text)
```

`output_config.format` guarantees the response text is valid JSON matching your schema — you
still call `json.loads()` yourself, but you never have to handle "Claude added a sentence before
the JSON" or "Claude used single quotes." Two rules for the schema: every object needs
`"additionalProperties": False`, and you must list every field in `"required"`. These are
requirements of the structured-output feature, not style choices — leaving them out causes the
API to reject the schema.

**Important:** do not use a bare `output_format=` argument on `messages.create()`. That parameter
name is deprecated on `create()`. Use `output_config` there. `output_format=` is only for
`.parse()`, where it names a Pydantic class, as in Section 2.

Which one should you use? Default to `messages.parse()` with Pydantic — it is shorter, and you
get real typed objects. Reach for raw `output_config` when the schema is not known until runtime,
or when you are calling `.create()` anyway for another reason (streaming, or combining structured
output with tool use, covered next).

## 4. A preview: `strict` tool inputs

Tools (Section 5) take input too, and that input also needs a guaranteed shape. Setting
`"strict": True` on a tool definition gives the same guarantee as `output_config` — Claude's call
to that tool will match your schema exactly, every time. We come back to this once tools are
properly introduced, in Section 5.3.

## 5. Tool calling: letting Claude take real actions

Structured output controls the *shape* of an answer Claude already knows. Tool calling is
different: it lets Claude get an answer, or take an action, that it could not do on its own —
looking up live data, running a calculation, calling an external API. This is what turns a
chatbot into an assistant that actually does things.

### 5.1 What a tool call really is

It helps to be precise about what happens, because the wording "Claude calls a tool" is slightly
misleading. **Claude never runs any code.** Claude only *asks* — it outputs a structured request
naming a tool and giving arguments. Your code is the one that decides whether to run it, runs it,
and reports back what happened. Claude cannot reach outside the conversation on its own; every
tool call is a request your program chooses to honor.

```
┌──────────┐   1. "call get_order_status    ┌──────────────┐
│          │      with order_id=ORD-4471"   │              │
│  Claude  │ ──────────────────────────────▶│  Your code   │
│          │                                 │              │
│          │   4. reads the result, writes  │  2. runs the │
│          │◀─── the final answer ───────── │  real lookup │
└──────────┘   3. "the result was: ..."      └──────────────┘
```

### 5.2 Defining a tool

A tool definition has three required parts: a `name`, a `description`, and an `input_schema` — a
JSON Schema describing the arguments the tool takes.

```python
get_order_status_tool = {
    "name": "get_order_status",
    "description": (
        "Look up the current status of a GlobalMart order by its order ID. "
        "Use this whenever the user asks where an order is, or if it has shipped."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "order_id": {
                "type": "string",
                "description": "GlobalMart order ID, format ORD-XXXXX.",
            }
        },
        "required": ["order_id"],
    },
}
```

The `description` is not a comment for other developers — it is the only documentation Claude
reads before deciding whether, and how, to call this tool. Write it like a short API doc: state
what the tool does, and state *when* to use it. A vague description ("looks something up") leads
to a model that skips the tool, or calls it at the wrong moment. Chapter 8 goes deeper on tool
design; this chapter focuses on getting the mechanics right.

### 5.3 The full loop: `tool_use` → run the tool → `tool_result`

This is the core pattern of the whole book. Read it block by block.

You send a request with `tools=[...]`. If Claude decides it needs a tool, it does not answer
directly. Instead, `resp.stop_reason` comes back as `"tool_use"`, and `resp.content` contains a
`tool_use` block — a request, not an answer — with the tool's `name`, its `input` arguments, and
an `id`. Your code must:

1. Find every `tool_use` block in `resp.content`.
2. Run the real function, using the given `input`.
3. Send the result back as a `tool_result` block, tagged with the same `id` (`tool_use_id`), so
   Claude knows which call it answers.
4. Call the API again with the updated `messages` list.
5. Repeat until `resp.stop_reason` is `"end_turn"` — Claude has a final answer and is not asking
   for another tool.

Here is a complete, runnable script that gives the GlobalMart Assistant this one tool:

```python
import json
import anthropic

client = anthropic.Anthropic()

# --- A pretend backend. In a real system, this hits your order database. ---
_FAKE_ORDERS = {
    "ORD-48213": {"status": "shipped", "carrier": "FastShip", "eta": "2026-09-14"},
    "ORD-91002": {"status": "delivered", "carrier": "FastShip", "eta": "2026-09-02"},
}


def get_order_status(order_id: str) -> dict:
    order = _FAKE_ORDERS.get(order_id)
    if order is None:
        return {"error": f"No order found with ID {order_id}."}
    return {"order_id": order_id, **order}


TOOLS = [get_order_status_tool]  # from Section 5.2


def run_tool(name: str, tool_input: dict) -> dict:
    if name == "get_order_status":
        return get_order_status(**tool_input)
    return {"error": f"Unknown tool: {name}"}


def ask_globalmart(question: str) -> str:
    messages = [{"role": "user", "content": question}]

    while True:
        resp = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system="You are the GlobalMart Assistant. Use tools to answer order questions.",
            tools=TOOLS,
            messages=messages,
        )

        if resp.stop_reason != "tool_use":
            # Claude has a final answer — no more tools needed.
            return "".join(b.text for b in resp.content if b.type == "text")

        # Claude asked for one or more tools. Append its request to the history first.
        messages.append({"role": "assistant", "content": resp.content})

        # Run every tool_use block, build a tool_result for each.
        tool_results = []
        for block in resp.content:
            if block.type == "tool_use":
                result = run_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,       # must match the tool_use block's id
                    "content": json.dumps(result),
                })

        # All results go back in ONE new user message.
        messages.append({"role": "user", "content": tool_results})
        # Loop: call the API again, now with the tool result in history.


if __name__ == "__main__":
    print(ask_globalmart("Where is my order ORD-48213?"))
```

Walk through what happens on each call. First call: Claude reads the question, sees the
`get_order_status` tool, and decides it needs the real order data — it cannot know this from
training. `resp.stop_reason` is `"tool_use"`. Your code runs `get_order_status("ORD-48213")`,
gets back a dict, and sends it as a `tool_result`. Second call: Claude now has the real data in
its context, and writes a normal text answer. `resp.stop_reason` is `"end_turn"`, the loop exits,
and you return the text.

Four rules from this example that matter every time you write this loop:

- **Always append `resp.content` as-is** to `messages`, not just the text parts. It contains the
  `tool_use` block Claude needs to see again later in the conversation.
- **`tool_use_id` must match exactly.** Claude uses it to line up which result answers which
  request. Get this wrong and the API rejects the request.
- **All tool results for one turn go in a single `user` message,** as a list of `tool_result`
  blocks — even if there was only one tool call. Do not send several separate messages.
- **Loop on `stop_reason`, not on a fixed number of tries.** Some questions need zero tool calls,
  some need one, some (in later chapters) need several rounds. The loop should keep going exactly
  as long as Claude keeps asking for tools.

### 5.4 Running multiple tools in one turn

Claude can request more than one tool in a single response — for example, if a user asks "where
is my order, and do you sell phone cases?" in one message. When this happens, `resp.content`
contains multiple `tool_use` blocks. The loop above already handles this correctly: the `for
block in resp.content` step collects a result for *every* `tool_use` block, and all of them go
back together in one `user` message. This matters — splitting them across separate messages, or
answering only one, confuses Claude about which calls were actually resolved.

For independent tool calls that are slow (a network call each), you can run them concurrently
with a thread pool instead of one after another, then assemble the same single `tool_results`
list. Chapter 8 shows this pattern in full; the shape of the message you send back is identical.

### 5.5 Returning errors as tool results

Real lookups fail: an order ID does not exist, a downstream service times out. **Never let this
raise an exception that crashes your loop.** Instead, catch the failure and send it back as a
normal tool result, with `"is_error": true`:

```python
tool_results.append({
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": "Error: no order found with ID ORD-99999. Please check the order ID.",
    "is_error": True,
})
```

Claude treats an error result like any other result — it reads it and reacts, usually by
apologizing and asking the user to double-check, or by trying a different approach. This is much
better than your program crashing, and better than silently returning an empty result, which
Claude might misread as "order status: nothing." Wrap your `run_tool` dispatcher in a `try`/
`except` so *no* tool failure can ever escape the loop:

```python
def run_tool_safely(name: str, tool_input: dict) -> tuple[dict, bool]:
    """Returns (result, is_error). Never raises."""
    try:
        return run_tool(name, tool_input), False
    except Exception as exc:
        return {"error": f"Tool '{name}' failed: {exc}"}, True
```

### 5.6 Strict tool inputs

By default, Claude's tool arguments follow your `input_schema` closely, but not with a hard
guarantee — like a keen intern following a spec, mostly correctly. Setting `"strict": True` on the
tool definition makes that guarantee absolute: the arguments will validate against your schema
every time, no exceptions.

```python
book_flight_tool = {
    "name": "book_flight",
    "description": "Book a flight to a destination on a given date.",
    "strict": True,
    "input_schema": {
        "type": "object",
        "properties": {
            "destination": {"type": "string"},
            "date": {"type": "string", "format": "date"},
            "passengers": {"type": "integer", "enum": [1, 2, 3, 4]},
        },
        "required": ["destination", "date", "passengers"],
        "additionalProperties": False,
    },
}
```

Use `strict: True` for any tool where a malformed argument would be expensive or dangerous — a
booking, a payment, a database write. It costs nothing extra to set, and the schema rules are the
same as for `output_config` (Section 3): every field listed in `required`, and
`additionalProperties: False`.

## 6. Less boilerplate: the Tool Runner

The loop in Section 5.3 is worth writing by hand once, so you understand exactly what is
happening. In real code, you often do not need to write it again. The **Tool Runner** is a beta
helper in the SDK that drives that same loop for you.

```python
from anthropic import beta_tool

client = anthropic.Anthropic()


@beta_tool
def get_order_status(order_id: str) -> dict:
    """Look up the current status of a GlobalMart order by its order ID.

    Args:
        order_id: GlobalMart order ID, format ORD-XXXXX.
    """
    order = _FAKE_ORDERS.get(order_id)
    if order is None:
        return {"error": f"No order found with ID {order_id}."}
    return {"order_id": order_id, **order}


runner = client.beta.messages.tool_runner(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=[get_order_status],
    messages=[{"role": "user", "content": "Where is my order ORD-48213?"}],
)

for message in runner:
    pass  # the runner calls get_order_status for you and loops automatically

final_text = "".join(b.text for b in message.content if b.type == "text")
print(final_text)
```

`@beta_tool` builds the JSON schema from your function's type hints and docstring — you never
write `input_schema` by hand. The runner calls the API, notices `tool_use`, calls your Python
function directly, sends the result back, and repeats, stopping automatically once Claude has a
final answer. It is genuinely a beta feature (hence `client.beta...`), but it is the right default
for new tool-calling code once you understand the manual loop it replaces. Reach for the manual
loop instead when you need control the runner does not expose — a custom transport, or a request
shape the runner cannot build.

## 7. Server-side tools: no loop required

So far, every tool has been *your* code — Claude asks, you run it. Some tools instead run
entirely on Anthropic's servers, with no round trip through your application at all. You just
declare them, and the result appears directly in the response.

```python
resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=[
        {"type": "web_search_20260209", "name": "web_search"},
        {"type": "web_fetch_20260209", "name": "web_fetch"},
    ],
    messages=[{"role": "user", "content": "What shipping carriers operate in Berlin right now?"}],
)
```

Web search looks things up on the live web; web fetch retrieves a specific URL's content; code
execution runs Python in a sandboxed container for math, data analysis, or file processing. None
of these need a `tool_use` → `tool_result` round trip from you — Claude runs them and folds the
result straight into its answer. Use them when the task genuinely needs live web data or real
computation, not as a replacement for your own tools like `get_order_status`, which only your
backend can answer.

## 8. MCP: a standard way to plug in tools

Writing a hand-rolled tool for every internal system does not scale past a handful of tools.
**MCP (Model Context Protocol)** is an open standard for exposing tools, so any MCP-compatible
client — including Claude — can call them without custom code for each one. Think of it as a
plug shape that many tools can share, instead of a different plug for every system.

The Messages API can connect straight to a remote MCP server, with no client-side tool-execution
loop, using a beta parameter pair:

```python
resp = client.beta.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    betas=["mcp-client-2025-11-20"],
    mcp_servers=[{"type": "url", "url": "https://example.com/mcp", "name": "globalmart-mcp"}],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "globalmart-mcp"}],
    messages=[{"role": "user", "content": "List today's delayed shipments."}],
)
```

`mcp_servers` names the connection; the matching `mcp_toolset` entry in `tools` turns on every
tool that server exposes. For now, remember MCP as "the standard way to share tools across
systems, instead of writing one-off integrations." Chapter 8 and the multi-agent chapters
(Chapter 9) come back to it once you have more tools worth sharing.

## 9. Hands-on: GlobalMart, end to end

Let's combine both halves of this chapter into one small, realistic feature: a customer support
intake step. First, extract a structured ticket from free text (Section 2). Then, if the message
mentions an order, use the `get_order_status` tool (Section 5) to check on it before replying.

```python
import json
import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()


class SupportTicket(BaseModel):
    customer_name: str
    order_id: str | None
    issue_summary: str
    urgency: str


def extract_ticket(message: str) -> SupportTicket:
    resp = client.messages.parse(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{"role": "user", "content": f"Extract a support ticket: {message}"}],
        output_format=SupportTicket,
    )
    return resp.parsed_output


def answer_with_order_status(ticket: SupportTicket) -> str:
    messages = [{
        "role": "user",
        "content": (
            f"Customer {ticket.customer_name} reports: {ticket.issue_summary}. "
            f"Order ID: {ticket.order_id}. Check the order and give a short, helpful reply."
        ),
    }]
    while True:
        resp = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system="You are the GlobalMart Assistant.",
            tools=[get_order_status_tool],
            messages=messages,
        )
        if resp.stop_reason != "tool_use":
            return "".join(b.text for b in resp.content if b.type == "text")

        messages.append({"role": "assistant", "content": resp.content})
        tool_results = [
            {
                "type": "tool_result",
                "tool_use_id": b.id,
                "content": json.dumps(run_tool(b.name, b.input)),
            }
            for b in resp.content if b.type == "tool_use"
        ]
        messages.append({"role": "user", "content": tool_results})


raw_message = "Hi, I'm Priya. Order ORD-48213 seems delayed and I'm worried, please check."
ticket = extract_ticket(raw_message)
print(ticket)  # a validated SupportTicket — ready to save to a database
print(answer_with_order_status(ticket))
```

Notice the split of responsibilities: `extract_ticket` turns messy customer text into a typed
object your database can store, with structured output. `answer_with_order_status` uses a tool to
check a real, current fact before replying. Neither step trusts Claude's unaided memory for
something it cannot actually know — that is the whole point of both features in this chapter.

## What You Built / Learned

- Why plain text answers are hard for code to act on, and how structured output solves that.
- How to get typed, validated Python objects with `client.messages.parse(..., output_format=MyModel)`
  and `resp.parsed_output`, using a Pydantic model.
- How to get raw JSON with `output_config={"format": {"type": "json_schema", "schema": {...}}}`
  on `client.messages.create()`, and why `output_format=` on `create()` is wrong (deprecated).
- What a tool call really is: Claude only requests a call; your code decides, runs it, and reports
  back — Claude never executes anything itself.
- How to define a tool with `name`, `description`, and `input_schema`, and why the description is
  the model's only documentation.
- The full `tool_use` → run the tool → `tool_result` loop, written by hand: append the assistant's
  content, match `tool_use_id`, send all results in one message, and loop until `"end_turn"`.
- How to handle several tool calls from one turn, and how to return failures as `tool_result`
  blocks with `is_error: true` instead of raising exceptions.
- `strict: True` for tool inputs that must validate exactly, every time.
- The beta **Tool Runner** (`@beta_tool` + `client.beta.messages.tool_runner`), which drives the
  same loop for you from plain Python functions.
- Server-side tools (web search, web fetch, code execution) that need no client-side loop at all.
- MCP (Model Context Protocol) as the standard way to plug in external tools without one-off
  integration code for each system.
- Built a `get_order_status` tool for the GlobalMart Assistant, and a structured-extraction step
  that turns a free-text message into a validated `SupportTicket`.

## Production Notes & Pitfalls

- **A crashed tool should never crash the loop.** Wrap every tool call in a `try`/`except` and
  return failures as a normal `tool_result` with `is_error: true`. A model that sees an error can
  recover gracefully; a program that raises cannot.
- **Never skip `tool_use_id` matching.** If a result's `tool_use_id` does not match a pending
  call, the API rejects the request. This bug is easy to introduce when you refactor the loop —
  test it with more than one tool call in flight.
- **All results for one turn go in one message.** Sending them as separate `user` messages, one
  per tool, is a common mistake that confuses the model about which calls are still open.
- **Structured output is not a substitute for validation.** `output_config` and `strict` guarantee
  *shape* — the right fields, the right types. They do not guarantee the *values* are correct or
  safe (a real order ID that belongs to this user, a date that is not in the past). Validate
  values yourself before acting on them, especially for anything that writes data.
- **`messages.parse()` can still raise.** If Claude cannot produce a reply matching your schema
  (rare, but possible on a refusal or a hit against `max_tokens`), the SDK raises rather than
  silently returning a broken object. Catch this the same way you catch other SDK exceptions
  (Chapter 1).
- **The Tool Runner is beta — know what that means for you.** It saves real code, but beta
  features can change behavior between SDK versions. Pin your `anthropic` package version in
  production, and re-test the tool loop after upgrades, the same as any other dependency.
- **Server-side tools bill differently and can pause.** A long server-side tool turn can end with
  `stop_reason: "pause_turn"` instead of `"end_turn"`. Chapter 8 and Chapter 13 cover resuming a
  paused turn correctly — for now, just know `"end_turn"` is not the only "stop and look at this"
  signal you will see.
- **This chapter's loop is the seed of every agent in this book.** Chapter 7 turns this exact
  pattern into a full ReAct-style agent loop, and Chapter 8 adds tool design and memory on top. If
  something breaks in a later chapter's agent, the bug is almost always in this loop's basic
  contract: matching IDs, appending full content blocks, and looping on `stop_reason`.
