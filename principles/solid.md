# SOLID — The Five Object-Oriented Principles

> Popularized by **Robert C. Martin** ("Uncle Bob") around 2000, SOLID is an acronym for the five most influential principles of object-oriented design. Master them and you'll write code that is **easy to change, easy to test, and easy to understand**.

---

## Table of Contents

1. [S — Single Responsibility Principle (SRP)](#1-s--single-responsibility-principle-srp)
2. [O — Open/Closed Principle (OCP)](#2-o--openclosed-principle-ocp)
3. [L — Liskov Substitution Principle (LSP)](#3-l--liskov-substitution-principle-lsp)
4. [I — Interface Segregation Principle (ISP)](#4-i--interface-segregation-principle-isp)
5. [D — Dependency Inversion Principle (DIP)](#5-d--dependency-inversion-principle-dip)
6. [Summary Table](#6-summary-table)

---

## 1. S — Single Responsibility Principle (SRP)

> *"A class should have only **one reason to change**."* — Robert C. Martin

A "reason to change" is a **stakeholder** or **axis of change**, not a method count.

### 1.1 Bad — Multiple Reasons to Change

```csharp
public class Report
{
    public string Title { get; set; }
    public List<Row> Rows { get; set; }

    public decimal CalculateTotal()           { /* business rule */ }
    public string  RenderHtml()                { /* presentation */ }
    public void    SaveToDatabase()            { /* persistence  */ }
    public void    EmailTo(string address)     { /* delivery     */ }
}
```

Changes for: tax law (business), brand redesign (HTML), DB switch, SMTP provider — four orthogonal forces, one class.

### 1.2 Good — One Reason Each

```csharp
public record Report(string Title, IReadOnlyList<Row> Rows);

public class ReportCalculator   { public decimal Total(Report r) { ... } }
public class ReportRenderer     { public string  Html(Report r)  { ... } }
public class ReportRepository   { public Task SaveAsync(Report r) { ... } }
public class ReportMailer       { public Task SendAsync(Report r, string to) { ... } }
```

### 1.3 Heuristic

> *"If you can write a sentence describing the class using **and**, it probably violates SRP."*

---

## 2. O — Open/Closed Principle (OCP)

> *"Software entities should be **open for extension** but **closed for modification**."*
> — Bertrand Meyer (1988), refined by Robert C. Martin

You should be able to add new behavior **without editing existing, tested code**.

### 2.1 Bad — Switch That Grows With Every New Type

```csharp
public class Discount
{
    public decimal For(Customer c)
    {
        return c.Type switch
        {
            "Regular" => 0m,
            "Premium" => 0.10m,
            "Vip"     => 0.20m,
            _         => 0m
        };
    }
}
```

Every new customer type means editing this method and re-testing it.

### 2.2 Good — Open via Polymorphism

```csharp
public interface ICustomer { decimal DiscountRate(); }

public class Regular : ICustomer { public decimal DiscountRate() => 0m;    }
public class Premium : ICustomer { public decimal DiscountRate() => 0.10m; }
public class Vip     : ICustomer { public decimal DiscountRate() => 0.20m; }

public class Discount
{
    public decimal For(ICustomer c) => c.DiscountRate();
}
```

Adding a `Platinum` customer means **adding one class** — `Discount` is untouched.

### 2.3 Common Tools to Achieve OCP

* **Polymorphism / inheritance**
* **Strategy pattern**
* **Plugin architectures**
* **Decorator pattern**
* **Configuration over conditionals**

### 2.4 Pitfall

Don't *pre-emptively* make everything extensible. Combine with **YAGNI** — wait for the second variant.

---

## 3. L — Liskov Substitution Principle (LSP)

> *"Subtypes must be **substitutable** for their base types without altering the correctness of the program."*
> — Barbara Liskov, 1987

If `S` extends `T`, every place that uses `T` must keep working when given an `S` — same behavior contract, same invariants, same exceptions.

### 3.1 Bad — Rectangle / Square

```csharp
public class Rectangle
{
    public virtual int Width  { get; set; }
    public virtual int Height { get; set; }
}

public class Square : Rectangle
{
    public override int Width  { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}

void Test(Rectangle r)
{
    r.Width  = 5;
    r.Height = 4;
    Debug.Assert(r.Width * r.Height == 20); // FAILS for Square — area is 16
}
```

A `Square` is mathematically a Rectangle, but **behaviorally** it isn't substitutable.

### 3.2 Good — Don't Force the "Is-A"

Model `Square` and `Rectangle` independently (or share an immutable `IShape` interface that returns `Area()`).

### 3.3 LSP Rules in Practice

A subclass must:

* **Not strengthen preconditions** (don't demand more than the parent).
* **Not weaken postconditions** (don't deliver less).
* **Preserve invariants** of the parent.
* **Not throw new exception types** the parent doesn't.

### 3.4 Real-World Smell

```csharp
public class ReadOnlyList<T> : List<T>
{
    public new void Add(T item) => throw new NotSupportedException();
}
```

Hides a `NotSupportedException` behind a base class signature — a textbook LSP violation. Use a smaller interface (`IReadOnlyList<T>`) instead.

---

## 4. I — Interface Segregation Principle (ISP)

> *"Clients should not be forced to depend on interfaces they do not use."*
> — Robert C. Martin

Fat interfaces become coupling magnets. Prefer **many small, role-specific interfaces**.

### 4.1 Bad — Fat Interface

```csharp
public interface IMultiFunctionDevice
{
    void Print(Document d);
    void Scan(Document d);
    void Fax(Document d);
    void Staple(Document d);
}

public class HomePrinter : IMultiFunctionDevice
{
    public void Print(Document d) { /* ok */ }
    public void Scan(Document d)  => throw new NotSupportedException();
    public void Fax(Document d)   => throw new NotSupportedException();
    public void Staple(Document d)=> throw new NotSupportedException();
}
```

### 4.2 Good — Segregated Interfaces

```csharp
public interface IPrinter { void Print(Document d); }
public interface IScanner { void Scan(Document d);  }
public interface IFax     { void Fax(Document d);   }
public interface IStapler { void Staple(Document d);}

public class HomePrinter   : IPrinter { ... }
public class OfficeMfp     : IPrinter, IScanner, IFax, IStapler { ... }
```

Clients depend only on what they actually use.

### 4.3 Heuristic

> *"Interfaces should be **role-based**, not **technology-based**."*

A `IUserRepository` with 14 methods often hides 3 distinct roles (`IUserReader`, `IUserWriter`, `IUserAuthenticator`).

---

## 5. D — Dependency Inversion Principle (DIP)

> *"High-level modules should not depend on low-level modules. **Both should depend on abstractions**. Abstractions should not depend on details. Details should depend on abstractions."*
> — Robert C. Martin

DIP is the principle behind **dependency injection**, **clean architecture**, and **hexagonal/ports-and-adapters**.

### 5.1 Bad — Concrete Dependency

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repo = new();   // concrete coupling
    private readonly SmtpEmailSender    _mail = new();   // concrete coupling

    public void Place(Order o) { _repo.Save(o); _mail.Send(o.Email, "Thanks"); }
}
```

You can't:

* Swap SQL for Mongo without editing this class.
* Unit-test without a real SMTP server.

### 5.2 Good — Depend on Abstractions

```csharp
public interface IOrderRepository { void Save(Order o); }
public interface IEmailSender     { void Send(string to, string body); }

public class OrderService(IOrderRepository repo, IEmailSender mail)
{
    public void Place(Order o) { repo.Save(o); mail.Send(o.Email, "Thanks"); }
}

// Composition root
var service = new OrderService(new SqlOrderRepository(), new SmtpEmailSender());
```

### 5.3 The Two "Inversions"

1. **Inversion of source-code dependency** — high-level code no longer imports low-level details.
2. **Inversion of control** — the framework or composition root *gives* dependencies instead of the class *fetching* them.

### 5.4 Where the Abstraction Lives

> *"The abstraction belongs in the layer that **uses** it, not the one that **implements** it."*

`IOrderRepository` lives next to `OrderService`, not next to `SqlOrderRepository`.

---

## 6. Summary Table

| Letter | Principle               | Smell it prevents              | Tool it enables                  |
| ------ | ----------------------- | ------------------------------ | -------------------------------- |
| **S**  | Single Responsibility   | God class, shotgun surgery     | Cohesive, testable modules       |
| **O**  | Open/Closed             | Edits propagate through code   | Strategy, plugins, extensions    |
| **L**  | Liskov Substitution     | Surprise behavior in subclasses| Safe polymorphism                |
| **I**  | Interface Segregation   | Bloated, coupling interfaces   | Role-based abstractions          |
| **D**  | Dependency Inversion    | Concrete coupling, untestable  | DI, clean architecture, mocking  |

---

> **Mnemonic:** *"**S**ingle, **O**pen, **L**iskov, **I**nterface, **D**ependency."*
> Apply SOLID in service of clarity — never as bureaucracy. If a principle complicates today's code without paying off, you may not need it *yet*.
