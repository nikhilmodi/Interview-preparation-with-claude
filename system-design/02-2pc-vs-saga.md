# Two-Phase Commit (2PC) vs Saga Pattern

*Frontend Team Lead's notes — distributed transactions deep dive*

## Two-Phase Commit (2PC)
- A **synchronous, blocking protocol** for distributed transactions
- A **central coordinator** asks all participants "can you commit?" (phase 1 — prepare)
- Once all say yes, coordinator tells everyone to actually commit (phase 2 — commit)
- Provides **strong atomicity** — all succeed or all fail, like a single ACID transaction spread across systems
- **Downsides:** participants hold locks and wait on the coordinator the whole time → latency, deadlocks, reduced availability. If the coordinator fails mid-process, participants can get stuck holding locks indefinitely.

## Saga Pattern
- Replaces locking with a sequence of **local transactions**, each committed independently
- If a step fails, **compensating transactions** undo prior steps → **eventual consistency** instead of strong atomicity
- No cross-service locking → faster, more resilient, more scalable than 2PC

## Two Ways to Implement a Saga

| | Orchestrated Saga | Choreographed Saga |
|---|---|---|
| **Who's in control** | A central orchestrator explicitly tells each service what to do | No central controller — services react autonomously to events |
| **Traceability** | Easy to audit — one place to see the full flow | Harder to trace — logic is scattered across services |
| **Coupling** | Services depend on the orchestrator | Services stay loosely coupled |
| **Scalability** | Orchestrator can become a bottleneck | Scales better — no single point of coordination |
| **Best for** | Simpler flows where visibility/audit matters | Complex flows with many services, prioritizing decoupling |

*Frontend analogy:* Orchestrated = parent component explicitly calling child functions, controlling the sequence. Choreographed = event bus / pub-sub, where components just listen and react independently with no single owner of the full flow.

## 2PC vs Saga — Full Comparison

| Aspect | Two-Phase Commit (2PC) | Saga |
|---|---|---|
| Coordination | Central coordinator | Choreographed (events) or orchestrated (central saga) |
| Communication | Synchronous | Asynchronous |
| Atomicity | Strong atomicity | Eventual consistency, compensating transactions |
| Flexibility/Resilience | Less flexible, single point of failure | More flexible, resilient, no single point of failure |
| Performance | Slower, participants wait | Faster, independent operations |
| Locking | Involves resource locking → contention | No resource locking, separate local transactions |
| Use Cases | Strong consistency, limited participants | Long-running, complex transactions, multiple services, eventual consistency preferred |

## When to Choose Choreography over Orchestration
- **High scalability needs** — no central orchestrator = no single performance bottleneck
- **Want loose coupling** — services react to events instead of being commanded
- **Complex workflows with many services** — avoids maintaining one giant central state machine

**Trade-off to know:** choreography adds *implementation complexity* (every service needs to understand and react to more event types) and makes tracing a transaction's full journey harder than with an orchestrator's clear audit trail.

## Interview Questions
- Why is 2PC considered a poor fit for microservices at scale?
- What specifically causes 2PC to risk deadlocks?
- When would you choose Orchestrated Saga over Choreographed, despite the bottleneck risk?
- How do you debug/trace a transaction in a choreographed saga without a central log of "what happened"? (hint: distributed tracing / correlation IDs — worth researching further)
- If eventual consistency is "good enough," why does 2PC still get used anywhere?

---
*Source: Supplementary video — "Sagas vs Two-Phase Commit (2PC)"*
