# Object-Oriented Programming — Complete Guide

> A practical, end-to-end reference for OOP: what it is, the four pillars, key concepts, a realistic project walkthrough, and the must-know points for interviews and design discussions. Examples are in **C#**, but the ideas apply to Java, TypeScript, Kotlin, Python, and any OO language.

---

## Table of Contents

1. [What Is OOP?](#1-what-is-oop)
2. [Why OOP?](#2-why-oop)
3. [The Four Pillars (Summary)](./pillars.md)
4. [Core Concepts (class, object, interface, …)](./concepts.md)
5. [Real Project Walkthrough — E-Commerce Checkout](./real-project.md)
6. [Key Points Cheat Sheet](./key-points.md)
7. [OOP vs Other Paradigms](#3-oop-vs-other-paradigms)
8. [When OOP Is the Wrong Tool](#4-when-oop-is-the-wrong-tool)
9. [Further Reading](#5-further-reading)

---

## 1. What Is OOP?

**Object-Oriented Programming** is a paradigm where software is modeled as a collection of **objects** that bundle **data** (state) and **behavior** (methods), interact through **well-defined interfaces**, and are organized around **abstractions** of real-world or domain concepts.

The paradigm is built on **four pillars**:

| Pillar              | One-line meaning                                                              |
| ------------------- | ----------------------------------------------------------------------------- |
| **Encapsulation**   | Bundle state and behavior; hide internals.                                    |
| **Abstraction**     | Expose **what** an object does, hide **how**.                                 |
| **Inheritance**     | Specialize an existing type by adding/overriding behavior.                    |
| **Polymorphism**    | One interface, many implementations selected at runtime.                      |

→ Detailed treatment in [`pillars.md`](./pillars.md).

### 1.1 A Tiny Example

```csharp
public abstract class PaymentMethod
{
    public abstract Task<PaymentResult> ChargeAsync(decimal amount);
}

public class CreditCard : PaymentMethod
{
    public override Task<PaymentResult> ChargeAsync(decimal amount) { /* Stripe call */ }
}

public class PayPal : PaymentMethod
{
    public override Task<PaymentResult> ChargeAsync(decimal amount) { /* PayPal call */ }
}

// Caller does not know or care which one it has:
async Task Checkout(PaymentMethod m, decimal total) => await m.ChargeAsync(total);
```

All four pillars are visible:
* **Encapsulation** — each class hides its API keys, HTTP plumbing, retries.
* **Abstraction** — the base class declares only `ChargeAsync`.
* **Inheritance** — `CreditCard` and `PayPal` extend `PaymentMethod`.
* **Polymorphism** — `Checkout` calls the right `ChargeAsync` based on the runtime type.

---

## 2. Why OOP?

OOP became dominant because it provides good answers to four perennial problems of software:

| Problem                          | OOP's answer                                                |
| -------------------------------- | ----------------------------------------------------------- |
| **Managing complexity**          | Decompose into small, named, focused objects.               |
| **Modeling the real world**      | Domain concepts map to classes (Customer, Order, Invoice).  |
| **Reuse without copy-paste**     | Inheritance, composition, and polymorphism.                 |
| **Surviving change**             | Encapsulation isolates changes inside boundaries.           |

OOP is **not** the only way to solve these — but it's the most common paradigm in enterprise software for a reason.

---

## 3. OOP vs Other Paradigms

| Paradigm          | Core unit            | Models the world as…           | Strength                              |
| ----------------- | -------------------- | ------------------------------ | ------------------------------------- |
| **OOP**           | Object               | Things with state and behavior | Large, evolving, stateful systems     |
| **Functional**    | Function             | Transformations of values      | Data pipelines, concurrency           |
| **Procedural**    | Procedure            | A sequence of steps            | Small scripts, embedded               |
| **Logic**         | Predicate            | Facts and rules                | Constraint solving, AI                |
| **Data-oriented** | Plain data + behavior| Arrays, records, transforms    | Games, high-performance, ECS          |

> Modern languages are **multi-paradigm**. C# / Java / TypeScript all embrace functional features (LINQ / streams / map-reduce, records, pattern matching). Treat OOP as one tool — not as religion.

---

## 4. When OOP Is the Wrong Tool

* **Pure data transforms** (CSV → JSON, aggregations). Functional / pipeline style wins.
* **Simple scripts** (under 200 lines). A function is enough.
* **Performance-critical inner loops** (games, signal processing). Data-oriented design beats OO indirection.
* **Math-heavy domains** (statistics, ML). Vector / matrix functions, not objects.

A useful rule: **if your "objects" never change state and have one method, they're just functions**. Don't dress them up as classes.

---

## 5. Further Reading

* **Grady Booch** — *Object-Oriented Analysis and Design with Applications*
* **Bertrand Meyer** — *Object-Oriented Software Construction*
* **Eric Evans** — *Domain-Driven Design*
* **Martin Fowler** — *Refactoring*, *Patterns of Enterprise Application Architecture*
* **Robert C. Martin** — *Clean Code*, *Agile Software Development: Principles, Patterns, and Practices*
* **Joshua Bloch** — *Effective Java* (still gold even for non-Java)

---

> **Next:** read [`pillars.md`](./pillars.md) for the four pillars, [`concepts.md`](./concepts.md) for the language-level constructs, [`real-project.md`](./real-project.md) for a full project walkthrough, and [`key-points.md`](./key-points.md) for the cheat sheet.
