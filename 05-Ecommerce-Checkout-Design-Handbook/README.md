# E-commerce Checkout System Design Handbook

A complete, interview-oriented walkthrough of designing a **hyperscale e-commerce checkout
system** — anchored to one concrete scenario ("**GlobalMart Checkout**": a global multi-seller
marketplace, ~200M orders/day, ~50K order-placements/sec peak, multi-region active-active) so
every chapter's numbers, APIs, and data model stay mutually consistent.

Checkout is the **correctness-critical, revenue-critical** path: the whole design turns on
**never double-charging a buyer and never overselling a seller's inventory**, even under
20× flash-sale spikes. The recurring engineering ideas are the **saga pattern with
compensating transactions**, **idempotency keys**, **inventory reservation**, and
**reconciliation** — and a deliberate **CP (strong-consistency) stance** on the money path,
in pointed contrast to the AP stance of the companion Search handbook.

## How to use this handbook
Read top-to-bottom the first time (chapters build on each other). Later, use it as a
reference: each chapter is self-contained, cross-links the others, and ends with **Interview
Tips** and **Key Takeaways**.

The file [`_DESIGN-BRIEF.md`](./_DESIGN-BRIEF.md) is the **canonical source of truth** — the
fixed scenario, scale numbers, API surface, data model, saga, and architecture that all
chapters reference. Start there for the 5-minute summary.

## Table of Contents
| # | Chapter | Pages | Focus |
|---|---------|:----:|-------|
| 1 | [Requirement Gathering](./Chapter-01-Requirement-Gathering.md) | 10 | Scoping, functional/non-functional requirements, correctness-first priorities |
| 2 | [Capacity Estimation](./Chapter-02-Capacity-Estimation.md) | 10 | Orders/sec, peak spikes, storage, payment/inventory throughput |
| 3 | [APIs](./Chapter-03-APIs.md) | 8 | Session, shipping, payment, idempotent place-order contracts |
| 4 | [Data Model](./Chapter-04-Data-Model.md) | 8 | Session, Order, Reservation, Payment, Idempotency schemas |
| 5 | [High-Level Design](./Chapter-05-High-Level-Design.md) | 15 | End-to-end architecture, services, the place-order flow |
| 6 | [Checkout Orchestration & the Saga](./Chapter-06-Checkout-Orchestration-Saga.md) | 15 | Saga vs choreography, state machine, compensations |
| 7 | [Inventory Reservation & Consistency](./Chapter-07-Inventory-Reservation-Consistency.md) | 15 | No-oversell, reservations, hot-SKU contention, TTL |
| 8 | [Payments, Idempotency & Money Correctness](./Chapter-08-Payments-Idempotency-Money-Correctness.md) | 20 | PSP integration, idempotency keys, exactly-once charge, PCI, reconciliation |
| 9 | [Scaling](./Chapter-09-Scaling.md) | 15 | Sharding, flash-sale surge, multi-region active-active, hot SKUs |
| 10 | [Failure Handling](./Chapter-10-Failure-Handling.md) | 10 | Partial saga failure, PSP outages, DLQ, graceful degradation |
| 11 | [Trade-offs](./Chapter-11-Trade-Offs.md) | 10 | Saga vs 2PC, CP vs AP, reserve-vs-oversell, sync vs async capture |
| 12 | [Mock Interview](./Chapter-12-Mock-Interview.md) | 20 | Full 45-min interview transcript with commentary |

*Total ≈ 156 pages.*

## Conventions
- A "page" ≈ 450–550 words of substantive content.
- Diagrams are ASCII/mermaid; snippets are JSON / SQL / pseudo-code.
- Canonical names/numbers come from `_DESIGN-BRIEF.md`.
