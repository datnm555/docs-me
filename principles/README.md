# Software Engineering Principles — Complete Guide

> A practical, language-agnostic reference for the principles every developer should know — from **DRY** and **KISS** to **SOLID**, **GRASP**, and beyond. Examples are in C# / Java / TypeScript but the ideas apply everywhere.

---

## Quick Reference (What · Why · When · Where)

- **What** — A language-agnostic reference for software-engineering principles: **DRY**, **KISS**, **YAGNI**, **SoC** (core); **SOLID** (5 OO principles); **GRASP** (9 responsibility-assignment principles); plus LoD, POLA, Fail Fast, Composition over Inheritance, Tell-Don't-Ask, and more.
- **Why** — Principles are *guidelines for good design* — the **why** behind every pattern and architecture. Internalize them and you'll reinvent half the patterns on demand; ignore them and even the best patterns won't save the system.
- **When** — Every design decision; code reviews ("which principle does this violate?"); explaining trade-offs to junior teammates; deciding whether to apply a pattern at all.
- **Where** — Above patterns and architectures in the hierarchy. Foundation for `oop/`, `dot-net/docs/design-pattern/`, `dot-net/docs/architectural-style/`, and `dot-net/docs/ddd/`.

---

## Table of Contents

1. [What Is a Principle?](#1-what-is-a-principle)
2. [Principles vs. Patterns vs. Architectures](#2-principles-vs-patterns-vs-architectures)
3. [The Full Catalog](#3-the-full-catalog)
4. [Core Principles (DRY, KISS, YAGNI, SoC)](./core.md)
5. [SOLID — 5 Object-Oriented Principles](./solid.md)
6. [GRASP — 9 Responsibility Assignment Principles](./grasp.md)
7. [Additional Principles (LoD, CoI, POLA, …)](./additional.md)
8. [How to Apply Principles in Practice](#4-how-to-apply-principles-in-practice)
9. [Anti-Patterns: What Happens When You Ignore Them](#5-anti-patterns)
10. [Further Reading](#6-further-reading)

---

## 1. What Is a Principle?

A **principle** is a *general rule of thumb* that guides design decisions. Unlike a pattern (a concrete solution) or an architecture (a high-level structure), a principle is a **value or heuristic** you apply when deciding *how* to write code.

Key properties:

* **Universal** — language- and framework-independent.
* **Trade-off based** — every principle has a cost; rigid application hurts.
* **Compositional** — principles reinforce each other (e.g. SRP makes OCP easier).
* **Heuristic, not law** — a principle is a default, not a contract.

> *"Principles are the **why**. Patterns are the **how**. Architectures are the **where**."*

---

## 2. Principles vs. Patterns vs. Architectures

| Layer            | What it is                                             | Example                                |
| ---------------- | ------------------------------------------------------ | -------------------------------------- |
| **Principle**    | A rule of thumb for *good design*                      | DRY, SRP, Tell-Don't-Ask               |
| **Pattern**      | A reusable, named *solution template*                  | Strategy, Factory, Repository          |
| **Architecture** | A high-level *structural style* of the whole system    | Clean Architecture, N-Layer, Hexagonal |

Patterns implement principles. Architectures organize patterns. Principles judge them all.

---

## 3. The Full Catalog

### 3.1 Core Principles (4)

| Acronym   | Name                            | One-line intent                                                       |
| --------- | ------------------------------- | --------------------------------------------------------------------- |
| **DRY**   | Don't Repeat Yourself           | Every piece of knowledge has one authoritative representation.        |
| **KISS**  | Keep It Simple, Stupid          | Prefer the simplest solution that works.                              |
| **YAGNI** | You Aren't Gonna Need It        | Don't build for hypothetical future needs.                            |
| **SoC**   | Separation of Concerns          | Split a program into parts that each address one concern.             |

→ See [`core.md`](./core.md)

### 3.2 SOLID (5)

| Letter  | Name                                | One-line intent                                                              |
| ------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| **S**   | Single Responsibility Principle     | A class should have only **one reason to change**.                           |
| **O**   | Open/Closed Principle               | Open for extension, **closed for modification**.                             |
| **L**   | Liskov Substitution Principle       | Subtypes must be **substitutable** for their base types.                     |
| **I**   | Interface Segregation Principle     | Many client-specific interfaces beat one fat interface.                      |
| **D**   | Dependency Inversion Principle      | Depend on **abstractions**, not concretions.                                 |

→ See [`solid.md`](./solid.md)

### 3.3 GRASP (9)

General Responsibility Assignment Software Patterns — by **Craig Larman**.

| # | Principle                  | One-line intent                                                                |
| - | -------------------------- | ------------------------------------------------------------------------------ |
| 1 | **Information Expert**     | Assign responsibility to the class with the data needed to fulfill it.         |
| 2 | **Creator**                | The class that aggregates/uses B should create B.                              |
| 3 | **Controller**             | Route system events to a non-UI "controller" class.                            |
| 4 | **Low Coupling**           | Minimize dependencies between classes.                                         |
| 5 | **High Cohesion**          | Keep related responsibilities together; unrelated ones apart.                  |
| 6 | **Polymorphism**           | Use polymorphic operations to handle type-based variation.                     |
| 7 | **Pure Fabrication**       | Invent a class with no domain analog to preserve cohesion/coupling.            |
| 8 | **Indirection**            | Add a middle component to decouple two parties.                                |
| 9 | **Protected Variations**   | Wrap unstable elements with a stable interface.                                |

→ See [`grasp.md`](./grasp.md)

### 3.4 Additional Principles

| Principle                              | One-line intent                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------- |
| **LoD** — Law of Demeter               | Talk only to immediate friends.                                              |
| **Composition over Inheritance**       | Prefer "has-a" relationships over "is-a".                                    |
| **Tell, Don't Ask**                    | Tell objects what to do; don't query state to decide for them.               |
| **POLA** — Principle of Least Astonishment | Behavior should match the user's intuitive expectations.                  |
| **Fail Fast**                          | Detect errors at the earliest possible point.                                |
| **CoC** — Convention over Configuration | Sensible defaults beat endless configuration knobs.                         |
| **Boy Scout Rule**                     | Leave the code cleaner than you found it.                                    |
| **Encapsulation**                      | Hide internals; expose behavior.                                             |
| **Principle of Least Privilege**       | Grant the minimum access necessary.                                          |
| **SoT** — Single Source of Truth       | Every fact lives in exactly one place.                                       |

→ See [`additional.md`](./additional.md)

---

## 4. How to Apply Principles in Practice

Principles often conflict. A few practical heuristics:

1. **Make it work, then make it right, then make it fast.** Don't optimize for SOLID on day one.
2. **Pay the abstraction cost only when the second use case appears.** (DRY + YAGNI tension.)
3. **Coupling and cohesion trump every other metric.** Most "bad code" is really one of these two.
4. **Prefer deletion over refactor.** Less code obeys every principle automatically.
5. **Rule of three.** Duplicate code is fine twice; refactor on the third occurrence.

---

## 5. Anti-Patterns

What goes wrong when you ignore principles:

| Smell                          | Violates                                  |
| ------------------------------ | ----------------------------------------- |
| **God Class**                  | SRP, High Cohesion                        |
| **Copy-Paste Programming**     | DRY                                       |
| **Shotgun Surgery**            | SRP, Low Coupling                         |
| **Feature Envy**               | Information Expert, Tell-Don't-Ask        |
| **Speculative Generality**     | YAGNI, KISS                               |
| **Long Parameter List**        | SRP, ISP                                  |
| **Inappropriate Intimacy**     | Law of Demeter, Encapsulation             |
| **Refused Bequest**            | LSP                                       |
| **Switch on Type**             | OCP, Polymorphism                         |
| **Primitive Obsession**        | Encapsulation                             |

---

## 6. Further Reading

* Robert C. Martin — *Clean Code*, *Clean Architecture*, *Agile Software Development: Principles, Patterns, and Practices*
* Craig Larman — *Applying UML and Patterns* (GRASP)
* Andy Hunt & Dave Thomas — *The Pragmatic Programmer* (DRY, orthogonality)
* Martin Fowler — *Refactoring* (code smells), *Patterns of Enterprise Application Architecture*
* Steve McConnell — *Code Complete*
* Eric Evans — *Domain-Driven Design*

---

> **Bottom line:** principles aren't rules to obey blindly — they're lenses to evaluate trade-offs. Internalize them so you can break them *knowingly* when context demands.
