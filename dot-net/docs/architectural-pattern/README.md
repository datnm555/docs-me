# Architectural Patterns in .NET

> Patterns that operate at the **service / distributed-system level** — bigger than GoF class-level patterns, smaller than full architectural styles. They solve recurring problems in microservices, messaging, data consistency, and resilience.

---

## What Counts as an "Architectural Pattern"?

* **Scope:** a service, a workflow across services, or a piece of cross-cutting infrastructure.
* **Granularity:** bigger than a class collaboration (GoF), smaller than a system-wide style (Hexagonal, EDA).
* **Canonical sources:** Chris Richardson — *Microservices Patterns* (2018); Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003); Microsoft — *Cloud Design Patterns*.

---

## Index

| Pattern               | Problem it solves                                                                            | File                                                  |
| --------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Saga**              | Distributed transactions across microservices (no 2PC).                                       | [`saga.md`](./saga.md)                                |
| **CQRS**              | Read and write workloads have different shapes and scaling needs.                            | [`cqrs.md`](./cqrs.md)                                |
| **Event Sourcing**    | Need a perfect audit trail and the ability to rebuild state from history.                    | [`event-sourcing.md`](./event-sourcing.md)            |
| **Outbox / Inbox**    | Atomically commit a DB write **and** publish a message (no dual-write problem).               | [`outbox.md`](./outbox.md)                            |

## Other Common Architectural Patterns (not yet expanded)

* **Circuit Breaker** — fail fast when a downstream is unhealthy. Use Polly.
* **Retry / Timeout / Bulkhead** — resilience primitives. Use Polly / `Microsoft.Extensions.Resilience`.
* **API Gateway / BFF** — single entry point per client type.
* **Strangler Fig** — incrementally replace a legacy system.
* **Sidecar** — co-deploy cross-cutting concerns (Envoy, Dapr).
* **Materialized View** — pre-compute read models.
* **Idempotent Receiver** — handle duplicates safely (often via Inbox).

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
