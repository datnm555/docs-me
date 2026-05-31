# Architectural Patterns in .NET

> Patterns that operate at the **service / distributed-system level** — bigger than GoF class-level patterns, smaller than full architectural styles. They solve recurring problems in microservices, messaging, data consistency, and resilience.

---

## Quick Reference (What · Why · When · Where)

- **What** — Patterns that operate at the service / distributed-system level — Saga, CQRS, Event Sourcing, Outbox/Inbox, plus brief notes on Circuit Breaker, API Gateway, Strangler Fig, Sidecar.
- **Why** — Solves the recurring problems of microservices: distributed transactions, read/write separation, atomic publish-and-save, resilience.
- **When** — Building or splitting a microservices system; introducing event-driven messaging; needing reliable cross-service workflows; replacing 2PC.
- **Where** — Between architectural style (Hexagonal, EDA — `architectural-style/`) and class-level patterns (GoF — `design-pattern/`). Often implemented using Repository (`enterprise-pattern/`) and DDD aggregates (`ddd/`).

---

## What Counts as an "Architectural Pattern"?

* **Scope:** a service, a workflow across services, or a piece of cross-cutting infrastructure.
* **Granularity:** bigger than a class collaboration (GoF), smaller than a system-wide style (Hexagonal, EDA).
* **Canonical sources:** Chris Richardson — *Microservices Patterns* (2018); Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003); Microsoft — *Cloud Design Patterns*.

---

## Index

### Distributed transactions & state coordination

| Pattern               | Problem it solves                                                                            | File                                                  |
| --------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Saga**              | Distributed transactions across microservices (no 2PC).                                       | [`saga.md`](./saga.md)                                |
| **2PC / TCC**         | Strong-consistency cross-service writes — and why microservices avoid 2PC.                    | [`two-phase-commit-tcc.md`](./two-phase-commit-tcc.md)|
| **CQRS**              | Read and write workloads have different shapes and scaling needs.                            | [`cqrs.md`](./cqrs.md)                                |
| **Event Sourcing**    | Need a perfect audit trail and the ability to rebuild state from history.                    | [`event-sourcing.md`](./event-sourcing.md)            |
| **Outbox / Inbox**    | Atomically commit a DB write **and** publish a message (no dual-write problem).               | [`outbox.md`](./outbox.md)                            |

### Delivery semantics & resilience

| Pattern               | Problem it solves                                                                            | File                                                  |
| --------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Idempotency**       | Safe retries — repeated execution = same effect. Foundation of every other resilience pattern. | [`idempotency.md`](./idempotency.md)                  |
| **Retry / Backoff / Jitter / DLQ** | Transient failure on async consumers — and parking what we can't fix.            | [`resilience-messaging.md`](./resilience-messaging.md)|
| **Timeout / Circuit Breaker / Bulkhead** | Synchronous cross-boundary calls — fail fast and stay alive.               | [`resilience-sync.md`](./resilience-sync.md)         |

### Decision guide

| Doc                                         | Use when                                                                                       |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| [Microservice Reliability Comparison](./microservice-reliability-comparison.md) | You're picking *which* patterns to use, or auditing a service for missing layers. |

## Other Common Architectural Patterns (not yet expanded)

* **API Gateway / BFF** — single entry point per client type.
* **Strangler Fig** — incrementally replace a legacy system.
* **Sidecar** — co-deploy cross-cutting concerns (Envoy, Dapr).
* **Materialized View** — pre-compute read models.

---

## How They Relate

```
                       ┌──────────────────────┐
                       │   Architectural      │
                       │      Style           │  Hexagonal / EDA / Microservices
                       └──────────┬───────────┘
                                  │ uses
                                  ▼
                       ┌──────────────────────┐
                       │  Architectural       │
                       │      Pattern         │  Saga / CQRS / Outbox / Circuit Breaker
                       └──────────┬───────────┘
                                  │ implemented with
                                  ▼
                       ┌──────────────────────┐
                       │  Enterprise +        │
                       │   GoF Patterns       │  Repository / Strategy / Decorator…
                       └──────────────────────┘
```

A microservice that uses **EDA** (style) might implement **Saga** (pattern) using **Mediator + Strategy** (GoF) and **Outbox** (pattern) backed by a **Repository** (enterprise pattern).
