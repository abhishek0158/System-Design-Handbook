# _DESIGN BRIEF — Canonical Source of Truth

> **Purpose:** Fixes the shared rules for the *Database Quick-Revision Handbook*. Every chapter
> follows the language rules (§3) and the light format (§4). Keep it short and reasoning-first.

## 0. Goal
A **focused, light** interview-revision handbook on a fixed list of database topics. The reader
is preparing for an interview soon. Two hard rules:
1. **Enough for the interview, plus the REASONING.** Do not just state facts — explain the *why*
   and the *trade-off* behind each idea, because interviewers reward reasoning.
2. **Not a heavy read.** Short chapters, plain words, small examples. No deep engine internals
   unless they directly help the reasoning.

**Reader:** a backend engineer (~3–4 years). Skip beginner filler; keep it crisp.

## 1. Topics (one chapter each — do not add or drop)
1. Primary / Foreign Keys & Constraints
2. Indexes — B-tree, Composite, Covering, and Index Selectivity
3. Transactions & ACID
4. Isolation Levels & Anomalies
5. Locks & MVCC
6. Optimistic vs Pessimistic Locking
7. Deadlocks
8. Normalization & Denormalization
9. Pagination

## 2. SQL dialect & examples
- Use **PostgreSQL** for the few SQL snippets. Keep examples tiny (a couple of tables like
  `orders(order_id, customer_id, order_date, status, total_amount)` or `employees(emp_id,
  dept_id, salary)`). Do not build big schemas — this is a light book.

## 3. LANGUAGE — SIMPLE ENGLISH (most important)
- Short sentences, one idea each. Common words. No idioms, slang, or metaphors.
- Define each term in one simple sentence the first time it appears.
- Explain the idea first, then a tiny example. Short paragraphs.
- Keep the depth honest but light — do not go down rabbit holes.

## 4. Per-chapter format (use these exact headings, keep it SHORT)
1. A 1–2 line intro: what this is and why interviewers ask it.
2. `## The Idea` — what it is, in plain words.
3. `## Why It Matters — the Reasoning` — the trade-off and the "why" behind it. **This section is
   the point of the book — make the reasoning clear.**
4. `## Common Interview Questions` — 4–6 likely questions, each with a short, crisp answer that
   *includes the reasoning*, not just the fact.
5. `## Quick Recall` — a few one-line bullets and the single biggest gotcha to remember.

Target length per chapter: **~1,500–2,200 words** (short). Prefer clear reasoning over length.
Cross-reference the other chapters by number where it helps.
