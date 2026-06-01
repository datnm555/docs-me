# Enterprise Patterns (Fowler PoEAA)

> Patterns catalogued by Martin Fowler in **Patterns of Enterprise Application Architecture** (2002). They sit *between* GoF design patterns and architectural styles — they shape how an application **interacts with data and other applications** at the layer-and-component level.

---

## Quick Reference (What · Why · When · Where)

- **What** — Patterns from Fowler's *PoEAA* (2002) plus community-driven follow-ons — **Repository**, **Unit of Work**, **IoC/DI**, **Service Lifetimes**, **Data Access patterns** (Lazy/Eager Loading, Identity Map, Active Record, Data Mapper), and **Specification/Mediator/CQRS/Result/Options/Decorator pipeline**.
- **Why** — They sit between GoF (class level) and architectural styles (system level). They shape how an application interacts with data and other applications.
- **When** — Building application services, data-access code, dependency wiring; deciding between Active Record and Data Mapper; choosing a Repository style; structuring cross-cutting concerns.
- **Where** — Live in the **Application / Infrastructure** layers of Clean/Hexagonal architecture. Many are already implemented in modern .NET (`DbContext` is Unit of Work, `DbSet<T>` is generic Repository, MediatR is Mediator, `IOptions<T>` is the Options pattern).

---

## What Counts as an "Enterprise Pattern"?

* **Scope:** the data-access layer, application service layer, and presentation/integration boundaries of a typical enterprise application.
* **Granularity:** bigger than class collaboration (GoF), smaller than a service or system style.
* **Canonical source:** Martin Fowler — *Patterns of Enterprise Application Architecture* (PoEAA).

These are **NOT** GoF design patterns, even though they look similar. They're a separate catalog with their own names. The most famous member, **Repository**, is also reused by DDD as a *tactical pattern*.

---

## Index

| Pattern                  | Problem it solves                                                            | File                                                |
| ------------------------ | ---------------------------------------------------------------------------- | --------------------------------------------------- |
| **Repository**           | Decouple the domain from data access; query as if in-memory.                  | [`repository.md`](./repository.md)                  |
| **Unit of Work**         | Track changes across multiple objects and commit them as one transaction.    | [`unit-of-work.md`](./unit-of-work.md)              |
| **IoC / DI**             | Invert object construction; depend on abstractions, not concretions.          | [`ioc-di.md`](./ioc-di.md)                          |
| **Service Lifetimes**    | Transient / Scoped / Singleton — when each instance is created.               | [`lifetimes.md`](./lifetimes.md)                    |
| **Data Access (Lazy/Eager/Identity Map/…)** | Loading strategies and persistence-related patterns.      | [`data-access.md`](./data-access.md)                |
| **Other Enterprise Patterns** | Specification, Mediator, Options, Result, Decorator pipeline, …          | [`other-enterprise.md`](./other-enterprise.md)      |

## Other PoEAA Patterns (not yet expanded, summarized)

| Pattern              | One-line                                                                  |
| -------------------- | ------------------------------------------------------------------------- |
| **Active Record**    | One object per table; the object knows how to save/load itself. (RoR style.) |
| **Data Mapper**      | Domain objects ignorant of persistence; a mapper moves data in/out.        |
| **Service Layer**    | A thin layer exposing application use cases over the domain.               |
| **Specification**    | Encapsulate a query predicate as a first-class object (composable).        |
| **Identity Map**     | One object per identity within a session — EF Core's change tracker.       |
| **Lazy Load**        | Defer loading related data until needed — EF Core lazy proxies.            |
| **DTO**              | Carry data between layers / over the wire without leaking domain types.    |
| **Front Controller** | Single entry point routes all requests — ASP.NET MVC/routing.              |
| **MVC / MVP / MVVM** | UI separation patterns.                                                   |

---

## How They Relate to Modern .NET

Many of these patterns are **already implemented inside the framework**. Recognize them before re-inventing them.

| Pattern         | Already present in .NET as…                                  |
| --------------- | ------------------------------------------------------------ |
| Repository      | `DbContext.Set<T>()` already gives an `IQueryable<T>`         |
| Unit of Work    | `DbContext` itself — `SaveChanges` commits all tracked changes |
| Identity Map    | EF Core change tracker                                        |
| Lazy Load       | `UseLazyLoadingProxies()` in EF Core                          |
| Data Mapper     | EF Core / Dapper                                              |
| Service Layer   | Application services / MediatR handlers                       |
| Specification   | LINQ `Expression<Func<T,bool>>` predicates                    |
| DTO             | Records / API models                                          |
| Front Controller| ASP.NET Core routing pipeline                                 |

This is why **applying these patterns naively on top of EF Core often adds no value and hurts** — see the discussion of "Generic Repository" in [`repository.md`](./repository.md).

---

## References

* Martin Fowler — *Patterns of Enterprise Application Architecture* (2002). The canonical reference.
* Online catalog: [martinfowler.com/eaaCatalog](https://martinfowler.com/eaaCatalog/).
