# Context Distiller — Project Pack (for the person leading it)

You are going to **lead a small team** (developer + QA + product) to build this in about
**5 weeks**. You are not sure yet what it is or how to run it. These files fix that.

Read them in this order:

| # | File | What it gives you |
|---|------|-------------------|
| 1 | [Project-Playbook.md](./Project-Playbook.md) | The full plan. What we build, the two phases (offline preprocess + online retrieval), and for **each stage**: what it is, **why** we do it, **how** to do it, which tools, who owns it, and when it is "done". Plus a small LangGraph code skeleton for the agentic loop, a risk table, the 5-week day-by-day plan, and how we measure success. |
| 2 | [Worked-Example.md](./Worked-Example.md) | A full sample: one task ("add rate limiting to the login endpoint" in a small Flask app), showing the **real data** at each stage — from a raw file, to its skeleton, to a cached summary, to the ranked symbols, then the full online retrieval loop, ending in a token count comparison. |

**One-line summary of the project:**
Preprocess a code repository **once** into a compact, searchable form (signatures, summaries,
a call graph, embeddings). Then, for every coding task, serve the coding assistant only the
**small, ranked slice of context it actually needs** — through an MCP server (Model Context
Protocol server — a standard way for tools like Claude Code, Copilot, and Cursor to call an
outside tool). The result: the assistant uses **far fewer tokens** (tokens = the units an LLM
is billed and limited by) and costs less, while giving an equal-or-better answer.

**Why this is not "just a repo map":** tools like Aider already build a static repo map.
Our real difference is the **agentic loop** — the LLM itself decides, step by step, what more
context it needs and when it has enough — plus a real evaluation harness that proves the
token savings with numbers. Part A of the Playbook explains this honestly.

Everything here is written in simple English on purpose. Use it as-is with your team.
