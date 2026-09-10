# Chapter 10 — LangChain

In Chapters 1–9 you built everything by hand: raw calls to the Claude API, your own retrieval
code, your own tool loop, your own multi-agent patterns. That was on purpose. It taught you what
is really happening under the hood. Now we look at **LangChain**, a framework that packages those
same ideas into reusable pieces.

LangChain is a Python (and JavaScript) library. It gives you a common set of building blocks —
chat models, prompts, parsers, tools, retrievers — plus a way to snap them together. It does not
give you new AI capability. Claude still does the thinking. LangChain gives you glue code that
many teams have already written, tested, and reused.

This chapter covers:
- What problem LangChain solves, and why it exists.
- The five core pieces: chat models, prompt templates, output parsers, tools, retrievers.
- **LCEL** (LangChain Expression Language), the pipe-style way to chain pieces together.
- Rebuilding the GlobalMart RAG pipeline from Chapters 4–5, this time in LangChain.
- An honest comparison: when the framework helps, and when the plain SDK is the better choice.

Install the packages you need for this chapter:

```bash
pip install langchain langchain-anthropic langchain-chroma langchain-voyageai chromadb
```

## 1. The problem LangChain solves

Look back at Chapter 4. To build a RAG pipeline by hand you wrote: a text splitter, an embedding
call, a Chroma client, a retrieval function, a prompt template (an f-string), and a call to
`client.messages.create`. Every piece was simple. But every team building a GenAI app writes
almost the same pieces, over and over, in slightly different ways.

LangChain's core idea is a **Runnable**: a small object with one method, `invoke`, that takes an
input and returns an output. A chat model is a Runnable. A prompt template is a Runnable. A
retriever is a Runnable. Because they all share this one shape, you can connect them with a
single operator — the pipe, `|` — instead of writing glue code for each combination.

A second idea is **swappable parts**. If every vector store, every embedding model, and every
chat model implements the same small interface, you can switch Chroma for pgvector, or Claude for
another model, by changing one line. In practice you rarely swap the LLM — this handbook always
uses Claude — but swapping the vector store, the document loader, or the retriever strategy is
common as a project matures.

A third idea is the **ecosystem**. LangChain ships hundreds of integrations: document loaders for
PDFs, websites, and databases; retrievers for many vector stores; tracing through **LangSmith**
(a hosted tool that records every step of a chain, similar in spirit to the tracing ideas we cover
in Chapter 15). If you need a loader for Notion pages or Slack messages, someone has likely
already written it.

Keep the recurring theme in mind: **use the simplest thing that works**. LangChain is a tool, not
a requirement. For a single Claude call with one prompt, plain `anthropic` SDK code (Chapter 1) is
often clearer. LangChain earns its cost when a project has several moving parts — multiple
retrievers, multiple prompts, a swappable vector store — and you want less glue code to
maintain yourself.

It helps to know where LangChain came from. It started as a thin helper library around early LLM
APIs, when every provider had a different request format and almost no one had a vector store
wrapper yet. As the field moved fast, LangChain grew fast too: new integrations, new chain types,
new agent styles, added one on top of another. That history explains both its strength (a huge
catalog of ready-made pieces) and its main weakness (an API surface that has been reshaped more
than once, covered in Section 6). Treat it as a well-stocked toolbox built by a large community,
not as a single, tightly designed system — and read the current docs for the version you install,
not just this chapter.

## 2. The core pieces

LangChain has a large surface area, but almost every app is built from five kinds of pieces.

| Piece | What it is | Chapter 1–9 equivalent |
|---|---|---|
| Chat model | A wrapper around an LLM's chat API | `client.messages.create(...)` |
| Prompt template | A reusable prompt with placeholders | An f-string or `.format()` call |
| Output parser | Turns raw model output into a Python value | Manual JSON parsing (Chapter 3) |
| Tool | A Python function the model can call | Your tool-use loop (Chapter 8) |
| Retriever | Fetches relevant chunks for a query | Your `retrieve()` function (Chapter 4) |

We walk through each one with a small, runnable example, then compose them together.

### 2.1 Chat models: `ChatAnthropic`

A **chat model** in LangChain is an object that sends messages to an LLM and returns a response.
For Claude, the class is `ChatAnthropic`, from the `langchain-anthropic` package. It wraps the
same Claude API you used directly in Chapter 1 — it does not add a different model or a different
provider.

```python
from langchain_anthropic import ChatAnthropic

# ChatAnthropic reads ANTHROPIC_API_KEY from the environment, same as the raw SDK.
llm = ChatAnthropic(model="claude-opus-4-8", max_tokens=1024)

response = llm.invoke("In one sentence, what is a vector database?")
print(response.content)
```

A few points that matter, given the current Claude API rules from this handbook (see the design
brief, §4):

- **Do not pass `temperature`, `top_p`, or `top_k`** to `ChatAnthropic` for `claude-opus-4-8` or
  `claude-sonnet-5`. LangChain will pass them straight through to the API, and the API will reject
  them with a 400 error. Steer the model with the prompt instead.
- Use the exact model IDs: `claude-opus-4-8`, `claude-sonnet-5`, `claude-haiku-4-5`. LangChain does
  not validate model names for you — a typo, or an old id like `claude-3-5-sonnet`, fails only when
  you call the model.
- `response` is an `AIMessage` object, not a plain string. `response.content` holds the text.
  `response.usage_metadata` holds token counts, useful for the cost tracking in Chapter 12.

You can also stream a response, matching the streaming rule from Chapter 1:

```python
for chunk in llm.stream("List three benefits of retrieval-augmented generation."):
    print(chunk.content, end="", flush=True)
```

### 2.2 Prompt templates

A **prompt template** is a prompt with blanks you fill in at run time. Instead of writing a new
f-string every time, you define the template once and reuse it.

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a support assistant for the GlobalMart marketplace. "
               "Answer in two sentences or fewer."),
    ("user", "{question}"),
])

# .invoke fills in the placeholders and returns a list of chat messages.
messages = prompt.invoke({"question": "Can I return a used blender?"})
print(messages.to_messages())
```

This looks like a small win over an f-string, and for one prompt, it is a small win. The value
grows when a template is reused across many chains, or when you compose it directly into a chain
(shown in Section 3) instead of building messages by hand every time.

### 2.3 Output parsers and structured output

An **output parser** takes the raw text a model returns and turns it into a Python value: a
string, a list, a JSON object, or a typed object. Chapter 3 covered structured output using the
raw SDK, with `client.messages.parse(..., output_format=MyPydanticModel)`. LangChain gives you the
same result through `with_structured_output`.

```python
from pydantic import BaseModel, Field
from langchain_anthropic import ChatAnthropic

class TicketSummary(BaseModel):
    """A short summary of a customer support issue."""
    category: str = Field(description="One of: shipping, returns, billing, product, other")
    urgent: bool = Field(description="True if the customer needs a same-day reply")
    summary: str = Field(description="One sentence summary of the issue")

llm = ChatAnthropic(model="claude-opus-4-8", max_tokens=1024)
structured_llm = llm.with_structured_output(TicketSummary)

result = structured_llm.invoke(
    "My order #4821 arrived broken and I need a replacement before Friday."
)
print(result)
# TicketSummary(category='shipping', urgent=True, summary='Order #4821 arrived broken...')
```

`result` is a real `TicketSummary` instance, not a dictionary you must validate yourself. Under
the hood, `with_structured_output` builds a Claude tool definition from your Pydantic model, sends
it as a forced tool call, and parses the tool input back into your class. This is the same
mechanism from Chapter 3 — LangChain just writes the schema-building and parsing code for you.

There is also a simpler parser for plain text: `StrOutputParser`, which just extracts
`response.content` as a string. You will see it in the chains below.

### 2.4 Tools

A **tool** is a Python function the model can decide to call, with a name, a description, and a
typed input, exactly like the tools you defined by hand in Chapter 8. LangChain's `@tool` decorator
turns a normal function into something a chat model can call.

```python
from langchain_core.tools import tool

@tool
def get_order_status(order_id: str) -> str:
    """Look up the current status of a GlobalMart order by its order ID."""
    # In a real system this calls the orders service. Here we fake it.
    fake_statuses = {"4821": "shipped", "9002": "delivered"}
    return fake_statuses.get(order_id, "order not found")

llm = ChatAnthropic(model="claude-opus-4-8", max_tokens=1024)
llm_with_tools = llm.bind_tools([get_order_status])

response = llm_with_tools.invoke("What's the status of order 4821?")
print(response.tool_calls)
# [{'name': 'get_order_status', 'args': {'order_id': '4821'}, 'id': 'toolu_...'}]
```

`bind_tools` attaches the tool schema to every request this model makes. The model does not run
the function — it only asks for it to be run, through `response.tool_calls`, the same
`stop_reason == "tool_use"` pattern from Chapter 8. You still write the loop: run the tool, send
the result back as a `ToolMessage`, and call the model again. LangChain gives you the schema
plumbing, not the loop itself — Chapter 11 (LangGraph) is where a full agent loop with tools
becomes a first-class structure with cycles and state.

### 2.5 Retrievers

A **retriever** is a Runnable that takes a text query and returns a list of relevant documents. It
is the LangChain name for the `retrieve()` function you wrote by hand in Chapter 4.

```python
from langchain_chroma import Chroma
from langchain_voyageai import VoyageAIEmbeddings

embeddings = VoyageAIEmbeddings(model="voyage-3")
vectorstore = Chroma(collection_name="globalmart_policies", embedding_function=embeddings)

retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
docs = retriever.invoke("return policy for electronics")
for doc in docs:
    print(doc.page_content[:100])
```

Note the embedding rule from the design brief: Claude does not provide embeddings. We use
`VoyageAIEmbeddings` here, matching Chapters 4–5. If you prefer a free, local model instead of a
paid API, swap in `langchain_huggingface.HuggingFaceEmbeddings` with a `sentence-transformers`
model name — the rest of the code does not change. That swap is exactly the "swappable parts"
benefit from Section 1.

## 3. Composition: LCEL and the pipe operator

Every piece above is a Runnable. **LCEL** (LangChain Expression Language) lets you connect
Runnables with the `|` operator, the same way Unix pipes connect commands. The output of the
piece on the left becomes the input of the piece on the right.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_anthropic import ChatAnthropic

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer in one short paragraph."),
    ("user", "{question}"),
])
llm = ChatAnthropic(model="claude-opus-4-8", max_tokens=1024)
parser = StrOutputParser()

chain = prompt | llm | parser

answer = chain.invoke({"question": "Why do teams use retrieval-augmented generation?"})
print(answer)
```

Read `prompt | llm | parser` from left to right: build the messages, send them to Claude, parse
the reply into a plain string. Each `|` calls `.invoke()` on the right-hand piece, passing in the
left-hand piece's output.

Why this matters more than it looks: the whole chain is *itself* a Runnable. You can call
`chain.invoke(...)`, but also `chain.stream(...)` for token-by-token output, `chain.batch([...])`
to run many inputs at once with automatic concurrency, and — if you connect it to LangSmith — get
a full trace of every step, with the exact prompt and the exact model response, without adding any
logging code yourself.

This composability is the main technical argument for LangChain. Without it, "batch this chain
over 500 inputs with concurrency limits and retries" is a few dozen lines of your own code, as you
saw in Chapter 8. With LCEL, it is one method call: `chain.batch(inputs, config={"max_concurrency": 8})`.

## 4. Rebuilding the GlobalMart RAG pipeline in LangChain

Chapters 4 and 5 built the GlobalMart RAG pipeline by hand: split documents into chunks, embed
them, store them in Chroma, retrieve the top chunks for a question, and ask Claude to answer using
only those chunks. Let's rebuild the same pipeline in LangChain, so you can compare the two
directly.

**Step 1 — load and split the documents.** GlobalMart's policy documents live as plain text files.
We split them into chunks with `RecursiveCharacterTextSplitter`, the LangChain version of the
chunking logic from Chapter 4.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document

raw_docs = [
    Document(
        page_content=open(path, encoding="utf-8").read(),
        metadata={"source": path},
    )
    for path in ["policies/returns.txt", "policies/shipping.txt", "policies/warranty.txt"]
]

splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
chunks = splitter.split_documents(raw_docs)
print(f"{len(raw_docs)} documents split into {len(chunks)} chunks")
```

**Step 2 — embed and store the chunks in Chroma.** Same embedding rule as before: Voyage AI, not
the Anthropic client.

```python
from langchain_chroma import Chroma
from langchain_voyageai import VoyageAIEmbeddings

embeddings = VoyageAIEmbeddings(model="voyage-3")
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    collection_name="globalmart_policies",
    persist_directory="./chroma_globalmart",
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
```

**Step 3 — build the RAG chain with LCEL.** This is the part that replaces the manual
"retrieve, then format a prompt, then call Claude" code from Chapter 4.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_anthropic import ChatAnthropic

def format_docs(docs) -> str:
    """Join retrieved chunks into one context block, with their source noted."""
    return "\n\n".join(f"[Source: {d.metadata['source']}]\n{d.page_content}" for d in docs)

rag_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are the GlobalMart support assistant. Answer the question using ONLY "
               "the context below. If the context does not have the answer, say you don't know. "
               "Context:\n{context}"),
    ("user", "{question}"),
])

llm = ChatAnthropic(model="claude-opus-4-8", max_tokens=1024)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("What is the return policy for electronics?")
print(answer)
```

The dictionary at the start of the chain runs both branches on the same input: the `question`
string flows straight through `RunnablePassthrough()`, while it is also sent into `retriever`,
whose output then flows through `format_docs`. LangChain runs independent branches like this
concurrently where it can, which is a small free speed-up over the sequential code from Chapter 4.

**Step 4 — add source citations with structured output.** Chapter 5 discussed returning citations
alongside the answer. Here is the same idea, using `with_structured_output` instead of manual JSON
parsing:

```python
from pydantic import BaseModel, Field

class CitedAnswer(BaseModel):
    """An answer to a GlobalMart policy question, with its sources."""
    answer: str = Field(description="The answer, in two sentences or fewer")
    sources: list[str] = Field(description="File names of the sources used")

structured_llm = llm.with_structured_output(CitedAnswer)

rag_chain_with_citations = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | structured_llm
)

result = rag_chain_with_citations.invoke("What is the return policy for electronics?")
print(result.answer)
print(result.sources)
```

Compare this to Chapters 4–5: the retrieval logic, the chunking, and the prompt design are
identical in spirit. What changed is who writes the glue: `Chroma.from_documents`,
`as_retriever`, and the `|` chain replace your own `retrieve()` function, your own prompt
f-string, and your own `client.messages.create` call. The underlying Claude API calls, the vector
math, and the ranking are the same as what you built by hand — LangChain did not make retrieval
smarter, it made the wiring shorter.

## 5. When LangChain helps, and when it does not

Be honest with yourself about this trade-off, project by project.

**LangChain tends to help when:**
- You want a **fast start**. A retriever, a chat model, and a chain, in under 20 lines, is hard to
  beat for a prototype or a proof of concept.
- You expect to **swap pieces**. Switching Chroma for pgvector, or adding a re-ranker, is a small
  change if you built on LangChain's retriever interface from the start.
- You want the **ecosystem**: document loaders for dozens of file types and services, ready-made
  retrievers, and LangSmith tracing without writing your own logging.
- Your team already standardized on it, and the shared vocabulary ("chain", "retriever", "tool")
  helps engineers move between projects.

**The plain SDK (Chapters 1–9) tends to be clearer when:**
- The app is **simple**: one or two prompts, one model, no swapping planned. A framework adds
  indirection with no matching benefit.
- You want **full control** over the exact request sent to Claude — the exact system prompt, the
  exact tool schema, the exact retry behavior — without a framework's own defaults sitting between
  you and the API.
- You want **fewer surprises**. Framework abstractions can hide a decision you needed to see, as
  the next section explains.
- You are debugging a subtle issue and need to see precisely what bytes went over the wire.

Most production teams end up with a **mix**: plain SDK code for the parts that need precision (a
tool loop with strict validation, a cost-sensitive routing step), and LangChain for the parts that
benefit from its ecosystem (a document ingestion pipeline with many file-type loaders, a
retriever that might later move to a different vector store). Chapter 16 revisits this choice for
the GlobalMart Assistant's final architecture.

## 6. Two honest warnings

**Version churn.** LangChain has changed its own APIs many times. Whole modules — early "Chain"
classes, the original agent executor, a separate memory module — have been deprecated and replaced,
often within the same major version line, as the LCEL style you learned in this chapter became the
preferred way to build. Code you find in an older blog post or Stack Overflow answer may import a
class that no longer exists, or that now issues a deprecation warning. Pin your LangChain and
`langchain-anthropic` versions in `requirements.txt`, and re-test after every upgrade. Read the
changelog before you upgrade a production system — do not assume old code still works.

Concretely: the package itself has been split apart more than once. Model integrations that used
to live inside the main `langchain` package moved into separate partner packages, like
`langchain-anthropic` and `langchain-chroma`, the ones this chapter installs. An import such as
`from langchain.chat_models import ChatAnthropic` may still work in an old tutorial, may warn, or
may fail outright, depending on the version installed. When code from the internet does not
import cleanly, check the package split for your version before assuming your own code is wrong.

**Hidden prompts.** `with_structured_output`, and any higher-level helper like it, builds a tool
schema and sometimes extra system instructions for you, behind the method call. This is convenient,
but it means the exact prompt sent to Claude is not the text you wrote — it is that text, plus
whatever LangChain added. For most calls this is harmless. But when you are debugging an odd
answer, or trying to control token cost precisely (Chapter 12), you need to see the *real* request.
Two ways to do that:

```python
import langchain
langchain.debug = True   # prints every step of every chain, including the exact messages sent

# or, more surgically, inspect what a chain will send without calling the model:
print(rag_prompt.invoke({"context": "...", "question": "..."}))
```

Turn on tracing (LangSmith, or `langchain.debug`) whenever you are not sure what a chain actually
sent. Never assume — check. The same discipline from Chapter 1, where you learned to read
`resp.usage` instead of guessing at token counts, applies here: verify what the framework is doing,
don't trust the abstraction blindly.

## What You Built / Learned

- What LangChain is: a set of shared building blocks (chat models, prompts, parsers, tools,
  retrievers) plus LCEL, a pipe-based way to connect them, all built on a common `Runnable`
  interface.
- `ChatAnthropic`, LangChain's wrapper around the Claude API, and the rule that it still cannot
  take `temperature`, `top_p`, or `top_k` on `claude-opus-4-8` / `claude-sonnet-5`.
- Prompt templates, output parsers, and `with_structured_output` as the LangChain equivalents of
  work you did by hand in Chapters 1–3.
- Tools with the `@tool` decorator and `bind_tools`, and why you still write the run-and-respond
  loop yourself (full agent loops arrive properly in Chapter 11, with LangGraph).
- Retrievers, backed by Chroma and Voyage AI embeddings, matching the RAG stack from Chapters 4–5.
- The full GlobalMart RAG pipeline, rebuilt with LCEL: `{"context": ..., "question": ...} | prompt
  | llm | parser`.
- A framework decision rule: fast start, swappable parts, and ecosystem favor LangChain; simple
  apps, full control, and fewer surprises favor the plain SDK.

## Production Notes & Pitfalls

- **Pin your versions.** LangChain's APIs move fast. A chain built on an old tutorial can fail to
  import on a fresh install. Pin `langchain`, `langchain-anthropic`, `langchain-chroma`, and
  `langchain-voyageai` in `requirements.txt`, and treat upgrades as a task with its own test pass,
  not a routine `pip install --upgrade`.
- **Turn on tracing before you debug, not after.** `langchain.debug = True` or a LangSmith trace
  shows you the exact prompt and tool schema a chain sent. Guessing at what a framework "probably"
  sent wastes time — check it directly.
- **Structured output still costs tokens and can still fail to match your schema** — with the
  extra step of an abstraction between you and the failure. If `with_structured_output` throws a
  parsing error, look at the raw model output (`with_structured_output(Model, include_raw=True)`)
  before assuming your Pydantic schema is at fault.
- **Don't let the framework hide cost.** Every extra system instruction LangChain adds is extra
  input tokens. For a high-volume, cost-sensitive path, measure the real request size (Chapter 12)
  rather than trusting that a convenience method is free.
- **Retrievers still need the Chapter 5 techniques.** Wrapping Chroma in `as_retriever()` does not
  add hybrid search, re-ranking, or query rewriting for you. You still design the retrieval quality
  yourself — LangChain gives you the plumbing to swap components, not better retrieval by default.
- **A chain is not an agent.** `prompt | llm | parser` runs once, start to end, with no branching
  or looping. If your GlobalMart Assistant needs to decide whether to retrieve, call a tool, or
  ask a follow-up question, that decision needs a loop or a graph — the subject of Chapters 7–8
  and, in the LangChain/LangGraph ecosystem, Chapter 11.
- **Test the underlying behavior, not the framework's API.** When you write evals (Chapter 6), test
  what the RAG chain answers, not that `rag_chain.invoke` was called correctly. That way, if you
  later replace LangChain with plain SDK code, or the reverse, your evals still tell you if the
  system still works.
