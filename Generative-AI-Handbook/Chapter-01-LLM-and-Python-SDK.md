# Chapter 1 — LLM & the Python SDK

## What is an LLM?

An LLM (large language model) is a program that predicts the next word in a piece of text.
It reads the words you gave it, then guesses the most likely next word. It repeats this,
one word at a time, until it has written a full answer. This simple trick, done at huge
scale, can answer questions, write code, and hold a conversation.

The model does not read plain words. It reads **tokens**. A token is a small piece of text,
often a word or part of a word. For example, "GlobalMart" might split into two tokens:
"Global" and "Mart". Short common words are usually one token each. Counting tokens matters
because Claude's pricing and limits are based on tokens, not words or characters.

Every model has a **context window**. This is the maximum number of tokens the model can
"see" at once — your system prompt, your message history, and its own reply, all added
together. Claude's models support large context windows (hundreds of thousands of tokens),
but going near the limit costs more and can slow down the response. Think of the context
window as the model's short-term memory for one conversation. It forgets everything once
the conversation ends, unless you save and resend it yourself.

**Next-token prediction** means the model has no real memory, no database, and no built-in
facts about "today". It only predicts text based on patterns learned during training, plus
whatever text you put in the prompt. This gives Claude two clear strengths and two clear
limits:

| Claude is good at | Claude is not good at (on its own) |
|---|---|
| Reading and summarizing text you give it | Knowing facts newer than its training, or private company data |
| Writing, explaining, and reasoning over language | Doing exact math or running real code reliably |
| Following instructions in a prompt | Taking real actions (sending an email, updating a database) |
| Producing structured output (JSON, code) | Remembering past conversations by itself |

This is why later chapters add **RAG** (Retrieval-Augmented Generation — giving the model
your company's documents at query time, Chapter 4) and **tools** (letting the model call
real functions, Chapter 3 and 7). An LLM call alone is a good starting point, but production
systems usually combine it with retrieval, tools, and memory. Keep this rule in mind for the
whole book: **use the simplest thing that works — a single call, then a workflow, then a
full agent** — only add complexity when a single call is not enough.

## Setting up the Anthropic Python SDK

Anthropic is the company that builds Claude. It ships an official Python library called
`anthropic`. Install it with pip:

```bash
pip install anthropic
```

Next, get an API key from the Anthropic Console and put it in your environment. Never write
the key directly in your code — if you commit it to git, anyone can find and misuse it.

On macOS/Linux:

```bash
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
```

On Windows PowerShell:

```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-your-key-here"
```

For real projects, put the key in a `.env` file (added to `.gitignore`) and load it with a
package like `python-dotenv`. The SDK automatically reads the `ANTHROPIC_API_KEY`
environment variable, so you rarely need to pass the key in code:

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment
```

That single line is enough to set up the client for the rest of this book.

## Your first API call

A call to Claude uses the `messages.create` method. You give it a model name, a limit on
reply length (`max_tokens`), and a list of messages. Here is the smallest working example:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system="You are a helpful assistant for the GlobalMart marketplace.",
    messages=[
        {"role": "user", "content": "In one sentence, what is GlobalMart?"}
    ],
)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

Run this, and Claude replies with one sentence. Let's break down each part.

## The messages format: roles and content blocks

Claude conversations are a list of turns. Each turn is a dictionary with a **role** and
**content**. There are two roles you send: `"user"` (what the human said) and `"assistant"`
(what Claude said before, if you are continuing a conversation). The **system prompt** is
different — it is not a message. It is a separate top-level argument that sets the model's
overall behavior, like "You are a support agent for GlobalMart. Be concise."

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system="You are a support assistant for GlobalMart. Be concise and polite.",
    messages=[
        {"role": "user", "content": "My order hasn't arrived yet."},
        {"role": "assistant", "content": "I'm sorry to hear that. Can you share your order ID?"},
        {"role": "user", "content": "It's ORD-4471."},
    ],
)
```

Each message's `content` can be a plain string, like above. It can also be a list of
**content blocks** — small typed pieces, like a text block, an image, or a tool result. Using
a list of blocks becomes important once you add images or tool calls (Chapter 3). For now,
know that a message is really `{"role": ..., "content": [list of blocks]}`, and a plain
string is just a shortcut for a single text block.

The **response** you get back, `response`, also holds its content as a list of blocks — not
as one plain string. This matters, because future models may add new block types (for
example, a "thinking" block or a tool-use block). Your code should check each block's `type`
before reading it, instead of assuming the first block is always text:

```python
for block in response.content:
    if block.type == "text":
        print(block.text)
    # other block types (e.g. "tool_use") are covered in Chapter 3
```

This small habit — check `block.type` first — avoids crashes when Claude's response contains
more than plain text. Never assume `response.content[0].text` is safe; always loop and check.

The response object also has useful fields beyond `content`:

```python
print(response.stop_reason)   # why the model stopped, e.g. "end_turn", "max_tokens"
print(response.usage)         # input_tokens and output_tokens used in this call
print(response.model)         # which model actually answered
```

`stop_reason` tells you if the reply is complete. If it is `"max_tokens"`, the reply was cut
off because it hit your `max_tokens` limit — a common bug in real systems. Always check this
field if replies seem to end mid-sentence.

## Streaming responses

By default, `messages.create` waits for Claude to finish the whole reply, then returns it all
at once. For short answers this is fine. For long answers — a generated report, a long
explanation — the user would stare at a blank screen for a while. **Streaming** sends the
reply back piece by piece, as the model generates it, so you can show text appearing live,
like a typing effect.

Use `messages.stream` for this:

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=2048,
    system="You are a helpful assistant for GlobalMart.",
    messages=[{"role": "user", "content": "Explain our return policy in detail."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

    final_message = stream.get_final_message()
    print("\n\nTokens used:", final_message.usage)
```

`stream.text_stream` gives you plain text pieces as they arrive, in order — good for printing
straight to a terminal or a chat UI. After the loop ends, `stream.get_final_message()` gives
you the full response object, with the same `content`, `stop_reason`, and `usage` fields as a
normal call. Use it to log usage or check the final `stop_reason`.

**When to stream:** use streaming for any reply that might be long, or for any user-facing
chat UI, where users expect to see text appear quickly. Skip streaming for short internal
calls — like classification or routing (Chapter 9) — where you just need the final short
answer and streaming adds no benefit.

## Counting tokens and estimating cost

Claude bills you separately for **input tokens** (everything you send: system prompt,
message history, tool definitions) and **output tokens** (what Claude generates back). Output
tokens usually cost more than input tokens, because generating text is more expensive than
reading it.

You already saw token counts on any response, in `response.usage`:

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "What is a token in an LLM?"}],
)
print(response.usage.input_tokens, response.usage.output_tokens)
```

But sometimes you want to know the token count **before** you make a call — for example, to
check if a document fits in the context window, or to estimate cost ahead of time. Use
`client.messages.count_tokens`:

```python
count = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": "What is a token in an LLM?"}],
)
print(count.input_tokens)
```

Do not use `tiktoken` for this. `tiktoken` is OpenAI's tokenizer. It splits text differently
than Claude's tokenizer, so it gives wrong counts for Claude. Always use the SDK's own
`count_tokens` method.

With a token count and the model's price per token, you can estimate cost:

```python
def estimate_cost(input_tokens, output_tokens, price_in_per_m, price_out_per_m):
    input_cost = (input_tokens / 1_000_000) * price_in_per_m
    output_cost = (output_tokens / 1_000_000) * price_out_per_m
    return input_cost + output_cost

# Example using Opus 4.8 rough pricing: $5 per 1M input tokens, $25 per 1M output tokens
cost = estimate_cost(input_tokens=1500, output_tokens=400, price_in_per_m=5, price_out_per_m=25)
print(f"Estimated cost: ${cost:.4f}")
```

Prices change over time, so always check current pricing on Anthropic's site before you rely
on exact numbers in production. Chapter 12 covers cost control in depth — caching, batching,
and model routing.

## Choosing a model

Anthropic offers three model tiers. They trade capability against cost and speed. Pick the
smallest model that reliably does the job — this is the same "simplest thing that works"
rule from earlier, applied to model choice.

| Model | ID | Best for | Rough price (input / output, per 1M tokens) |
|---|---|---|---|
| Claude Opus 4.8 | `claude-opus-4-8` | Hardest reasoning, complex agents, the main "brain" of a system | $5 / $25 |
| Claude Sonnet 5 | `claude-sonnet-5` | Balanced quality and cost, high-volume steps (e.g., RAG answer generation) | $3 / $15 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | Simple, fast tasks: routing, classification, short extraction | $1 / $5 |

A practical pattern used later in this book (Chapter 9 and 12): use Haiku to quickly decide
*what kind* of request came in (a routing step), then hand harder requests to Sonnet or Opus.
This can cut cost a lot, since most real traffic is simple.

For the GlobalMart Assistant, default to `claude-opus-4-8` while you are learning and testing
correctness. Once the system works, measure whether Sonnet 5 or Haiku 4.5 gives "good enough"
answers for cheaper, and switch the simpler steps over.

Never use older model names like `claude-3-5-sonnet` or `claude-3-opus`. Those are prior
generations and are not used anywhere in this book.

## `max_tokens` and the system prompt

`max_tokens` is not optional — every call needs it. It sets the **maximum** number of tokens
Claude is allowed to generate in its reply. It is a safety limit, not a target length. Claude
may stop earlier on its own (`stop_reason == "end_turn"`) once it finishes its answer. If
Claude hits your limit before finishing, `stop_reason` will be `"max_tokens"`, and the reply
will be cut off mid-sentence.

A short chat reply needs a small `max_tokens`, like 1024. A long generated document needs
more, like 4096 or higher. If you are unsure, start around 4096 for general use, and raise it
if you see truncated replies.

The **system prompt** is a separate top-level string, not a message in the `messages` list.
Use it to set Claude's role, tone, and rules for the whole conversation:

```python
system_prompt = (
    "You are the GlobalMart Assistant. "
    "Answer questions about orders, products, and policies. "
    "Be concise. If you do not know something, say so clearly."
)
```

Keep the system prompt stable across calls in one feature — this also helps with **prompt
caching** later (Chapter 12), which reuses a stable system prompt across calls to save cost.

## Basic error handling

Real systems fail sometimes: the network drops, you send too many requests too fast, or the
API is briefly down. The `anthropic` SDK gives you **typed exceptions** — specific error
classes you can catch by name, instead of guessing what went wrong from a generic error.

```python
import anthropic

client = anthropic.Anthropic()

try:
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello"}],
    )
except anthropic.RateLimitError:
    print("Too many requests right now. Wait and try again.")
except anthropic.APIConnectionError:
    print("Could not reach the API. Check your network.")
except anthropic.APIStatusError as e:
    print(f"API returned an error: {e.status_code} — {e.message}")
```

`RateLimitError` means you sent requests too fast for your account's limit.
`APIConnectionError` means the request never reached Anthropic's servers — a network problem
on your end. `APIStatusError` is the general case: the API responded, but with an error status
code (like a 400 for a bad request, or a 500 for a server problem).

The good news: the SDK already retries some errors for you. It automatically retries on
`429` (rate limit) and `5xx` (server error) responses, with a growing wait time between
tries. You do not need to write your own retry loop for these common, temporary failures.
Chapter 12 goes further, into fallback models and circuit breakers for larger systems.

## Putting it together: a GlobalMart "ask a question" script

Here is a small, complete script. It ties together everything from this chapter: setup, a
system prompt, a safe response loop, token usage, and error handling. This is the seed the
GlobalMart Assistant grows from in later chapters.

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT = (
    "You are the GlobalMart Assistant. "
    "Answer questions about GlobalMart's products, orders, and policies. "
    "Be concise and clear. If you are not sure, say so."
)


def ask_globalmart(question: str) -> str:
    try:
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            messages=[{"role": "user", "content": question}],
        )
    except anthropic.RateLimitError:
        return "The assistant is busy right now. Please try again shortly."
    except anthropic.APIConnectionError:
        return "Could not reach the assistant. Check your network."
    except anthropic.APIStatusError as e:
        return f"The assistant returned an error: {e.status_code}"

    answer_parts = [block.text for block in response.content if block.type == "text"]
    answer = "".join(answer_parts)

    print(f"[tokens used — input: {response.usage.input_tokens}, "
          f"output: {response.usage.output_tokens}]")

    return answer


if __name__ == "__main__":
    question = "What is the return policy for electronics?"
    print(ask_globalmart(question))
```

Right now, Claude answers from general knowledge only — it has never seen GlobalMart's real
return policy. Chapter 4 fixes this with RAG, feeding Claude real GlobalMart documents at
query time. Chapter 3 adds tools, so the assistant can call `get_order_status(order_id)`
instead of guessing. This chapter's script is the small, working foundation both build on.

## What You Built / Learned

- What an LLM does: it predicts the next token, based on the tokens you give it plus what it
  learned in training. It has no built-in memory or live facts.
- What a token and a context window are, and why both drive cost and limits.
- How to install the `anthropic` SDK and read the API key safely from the environment.
- How to make a basic `messages.create` call, with a system prompt and a user message.
- How messages use roles (`user`, `assistant`) and content blocks, and why you must check
  `block.type` before reading a response instead of assuming it is plain text.
- How to stream long replies with `messages.stream`, and when streaming is worth the extra
  code.
- How to count tokens ahead of time with `count_tokens`, and roughly estimate cost.
- How to pick between Opus 4.8, Sonnet 5, and Haiku 4.5, based on task difficulty and cost.
- Why `max_tokens` is a safety cap, not a target, and how to read `stop_reason`.
- How to catch typed SDK exceptions (`RateLimitError`, `APIConnectionError`,
  `APIStatusError`), and that the SDK already retries rate limits and server errors for you.
- A first working "ask a question" script for the GlobalMart Assistant.

## Production Notes & Pitfalls

- **Never hardcode API keys.** Always read them from the environment or a secret manager. A
  key committed to git can be found and used by anyone, even after you delete the commit.
- **Do not assume response shape.** Always loop over `response.content` and check
  `block.type`. Future model versions can add new block types; code that assumes
  `content[0].text` will break without warning.
- **Watch `stop_reason`, not just the text.** A reply that looks complete can still be cut off
  if `max_tokens` was too low. Log `stop_reason` in production, and alert if
  `"max_tokens"` shows up often — it means real answers are getting truncated.
- **Token counts are not free to skip.** For anything sent often (a chatbot, a batch job),
  count tokens and estimate cost before shipping. Surprise bills usually come from unbounded
  context — like appending the full chat history forever without trimming it.
- **Do not default every call to the biggest model.** Opus 4.8 is capable but costs more per
  call and can be slower. For high-volume, simple steps (routing, short lookups), Haiku 4.5 or
  Sonnet 5 is often "good enough" and much cheaper at scale. Measure quality with real
  examples before downgrading a model — do not guess.
- **Rely on the SDK's retries, but still handle exceptions.** The SDK retries `429` and `5xx`
  automatically, but only up to a limit. Your code still needs a fallback path (a friendly
  error message, or a fallback model) for when retries are exhausted. Chapter 12 covers this
  in more depth, with fallback models and circuit breakers.
- **A single LLM call has real limits.** It cannot know private company facts (fixed by RAG in
  Chapter 4) or take real actions (fixed by tools in Chapter 3). Do not try to solve these with
  a bigger prompt alone — use the right building block instead.
