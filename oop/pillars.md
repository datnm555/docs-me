# The Four Pillars of OOP

> **Encapsulation, Abstraction, Inheritance, Polymorphism.** Master these and you understand the *why* behind every OO design decision.

---

## Quick Reference (What · Why · When · Where)

- **What** — The four pillars: **Encapsulation** (hide state behind behavior), **Abstraction** (expose what, hide how), **Inheritance** (specialize an existing type), **Polymorphism** (one interface, many runtime behaviors).
- **Why** — These four ideas together explain *every* OO design decision. Master them and patterns like Strategy, Template Method, and Decorator become natural rather than memorized.
- **When** — Before learning design patterns, SOLID, or DDD; explaining why an `if (x is Y)` is a code smell; deciding between subclassing and composition.
- **Where** — Language fundamentals (the language's `class`, `interface`, `virtual` keywords are how each pillar shows up). Foundation for everything in `principles/`, `design-pattern/`, and `ddd/`.

---

## Table of Contents

1. [Encapsulation](#1-encapsulation)
2. [Abstraction](#2-abstraction)
3. [Inheritance](#3-inheritance)
4. [Polymorphism](#4-polymorphism)
5. [How the Pillars Relate](#5-how-the-pillars-relate)

---

## 1. Encapsulation

> *"Bundle the data, and the operations on that data, into a single unit. Hide the internals behind a public interface."*

Encapsulation is the **load-bearing pillar**. The other three depend on it.

### 1.1 What It Means in Code

* **Fields are `private`**, never touched from outside.
* **Methods enforce invariants** (rules that must always hold).
* **Public surface = behavior**, not data.

### 1.2 Bad — No Encapsulation (Anemic Object)

```csharp
public class BankAccount
{
    public decimal Balance;          // raw public field
}

// Anywhere in the codebase:
account.Balance -= 100;              // no validation, no audit, no concurrency
```

A negative balance can appear. The rule "no overdraft" lives nowhere — and so it lives nowhere reliably.

### 1.3 Good — Behavior Owns Data

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        if (amount > Balance)
            throw new InvalidOperationException("Insufficient funds");

        Balance -= amount;
    }
}
```

Now the rule is enforced in **one place** — the `Withdraw` method.

### 1.4 Levels of Access Control

| Modifier (C#)           | Visible to                                                |
| ----------------------- | --------------------------------------------------------- |
| `private`               | Same class only                                           |
| `protected`             | Same class + subclasses                                   |
| `internal`              | Same assembly (≈ package)                                 |
| `protected internal`    | Same assembly **or** subclass                             |
| `private protected`     | Same assembly **and** subclass                            |
| `public`                | Everyone                                                  |

> **Default to the *strictest* modifier**. Open access only when justified.

### 1.5 Encapsulation in Modern C#

```csharp
public class Order
{
    private readonly List<LineItem> _items = new();

    public IReadOnlyList<LineItem> Items => _items;   // expose read-only view
    public decimal Total => _items.Sum(i => i.Price * i.Qty);

    public void Add(LineItem item)
    {
        ArgumentNullException.ThrowIfNull(item);
        _items.Add(item);
    }
}
```

The `_items` list is fully owned. Callers can read it but cannot append, clear, or sort it.

---

## 2. Abstraction

> *"Expose **what** an object does. Hide **how** it does it."*

Abstraction works one level above encapsulation: it asks *which* operations to expose at all.

### 2.1 Encapsulation vs Abstraction

| Concept           | Question it answers                                |
| ----------------- | -------------------------------------------------- |
| **Encapsulation** | *How do I hide and protect this data?*             |
| **Abstraction**   | *Which operations are even worth exposing?*        |

Encapsulation is mechanical; abstraction is a **design decision**.

### 2.2 Example — Two Levels of Abstraction

```csharp
// Low-level (concrete)
public class SmtpClient
{
    public void Connect(string host, int port);
    public void Authenticate(string user, string pass);
    public void Send(MailMessage msg);
    public void Disconnect();
}

// High-level abstraction the application actually wants
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body);
}
```

The application doesn't care about ports, TLS, or auth — it just wants to *send an email*.

### 2.3 How to Find the Right Abstraction

* Name the **role**, not the technology — `IEmailSender`, not `ISmtpClient`.
* Describe the **intent**, not the mechanism — `IClock`, not `IDateTimeWrapper`.
* Resist *speculative* abstractions; wait until you have **two real implementations** (or one real + a test fake).

### 2.4 Abstract Class vs Interface

| Need                                       | Use              |
| ------------------------------------------ | ---------------- |
| Pure contract, no shared state             | `interface`      |
| Shared state OR shared default behavior    | `abstract class` |
| Multiple "inheritance" of capability       | Multiple interfaces |
| Marker for a category                      | `interface`      |

> Modern C# allows **default interface methods** — but if you need them, you probably want an abstract base class.

---

## 3. Inheritance

> *"Define a new class as a **specialization** of an existing class — inheriting its members and refining or adding behavior."*

Inheritance is the **most-misused** pillar. Used well, it expresses **is-a**. Used badly, it becomes a tangled mess.

### 3.1 Anatomy of Inheritance

```csharp
public class Animal
{
    public string Name { get; set; }
    public virtual void Speak() => Console.WriteLine("...");
}

public class Dog : Animal
{
    public override void Speak() => Console.WriteLine("Woof!");
}

public class Cat : Animal
{
    public override void Speak() => Console.WriteLine("Meow!");
}
```

* `virtual` — base method may be overridden.
* `override` — derived method replaces the base.
* `sealed` — prevents further inheritance/override.

### 3.2 The "is-a" Test

If you can't honestly say *"a `Dog` **is-a** `Animal`"*, don't inherit. A common failed test:

```csharp
public class Stack<T> : List<T> { ... }   // BAD: a Stack is NOT a List
```

A stack disallows random-access mutation; inheriting from `List<T>` exposes operations that violate the stack's invariants. Use **composition** instead.

### 3.3 Inheritance Pitfalls

* **Fragile base class** — a small change in the base can break every child.
* **Yo-yo problem** — to understand a class you bounce up and down the hierarchy.
* **Forced generalization** — features added "just in case" they're shared.
* **Diamond problem** — multiple inheritance ambiguity (avoided in C# / Java with interfaces).

### 3.4 Rule of Thumb

> *"Favor **composition over inheritance**. Use inheritance only for **stable, true is-a** relationships."*
> — Gang of Four

---

## 4. Polymorphism

> *"The same operation behaves differently on different types."*

Polymorphism is what makes inheritance and abstraction **useful at runtime**.

### 4.1 Four Kinds of Polymorphism

| Kind            | Mechanism in C#                          | Example                                            |
| --------------- | ---------------------------------------- | -------------------------------------------------- |
| **Subtype**     | `virtual` / `override`                   | `Animal.Speak()` dispatched to `Dog` or `Cat`      |
| **Parametric**  | Generics `<T>`                           | `List<T>`, `Dictionary<TKey, TValue>`              |
| **Ad-hoc**      | Method overloading, operator overloading | `Print(int)` vs `Print(string)`                    |
| **Coercion**    | Implicit type conversions                | `int → long`, `string ↔ ReadOnlySpan<char>`        |

When OOP people say "polymorphism", they almost always mean **subtype polymorphism**.

### 4.2 Subtype Polymorphism in Action

```csharp
public abstract class Shape { public abstract double Area(); }

public class Circle(double r)    : Shape { public override double Area() => Math.PI * r * r; }
public class Square(double side) : Shape { public override double Area() => side * side; }

double TotalArea(IEnumerable<Shape> shapes) => shapes.Sum(s => s.Area());
```

`TotalArea` doesn't know — or care — *which* shape it has. Adding `Triangle` means writing one class. No `switch`, no `if`.

### 4.3 Why Polymorphism Matters

* **Open/Closed Principle** — new variants added without editing existing code.
* **Testability** — substitute fakes for real dependencies.
* **Clarity** — replaces long `switch` ladders with type-driven behavior.

### 4.4 When NOT to Use Polymorphism

* **Two-case logic with no future variants** — a simple `if` is clearer than two subclasses.
* **Performance-critical loops** — virtual calls cost an indirection. (Rarely a real issue, but real in tight inner loops.)
* **Cross-type behavior** — pattern matching may read better than polymorphism for one-off code paths.

---

## 5. How the Pillars Relate

```
                ┌──────────────┐
                │ ENCAPSULATION│ (hides data)
                └──────┬───────┘
                       │ makes possible
                       ▼
                ┌──────────────┐
                │ ABSTRACTION  │ (chooses what to expose)
                └──────┬───────┘
                       │ realized by
              ┌────────┴─────────┐
              ▼                  ▼
       ┌────────────┐     ┌────────────────┐
       │INHERITANCE │     │  COMPOSITION   │
       │ (is-a)     │     │  (has-a)       │
       └─────┬──────┘     └────────┬───────┘
             │                     │
             └──────── enables ────┘
                        ▼
                 ┌──────────────┐
                 │POLYMORPHISM  │ (different behaviors via same contract)
                 └──────────────┘
```

* **Encapsulation** is the soil.
* **Abstraction** is the design choice grown in that soil.
* **Inheritance and composition** are the structures built from abstractions.
* **Polymorphism** is the behavior those structures exhibit.

---

> **Next step:** read [`concepts.md`](./concepts.md) for the language-level constructs that implement these pillars in C# / Java.
