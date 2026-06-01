# Additional Principles — Beyond DRY, SOLID, and GRASP

> A catalogue of the other principles every working engineer eventually internalizes. None are part of SOLID or GRASP, but together they cover the *behavioral*, *organizational*, and *defensive* aspects of good code.

---

## Quick Reference (What · Why · When · Where)

- **What** — The other principles every working engineer eventually internalizes: **Law of Demeter**, **Composition over Inheritance**, **Tell, Don't Ask**, **POLA** (Least Astonishment), **Fail Fast**, **Convention over Configuration**, **Boy Scout Rule**, **Encapsulation**, **Least Privilege**, **Single Source of Truth**, **Curly's Law**, **Hollywood Principle**, **Worse Is Better**.
- **Why** — None of these are part of SOLID or GRASP, but together they cover the *behavioral*, *organizational*, and *defensive* aspects of good code that the famous-acronym principles miss.
- **When** — Writing or reviewing any non-trivial code. These principles catch the smells that SOLID alone doesn't name (train wrecks, anemic models, time-coupled tests, hidden globals).
- **Where** — Code review checklists; mentoring conversations; daily coding decisions. Most of these reinforce or refine the SOLID and GRASP rules from a different angle.

---

## Table of Contents

1. [Law of Demeter (LoD)](#1-law-of-demeter-lod)
2. [Composition over Inheritance](#2-composition-over-inheritance)
3. [Tell, Don't Ask](#3-tell-dont-ask)
4. [Principle of Least Astonishment (POLA)](#4-principle-of-least-astonishment-pola)
5. [Fail Fast](#5-fail-fast)
6. [Convention over Configuration (CoC)](#6-convention-over-configuration-coc)
7. [Boy Scout Rule](#7-boy-scout-rule)
8. [Encapsulation](#8-encapsulation)
9. [Principle of Least Privilege](#9-principle-of-least-privilege)
10. [Single Source of Truth (SoT)](#10-single-source-of-truth-sot)
11. [Curly's Law (Do One Thing)](#11-curlys-law-do-one-thing)
12. [Hollywood Principle](#12-hollywood-principle)
13. [Worse Is Better](#13-worse-is-better)

---

## 1. Law of Demeter (LoD)

> *"Only talk to your immediate friends."* — Northeastern University, 1987

A method `m` of class `C` should only call methods of:

1. `C` itself
2. Its fields
3. Its parameters
4. Objects it creates

Not on objects returned by those — that's reaching through one object to grab another (a.k.a. **train wreck** code).

### Bad — Train Wreck

```csharp
decimal total = order.Customer.Address.PostalCode.Zone.TaxRate * order.Subtotal;
```

Six classes coupled together. Any rename anywhere breaks this.

### Good — Tell, Don't Reach

```csharp
decimal total = order.CalculateTotal();   // Order knows its customer, who knows its address...
```

### Caveat

LoD does **not** ban method chaining where the same object is returned (fluent APIs, LINQ). It bans chaining through unrelated objects.

---

## 2. Composition over Inheritance

> *"Favor object **composition** over class **inheritance**."* — Gang of Four

Inheritance creates a permanent, compile-time bond. Composition is a runtime relationship — flexible, replaceable, mockable.

### Bad — Inheritance Misuse

```csharp
public class Bird { public virtual void Fly() { ... } }
public class Penguin : Bird { public override void Fly() => throw new NotSupportedException(); }
```

Penguin is a bird **biologically** but not **behaviorally** (LSP violation too).

### Good — Compose Capabilities

```csharp
public interface IFlyBehavior { void Fly(); }
public class CanFly  : IFlyBehavior { public void Fly() { /* lift off */ } }
public class CantFly : IFlyBehavior { public void Fly() { /* no-op */ } }

public class Bird(IFlyBehavior fly)
{
    public void Fly() => fly.Fly();
}

var sparrow = new Bird(new CanFly());
var penguin = new Bird(new CantFly());
```

This is exactly the **Strategy pattern** — and it's almost always more flexible than inheritance.

### When Inheritance Is Right

* True *is-a* relationships (a `SqlConnection` IS-A `DbConnection`).
* You want polymorphism without per-instance configuration.
* The base class is stable and the child only specializes.

---

## 3. Tell, Don't Ask

> *"Procedural code gets information then makes decisions. Object-oriented code tells objects to do things."* — Alec Sharp

### Bad — Ask

```csharp
if (order.Status == OrderStatus.Pending && order.TotalAmount > 1000)
{
    order.Status = OrderStatus.RequiresApproval;
    notifier.NotifyManager(order);
}
```

The caller is making decisions that belong to `Order`.

### Good — Tell

```csharp
order.RequestApprovalIfNeeded(notifier);
```

```csharp
public void RequestApprovalIfNeeded(IManagerNotifier notifier)
{
    if (Status == OrderStatus.Pending && TotalAmount > 1000)
    {
        Status = OrderStatus.RequiresApproval;
        notifier.NotifyManager(this);
    }
}
```

Tied to: **Information Expert**, **Encapsulation**, and **Law of Demeter**.

---

## 4. Principle of Least Astonishment (POLA)

> *"A component should behave the way most users will expect it to behave."*

If a method named `GetUser` *also* writes to a log, sends an email, and updates the cache — it's astonishing. Astonishing code becomes **bug-attracting code**.

### Practical Rules

* `Get*` methods don't mutate state.
* `Save*` methods don't make HTTP calls to third-parties silently.
* Operators (`==`, `+`) behave like their primitive analogues.
* Equal inputs produce equal outputs (purity where possible).

### Bad

```csharp
public bool IsAuthorized(User u)
{
    if (u == null) Environment.Exit(1);   // astonishing — kills the process
    ...
}
```

### Good

```csharp
public bool IsAuthorized(User u)
{
    ArgumentNullException.ThrowIfNull(u);
    ...
}
```

---

## 5. Fail Fast

> *"Errors should be detected as **early as possible** — preferably at compile time, otherwise at startup, otherwise at the first invalid call."*

Bugs that surface near their cause are cheap. Bugs that surface in production at 3 a.m. cost money.

### Bad — Silent Drift

```csharp
public class Config
{
    public string ConnectionString { get; set; }   // null until something tries to use it
}
```

A typo in `appsettings.json` won't be noticed until a query runs.

### Good — Validate at Startup

```csharp
public class Config
{
    [Required, MinLength(1)] public required string ConnectionString { get; init; }
}

builder.Services.AddOptions<Config>()
    .Bind(builder.Configuration.GetSection("Db"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Fails immediately at boot — visible, fixable, not in production.

### Where to Apply

* Constructor argument checks (`ArgumentNullException`, `ArgumentException`).
* Strong types over primitives (e.g. `EmailAddress` value object).
* Schema validation on incoming requests.
* `readonly` / `init` fields to prevent later mutation.

---

## 6. Convention over Configuration (CoC)

> *"A developer only needs to specify the **unconventional** aspects of the application."* — David Heinemeier Hansson, Rails

If a framework uses sensible defaults, the developer writes less code. Only deviations need configuration.

### Examples

* **ASP.NET Core**: `Controllers/HomeController.cs` automatically maps to `/Home/*`.
* **Entity Framework**: `Id` is the primary key unless you say otherwise.
* **Maven / npm**: standard project layout means no setup.

### Trade-off

Magical defaults are great until they surprise you. POLA limits CoC: conventions should be **discoverable**, **documented**, and **overridable**.

---

## 7. Boy Scout Rule

> *"Always leave the campground cleaner than you found it."* — Robert C. Martin

When you touch a file, make a small improvement: a rename, a missing `null` check, a comment removed, a method extracted. Small, continuous improvements compound — they're how legacy codebases recover without dedicated "refactor sprints".

### Rules of Engagement

* Don't bundle drive-by refactors into feature PRs — open a separate one.
* Don't fix the whole file; fix what's near your change.
* Don't reformat code you didn't change (kills `git blame`).

---

## 8. Encapsulation

> *"Bundle data and behavior; expose only what callers need."*

Encapsulation is the **load-bearing wall** of OO design — almost every other principle assumes it.

### Bad — Anemic Object

```csharp
public class BankAccount
{
    public decimal Balance;
}

// Anywhere in the system:
account.Balance -= 100;  // no validation, no audit, no concurrency
```

### Good — Behavior Owns Data

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Must be positive");
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}
```

Now invariants are enforced in **one place**, and callers can't break them.

---

## 9. Principle of Least Privilege

> *"Every module should be able to access only the information and resources necessary for its legitimate purpose."* — Saltzer & Schroeder, 1975

Originally a security principle, but equally applicable to code design:

* `private` over `public`.
* `readonly` over mutable.
* `IReadOnlyList<T>` over `List<T>` in return values.
* Database users with the *minimum* grants needed.
* Service accounts scoped to the *minimum* APIs they touch.

Each closed door is one less attack surface and one less footgun.

---

## 10. Single Source of Truth (SoT)

> *"Every piece of data should have exactly **one** authoritative home."*

A close cousin of DRY, but specifically about *state* (DRY is about *knowledge*).

### Bad

```csharp
public class Order
{
    public List<LineItem> Items { get; }
    public int TotalQuantity { get; set; }   // duplicates Items.Sum(i => i.Qty)
}
```

`TotalQuantity` can drift from reality. Two sources of truth = one will be wrong.

### Good

```csharp
public class Order
{
    public IReadOnlyList<LineItem> Items { get; }
    public int TotalQuantity => Items.Sum(i => i.Qty);   // derived, never stored
}
```

---

## 11. Curly's Law (Do One Thing)

> *"A variable, function, or class should mean **one thing** and only one thing."* — Tim Ottinger / Jeff Atwood

A close relative of SRP, but at every granularity:

* A **variable** should not be reused for two purposes (`var x = ...; x = ...somethingElse...`).
* A **function** should not "save and return formatted" — split it.
* A **class** should have a single, nameable purpose.

If you can't name something with a noun or a verb-noun, it's doing too many things.

---

## 12. Hollywood Principle

> *"Don't call us — we'll call you."*

Frameworks invert control: rather than your code calling into the library, the library calls your code. This is the foundation of:

* **Dependency Injection containers**
* **Event-driven systems**
* **Template Method pattern**
* **Lifecycle hooks** (`OnInit`, `OnDestroy`, `BeforeSave`)
* **Callbacks / observers**

It's the same idea as **DIP** seen from the framework author's perspective.

---

## 13. Worse Is Better

> *"A program that is 90% correct, simple, and fast often beats one that is 100% correct, complex, and slow."* — Richard Gabriel, 1989

A counterweight to perfectionism. *Pragmatic* wins over *pure*. Ship the boring solution that solves 95% of the cases; revisit when the remaining 5% becomes a real problem.

### Translations

* Worse Is Better → **YAGNI** + **KISS** + **Lean**
* A perfect framework that ships in 18 months loses to a flawed one shipped in 3.
* Library design should optimize for **adoption**, not for theoretical elegance.

---

## Quick Reference

| Principle                    | One-line takeaway                                                   |
| ---------------------------- | ------------------------------------------------------------------- |
| Law of Demeter               | Only talk to immediate friends.                                     |
| Composition over Inheritance | Prefer has-a to is-a.                                               |
| Tell, Don't Ask              | Send commands; don't probe state.                                   |
| POLA                         | No surprises in public behavior.                                    |
| Fail Fast                    | Catch bugs at the earliest possible moment.                         |
| Convention over Configuration| Sensible defaults; configure only deviations.                       |
| Boy Scout Rule               | Leave the file cleaner than you found it.                           |
| Encapsulation                | Behavior owns the data it depends on.                               |
| Least Privilege              | Grant the minimum access necessary.                                 |
| Single Source of Truth       | Every fact lives in exactly one place.                              |
| Curly's Law                  | One name = one purpose.                                             |
| Hollywood Principle          | Don't call us — we'll call you.                                     |
| Worse Is Better              | Ship something simple over something perfect.                       |

---

> **Final thought:** principles compound. Encapsulation enables Tell-Don't-Ask. Tell-Don't-Ask enables Law of Demeter. Law of Demeter lowers coupling. Lower coupling makes SRP easier. SRP makes OCP painless. Start anywhere; the whole edifice gets stronger together.
