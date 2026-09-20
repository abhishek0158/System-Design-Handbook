# Database Interview Handbook

A practice-first guide to **clearing database interview rounds** for a backend engineer
(~3–4 years). It is organized around the three real rounds: the **SQL round**, the
**internals / concepts round**, and the **database design round**.

Every chapter uses the same format — **Key Concepts → The Questions They Ask (with answers)
→ Rapid-Fire → Common Traps** — with runnable PostgreSQL and worked schema designs. The
language is kept simple and easy to read.

See [`_DESIGN-BRIEF.md`](./_DESIGN-BRIEF.md) for the SQL dialect, the shared sample schemas
used across all SQL examples, and the house style.

## Table of Contents

### Part A — The SQL Round (write queries live)
| # | Chapter | Pages |
|---|---------|:----:|
| 1 | [SQL Refresher (fast)](./Chapter-01-SQL-Refresher.md) | 5 |
| 2 | [Joins](./Chapter-02-Joins.md) | 7 |
| 3 | [Aggregation, GROUP BY & HAVING](./Chapter-03-Aggregation-Group-By-Having.md) | 6 |
| 4 | [Subqueries & CTEs](./Chapter-04-Subqueries-and-CTEs.md) | 6 |
| 5 | [Window Functions](./Chapter-05-Window-Functions.md) | 8 |
| 6 | [SQL Problem Patterns](./Chapter-06-SQL-Problem-Patterns.md) | 10 |
| 7 | [Query Optimization for Interviews](./Chapter-07-Query-Optimization-for-Interviews.md) | 7 |

### Part B — The Internals / Concepts Round
| # | Chapter | Pages |
|---|---------|:----:|
| 8 | [Indexing](./Chapter-08-Indexing.md) | 8 |
| 9 | [Transactions & ACID](./Chapter-09-Transactions-and-ACID.md) | 6 |
| 10 | [Isolation Levels & Anomalies](./Chapter-10-Isolation-Levels-and-Anomalies.md) | 7 |
| 11 | [Locking, Concurrency & MVCC](./Chapter-11-Locking-Concurrency-and-MVCC.md) | 8 |
| 12 | [Storage Engines: B-tree, LSM & WAL](./Chapter-12-Storage-Engines-BTree-LSM-WAL.md) | 8 |
| 13 | [Replication](./Chapter-13-Replication.md) | 7 |
| 14 | [Partitioning & Sharding](./Chapter-14-Partitioning-and-Sharding.md) | 8 |
| 15 | [CAP, Consistency & Quorum](./Chapter-15-CAP-Consistency-and-Quorum.md) | 7 |

### Part C — The "Which Database?" Round
| # | Chapter | Pages |
|---|---------|:----:|
| 16 | [NoSQL Types & When to Use](./Chapter-16-NoSQL-Types-and-When-to-Use.md) | 7 |
| 17 | [SQL vs NoSQL Decision Framework](./Chapter-17-SQL-vs-NoSQL-Decision-Framework.md) | 6 |

### Part D — The Database Design Round
| # | Chapter | Pages |
|---|---------|:----:|
| 18 | [How to Approach a DB Design Question](./Chapter-18-How-to-Approach-DB-Design.md) | 6 |
| 19 | [Data Modeling & Normalization](./Chapter-19-Data-Modeling-and-Normalization.md) | 7 |
| 20 | [Design Case Studies (worked)](./Chapter-20-Design-Case-Studies.md) | 12 |
| 21 | [Scaling the Data Layer](./Chapter-21-Scaling-the-Data-Layer.md) | 8 |

### Part E — Practical + Final Prep
| # | Chapter | Pages |
|---|---------|:----:|
| 22 | [ORMs, JPA & Hibernate Pitfalls](./Chapter-22-ORMs-JPA-Hibernate-Pitfalls.md) | 6 |
| 23 | [Rapid-Fire Q&A](./Chapter-23-Rapid-Fire-QA.md) | 7 |
| 24 | [Mock Interview](./Chapter-24-Mock-Interview.md) | 12 |

*Total ≈ 180 pages.*

## Conventions
- SQL is PostgreSQL; MySQL differences are noted where they matter.
- All SQL examples use the shared schemas in `_DESIGN-BRIEF.md` §3.
- Simple, easy English; every term defined on first use.
- Each chapter ends with Rapid-Fire and Common Traps for quick revision.
