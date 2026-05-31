# Microservice Reliability Patterns — Comparison & Decision Guide

> A map of the reliability patterns covered in this folder — Saga, 2PC, TCC, Outbox, Inbox, Idempotency, Retry/Backoff/DLQ, Circuit Breaker, Bulkhead, Timeout — organized by **the problem they solve** and **how they compose**. None of them are alternatives in isolation; a production service uses a layered stack of them.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — A decision guide and decision matrix for choosing the right reliability pattern for the right failure mode in a microservice. Cross-references every pattern in `dot-net/docs/architectural-pattern/` plus the canonical "what about 2PC?" question.
- **Why** — These patterns get mistaken for alternatives. They are not. Saga is not "instead of" Idempotency; Circuit Breaker is not "instead of" Retry. Knowing where each fits stops you from over-engineering or leaving a layer unprotected.
- **When** — Designing a new microservice; reviewing an incident postmortem ("which layer failed?"); deciding whether to introduce a new pattern or fix an existing one; explaining the trade-off to a team.
- **Where** — `dot-net/docs/architectural-pattern/` — every entry in the [Index](./README.md#index) is referenced here. Cross-links also point into `architectural-style/event-driven.md` and `ddd/`.
- **How** — Identify the failure mode first (commit coordination? at-least-once delivery? downstream slowdown?). Match it to the pattern layer below. Most services need at least one pattern from each layer.

---

## The Four Layers of Reliability

Every microservice that talks to another microservice has to answer **four** independent questions. Most outages happen because one of the four was left unanswered.

```
┌───────────────────────────────────────────────────────────────────────────┐
│ Layer 4 — Synchronous resilience                                          │
│   "Don't let a slow downstream kill us."                                  │
│   ► Timeout · Circuit Breaker · Bulkhead                                  │
├───────────────────────────────────────────────────────────────────────────┤
│ Layer 3 — Messaging resilience                                            │
│   "Don't let transient failure or poison messages stall us."              │
│   ► Retry · Backoff+Jitter · DLQ                                          │
├───────────────────────────────────────────────────────────────────────────┤
│ Layer 2 — Delivery semantics                                              │
│   "Make duplicate delivery harmless."                                     │
│   ► Idempotency · Inbox                                                   │
├───────────────────────────────────────────────────────────────────────────┤
│ Layer 1 — Write coordination                                              │
│   "Make state consistent across services."                                │
│   ► Saga · TCC · (2PC, mostly avoided) · Outbox · Event Sourcing          │
└───────────────────────────────────────────────────────────────────────────┘
```

Notice the layers are **independent**. You can have a perfect Saga and still lose data if the participating services have no Outbox. You can have a flawless Outbox + Inbox and still suffer a thundering-herd outage if the consumers have no backoff. Each layer fails on its own and must be addressed on its own.

---

## Pattern Map by Problem

### Layer 1 — Cross-service write coordination

| Problem                                                                          | Pattern                                            | Why                                                                                       |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Atomic write across services with strict ACID                                    | **2PC** ([`two-phase-commit-tcc.md`](./two-phase-commit-tcc.md)) | Only viable in single-vendor / same-process scenarios. Blocks under failure. **Avoid for microservices.** |
| Long-running business workflow with semantic rollback                            | **Saga** ([`saga.md`](./saga.md))                  | Local TX + compensating action. Default for distributed transactions in microservices.    |
| "Reserve then confirm" business model (travel, ticketing)                        | **TCC** ([`two-phase-commit-tcc.md`](./two-phase-commit-tcc.md)) | Three-phase API per service. Lighter than 2PC, stricter than Saga.                       |
| Atomically (1) write to DB and (2) publish message                               | **Outbox** ([`outbox.md`](./outbox.md))            | The dual-write problem. Mandatory for at-least-once messaging.                            |
| Need a perfect audit trail; need to rebuild state from history                   | **Event Sourcing** ([`event-sourcing.md`](./event-sourcing.md)) | Events are source of truth, publish is free, audit is free. Big trade-offs.           |

### Layer 2 — Delivery semantics

| Problem                                                                          | Pattern                                            | Why                                                                                       |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| HTTP retries cause duplicate side effects (`POST /payments`)                     | **Idempotency Key** ([`idempotency.md`](./idempotency.md)) | Server dedupes by client-supplied key.                                              |
| At-least-once message broker redelivers the same message                          | **Inbox** ([`outbox.md`](./outbox.md))             | Consumer records processed message IDs. Symmetric with Outbox.                            |

These two are the same idea applied at different boundaries: HTTP edge vs. message consumer. Without them, every retry in Layer 3 is a duplication bug.

### Layer 3 — Messaging resilience (async)

| Problem                                                                          | Pattern                                            | Why                                                                                       |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Transient downstream failure (rate limit, 5xx, network blip)                     | **Retry + Exponential Backoff + Jitter** ([`resilience-messaging.md`](./resilience-messaging.md)) | Survive the blip without thundering herd.    |
| Message we'll never be able to handle (validation, schema, bug)                  | **Dead Letter Queue** ([`resilience-messaging.md`](./resilience-messaging.md)) | Park the poison, page on-call, re-drive after fix.                 |

### Layer 4 — Synchronous resilience (sync)

| Problem                                                                          | Pattern                                            | Why                                                                                       |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Slow downstream holds our threads / connections                                  | **Timeout** ([`resilience-sync.md`](./resilience-sync.md))    | Hard deadline. Required for every cross-boundary call.                              |
| Downstream is consistently failing — stop hitting it                              | **Circuit Breaker** ([`resilience-sync.md`](./resilience-sync.md)) | Fail-fast + half-open probe. Pairs with a fallback.                              |
| One slow dependency starves the whole service                                    | **Bulkhead** ([`resilience-sync.md`](./resilience-sync.md))   | Per-dependency thread / connection pool.                                            |

---

## Decision Matrix

### "Pick one transaction strategy"

| If your problem is…                                                          | Use                                | Don't use                       |
| ----------------------------------------------------------------------------- | ---------------------------------- | ------------------------------- |
| One service, two local DBs                                                    | Local TX or MSDTC                  | Saga (overkill)                 |
| Many services, eventual consistency is fine                                   | **Saga**                           | 2PC, TCC                        |
| Many services, business *already* means "reserve then confirm"               | **TCC**                            | Plain Saga (loses reservation semantics) |
| Many services, must be strict ACID                                            | Re-evaluate boundaries; or 2PC inside the same vendor | Greenfield 2PC across services |
| Audit-heavy domain, rebuilding history matters                                | **Event Sourcing** + Saga          | Plain Saga (loses history)      |

### "Pick reliability primitives" (every service needs these)

| Failure mode                                       | Sync (HTTP/gRPC)            | Async (queue/topic)                                |
| -------------------------------------------------- | --------------------------- | -------------------------------------------------- |
| Duplicate request / message                        | Idempotency Key             | Inbox                                              |
| Transient failure                                  | Retry + backoff + jitter    | Retry + backoff + jitter                           |
| Persistent failure                                 | Circuit Breaker + fallback  | DLQ + alert + re-drive                             |
| Slowness                                           | Timeout, Bulkhead           | Visibility timeout / lease                         |
| Atomic write + publish                             | n/a                         | **Outbox**                                         |
| Coordinator failure mid-workflow                   | Stateless retry             | Durable saga state (event-sourced or DB-persisted) |

### "Pick the right pattern when X tempts you"

| You're tempted to…                                            | But you really need…                                            |
| ------------------------------------------------------------- | ---------------------------------------------------------------- |
| Use 2PC across microservices                                  | Saga (or rethink boundaries)                                     |
| Just publish to the broker after `SaveChangesAsync`            | Outbox                                                           |
| Just check "already handled?" in memory                       | Inbox (durable, transactional)                                   |
| Retry forever until it works                                  | Cap + backoff + DLQ                                              |
| Catch all exceptions and retry                                | Classify transient vs terminal, DLQ terminal                     |
| Set `HttpClient.Timeout = TimeSpan.FromMinutes(5)`            | A real per-attempt timeout sized off downstream p99               |
| Add a circuit breaker because a call sometimes fails           | Confirm there's a fallback first; otherwise the breaker just changes the error |
| Share `HttpClient` between Pricing and Catalog                | Named client per downstream + bulkhead                           |

---

## How the Patterns Compose — E-Commerce Checkout End-to-End

A single "Place Order" use case that exercises every layer:

```
                          ┌─────────────────────────────────────────────┐
                          │  Checkout Service                           │
                          │                                             │
HTTP  POST /orders ──────►│  [Idempotency middleware] (Layer 2)         │
   Idempotency-Key: K     │      │                                      │
                          │      ▼                                      │
                          │  Validate cart                              │
                          │      │                                      │
                          │      ▼                                      │
                          │  Call Pricing  (Layer 4: Timeout +          │
                          │      │          Circuit Breaker + Bulkhead) │
                          │      ▼                                      │
                          │  Start Saga (Layer 1)                        │
                          │      │                                      │
                          │      ▼                                      │
                          │  Local TX:                                  │
                          │    • Save Order                             │
                          │    • Save Outbox row OrderPlaced (Layer 1)  │
                          │      │                                      │
                          │      ▼                                      │
                          │  Outbox Publisher ── publish → [Broker]     │
                          └─────────────────────────────────────────────┘
                                                            │
                                                            │ at-least-once
                                                            ▼
                          ┌─────────────────────────────────────────────┐
                          │  Payment Service                            │
                          │      │                                      │
                          │      ▼                                      │
                          │  [Inbox check] (Layer 2)                    │
                          │      │                                      │
                          │      ▼                                      │
                          │  Charge via Stripe (Layer 4 on Stripe call) │
                          │      │ fail with 429? → Retry+backoff+jitter (Layer 3)
                          │      │ fail terminal? → DLQ (Layer 3)        │
                          │      ▼                                      │
                          │  Local TX:                                  │
                          │    • Save Payment                           │
                          │    • Save Inbox row                         │
                          │    • Save Outbox row PaymentCompleted        │
                          └─────────────────────────────────────────────┘
                                                            │
                                                            ▼
                                                     [Broker] … saga continues
```

What's protecting each step:

* **Layer 1** — Outbox guarantees `Order` and `OrderPlaced` event are written atomically. Saga sequences `OrderPlaced → ChargePayment → ReserveStock → ShipOrder` with compensations if any step fails.
* **Layer 2** — Idempotency middleware ensures `POST /orders` retried by the client doesn't double-charge. Inbox in PaymentService dedupes the redelivered `OrderPlaced`.
* **Layer 3** — Retry+backoff+jitter handles Stripe rate-limits. DLQ catches Stripe responses that won't ever succeed (invalid card metadata).
* **Layer 4** — Timeout caps each call to Pricing/Stripe. Circuit Breaker opens if Pricing is consistently down → checkout falls back to last-known price. Bulkhead isolates the Pricing client so its slowness can't starve Stripe calls.

Remove any one layer and you have a class of outages:

| Removed                | Outage type                                                                  |
| ---------------------- | ---------------------------------------------------------------------------- |
| Outbox                 | Lost events — Order saved but no downstream notified. Silent.                |
| Idempotency / Inbox    | Duplicate charges, duplicate orders.                                          |
| Retry+backoff          | Single transient blip = lost message. No jitter = thundering herd on recovery. |
| DLQ                    | One poison message stalls the partition.                                      |
| Timeout                | A slow Pricing → checkout threads exhausted → 503 storm.                      |
| Circuit Breaker        | Repeated calls to a dead Pricing → repeated 5s timeouts hurt user experience. |
| Bulkhead               | Pricing slowness leaks into Stripe call latency.                              |

---

## Anti-Patterns: Visible Holes

* **Using one pattern as a substitute for another layer.** "We have Retry, we don't need a Circuit Breaker." Retry without a breaker is a thundering herd.
* **"Distributed transactions are hard, so we just won't coordinate."** Lost orders aren't a "trade-off." They're a bug. Use Saga + Outbox.
* **Choreography saga + at-least-once + no idempotency.** Duplicates will happen. Always.
* **No timeout because "we'll add it later."** "Later" is the postmortem.
* **DLQ with no alert.** A black hole. The bug is invisible until a customer complains.
* **Idempotency key as the request body hash.** Whitespace differences look like different keys. Use an explicit client-supplied UUID.

---

## References

* Chris Richardson — *Microservices Patterns* (2018). Canonical mapping of these patterns to microservices.
* Michael Nygard — *Release It!* (2nd ed.). Stability Patterns — Timeout, Circuit Breaker, Bulkhead, Fail Fast.
* Sam Newman — *Building Microservices* (2nd ed.). Chapter 12 (Resilience), Chapter 6 (Workflow).
* Gregor Hohpe, Bobby Woolf — *Enterprise Integration Patterns* (2003). Inbox, Outbox, Idempotent Receiver, Dead Letter Channel.
* Pat Helland — [*Life beyond Distributed Transactions: an Apostate's Opinion*](https://queue.acm.org/detail.cfm?id=3025012). Why we stopped doing 2PC.
* [microservices.io/patterns](https://microservices.io/patterns/) — patterns catalog cross-referenced throughout.
