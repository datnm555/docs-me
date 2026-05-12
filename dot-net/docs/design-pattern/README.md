# Design Patterns in .NET — Complete Guide

> A practical reference for the 23 classic **Gang of Four (GoF)** design patterns, with idiomatic C# samples for modern .NET (.NET 6 / 7 / 8 / 9).

---

## Table of Contents

1. [What Is a Design Pattern?](#1-what-is-a-design-pattern)
2. [How Many Design Patterns Are There?](#2-how-many-design-patterns-are-there)
3. [The Three Categories](#3-the-three-categories)
4. [Full Catalog (23 GoF Patterns)](#4-full-catalog-23-gof-patterns)
5. [How to Choose a Pattern](#5-how-to-choose-a-pattern)
6. [Most Frequently Used Patterns in Real Projects](#6-most-frequently-used-patterns-in-real-projects)
7. [Patterns vs. Principles vs. Architectures](#7-patterns-vs-principles-vs-architectures)
8. [Modern .NET Already Implements Many for You](#8-modern-net-already-implements-many-for-you)
9. [Anti-Patterns to Watch For](#9-anti-patterns-to-watch-for)
10. [Further Reading](#10-further-reading)

---

## 1. What Is a Design Pattern?

A **design pattern** is a *reusable, named solution* to a *recurring design problem* in object-oriented software. Patterns are not finished code — they are **templates** describing how to structure classes and objects to solve a problem in a flexible, maintainable way.

Key properties of a true pattern:

* **Recurring problem** — appears in many projects.
* **Proven solution** — has been used and refined over time.
* **Trade-offs** — every pattern adds indirection; use it only when it pays off.
* **Vocabulary** — gives developers a shared language ("use a Strategy here", "this is a Decorator").

---

## 2. How Many Design Patterns Are There?

The canonical answer is **23 patterns**, defined by the *Gang of Four* (Gamma, Helm, Johnson, Vlissides) in **"Design Patterns: Elements of Reusable Object-Oriented Software"** (1994).

| Category       | Count  | Purpose                                       |
| -------------- | ------ | --------------------------------------------- |
| **Creational** | 5      | How objects are **created**                   |
| **Structural** | 7      | How objects are **composed**                  |
| **Behavioral** | 11     | How objects **communicate** and share work    |
| **Total**      | **23** |                                               |

Beyond the GoF 23, the industry has added many more (e.g. **Repository**, **Unit of Work**, **Specification**, **CQRS**, **Mediator (MediatR-style)**, **Null Object**, **Dependency Injection**, **Service Locator**, **Lazy Load**) — these are usually grouped under *Enterprise Patterns* (Martin Fowler, *PoEAA*) rather than GoF.

This guide focuses on the **23 GoF patterns** as the foundation.

---

## 3. The Three Categories

### 3.1 Creational Patterns (5)

Deal with **object creation** — abstracting and decoupling the *how* of instantiation. Useful when `new SomeClass()` becomes too rigid.

→ See [`creational.md`](./creational.md)

### 3.2 Structural Patterns (7)

Deal with **object composition** — how classes and objects are assembled into larger structures while keeping them flexible and efficient.

→ See [`structural.md`](./structural.md)

### 3.3 Behavioral Patterns (11)

Deal with **communication and responsibility** between objects — how they interact, distribute work, and manage algorithms.

→ See [`behavioral.md`](./behavioral.md)

---

## 4. Full Catalog (23 GoF Patterns)

### Creational (5)

| #  | Pattern              | One-Line Intent                                                                  |
| -- | -------------------- | -------------------------------------------------------------------------------- |
| 1  | **Singleton**        | Ensure a class has **one instance** and provide a global access point.           |
| 2  | **Factory Method**   | Let subclasses decide **which concrete class** to instantiate.                   |
| 3  | **Abstract Factory** | Create **families of related objects** without specifying concrete classes.      |
| 4  | **Builder**          | Construct a complex object **step-by-step**; same process, different results.    |
| 5  | **Prototype**        | Create new objects by **cloning** an existing one.                               |

### Structural (7)

| #  | Pattern         | One-Line Intent                                                                        |
| -- | --------------- | -------------------------------------------------------------------------------------- |
| 6  | **Adapter**     | Convert one interface into another the client expects (a translator).                  |
| 7  | **Bridge**      | Decouple an **abstraction** from its **implementation** so both can vary independently.|
| 8  | **Composite**   | Treat individual objects and compositions of objects **uniformly** (tree structure).   |
| 9  | **Decorator**   | Attach new responsibilities to an object **dynamically**.                              |
| 10 | **Facade**      | Provide a **simplified interface** over a complex subsystem.                           |
| 11 | **Flyweight**   | Share fine-grained objects efficiently to reduce memory.                               |
| 12 | **Proxy**       | Provide a **placeholder/surrogate** to control access to another object.               |

### Behavioral (11)

| #  | Pattern                    | One-Line Intent                                                                  |
| -- | -------------------------- | -------------------------------------------------------------------------------- |
| 13 | **Chain of Responsibility**| Pass a request along a chain until someone handles it.                           |
| 14 | **Command**                | Encapsulate a request as an object (queue, log, undo).                           |
| 15 | **Interpreter**            | Define a grammar and an interpreter for sentences in that grammar.               |
| 16 | **Iterator**               | Traverse a collection sequentially without exposing its internals.               |
| 17 | **Mediator**               | Centralize complex communications between objects.                               |
| 18 | **Memento**                | Capture and restore an object's internal state (undo).                           |
| 19 | **Observer**               | One-to-many notification when state changes (pub/sub).                           |
| 20 | **State**                  | Let an object alter its behavior when its internal state changes.                |
| 21 | **Strategy**               | Encapsulate interchangeable algorithms behind a common interface.                |
| 22 | **Template Method**        | Define the skeleton of an algorithm; let subclasses fill in steps.               |
| 23 | **Visitor**                | Add new operations to objects without modifying them.                            |

---

## 5. How to Choose a Pattern

Ask the question first; the pattern follows.

| If you need to…                                            | Consider                                           |
| ---------------------------------------------------------- | -------------------------------------------------- |
| Hide complex construction logic                            | **Builder**, **Factory Method**, **Abstract Factory** |
| Ensure exactly one instance                                | **Singleton** (or just DI as Singleton lifetime)    |
| Add behavior to an object without inheritance              | **Decorator**                                      |
| Make incompatible interfaces work together                 | **Adapter**                                        |
| Simplify a complicated API for clients                     | **Facade**                                         |
| Switch algorithms at runtime                               | **Strategy**                                       |
| React to state changes elsewhere                           | **Observer**                                       |
| Process a request through several optional handlers        | **Chain of Responsibility**                        |
| Support undo / redo                                        | **Command** + **Memento**                          |
| Walk a tree of mixed leaf/branch nodes                     | **Composite** + **Visitor**                        |
| Defer or wrap object access (cache, lazy, security)        | **Proxy**                                          |
| Reduce memory of many similar objects                      | **Flyweight**                                      |
| Change behavior based on lifecycle/state machine           | **State**                                          |

---

## 6. Most Frequently Used Patterns in Real Projects

Not all 23 patterns get equal use. In production .NET codebases (web APIs, microservices, line-of-business apps) the distribution is heavily skewed. Here is a pragmatic ranking — based on how often you'll actually meet each pattern in a real ASP.NET Core / EF Core / MediatR-style stack.

### Tier 1 — Used in **almost every** .NET project

These are so common you often use them without naming them.

| Pattern              | Where it shows up                                                   |
| -------------------- | ------------------------------------------------------------------- |
| **Strategy**         | Any time you register multiple implementations of an interface in DI (auth providers, pricing rules, exporters). |
| **Singleton**        | `AddSingleton<T>()` — cached config, HTTP factories, in-memory stores. |
| **Factory (Method)** | `IHttpClientFactory`, `ILoggerFactory`, keyed services, MediatR resolving handlers. |
| **Decorator**        | Cross-cutting concerns: caching/logging/retry around services; ASP.NET middleware; MediatR pipeline behaviors. |
| **Iterator**         | Every `foreach`; `IAsyncEnumerable<T>` for streaming queries.        |
| **Observer**         | `event`s, domain events, MediatR `INotification`, `IHostedService`, Reactive Extensions. |
| **Builder**          | `WebApplicationBuilder`, `HostBuilder`, `StringBuilder`, EF model builder, fluent test builders. |
| **Facade**           | "Application service" / "use case" classes that orchestrate many dependencies behind one method. |

### Tier 2 — **Common** in mid-to-large projects

You'll meet these every few weeks of work.

| Pattern                  | Where it shows up                                                |
| ------------------------ | ---------------------------------------------------------------- |
| **Command**              | CQRS, MediatR `IRequest`, message-bus payloads, audit logs.       |
| **Mediator**             | MediatR — the de-facto standard in modern .NET architecture.       |
| **Adapter**              | Wrapping 3rd-party SDKs (Stripe, Twilio, S3) behind your interface. |
| **Template Method**      | `BackgroundService.ExecuteAsync`, base controllers, base jobs.    |
| **Chain of Responsibility** | ASP.NET middleware, `HttpClient` `DelegatingHandler`.          |
| **Proxy**                | EF Core lazy loading, gRPC/Refit clients, authorization wrappers. |
| **State**                | Order/booking/loan workflows with `Stateless` or Workflow Core.   |

### Tier 3 — **Useful in specific niches**

Reach for these when the problem matches.

| Pattern             | When it earns its keep                                            |
| ------------------- | ----------------------------------------------------------------- |
| **Abstract Factory**| Theming, multi-tenant providers, swapping a whole family at once. |
| **Composite**       | Tree-like data: ASTs, file systems, permission/menu trees.        |
| **Bridge**          | Two independent axes of variation (e.g. message-type × channel).   |
| **Visitor**         | Roslyn analyzers, expression-tree rewriters, AST walkers.          |
| **Interpreter**     | DSL parsers, rule engines, search-query languages.                 |
| **Memento**         | Undo/redo in editors; transactional rollback of in-memory state.   |
| **Prototype**       | `record` + `with`; cloning expensive-to-build templates.           |

### Tier 4 — **Rare** in modern .NET app code

| Pattern       | Why it's rare                                                                  |
| ------------- | ------------------------------------------------------------------------------ |
| **Flyweight** | Memory pressure problems are uncommon in typical web/business apps. Reserved for rendering, gaming, large-document tooling. |

### The honest summary

If you only learn **8 patterns**, learn these — they cover ~80% of what you'll do in a real .NET codebase:

> **Strategy, Decorator, Factory, Singleton, Builder, Observer, Mediator/Command, Facade.**

Add **Adapter** and **Template Method** to reach ~90%. Everything else is situational.

---

## 7. Patterns vs. Principles vs. Architectures

These three concepts often get conflated. They operate at different levels.

| Concept           | Scope              | Examples                                                |
| ----------------- | ------------------ | ------------------------------------------------------- |
| **Principle**     | Guideline          | SOLID, DRY, KISS, YAGNI, Law of Demeter                 |
| **Pattern**       | Class/object level | Strategy, Observer, Builder (this guide)                |
| **Architecture**  | System level       | N-Layer, Clean, Onion, Hexagonal, CQRS, Microservices   |

Patterns **implement** principles; architectures **organize** patterns.

---

## 8. Modern .NET Already Implements Many for You

Don't reinvent. Recognize the patterns already baked into the framework:

| Pattern             | Built-in in .NET                                                       |
| ------------------- | ---------------------------------------------------------------------- |
| **Singleton**       | `services.AddSingleton<T>()` in DI container.                          |
| **Factory**         | `IHttpClientFactory`, `ILoggerFactory`, `IServiceProvider`.            |
| **Builder**         | `StringBuilder`, `HostBuilder`, `WebApplicationBuilder`, EF model builder. |
| **Iterator**        | `IEnumerable<T>` / `IEnumerator<T>` — every `foreach` uses it.          |
| **Observer**        | `IObservable<T>`/`IObserver<T>`, `events`, `IHostedService`, Reactive Extensions. |
| **Decorator**       | ASP.NET Core middleware pipeline; `Scrutor`'s `Decorate<T>()`.          |
| **Strategy**        | DI of an interface with multiple implementations + `IEnumerable<T>`.    |
| **Chain of Resp.** | ASP.NET middleware; delegating handlers in `HttpClient`.                |
| **Adapter**         | `IAsyncEnumerable<T>` wrappers; SDK adapters.                          |
| **Proxy**           | EF Core lazy loading proxies; Castle DynamicProxy; `dispatch_proxy`.    |
| **Command**         | MediatR's `IRequest`; CQRS commands.                                   |
| **Mediator**        | MediatR.                                                               |
| **Template Method** | Base controllers, `BackgroundService.ExecuteAsync`.                    |
| **State**           | `Stateless` library, workflow engines.                                 |

Lesson: most of the time you **consume** a pattern via the framework, not implement one from scratch.

---

## 9. Anti-Patterns to Watch For

| Anti-Pattern              | Description                                                                    |
| ------------------------- | ------------------------------------------------------------------------------ |
| **Patternitis**           | Using patterns for the sake of patterns. Adds indirection with no payoff.      |
| **Singleton abuse**       | Global mutable state. Hides dependencies and kills testability.                |
| **God Factory**           | A factory that knows about every concrete type — collapses to a switch.        |
| **Anemic Domain Model**   | Domain entities with only getters/setters, all logic in "service" classes.     |
| **Service Locator**       | Hiding dependencies behind a container lookup; prefer constructor injection.   |
| **Premature Abstraction** | Inventing strategies/factories before you have two real implementations.       |

**Rule of thumb:** wait until you have **at least two real variations** before extracting a pattern. The first occurrence is just code; the second is when a pattern earns its keep.

---

## 10. Further Reading

* Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (1994). The original GoF book.
* Freeman & Robson — *Head First Design Patterns* (2nd ed., 2020). Friendlier introduction with Java examples.
* Martin Fowler — *Patterns of Enterprise Application Architecture* (2002). Enterprise/persistence patterns.
* Mark Seemann — *Dependency Injection Principles, Practices, and Patterns* (2019). DI-aware patterns in .NET.
* [refactoring.guru/design-patterns](https://refactoring.guru/design-patterns) — visual, modern explanations with C# examples.

---

## File Index

* [`README.md`](./README.md) — this overview
* [`creational.md`](./creational.md) — 5 creational patterns with C# samples
* [`structural.md`](./structural.md) — 7 structural patterns with C# samples
* [`behavioral.md`](./behavioral.md) — 11 behavioral patterns with C# samples
