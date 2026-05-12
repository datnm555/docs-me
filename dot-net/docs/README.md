# .NET Architecture & Patterns — Documentation Map

> This folder organizes software patterns by their **proper category**. "Pattern" is an overloaded word — Saga, DDD, EDD, Repository, and the GoF 23 all live at different levels of abstraction. Use this map to find the right name for the right concept.

---

## The Pattern Taxonomy

| Level                          | Folder / File                                      | Examples                                                                                              |
| ------------------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Methodology / Practice**     | [`ddd/`](./ddd/)                                   | **DDD**, TDD, BDD, Agile                                                                              |
| **Architectural Style**        | [`architectural-style/`](./architectural-style/)   | Layered, Hexagonal, Onion, Clean, **EDA / EDD**, Microservices, REST                                  |
| **Architectural Pattern**      | [`architectural-pattern/`](./architectural-pattern/) | **Saga**, CQRS, Event Sourcing, Outbox/Inbox, Circuit Breaker, API Gateway, Strangler Fig             |
| **Enterprise Pattern**         | [`enterprise-pattern/`](./enterprise-pattern/)     | **Repository**, Unit of Work, Specification, Active Record, Data Mapper, Service Layer                |
| **Tactical DDD Pattern**       | [`ddd/tactical-patterns.md`](./ddd/tactical-patterns.md) | Aggregate, Entity, Value Object, Domain Event, Domain Service, Factory                            |
| **Strategic DDD Pattern**      | [`ddd/strategic-patterns.md`](./ddd/strategic-patterns.md) | Bounded Context, Ubiquitous Language, Context Map, Anti-Corruption Layer                          |
| **Design Pattern (GoF, 23)**   | [`design-pattern/`](./design-pattern/)             | Strategy, Observer, Decorator, Factory, Builder…                                                      |
| **Idiom / Language Technique** | (per language)                                     | Generic Repository, RAII, Disposable, `IAsyncEnumerable<T>` streaming                                 |

**Rule of thumb on scope:** the higher the level (system → service → class → line of code), the higher up the table you go. GoF patterns sit near the bottom — they describe class collaboration *inside one process*.

---

## Quick "Where Does X Belong?"

| Concept                | Category                                                         | See                                                       |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------- |
| **Saga**               | Architectural Pattern (distributed transaction)                  | [`architectural-pattern/saga.md`](./architectural-pattern/saga.md) |
| **DDD**                | Methodology (containing tactical + strategic *patterns*)         | [`ddd/README.md`](./ddd/README.md)                        |
| **EDA / EDD**          | Architectural Style                                              | [`architectural-style/event-driven.md`](./architectural-style/event-driven.md) |
| **CQRS**               | Architectural Pattern                                            | [`architectural-pattern/cqrs.md`](./architectural-pattern/cqrs.md) |
| **Event Sourcing**     | Architectural Pattern                                            | [`architectural-pattern/event-sourcing.md`](./architectural-pattern/event-sourcing.md) |
| **Outbox / Inbox**     | Architectural Pattern (messaging reliability)                    | [`architectural-pattern/outbox.md`](./architectural-pattern/outbox.md) |
| **Repository**         | Enterprise Pattern (also a Tactical DDD pattern)                 | [`enterprise-pattern/repository.md`](./enterprise-pattern/repository.md) |
| **Generic Repository** | **Implementation idiom** — often an anti-pattern in DDD          | [`enterprise-pattern/repository.md`](./enterprise-pattern/repository.md) |
| **Unit of Work**       | Enterprise Pattern                                               | [`enterprise-pattern/unit-of-work.md`](./enterprise-pattern/unit-of-work.md) |
| **Aggregate**          | Tactical DDD Pattern                                             | [`ddd/tactical-patterns.md`](./ddd/tactical-patterns.md)  |
| **Bounded Context**    | Strategic DDD Pattern                                            | [`ddd/strategic-patterns.md`](./ddd/strategic-patterns.md) |
| **Strategy / Observer / Decorator / …** | GoF Design Patterns                                 | [`design-pattern/`](./design-pattern/)                    |
| **N-Layer Architecture** | Architectural Style                                            | [`architectural-style/n-layer-architecture.md`](./architectural-style/n-layer-architecture.md) |
| **N-Tier Architecture** | Architectural Style (physical deployment)                        | [`architectural-style/n-tier-architecture.md`](./architectural-style/n-tier-architecture.md) |
| **Clean Architecture** | Architectural Style                                              | [`architectural-style/clean-architecture.md`](./architectural-style/clean-architecture.md) |
| **Hexagonal Architecture** | Architectural Style                                          | [`architectural-style/hexagonal-architecture.md`](./architectural-style/hexagonal-architecture.md) |
| **Vertical Slice Architecture** | Architectural Style                                       | [`architectural-style/vertical-slice-architecture.md`](./architectural-style/vertical-slice-architecture.md) |
| **Clean Slice Architecture** | Hybrid (Clean + Vertical Slice)                            | [`architectural-style/clean-slice-architecture.md`](./architectural-style/clean-slice-architecture.md) |
| **Architecture Comparison** | Decision guide across the styles                             | [`architectural-style/architecture-comparison.md`](./architectural-style/architecture-comparison.md) |
| **Five-Architectures Mixing** | How to combine architectural styles                        | [`architectural-style/five-architectures-comparison-and-mixing.md`](./architectural-style/five-architectures-comparison-and-mixing.md) |
| **IoC / Dependency Injection** | Enterprise pattern (foundation of modern .NET)            | [`enterprise-pattern/ioc-di.md`](./enterprise-pattern/ioc-di.md) |
| **Service Lifetimes**  | Enterprise pattern (Transient/Scoped/Singleton)                  | [`enterprise-pattern/lifetimes.md`](./enterprise-pattern/lifetimes.md) |
| **Data Access Patterns** | Enterprise patterns (Lazy/Eager, Identity Map, Active Record, …) | [`enterprise-pattern/data-access.md`](./enterprise-pattern/data-access.md) |
| **Other Enterprise Patterns** | Spec, Mediator (MediatR), Options, Result, …               | [`enterprise-pattern/other-enterprise.md`](./enterprise-pattern/other-enterprise.md) |

---

## Folder Map

* **[`design-pattern/`](./design-pattern/)** — the 23 GoF design patterns with .NET samples.
* **[`architectural-pattern/`](./architectural-pattern/)** — distributed-systems and service-level patterns (Saga, CQRS, Event Sourcing, Outbox).
* **[`architectural-style/`](./architectural-style/)** — system-wide structural choices. Contains all per-style deep-dives (N-Layer, N-Tier, Clean, Hexagonal, Vertical Slice, Clean Slice), the EDA / EDD deep-dive, and the comparison docs.
* **[`enterprise-pattern/`](./enterprise-pattern/)** — application-layer patterns (Repository, Unit of Work, IoC/DI, Service Lifetimes, Data Access, Specification, Mediator, …).
* **[`ddd/`](./ddd/)** — Domain-Driven Design as a methodology, plus its tactical and strategic patterns.

---

## How These Layers Interact in a Real .NET Solution

A typical microservice in production combines patterns from **multiple** levels:

* **Style** — Clean Architecture (dependencies point inward).
* **Architectural Pattern** — CQRS for read/write separation, Saga for cross-service transactions, Outbox for reliable publishing.
* **Methodology** — DDD shapes the domain model.
* **Enterprise Pattern** — Repository per aggregate, Unit of Work via `DbContext`.
* **GoF Patterns** — Strategy for pluggable rules, Decorator for caching, Observer for domain events, Mediator for command dispatch.

None of these are interchangeable — each one is solving a problem at a different scale.
