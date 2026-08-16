# Monolith vs Microservices

*Frontend Team Lead's notes — system design fundamentals for interview prep*

## Core Definitions

**Monolithic Architecture**
- All modules (auth, payments, orders) live in **one codebase, one database**
- Tightly coupled — shared memory space
- Simple to start, hard to scale/maintain as the app grows
- *Frontend analogy:* one giant SPA bundle — everything ships and breaks together

**Microservices Architecture**
- App broken into small, **loosely coupled, independently deployable services**
- Each service usually owns its **own dedicated database**
- Enables independent scaling — only scale the hot module, not the whole system
- *Frontend analogy:* micro-frontends — each feature is its own app, deployed separately

## Why Migrate? (Monolith Pain Points)

| Problem | Why it hurts |
|---|---|
| Single point of failure | One module crashing can take down the whole system |
| Deployment bottleneck | Any change = redeploy everything |
| Can't scale selectively | Must scale the entire app even if only one module is hot |

> **Interview tip:** Don't say "microservices are always better." Say monoliths are the right *starting* choice for small teams/simple products, and migration is justified once complexity/scale demands it. This nuance signals seniority.

## The Migration Strategy (6 Steps)

1. **Analyze the monolith** — find high-traffic/high-risk modules to prioritize
2. **Define API contracts** — REST/gRPC for sync calls, Kafka/message queues for async
3. **Code migration** — move module logic + its own DB to a new repo/service
4. **Traffic routing** — Strangler Fig + Canary Deployment (gradual cutover, not big-bang)
5. **Maintain consistency** — Outbox Pattern (keep old + new DB in sync during transition)
6. **Manage transactions** — Saga Pattern (handle multi-service transactions safely)

## Pattern Deep Dives

### 🌱 Strangler Fig Pattern
**Problem it solves:** How to cut over from monolith to microservice *safely*, without a risky big-bang release.

- A gateway/controller routes incoming requests
- Start small: e.g. 90% traffic → monolith, 10% → new microservice
- If the new service fails → instantly dial back to 0%, zero damage
- If stable → gradually increase to 100%
- At 100%, decommission the old monolithic module

*Frontend analogy:* feature-flagged canary rollout (e.g. LaunchDarkly rolling new checkout to 10% of users first).

### 🔗 Saga Pattern (intro — see `02-2pc-vs-saga.md` for the deep dive)
**Problem it solves:** Managing transactions that span multiple services, each with its own database (no single ACID transaction possible).

- Business process (e.g. an order) runs as a **chain of local transactions** across services
- If a later step fails, you can't "rollback" a DB that already committed elsewhere
- Instead, trigger **compensating transactions** — explicit reverse operations that undo prior steps
  - e.g. Order created ✅ → Payment fails ❌ → "cancel order" compensating action fires

*Frontend analogy:* optimistic UI updates — you don't rewind time on failure, you explicitly revert the UI state via a corrective action.

### 📤 Outbox Pattern
**Problem it solves:** Keeping the monolith's DB and the new microservice's DB in sync *during* the migration/transition period, without data drift.

- Writing to the DB and separately publishing an event (e.g. to Kafka) are two operations — if the app crashes between them, the two databases disagree
- Fix: write the actual data **and** an "event to send" row into the **same DB transaction** (the outbox table)
- A background process reliably reads the outbox and publishes events to Kafka → updates the microservice DB
- Guarantees atomicity: either both happen, or neither does

*Frontend analogy:* bundling a state update + a queued "event to emit" into one reducer action, instead of firing them as two separate, racy calls.

## Likely Interview Questions & Answers

**Q: When would you *not* migrate a monolith to microservices?**
When the team is small (roughly under 8-10 engineers), traffic is low/predictable, or the domain boundaries aren't well understood yet. Microservices add real operational cost — network calls, distributed debugging, more infra to run — that only pays off once a monolith's specific pain points (deploy contention, scaling one hot module, team-ownership conflicts) actually show up. Migrating too early is a common anti-pattern interviewers like to see you call out.

**Q: How do you decide which module to extract first?**
Look for the module that is simultaneously high-traffic/high-scaling-need AND relatively self-contained (few tight dependencies on other modules' data). Extracting a heavily-coupled module first maximizes migration pain for minimal benefit. A good signal: pick the module causing the most deploy bottlenecks or on-call pain today.

**Q: How does the Strangler Fig pattern reduce migration risk vs. a full cutover?**
A full cutover is all-or-nothing — if the new service has a bug, 100% of users are affected immediately with no fallback. Strangler Fig routes a small, controlled percentage of traffic first, so failures are caught while blast radius is small, and rollback is instant (just route traffic back to 0%) since the old monolith module is still running and untouched during the transition.

**Q: What are the trade-offs of eventual consistency introduced by Saga/Outbox?**
You gain availability, resilience, and independent scaling — but you give up the guarantee that all services see the same data at the exact same instant. There's a window where, e.g., an order shows "created" before payment has synced — so the UI/business logic has to be designed to tolerate brief staleness (e.g. showing "processing" states) rather than assuming instant cross-service consistency.

---
*Source: Ultimate System Design playlist, Video 1 (Monolith vs Microservices) — https://youtu.be/-r81RFGhVfw*
