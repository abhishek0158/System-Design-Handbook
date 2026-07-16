# Search System Design Handbook

A complete, interview-oriented walkthrough of designing a **hyperscale e-commerce product
search system** — anchored to one concrete scenario ("**GlobalMart Search**": 10B+ listings,
~100K QPS peak) so every chapter's numbers, APIs, and data model stay mutually consistent.

Built around **Elasticsearch** as the core search engine, with full coverage of the
indexing pipeline, the query/search pipeline, engine internals, scaling, failure handling,
and trade-offs — ending in a full mock interview.

## How to use this handbook
Read top-to-bottom the first time (chapters build on each other). Later, use it as a
reference: each chapter is self-contained, cross-links the others, and ends with **Interview
Tips** and **Key Takeaways**.

The file [`_DESIGN-BRIEF.md`](./_DESIGN-BRIEF.md) is the **canonical source of truth** — the
fixed scenario, scale numbers, API surface, data model, and architecture that all chapters
reference. Start there if you want the 5-minute summary.

## Table of Contents
| # | Chapter | Pages | Focus |
|---|---------|:----:|-------|
| 1 | [Requirement Gathering](./Chapter-01-Requirement-Gathering) | 10 | Scoping, functional/non-functional requirements, clarifying questions |
| 2 | [Capacity Estimation](./Chapter-02-Capacity-Estimation.md) | 10 | QPS, storage, bandwidth, memory, shard/node math |
| 3 | [APIs](./Chapter-03-APIs.md) | 8 | Search, autocomplete, indexing contracts; pagination, auth |
| 4 | [Data Model](./Chapter-04-Data-Model.md) | 8 | Document schema, mappings, analyzers, source of truth vs. index |
| 5 | [High-Level Design](./Chapter-05-High-Level-Design.md) | 15 | End-to-end architecture, read & write paths, components |
| 6 | [Indexing Pipeline](./Chapter-06-Indexing-Pipeline.md) | 15 | CDC → Kafka → transform → bulk index; NRT, versioning, reindex |
| 7 | [Search Pipeline](./Chapter-07-Search-Pipeline.md) | 15 | Query understanding, retrieval, ranking/LTR, faceting, caching |
| 8 | [Elasticsearch Internals](./Chapter-08-Elasticsearch-Internals.md) | 20 | Lucene, inverted index, segments, shards, scoring, refresh/merge |
| 9 | [Scaling](./Chapter-09-Scaling.md) | 15 | Sharding, replication, hot/warm, multi-region, caching tiers |
| 10 | [Failure Handling](./Chapter-10-Failure-Handling.md) | 10 | Failure modes, retries, DLQ, split-brain, graceful degradation |
| 11 | [Trade-offs](./Chapter-11-Trade-Offs.md) | 10 | Key decisions, alternatives, CAP, ES vs. alternatives |
| 12 | [Mock Interview](./Chapter-12-Mock-Interview.md) | 20 | Full 45-min interview transcript with commentary |

*Total ≈ 156 pages.*

## Conventions
- A "page" ≈ 450–550 words of substantive content.
- Diagrams are ASCII/mermaid; code snippets are Elasticsearch/JSON where relevant.
- Canonical names/numbers come from `_DESIGN-BRIEF.md`.
