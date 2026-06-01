# GRASP — 9 Responsibility-Assignment Principles

> **GRASP** = **G**eneral **R**esponsibility **A**ssignment **S**oftware **P**atterns.
> Introduced by **Craig Larman** in *Applying UML and Patterns* (1997). While SOLID asks *"is this class well designed?"*, GRASP asks the more fundamental question: ***"to which class should this responsibility belong?"***

---

## Quick Reference (What · Why · When · Where)

- **What** — The 9 GRASP principles (Craig Larman): **Information Expert**, **Creator**, **Controller**, **Low Coupling**, **High Cohesion**, **Polymorphism**, **Pure Fabrication**, **Indirection**, **Protected Variations**.
- **Why** — Where SOLID asks *"is this class designed well?"*, GRASP asks the more fundamental question: ***"which class should this responsibility belong to?"***. Together they cover both the design and the allocation of responsibility.
- **When** — Deciding where to put a new method; reviewing a "Feature Envy" smell; explaining why a Repository (Pure Fabrication) doesn't represent a real domain concept; designing a Mediator (Indirection).
- **Where** — Class-level responsibility assignment. Foundation for understanding *why* the GoF patterns are shaped the way they are.

---

## Table of Contents

1. [Information Expert](#1-information-expert)
2. [Creator](#2-creator)
3. [Controller](#3-controller)
4. [Low Coupling](#4-low-coupling)
5. [High Cohesion](#5-high-cohesion)
6. [Polymorphism](#6-polymorphism)
7. [Pure Fabrication](#7-pure-fabrication)
8. [Indirection](#8-indirection)
9. [Protected Variations](#9-protected-variations)
10. [Summary](#10-summary)

---

## 1. Information Expert

> *"Assign the responsibility to the class that **has the information** needed to fulfill it."*

If a class already owns the data, putting the behavior elsewhere creates **Feature Envy**.

### Bad

```csharp
public class Order { public IList<LineItem> Items { get; } = new List<LineItem>(); }

public class OrderTotalService
{
    public decimal Total(Order o) => o.Items.Sum(i => i.Price * i.Qty);
}
```

`OrderTotalService` only exists to read `Order`'s data — the order should compute itself.

### Good

```csharp
public class Order
{
    public IList<LineItem> Items { get; } = new List<LineItem>();
    public decimal Total() => Items.Sum(i => i.Price * i.Qty);
}
```

---

## 2. Creator

> *"Class B should create objects of class A when at least one of these is true:*
> *— B aggregates A*
> *— B contains A*
> *— B records A*
> *— B closely uses A*
> *— B has the initializing data for A."*

### Example

`Order` contains `LineItem`s — so `Order` should create them.

```csharp
public class Order
{
    private readonly List<LineItem> _items = new();

    public void AddItem(Product p, int qty)
    {
        _items.Add(new LineItem(p, qty, p.Price));   // Order is the Creator
    }
}
```

This avoids leaking construction logic into clients and keeps invariants centralized.

---

## 3. Controller

> *"Assign the responsibility for handling a **system event** to a class representing the **overall system, a use-case scenario, or a façade**."*

A Controller sits between the UI and the domain — it doesn't *do* the work, it **orchestrates** it.

### Example

```csharp
// ASP.NET Core — controller is thin, delegates to a use-case service
public class CheckoutController(ICheckoutUseCase useCase) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Place([FromBody] PlaceOrderRequest req)
    {
        var result = await useCase.PlaceOrderAsync(req);
        return result.IsSuccess ? Ok(result.Value) : BadRequest(result.Error);
    }
}
```

### Bad Form (Fat Controller)

A controller that validates, computes totals, charges payment, sends emails, and writes to the DB violates SRP **and** Controller.

---

## 4. Low Coupling

> *"Assign responsibilities so that **coupling remains low**."*

Coupling = how dependent a class is on other classes. Low coupling means:

* Easy to change one class without touching others.
* Easy to test in isolation.
* Easy to reuse.

### Symptoms of High Coupling

* Classes know each other's *internal* fields.
* A change in module A breaks tests in module B.
* You can't compile one module without ten others.

### Tools to Lower Coupling

* Depend on **interfaces**, not concretes (DIP).
* Use **events / messages** instead of direct calls.
* Apply **Indirection** (see below).
* Respect **Law of Demeter**.

---

## 5. High Cohesion

> *"Assign responsibilities so that **cohesion remains high**."*

Cohesion = how strongly the responsibilities inside one class belong together.

### Bad — Low Cohesion (Utility Class)

```csharp
public static class Helpers
{
    public static string  Capitalize(string s)        { ... }
    public static decimal CalculateTax(decimal d)      { ... }
    public static void    SendEmail(string to)         { ... }
    public static byte[]  CompressZip(byte[] data)     { ... }
}
```

Four unrelated responsibilities — one class. Any change touches it.

### Good — High Cohesion

```csharp
public static class StringFormatting { public static string Capitalize(string s) { ... } }
public static class TaxPolicy        { public static decimal Apply(decimal d) { ... } }
public class MailService             { public void Send(string to) { ... } }
public static class ZipCompression   { public static byte[] Compress(byte[] data) { ... } }
```

> **Low coupling + high cohesion** is the single most important pair of metrics in OO design. Almost every other principle exists to serve them.

---

## 6. Polymorphism

> *"When alternatives or behaviors vary by type, **use polymorphic operations**, not type checks."*

This is the **OCP enabler** at the responsibility-assignment level.

### Bad

```csharp
public decimal Area(Shape s)
{
    if (s is Circle c)    return Math.PI * c.R * c.R;
    if (s is Square sq)   return sq.Side * sq.Side;
    if (s is Triangle t)  return 0.5m * t.B * t.H;
    throw new InvalidOperationException();
}
```

### Good

```csharp
public abstract class Shape { public abstract decimal Area(); }
public class Circle(decimal r)               : Shape { public override decimal Area() => (decimal)Math.PI * r * r; }
public class Square(decimal side)            : Shape { public override decimal Area() => side * side; }
public class Triangle(decimal b, decimal h)  : Shape { public override decimal Area() => 0.5m * b * h; }
```

Adding a `Pentagon` no longer touches `Area()` — closed for modification, open for extension.

---

## 7. Pure Fabrication

> *"When no existing domain class fits, **invent a class** with no real-world counterpart to keep cohesion and coupling clean."*

Pure Fabrication = a *made-up* class that doesn't model a domain concept but earns its keep technically.

### Examples

* `OrderRepository` — there is no "repository" in the business domain, but it isolates persistence.
* `EmailSender` — a service object created purely for technical needs.
* `TaxCalculator` — extracted out so `Order` doesn't bloat.

### Why It Matters

Without Pure Fabrication you end up overloading domain classes (Order persists itself, mails itself, prints itself) — a clear SRP and Cohesion violation.

---

## 8. Indirection

> *"Assign a responsibility to an **intermediate object** to mediate between other components."*

This is the meta-principle behind:

* **Adapter pattern**
* **Mediator pattern**
* **Facade pattern**
* **Proxy / Decorator**
* **Service buses / message brokers**

### Example

```csharp
// Direct: ApplicationLayer talks to MailKit -> tight coupling to a library
// Indirection: ApplicationLayer talks to IEmailSender ; MailKitEmailSender implements it

public interface IEmailSender { Task SendAsync(string to, string subject, string body); }

public class MailKitEmailSender : IEmailSender { /* knows MailKit */ }
public class SendGridEmailSender : IEmailSender { /* knows SendGrid */ }
```

Swapping providers becomes a configuration change, not a refactor.

### Caution

Indirection has a cost: more files, more vocabulary. Apply only when **decoupling pays off**.

---

## 9. Protected Variations

> *"Identify points of predicted **variation or instability**; assign responsibilities to create a **stable interface** around them."*

Wrap volatility behind a contract. Inside the wall, things can change; outside, callers are protected.

### Examples

* `IPaymentGateway` shields the codebase from "Stripe", "Adyen", "PayPal" changes.
* `IClock` shields tests from `DateTime.Now`.
* `IFileStorage` shields code from "local filesystem vs S3".

### Relationship to Other Principles

Protected Variations is the **goal**. Indirection, Polymorphism, OCP, and DIP are the **means** to achieve it.

---

## 10. Summary

| #   | Principle                  | Question it answers                                              |
| --- | -------------------------- | ---------------------------------------------------------------- |
| 1   | **Information Expert**     | *Who has the data to do this?*                                   |
| 2   | **Creator**                | *Who should instantiate this?*                                   |
| 3   | **Controller**             | *Who handles this system event?*                                 |
| 4   | **Low Coupling**           | *Can we reduce inter-class dependencies?*                        |
| 5   | **High Cohesion**          | *Does everything in this class belong together?*                 |
| 6   | **Polymorphism**           | *How do we handle type-based variation without `switch`?*        |
| 7   | **Pure Fabrication**       | *No domain class fits — should we invent one?*                   |
| 8   | **Indirection**            | *Should we put a middleman between these two parties?*           |
| 9   | **Protected Variations**   | *What's likely to change, and how do we shield callers from it?* |

---

> **Mental model:**
> *SOLID describes good **classes**. GRASP decides **which class** each responsibility belongs to.* Use them together — they answer different questions.
