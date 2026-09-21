# Database Design Round Handbook

A **light, example-heavy** handbook for the **database design round** — the interview where you
design a schema for a system and defend it. It gives you a repeatable method, the reusable
building blocks, and **many worked designs**, each with the key decisions explained clearly and
the reasoning behind them.

Companion to `Database-Quick-Revision/` (concepts) and `Database-Interview-Handbook/` (the full
book). This one is all about **designing schemas well and explaining your choices**.

## How to use it
Read Chapters 1–2 first (the method and the building blocks). Then work through the case
studies. For each one, try to design the schema yourself first, then compare — and pay most
attention to the **Key Design Decisions — the Reasoning** section, because that is what the
interviewer really tests.

## Contents
| # | Chapter |
|---|---------|
| 1 | [How to Approach a Design Question](./Chapter-01-How-to-Approach-a-Design-Question.md) |
| 2 | [Design Building Blocks & Reusable Patterns](./Chapter-02-Design-Building-Blocks-and-Patterns.md) |
| 3 | [E-commerce / Orders](./Chapter-03-Ecommerce.md) |
| 4 | [Social News Feed](./Chapter-04-Social-News-Feed.md) |
| 5 | [Chat / Messaging](./Chapter-05-Chat-Messaging.md) |
| 6 | [Booking / Reservation (no double-booking)](./Chapter-06-Booking-Reservation.md) |
| 7 | [URL Shortener](./Chapter-07-URL-Shortener.md) |
| 8 | [Wallet / Ledger (money)](./Chapter-08-Wallet-Ledger.md) |
| 9 | [Ride-Hailing (Uber-style)](./Chapter-09-Ride-Hailing.md) |
| 10 | [Food Delivery](./Chapter-10-Food-Delivery.md) |
| 11 | [Q&A Site (Stack Overflow-style)](./Chapter-11-QA-Site.md) |
| 12 | [Notifications System](./Chapter-12-Notifications.md) |
| 13 | [Photo Sharing & Feed (Instagram-style)](./Chapter-13-Photo-Sharing-Feed.md) |
| 14 | [Reusable Patterns: Tagging, Threaded Comments, Multi-Tenancy](./Chapter-14-Reusable-Patterns.md) |
| 15 | [Subscription Billing / SaaS](./Chapter-15-Subscription-Billing.md) |
| 16 | [Common Mistakes & How to Ace the Design Round](./Chapter-16-Common-Mistakes-and-How-to-Ace-It.md) |

*Light and example-rich — about 45–55 pages total, with 13+ worked designs.*

## Conventions
- SQL is PostgreSQL; schemas are focused (key tables/columns, not every field).
- Money is integer minor units or NUMERIC, never float.
- Simple, easy English; reasoning-first — the "why" matters most.
