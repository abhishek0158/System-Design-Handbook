# _DESIGN BRIEF — Canonical Source of Truth

> **Purpose:** This file fixes every shared fact for the *Database Interview Handbook*. Every
> chapter MUST follow the language rules (§4), the per-chapter format (§5), the SQL dialect (§2),
> and the shared sample schemas (§3) so the book is consistent. Do not contradict these.

---

## 0. Goal of this handbook
Help a backend engineer **clear database interview rounds**. The three rounds it targets:
1. **The SQL round** — write queries live (joins, aggregation, window functions, problem patterns).
2. **The internals / concepts round** — explain how databases work (indexes, transactions,
   isolation, MVCC, storage engines, replication, sharding, CAP).
3. **The database design round** — design a schema for a given system and defend it.

It is **practice-first**: real interview questions with clear answers, worked SQL, and worked
schema designs — not just theory.

**Reader:** a backend engineer with ~3–4 years of experience (Java/Spring background). Skip
absolute-beginner filler. Focus on what interviews actually test. Explain each idea clearly.

---

## 2. SQL dialect
- **Default dialect: PostgreSQL.** Use Postgres syntax for all examples (it has window
  functions, `EXPLAIN ANALYZE`, MVCC, CTEs — everything interviews cover).
- When something differs in **MySQL** in a way interviews care about, add a short "MySQL note".
- Prefer standard ANSI SQL where possible so the queries are portable.

## 3. Shared sample schemas (USE THESE in every SQL chapter, so examples stay consistent)

**Schema A — HR (for joins, aggregation, window functions, self-join):**
```sql
departments(dept_id PK, dept_name)
employees(emp_id PK, emp_name, dept_id FK, manager_id FK->employees.emp_id,
          salary, hire_date)
```

**Schema B — E-commerce (for grouping, top-N, running totals, design):**
```sql
customers(customer_id PK, name, country, created_at)
products(product_id PK, name, category, price)
orders(order_id PK, customer_id FK, order_date, status, total_amount)
order_items(order_id FK, product_id FK, quantity, unit_price)   -- PK (order_id, product_id)
```

Reuse these tables in query examples. Only introduce a new table if a chapter truly needs it,
and say so.

---

## 4. LANGUAGE — SIMPLE ENGLISH (most important rule)
The reader prefers simple, easy-to-read English (non-native friendly).
- Short sentences, one idea each (~15–20 words). Common words. No idioms, slang, or metaphors.
- Define each technical term in one simple sentence the first time it appears.
- Explain the idea first, then show the SQL or the diagram. Short paragraphs.
- Keep full technical depth — only the language is simple, not the content.

## 5. Per-chapter format (use these exact section headings)
Each chapter has these sections, in this order:
1. A 2–3 line intro: what this topic is and why interviewers ask it.
2. `## Key Concepts` — the must-know points, with worked SQL / small diagrams / tables.
3. `## The Questions They Ask` — the real interview questions at 3–4 YOE, each with a clear,
   concise answer (and the follow-up questions interviewers add).
4. `## Rapid-Fire` — short question → 1–2 line answer, for quick revision.
5. `## Common Traps & Mistakes` — what candidates get wrong, and the subtle points.

Other rules:
- Give **runnable, correct SQL** (PostgreSQL) using the shared schemas (§3).
- For design chapters, give real **schema designs** (tables, keys, indexes) and the trade-offs.
- Refer back to other chapters by number.
- A "page" ≈ 450–550 words. Follow the per-chapter page targets in the README.
- Recurring interview framing to repeat: **say your assumptions out loud, reason about
  trade-offs, and state the "it depends" clearly** — interviewers reward reasoning, not just the
  final answer.
