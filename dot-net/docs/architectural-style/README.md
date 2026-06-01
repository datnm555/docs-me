# Architectural Styles in .NET

> Architectural **styles** define the **overall shape** of a system — how components are organized, where dependencies point, how communication happens. They are bigger than patterns and set the rules patterns operate inside.

---

## Quick Reference (What · Why · When · Where)

- **What** — Architectural **styles**: N-Layer, N-Tier, Clean, Hexagonal, Onion, Vertical Slice, Clean Slice, Event-Driven, plus the comparison and mixing docs.
- **Why** — Style sets the rules that patterns operate inside. Choosing the wrong style or mixing incompatible styles costs more than any pattern decision.
- **When** — Starting a new system; auditing whether your "Clean Architecture" actually points dependencies inward; deciding to mix Vertical Slice with Clean's outer rings; debating microservices vs. monolith.
- **Where** — Sits *above* architectural patterns (Saga, CQRS in `../architectural-pattern/`) and *below* methodology (DDD in `../ddd/`). Most apps pick one style on the principles axis + one on the logical axis + one tier count.

---

## Style vs. Pattern vs. Design Pattern

| Concept                    | Scope                  | Example                                                       |
| -------------------------- | ---------------------- | ------------------------------------------------------------- |
| **Architectural Style**    | System-wide structure  | Layered, Hexagonal, Onion, Clean, **EDA**, Microservices, REST |
| **Architectural Pattern**  | A service / workflow   | Saga, CQRS, Event Sourcing, Outbox, Circuit Breaker            |
| **GoF Design Pattern**     | Classes / objects      | Strategy, Observer, Decorator, …                              |

A system has **one or two styles** (e.g. "Clean Architecture + EDA"), uses **many patterns**, and contains **countless design patterns**.

---

## Index

| Style                              | File / Doc                                                       |
| ---------------------------------- | ---------------------------------------------------------------- |
| **N-Layer (Layered)**              | [`n-layer-architecture.md`](./n-layer-architecture.md)            |
| **N-Tier (physical deployment)**   | [`n-tier-architecture.md`](./n-tier-architecture.md)              |
| **Clean Architecture**             | [`clean-architecture.md`](./clean-architecture.md)                |
| **Hexagonal (Ports & Adapters)**   | [`hexagonal-architecture.md`](./hexagonal-architecture.md)        |
| **Vertical Slice**                 | [`vertical-slice-architecture.md`](./vertical-slice-architecture.md) |
| **Clean Slice** (hybrid)           | [`clean-slice-architecture.md`](./clean-slice-architecture.md)    |
| **Hexagonal vs. Onion vs. Clean** *(comparison)* | [`hexagonal-onion-clean.md`](./hexagonal-onion-clean.md) |
| **Style decision guide**           | [`architecture-comparison.md`](./architecture-comparison.md)      |
| **Five Architectures — Comparison & Mixing** | [`five-architectures-comparison-and-mixing.md`](./five-architectures-comparison-and-mixing.md) |
| **Event-Driven (EDA / EDD)**       | [`event-driven.md`](./event-driven.md)                            |
| Microservices                      | *(not yet expanded — see Chris Richardson's book)*               |
| REST / RPC / gRPC                  | *(communication styles)*                                         |
| Reactive                           | *(not yet expanded — see ReactiveX, Akka.NET)*                   |

---

## Choosing a Style

The choice is driven by **what changes most often** and **what you optimize for**:

| If your priority is…                                       | Lean toward…                               |
| ---------------------------------------------------------- | ------------------------------------------ |
| Simplicity, small team, CRUD app                           | **N-Layer** (Layered)                      |
| Testable domain, easy infra-swaps, complex business rules  | **Hexagonal / Clean / Onion**              |
| Loose coupling, scaling subsystems independently           | **Microservices** + **EDA**                |
| Real-time, audit-heavy, history-rich domains               | **EDA + Event Sourcing**                   |
| High-throughput streaming, push-based                      | **Reactive**                               |

You **can combine styles**: a Clean Architecture service inside an EDA microservices system is common and idiomatic.

---

## How Styles Compose with Patterns

```
Style       :  Clean Architecture  +  EDA
                        │
                        ▼
Pattern     :  CQRS, Saga, Outbox, Repository
                        │
                        ▼
Design      :  Strategy, Decorator, Observer (events), Mediator
```

Each layer's choices constrain the next.
