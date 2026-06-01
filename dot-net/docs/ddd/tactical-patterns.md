# Tactical DDD Patterns

> The **code-level building blocks** of Domain-Driven Design. Used *inside* one Bounded Context to model the domain richly and protect its invariants. Each block below has a precise meaning — don't conflate them.

---

## Quick Reference (What · Why · When · Where)

- **What** — The code-level building blocks of DDD: **Entity** (identity over time), **Value Object** (immutable, equal by value), **Aggregate** (consistency boundary with one root), **Domain Service** (behavior that doesn't fit one entity), **Domain Event** (something that happened), **Factory** (complex construction), **Repository** (collection-like access per aggregate root), **Module** (group by domain concept).
- **Why** — These patterns enforce domain invariants at the type system level, eliminate primitive obsession, and keep business logic out of services / controllers.
- **When** — Inside a Bounded Context where the domain has meaningful behavior (not just CRUD). Skip for trivial admin tools — the ceremony costs more than the benefit there.
- **Where** — Live in the **Domain layer** of Clean/Hexagonal/Onion architecture. Pair with strategic patterns (Bounded Contexts) and domain events flowing out via the Outbox pattern to other contexts.

---

## Index

1. [Entity](#1-entity)
2. [Value Object](#2-value-object)
3. [Aggregate & Aggregate Root](#3-aggregate--aggregate-root)
4. [Domain Event](#4-domain-event)
5. [Domain Service](#5-domain-service)
6. [Factory](#6-factory)
7. [Repository](#7-repository)
8. [Putting It All Together](#8-putting-it-all-together)

---

## 1. Entity

An **Entity** is an object with an **identity that persists over time**. Two entities are equal if their **IDs** are equal — even if every other field is identical.

* `Order #42` is the same order even after every line item changes.
* `Customer { Name = "Quan" }` and `Customer { Name = "Quan" }` are **different customers** if their IDs differ.

```csharp
public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;

    public override bool Equals(object? obj) =>
        obj is Entity<TId> other && EqualityComparer<TId>.Default.Equals(Id, other.Id);

    public override int GetHashCode() => Id?.GetHashCode() ?? 0;
}

public sealed class Customer : Entity<Guid>
{
    public string Name { get; private set; }
    public Email Email { get; private set; }

    public Customer(Guid id, string name, Email email)
    {
        Id = id;
        Name = name;
        Email = email;
    }
}
```

**Rules:**

* Identity is established at creation and **never changes**.
* Behavior lives **on the entity** (not on a separate "Service" class) when possible.
* Use private setters; mutate state through methods that enforce invariants.

---

## 2. Value Object

A **Value Object** has **no identity**. It is defined entirely by its **values**. Two value objects with the same data are interchangeable.

* `Money { Amount = 100, Currency = "USD" }` equals any other `Money { 100, USD }`.
* You don't say *"this $100"* vs *"that $100"* — they're the same.

### Properties

* **Immutable** — once created, never changes.
* **Equality by value** (all properties).
* **No side-effect-free methods only** — operations return new instances.
* **Encapsulates business rules** (validation, computation).

### C# — Value Object with `record`

```csharp
public sealed record Email
{
    public string Value { get; }

    public Email(string value)
    {
        if (!Regex.IsMatch(value, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
            throw new ArgumentException("Invalid email", nameof(value));
        Value = value.ToLowerInvariant();
    }

    public override string ToString() => Value;
}

public sealed record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentException("Amount cannot be negative");
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }

    public static Money operator +(Money a, Money b) => a.Add(b);
}
```

### Why Value Objects Matter

Without them, every method takes a primitive `string email, decimal amount, string currency` — and every method must validate. **Primitive obsession** spreads validation everywhere.

With Value Objects, the *type system itself* enforces correctness: you cannot construct an invalid `Email`. A method that takes `Money` cannot be called with a negative number.

---

## 3. Aggregate & Aggregate Root

This is the most important and most misunderstood DDD concept.

### Definitions

* An **Aggregate** is a cluster of related entities + value objects treated as **one unit** for data changes and invariant enforcement.
* The **Aggregate Root** is the *single entity* through which the outside world interacts with the aggregate. Internal entities are accessed *only* via the root.
* The aggregate has an **invariant boundary**: rules that must hold for the aggregate as a whole must be enforceable in **one local transaction**.

### Example

```
Order (Aggregate Root)
 ├── OrderLine (Entity inside aggregate, no outside reference)
 ├── ShippingAddress (Value Object)
 └── TotalAmount (Value Object — derived)

Customer (Different Aggregate Root)
```

Outside code can `orders.GetById(id)`, then `order.AddLine(...)`. It **cannot** load an `OrderLine` directly — `OrderLine` only exists in the context of an `Order`.

### C# Sample

```csharp
public sealed class Order : Entity<Guid>
{
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total => new(_lines.Sum(l => l.LineTotal.Amount), _lines.FirstOrDefault()?.LineTotal.Currency ?? "USD");

    private Order() { } // EF

    public static Order Place(Guid customerId)
    {
        return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Cannot modify a submitted order");
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive");
        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_lines.Count == 0) throw new InvalidOperationException("Cannot submit an empty order");
        Status = OrderStatus.Submitted;
    }
}

public sealed class OrderLine : Entity<Guid> // entity inside the aggregate
{
    public Guid ProductId { get; }
    public int Quantity { get; }
    public Money UnitPrice { get; }
    public Money LineTotal => new(UnitPrice.Amount * Quantity, UnitPrice.Currency);

    internal OrderLine(Guid productId, int quantity, Money unitPrice)
    {
        Id = Guid.NewGuid();
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }
}
```

Note:

* `_lines` is private; you can only read it.
* All mutations go through `Order` methods that enforce invariants.
* `OrderLine.ctor` is `internal` — outside code cannot bypass `Order.AddLine`.

### Rules of Thumb for Aggregate Design

* **Small aggregates**. Vaughn Vernon: *"design small aggregates."* Big aggregates become bottlenecks under concurrency.
* **One aggregate per transaction**. If a single use case modifies two aggregates, the rule is: each gets its own transaction; coordinate them via **Domain Events** (in-process) or **Sagas** (cross-service).
* **Reference other aggregates by ID, not navigation property.** `Order.CustomerId : Guid`, not `Order.Customer : Customer`. This prevents accidentally loading multi-aggregate graphs.
* **One Repository per Aggregate Root** — never a repository for an internal entity.

---

## 4. Domain Event

A **Domain Event** is something the domain cares about that **has happened** — past tense.

```csharp
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total, DateTime OccurredAt);
public sealed record OrderShipped(Guid OrderId, string TrackingNumber, DateTime OccurredAt);
public sealed record CustomerEmailChanged(Guid CustomerId, Email OldEmail, Email NewEmail);
```

**Properties:**

* **Past tense**, immutable.
* Raised by an **Aggregate** as a side effect of a business operation.
* Dispatched **after** the aggregate is persisted (commit-then-publish).
* Consumed by other aggregates (in the same context) or other services (integration events).

### C# — Raising Domain Events from an Aggregate

```csharp
public interface IHasDomainEvents
{
    IReadOnlyList<INotification> DomainEvents { get; }
    void ClearDomainEvents();
}

public abstract class Entity<TId> : IHasDomainEvents
{
    public TId Id { get; protected set; } = default!;
    private readonly List<INotification> _events = new();
    public IReadOnlyList<INotification> DomainEvents => _events;
    protected void Raise(INotification ev) => _events.Add(ev);
    public void ClearDomainEvents() => _events.Clear();
}

public sealed class Order : Entity<Guid>
{
    public void Submit()
    {
        // ... checks + state change ...
        Status = OrderStatus.Submitted;
        Raise(new OrderPlaced(Id, CustomerId, Total.Amount, DateTime.UtcNow));
    }
}

// Dispatch on SaveChangesAsync (in AppDbContext) — see architectural-style/event-driven.md
```

See [`../architectural-style/event-driven.md`](../architectural-style/event-driven.md) for the difference between **domain events** (in-process) and **integration events** (cross-service).

---

## 5. Domain Service

When a piece of domain behavior **doesn't naturally fit on any entity or value object**, model it as a **Domain Service**.

Heuristic: if you find yourself adding a method to an entity that takes *another entity from a different aggregate*, that's often a Domain Service.

```csharp
// Bad — Order shouldn't know how to look up a customer's credit profile
public class Order
{
    public void ApplyDiscount(Customer customer) { /* ... */ }
}

// Good — domain service coordinates two aggregates
public sealed class PricingService
{
    public Money CalculateDiscountedTotal(Order order, Customer customer, IReadOnlyList<Promotion> active)
    {
        // logic involving all three
    }
}
```

**Domain Service vs. Application Service:**

* **Domain Service** lives in the domain layer. Knows nothing about HTTP, EF, MediatR. Pure domain logic.
* **Application Service** lives in the application layer. Orchestrates use cases (load aggregate from repo, call domain methods, save, publish events). Knows about repositories and the unit of work.

---

## 6. Factory

When constructing an aggregate is complex (many invariants, multiple parts, derived values), encapsulate it in a **Factory**:

```csharp
public static class OrderFactory
{
    public static Order CreateFromCart(Customer customer, Cart cart, IReadOnlyList<DiscountRule> rules)
    {
        if (cart.IsEmpty) throw new InvalidOperationException("Cannot place an order with an empty cart");

        var order = Order.Place(customer.Id);
        foreach (var item in cart.Items)
            order.AddLine(item.ProductId, item.Quantity, item.UnitPrice);

        foreach (var rule in rules.Where(r => r.AppliesTo(order)))
            rule.Apply(order);

        return order;
    }
}
```

A static factory method on the aggregate itself (`Order.Place(...)`) also counts as a Factory and is often preferred.

---

## 7. Repository

See the full discussion in [`../enterprise-pattern/repository.md`](../enterprise-pattern/repository.md).

In DDD tactical terms:

* **One repository per aggregate root.** `IOrderRepository`, not `IOrderLineRepository`.
* Methods speak the **domain language**: `GetUnpaidByCustomerAsync`, not `Find(predicate)`.
* The repository is part of the **domain layer** (interface) and **infrastructure layer** (implementation).

---

## 8. Putting It All Together

A typical .NET DDD slice for "place an order" looks like this:

```
[ASP.NET Controller]
    └─▶ MediatR.Send(new PlaceOrderCommand(...))
            └─▶ PlaceOrderHandler                        ← Application Service
                  ├─▶ ICustomerRepository.GetById(...)   ← Repository (Domain interface)
                  ├─▶ OrderFactory.CreateFromCart(...)   ← Factory
                  │       └─▶ Order (Aggregate Root)     ← Tactical patterns at work
                  │              ├─ OrderLine (Entity)
                  │              ├─ Money (Value Object)
                  │              └─ raises OrderPlaced (Domain Event)
                  ├─▶ IOrderRepository.AddAsync(order)
                  └─▶ IUnitOfWork.SaveChangesAsync()      ← commit
                          └─▶ Domain events dispatched (in-process) or written to outbox (integration)
```

Each tactical pattern has one job. Together they protect domain invariants and keep business logic centralized in the domain layer.

---

## Common Anti-Patterns

* **Anemic Domain Model** — entities with only getters/setters, all logic in services. You have the structure of DDD without the substance.
* **Aggregates that span everything** — one giant aggregate root that contains the whole world. Concurrency dies.
* **Cross-aggregate references via navigation properties** — `Order.Customer` instead of `Order.CustomerId`. Loads explode; transactions widen.
* **Repositories with `Update` methods** — EF Core tracks changes; you don't need to "update". Mutate the aggregate and save.
* **Domain events leaked as integration events** — domain events are in-process; integration events cross service boundaries. Don't mix.

---

## References

* Eric Evans — *Domain-Driven Design* (2003). Part II.
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Chapters 5–11.
* Vaughn Vernon — *Effective Aggregate Design* (3-part paper, free). [vaughnvernon.com](https://www.dddcommunity.org/library/vernon_2011/).
