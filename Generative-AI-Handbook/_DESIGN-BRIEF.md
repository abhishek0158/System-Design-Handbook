# _DESIGN BRIEF — Canonical Source of Truth

> **Purpose:** This file fixes every shared fact for the *Generative AI Handbook (Curated,
> Python-first)*. Every chapter MUST follow the language rules (§5), use the SAME Python stack
> (§2), the SAME running example (§3), and the SAME current Claude API patterns (§4). Do not
> contradict these. If a chapter needs a new detail, derive it from here.

---

## 0. What this handbook is
A hands-on, Python-first guide to building **real generative-AI and agentic systems**. It is
curated: 17 chapters, no filler. Every chapter has runnable Python. Chapters 1–15 each add a
piece; chapters 16–17 tie everything into one production-style agent.

**Reader:** an experienced backend engineer (3–4 years) who is new to GenAI but strong in
systems. So: skip beginner filler, explain each new AI idea clearly, and lean into the
production/system-design angle.

---

## 3. The running example (the capstone we build toward)
Everything builds toward one concrete agent:

**"GlobalMart Assistant"** — an assistant for the GlobalMart marketplace (same fictional
company as the Search and Checkout handbooks). It does two things:
1. **Answers questions** from GlobalMart's internal documents and policies (RAG). Example:
   "What is the return policy for electronics?"
2. **Takes a few real actions** through tools. Example tools: `get_order_status(order_id)`,
   `create_support_ticket(user_id, issue)`, `search_products(query)`.

This scenario threads through the book: RAG chapters build its knowledge; agent chapters give
it tools and a loop; framework chapters rebuild it in LangChain/LangGraph; production chapters
make it reliable, cheap, safe, and observable; the capstone assembles the whole thing.

---

## 2. Canonical Python stack (use these in every chapter)
- **Python 3.11+**.
- **LLM calls:** the official `anthropic` Python SDK (`pip install anthropic`).
- **Frameworks:** `langchain` and `langgraph` (Python). Tracing with LangSmith-style tracing.
- **Vector store for RAG:** **Chroma** (`chromadb`) as the default; mention **pgvector** as the
  production alternative.
- **Embeddings:** the Claude API does **NOT** provide embeddings. Use a dedicated embeddings
  model. Default in examples: **Voyage AI** (`voyageai`, e.g. `voyage-3`) — this is the
  embeddings provider Anthropic recommends. Mention open-source `sentence-transformers` as a
  local/free alternative. **Never write `client.embeddings...` on the Anthropic client — it does
  not exist.**
- **Evaluation:** a small `pytest`-based harness plus LLM-as-judge.
- **Config:** read the API key from the environment (`ANTHROPIC_API_KEY`); never hardcode keys.

---

## 4. CURRENT CLAUDE API — USE EXACTLY THIS (do not use older/training-prior patterns)

> Many older Claude patterns are wrong now and will error. Follow this section exactly. If a
> chapter shows Claude code, it must match these rules.

### Models (use these exact IDs)
| Model | ID | Use it for |
|---|---|---|
| Claude Opus 4.8 | `claude-opus-4-8` | Default. The agent's main reasoning. Most capable Opus tier. |
| Claude Sonnet 5 | `claude-sonnet-5` | Balanced cost/quality for high-volume steps. |
| Claude Haiku 4.5 | `claude-haiku-4-5` | Cheap + fast: routing, classification, simple sub-tasks. |

- **Default to `claude-opus-4-8`** unless a chapter is specifically about saving cost (then show
  routing Haiku → Sonnet → Opus).
- **Never use old IDs** like `claude-3-5-sonnet`, `claude-3-opus`, or any date-suffixed alias.
- Rough pricing for the cost chapter (per 1M tokens, input/output): Opus 4.8 **$5 / $25**,
  Sonnet 5 **$3 / $15**, Haiku 4.5 **$1 / $5**. (Say "check current pricing" — do not over-anchor.)

### Basic call (Python)
```python
import anthropic
client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,
    system="You are a helpful assistant.",   # system prompt is a top-level field
    messages=[{"role": "user", "content": "Hello"}],
)
# resp.content is a list of blocks; check block.type before reading .text
for block in resp.content:
    if block.type == "text":
        print(block.text)
```

### Rules that are easy to get wrong
- **Streaming:** for long output use `with client.messages.stream(...) as stream: for text in
  stream.text_stream: ...`, then `stream.get_final_message()`. Default `max_tokens` ~4096 for
  short replies; use streaming when it may be large.
- **Thinking / reasoning:** opt in with `thinking={"type": "adaptive"}`. Control depth with
  `output_config={"effort": "low|medium|high|xhigh|max"}`. **Do NOT use `budget_tokens`** — it
  is removed on these models and returns a 400.
- **Sampling params:** **do NOT pass `temperature`, `top_p`, or `top_k`** on Opus 4.8 / Sonnet 5
  — they are removed and return a 400. Steer behaviour with the prompt instead.
- **Tool use (function calling):** define tools with `name`, `description`, `input_schema`
  (JSON schema). When `resp.stop_reason == "tool_use"`, run the tool and send back a
  `tool_result` block with the matching `tool_use_id`, then call again. Loop until
  `stop_reason == "end_turn"`. For less boilerplate, mention the beta **Tool Runner**
  (`from anthropic import beta_tool` + `client.beta.messages.tool_runner(...)`).
- **Structured output:** use `client.messages.parse(model=..., messages=..., output_format=MyPydanticModel)`
  and read `resp.parsed_output`, OR `output_config={"format": {"type": "json_schema", "schema": {...}}}`.
  Do **not** use a top-level `output_format=` on `create()` (deprecated). For strict tool inputs,
  set `strict: True` on the tool definition.
- **Prompt caching:** put `cache_control={"type": "ephemeral"}` on the large stable part
  (system prompt or a big context block). Check it worked with `resp.usage.cache_read_input_tokens`.
- **Token counting:** use `client.messages.count_tokens(model=..., messages=...)`. **Never use
  `tiktoken`** — it is OpenAI's tokenizer and is wrong for Claude.
- **Server-side tools** (run on Anthropic's side, no client loop): web search
  (`web_search_20260209`), web fetch (`web_fetch_20260209`), code execution. Declare them in
  `tools`. Good to mention in the tools/agent chapters.
- **MCP (Model Context Protocol):** the standard way to plug in external tools/context. The
  Messages API can connect to remote MCP servers (beta). Mention it in the tools chapter.
- **Errors:** use the SDK's typed exceptions (`anthropic.RateLimitError`,
  `anthropic.APIStatusError`, `anthropic.APIConnectionError`). The SDK auto-retries 429 and 5xx
  with backoff.
- **Long-running agents:** the API has beta **context editing** and **compaction** to keep long
  conversations inside the context window — mention these in the long-running/scaling chapter.
- **Managed Agents (a hosted option):** Anthropic offers a hosted agent platform where Anthropic
  runs the agent loop and a sandbox. Mention it in the deployment/scaling discussion as a
  build-vs-buy option, but the handbook's main path is building the agent ourselves in Python.

---

## 5. House style (every chapter)

### 5.0 LANGUAGE — SIMPLE ENGLISH (most important rule)
The reader prefers simple, easy-to-read English (non-native friendly).
- Short sentences, one idea each (~15–20 words). Common words. No idioms, slang, or metaphors.
- Define each new AI term in one simple sentence the first time it appears
  (for example: "An *embedding* is a list of numbers that captures the meaning of text.").
- Explain the idea first, then show the code. Short paragraphs. Keep full technical depth.

### 5.1 Structure & content
- **Hands-on Python:** every chapter has runnable, correct Python that follows §4 exactly.
- Build toward the GlobalMart Assistant (§3) where it fits.
- Use small diagrams (ASCII/mermaid), tables, and short code blocks.
- Each chapter ends with two sections: **"## What You Built / Learned"** (plain bullets) and
  **"## Production Notes & Pitfalls"** (what breaks in real systems, and the senior-level advice).
- Cross-reference other chapters by number.
- Recurring theme to repeat: **use the simplest thing that works — single call < workflow <
  agent** — and **evaluation + observability are what make it production.**
- Page target: a "page" ≈ 450–550 words. Follow the per-chapter targets in the README.
