# Generative AI Handbook (Curated, Python-first)

A hands-on guide to building **real generative-AI and agentic systems** in Python. It is
**curated** — 17 chapters, no filler. Every chapter has runnable Python, and the chapters build
toward one production-style agent: the **"GlobalMart Assistant"** (answers questions from
company documents with RAG, and takes real actions through tools).

Made for an experienced backend engineer who is new to GenAI but strong in systems. So it skips
beginner filler, explains each new AI idea in **simple English**, and leans into the
production / system-design side.

## How to use it
Read top to bottom the first time — each chapter adds a piece and the last two assemble the full
agent. Later, use it as a reference. The Python examples use the **Anthropic (Claude) SDK**,
**LangChain + LangGraph**, and a vector store (**Chroma**/pgvector). See
[`_DESIGN-BRIEF.md`](./_DESIGN-BRIEF.md) for the exact stack, the running example, and the
current Claude API patterns all chapters follow.

## Table of Contents
| # | Chapter | Pages | Focus |
|---|---------|:----:|-------|
| 1 | [LLM & the Python SDK](./Chapter-01-LLM-and-Python-SDK.md) | 7 | Messages, streaming, tokens/cost, model choice; first working calls |
| 2 | [Prompting That Works](./Chapter-02-Prompting-That-Works.md) | 6 | System/user, few-shot, structured prompts, reliability |
| 3 | [Structured Output & Tool Calling](./Chapter-03-Structured-Output-and-Tool-Calling.md) | 8 | Typed/JSON output; function/tool calling in Python |
| 4 | [RAG Fundamentals](./Chapter-04-RAG-Fundamentals.md) | 9 | Embeddings, vector DB, chunking, retrieve-and-augment |
| 5 | [Advanced RAG & Re-ranking](./Chapter-05-Advanced-RAG-and-Reranking.md) | 8 | Hybrid search, re-ranking, query rewriting, filtering |
| 6 | [Evaluating RAG](./Chapter-06-Evaluating-RAG.md) | 7 | Measuring quality, hallucination, LLM-as-judge, golden sets |
| 7 | [From Calls to Agents](./Chapter-07-From-Calls-to-Agents.md) | 8 | Call vs workflow vs agent; a ReAct loop from scratch |
| 8 | [Tools & Memory](./Chapter-08-Tools-and-Memory.md) | 8 | Tool design/execution/errors; short- and long-term memory |
| 9 | [Multi-Agent & Patterns](./Chapter-09-Multi-Agent-and-Patterns.md) | 8 | Orchestrator-workers, router, planner, reflection |
| 10 | [LangChain](./Chapter-10-LangChain.md) | 8 | Core building blocks, composition, when to use vs plain SDK |
| 11 | [LangGraph](./Chapter-11-LangGraph.md) | 9 | Nodes/edges/state/cycles, human-in-the-loop, checkpointing |
| 12 | [Reliability, Cost & Latency](./Chapter-12-Reliability-Cost-Latency.md) | 9 | Retries, fallbacks, circuit breakers; caching; model routing |
| 13 | [Scaling & Long-Running Agents](./Chapter-13-Scaling-and-Long-Running-Agents.md) | 8 | Durable state, checkpointing, concurrency, rate limits |
| 14 | [Guardrails & Security](./Chapter-14-Guardrails-and-Security.md) | 8 | Validation, prompt-injection defense, PII, tool permissions |
| 15 | [Observability & Evaluation in Production](./Chapter-15-Observability-and-Evaluation.md) | 8 | Tracing, logging, metrics, offline evals, regression |
| 16 | [Reference Architecture](./Chapter-16-Reference-Architecture.md) | 8 | The full production agent, component by component |
| 17 | [Capstone Build + Decision Cheat Sheet](./Chapter-17-Capstone-Build.md) | 12 | Assemble the GlobalMart Assistant end to end; decision guide |

*Total ≈ 130 pages.*

## Conventions
- A "page" ≈ 450–550 words of substantive content.
- Simple, easy English. Every AI term is defined on first use.
- All Claude SDK code follows the current API patterns in `_DESIGN-BRIEF.md` §4.
- Each chapter ends with **What You Built / Learned** and **Production Notes & Pitfalls**.
