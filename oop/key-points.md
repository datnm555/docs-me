# OOP Key Points — Interview & Design Cheat Sheet

> The high-density summary. If you only remember one page from this guide, make it this one. Every line is a claim you should be able to **defend with an example** and **counter with a trade-off**.

---

## Quick Reference (What · Why · When · Where)

- **What** — A high-density cheat sheet of OO design wisdom: the four pillars in one line each, class-design essentials, inheritance vs. composition, abstract class vs. interface, value vs. entity vs. aggregate, SOLID in 30 seconds, OO smells, top interview questions, and what juniors vs. seniors get right/wrong.
- **Why** — Every line is a *claim you should be able to defend with an example and counter with a trade-off*. Most "checklists" are useless; this one is meant for pre-interview review and code-review arguments.
- **When** — The day before an interview; preparing to lead a design review; mentoring a developer who's about to violate one of these rules.
- **Where** — Distillation of `pillars.md`, `concepts.md`, `real-project.md`, and `principles/`. The recall aid after you've internalized the longer docs.

---

## Table of Contents

1. [The Four Pillars in One Line Each](#1-the-four-pillars-in-one-line-each)
2. [Class Design — The Essentials](#2-class-design--the-essentials)
3. [Inheritance vs Composition](#3-inheritance-vs-composition)
4. [Abstract Class vs Interface](#4-abstract-class-vs-interface)
5. [Value vs Reference / Entity vs Value Object](#5-value-vs-reference--entity-vs-value-object)
6. [SOLID in 30 Seconds](#6-solid-in-30-seconds)
7. [Common OOP Smells](#7-common-oop-smells)
8. [The Mental Checklist Before Writing a Class](#8-the-mental-checklist-before-writing-a-class)
9. [Top Interview Questions With Tight Answers](#9-top-interview-questions-with-tight-answers)
10. [Things Juniors Get Wrong](#10-things-juniors-get-wrong)
11. [Things Seniors Care About](#11-things-seniors-care-about)

---

## 1. The Four Pillars in One Line Each

| Pillar             | One-liner                                                  | Tell-tale code                          |
| ------------------ | ---------------------------------------------------------- | --------------------------------------- |
| **Encapsulation**  | Hide state behind behavior.                                | `private` fields, `init` setters        |
| **Abstraction**    | Choose what to expose; hide what's not relevant.           | interfaces, abstract classes            |
| **Inheritance**    | "is-a" specialization of an existing type.                 | `class Dog : Animal`                    |
| **Polymorphism**   | Same call, different runtime behavior.                     | `virtual / override`, interfaces        |

---

## 2. Class Design — The Essentials

* **Default to `sealed` + `private` + `readonly`.** Open access only when justified.
* **Constructors enforce invariants.** A half-built object shouldn't exist.
* **Public surface = behavior**, not data. Properties are okay; raw fields are not.
* **Methods describe intent**, not implementation. `Order.MarkPaid()`, not `Order.SetStatus(2)`.
* **One reason to change.** If the class name needs "And" to describe it, split it.
* **Small > Smart.** Three short, well-named classes beat one clever one.
* **Tell, Don't Ask.** Don't reach into an object to make decisions for it.
* **Fail at construction.** Reject invalid state immediately, not later.

---

## 3. Inheritance vs Composition

| Inheritance ("is-a")                               | Composition ("has-a")                               |
| -------------------------------------------------- | --------------------------------------------------- |
| Compile-time, permanent                            | Runtime, swappable                                  |
| Single parent (in C# / Java)                       | Many collaborators                                  |
| Strong coupling to base class                      | Weak coupling via interfaces                        |
| Best for **stable, true** is-a relationships       | Best when behavior **varies** or **may grow**       |
| Fragile-base-class risk                            | More files, more wiring                             |

> **Default to composition.** Reach for inheritance only when *"X is-a Y"* is genuinely true *and* stable.

---

## 4. Abstract Class vs Interface

| Choose **abstract class** when…           | Choose **interface** when…                       |
| ----------------------------------------- | ------------------------------------------------ |
| You have shared **state or defaults**     | You need a pure **contract**                     |
| You want to control construction          | Multiple capabilities can be implemented at once |
| The hierarchy will be **closed** and stable| The contract spans **unrelated** implementers   |
| You can credibly say "is-a"               | You can credibly say "behaves-as"                |

> Modern code prefers **interfaces + composition** for flexibility, **abstract classes** for true taxonomy with shared internals.

---

## 5. Value vs Reference / Entity vs Value Object

### Value Object

* No identity. Two equal-value instances are **interchangeable**.
* **Immutable.** Mutating one would affect every "logical copy".
* Examples: `Money`, `Address`, `DateRange`, `PhoneNumber`, `Color`.

### Entity

* Has **identity** (an `Id`) that persists through change.
* Two entities can have the same data and still be different.
* Examples: `Customer`, `Order`, `Account`.

### Aggregate

* A cluster of entities + value objects with a single **root**.
* Outside code only talks to the **root**, never reaches inside.
* Boundary of consistency: all invariants enforced in one transaction.

---

## 6. SOLID in 30 Seconds

| Letter | Principle                       | "If I forget it…"                                                |
| ------ | ------------------------------- | ---------------------------------------------------------------- |
| **S**  | Single Responsibility           | I get God classes.                                               |
| **O**  | Open/Closed                     | Every new feature edits old code.                                |
| **L**  | Liskov Substitution             | Subclasses surprise callers.                                     |
| **I**  | Interface Segregation           | One change ripples through unrelated implementers.               |
| **D**  | Dependency Inversion            | Concrete coupling makes tests and swaps impossible.              |

See the full file at [`../principles/solid.md`](../principles/solid.md).

---

## 7. Common OOP Smells

| Smell                       | What it signals                          |
| --------------------------- | ---------------------------------------- |
| **God Class**               | SRP violated; cohesion low.              |
| **Anemic Domain Model**     | Data and behavior split apart.           |
| **Feature Envy**            | Method belongs on another class.         |
| **Train Wreck** (`a.b().c().d()`) | Law of Demeter broken.             |
| **Switch on Type**          | Polymorphism missing.                    |
| **Long Parameter List**     | Probably a missing value object.         |
| **Primitive Obsession**     | Strings/ints where types should live.    |
| **Speculative Generality**  | YAGNI violated.                          |
| **Refused Bequest**         | Subclass doesn't want what it inherits — wrong is-a. |
| **Singletonitis**           | Hidden global state; untestable.         |

---

## 8. The Mental Checklist Before Writing a Class

Before adding a new class, ask:

1. **What is its single responsibility?** (Name it as a noun + role.)
2. **What state does it own?** (And what should be derived?)
3. **What invariants does it guarantee?** (Where are they enforced?)
4. **Who creates it?** (Factory / constructor / aggregate root.)
5. **Who *can't* see it?** (Lock down with `internal` / `private`.)
6. **What changes might it absorb?** (Stable methods vs hot spots.)
7. **What does it depend on?** (Inject interfaces; never `new` a service.)
8. **How will I test it?** (If hard — design is probably wrong.)

If you can't answer 1-3 in one sentence each, you're not ready to write the class.

---

## 9. Top Interview Questions With Tight Answers

**Q: What's the difference between abstraction and encapsulation?**
Encapsulation hides *how*; abstraction decides *what to expose*. Encapsulation is mechanical (access modifiers); abstraction is a design choice.

**Q: When would you choose composition over inheritance?**
Whenever the relationship isn't a stable is-a, when you'd otherwise override to do "nothing", when behavior may need to vary at runtime, or when you want testable seams. In practice — almost always.

**Q: Explain polymorphism without using the word "polymorphism".**
Two different objects respond to the same message in their own way, and the caller doesn't need to know which one it has.

**Q: Why is `instanceof` / `is X` often a code smell?**
It encodes type knowledge into a caller, blocking OCP. Usually a missing virtual method or strategy.

**Q: What's a value object?**
An immutable type whose identity *is* its value — `Money(10, "USD")` is interchangeable with any other `Money(10, "USD")`.

**Q: Why prefer immutability?**
Threading safety, predictability, equality semantics, and easier reasoning about state.

**Q: When is OOP the wrong choice?**
Pure data transformation, math-heavy code, very short scripts, performance-critical inner loops — functional, procedural, or data-oriented styles fit better.

**Q: Is OOP about reuse?**
No. OOP is primarily about **managing dependencies and absorbing change**. Reuse is a side effect, and often overrated.

**Q: What's the relationship between SOLID and OOP?**
SOLID is the set of principles that turn ad-hoc OO code into *maintainable* OO code. OOP enables them; SOLID disciplines them.

**Q: Difference between `interface` and `abstract class`?**
Interface = contract, no state. Abstract class = partial implementation with shared state. Prefer interfaces; reach for abstract when you genuinely need shared internals.

---

## 10. Things Juniors Get Wrong

* Treating inheritance as the default reuse mechanism.
* Making everything `public` "in case I need it".
* Writing anemic classes with `get; set;` everywhere, then putting business rules in services.
* Using `static` for "convenience", then drowning in hidden coupling.
* Modeling with primitives (`string customerId`, `decimal price`) instead of value objects.
* Over-using `if (x is Y)` instead of polymorphism.
* Over-using polymorphism for two-case logic that an `if` would express clearly.
* Mocking everything in tests, including value objects.
* Building extensibility for variants that never come.

---

## 11. Things Seniors Care About

* **Coupling and cohesion** above almost everything else.
* **Stability of abstractions** — what's volatile lives behind a port.
* **Names that survive refactors** — domain language, not technical jargon.
* **Behavior over data** — methods, not properties, define the API.
* **Crisp layer boundaries** — the dependency rule is non-negotiable.
* **Testability as a design feedback signal** — if it's hard to test, the design is wrong.
* **Deletability** — code that's easy to delete is code that's easy to change.
* **Knowing when *not* to apply principles** — context > dogma.

---

## TL;DR

> **OOP isn't about objects — it's about boundaries. Encapsulation creates them, abstraction names them, inheritance and composition arrange them, polymorphism lets behavior cross them safely. The four pillars exist to manage *change*. Every principle in this docs folder is a heuristic for keeping those boundaries cheap to draw and cheap to redraw.**

Read in order:
1. [`README.md`](./README.md) — orientation
2. [`pillars.md`](./pillars.md) — the four pillars
3. [`concepts.md`](./concepts.md) — language constructs
4. [`real-project.md`](./real-project.md) — putting it all together
5. **This file** — the takeaway

Then practice. Theory is cheap; design is muscle memory.
