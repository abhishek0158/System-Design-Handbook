# <System Name> — System Design

> One-line summary of what we're designing and for whom.

---

## 1. Problem Statement & Clarifying Questions
- Restate the problem in my own words.
- **Clarifying questions I'd ask the interviewer** (and the assumptions I'll proceed with if unanswered):
    - Scope: which features are in / out?
    - Scale: how many users / requests / data volume?
    - Read vs. write heavy?
    - Consistency vs. availability priority?
    - Latency expectations?

## 2. Requirements
### Functional (what the system must *do*)
- FR1 …
- FR2 …

### Non-Functional (how *well* it must do it)
- Availability, latency, consistency, durability, scalability, security.
- Explicitly state which are prioritized and why.

### Out of Scope
- Things we consciously exclude to bound the discussion.

## 3. Back-of-the-Envelope Estimation
- **Traffic:** DAU → QPS (avg & peak), read:write ratio.
- **Storage:** per-object size × volume × retention.
- **Bandwidth:** ingress/egress.
- **Memory (cache):** working set, 80/20 rule.
- Note the assumptions behind each number.

## 4. API Design
- Public-facing contracts (REST/gRPC/GraphQL). Method, path, params, response.
- Auth model, pagination, idempotency where relevant.

## 5. Data Model
- Core entities & relationships.
- Schema sketch (tables/collections, key fields, indexes).
- SQL vs. NoSQL choice **with justification**.

## 6. High-Level Architecture
- Component diagram described in text (client → LB → services → data stores → async workers).
- Request flow for the 1–2 most important operations (read path + write path).

## 7. Deep Dives
> The part that separates senior candidates. Pick 2–4 areas and go deep.
- Chosen deep-dive topics and the design decisions within each.
- Concrete algorithms / data structures where relevant.

## 8. Scaling & Bottlenecks
- Where does it break as we grow 10×/100×?
- Techniques: sharding/partitioning, replication, caching layers, CDN, queues, load balancing.
- Hot-spot / celebrity / thundering-herd problems and mitigations.

## 9. Trade-offs & Alternatives
- Key decisions, the alternatives considered, and why we chose what we chose.
- CAP positioning, consistency model, sync vs. async.

## 10. Reliability & Operations
- Failure modes & handling (retries, circuit breakers, idempotency, dead-letter queues).
- Redundancy, failover, backups, disaster recovery.
- Observability: metrics, logging, tracing, alerting.

## 11. Summary
- 3–5 sentence wrap-up: the core design, the key trade-offs, and what I'd revisit with more time.

---
*Difficulty: <Easy/Medium/Hard> · Key concepts: <tags> · Date: <YYYY-MM-DD>*
