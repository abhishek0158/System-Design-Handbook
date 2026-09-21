# _DESIGN BRIEF — Canonical Source of Truth

> **Purpose:** Fixes the shared rules for the *Database Design Round Handbook*. Every chapter
> follows the language rules (§3) and the format (§4). It is light, clear, and example-rich.

## 0. Goal
A **light, reasoning-first, example-heavy** handbook for the **database design round** — the
interview where you design a schema for a system and defend it. Two hard rules:
1. **Lots of clear worked examples.** Show the real schema and, most importantly, explain the
   **key design decisions and the reasoning** behind them (the part interviewers test).
2. **Not a heavy read.** Short chapters, plain words, focused schemas (show the interesting
   tables and columns, not every field).

**Reader:** a backend engineer (~3–4 years) preparing for interviews. Keep it crisp.

## 1. SQL dialect & schema style
- Use **PostgreSQL** `CREATE TABLE` for schemas. Show primary keys, foreign keys, important
  columns, and the key indexes. Keep each design focused — do not list 30 columns; show the
  ones that matter and note the rest with "…".
- Money is always integer minor units (cents) or `NUMERIC`, never `float`.

## 2. Recurring design themes to reuse (mention where relevant)
- Model many-to-many with a junction table.
- Store history/status with an append-only table or a status column + `updated_at`.
- Keep a real event/ledger append-only when correctness matters (money).
- "Price/amount at time of the event" — copy the value onto the row, do not rely on the live
  price (it changes).
- Index for the main read pattern; denormalize only a hot read path, on purpose.
- Say assumptions and scale out loud; explain trade-offs — interviewers reward reasoning.

## 3. LANGUAGE — SIMPLE ENGLISH (most important)
- Short sentences, one idea each. Common words. No idioms, slang, or metaphors.
- Define each term in one simple sentence the first time it appears.
- Explain the idea first, then show the schema. Short paragraphs.

## 4. Per-chapter format
**Case-study chapters** (the worked designs) use these exact headings:
1. A 1–2 line intro: the system and why it is a common design question.
2. `## Requirements & Clarifying Questions` — key features, the questions to ask, rough scale.
3. `## The Schema` — PostgreSQL `CREATE TABLE`s with keys, plus a one-line note on each table.
4. `## Key Design Decisions — the Reasoning` — the 2–4 decisions the interviewer probes, each
   explained clearly with the WHY and the trade-off. **This is the most important section.**
5. `## Scaling It` — short: what breaks first and how to scale (index → cache → replica → shard).
6. `## Interview Tips & Common Mistakes`.

**The method / patterns / wrap-up chapters** (Ch1, Ch2, last chapter) use a light format that
fits their content (clear steps, a patterns list, or a checklist) — keep them short.

Target length: case studies ~1,200–1,700 words; method/patterns chapters ~2,000–2,500 words.
Prefer clear reasoning over length. Cross-reference other chapters by number where it helps.
