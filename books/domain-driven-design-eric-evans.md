# Domain-Driven Design — Summary

> **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software*, Addison-Wesley, 2003. Often called **"The Blue Book."**

> **Why this book matters:** the foundational text for modeling complex domains in object-oriented software. Evans introduced vocabulary the industry still uses daily — *Ubiquitous Language, Bounded Context, Aggregate, Domain Event, Anti-Corruption Layer, Core Domain*. The Blue Book is dense, sometimes meandering, and over 500 pages long; this summary distills its arguments into the structure most engineers need to know about, with examples in my own words and original code.

> **This summary covers:** the philosophy of model-driven design, Ubiquitous Language, the tactical patterns (Entity, Value Object, Aggregate, Factory, Repository, Domain Service, Module, Domain Event), the strategic patterns (Bounded Context, Context Map, Core/Supporting/Generic subdomains, Anti-Corruption Layer, and the rest), supple design principles, and how the book has been received and extended (Vernon, EventStorming) since 2003.

> 🇻🇳 Vietnamese version: [`domain-driven-design-eric-evans-vi.md`](./domain-driven-design-eric-evans-vi.md)

---

## Quick Reference (What · Why · When · Where)

- **What** — A digest of Eric Evans' 2003 Blue Book — Ubiquitous Language, Model-Driven Design, the tactical patterns (Entity, Value Object, Aggregate, Domain Service, Repository, Factory, Domain Event), and the strategic patterns (Bounded Context, Context Map, Anti-Corruption Layer, Core/Supporting/Generic subdomains).
- **Why** — Gives you the vocabulary and discipline to model complex domains so the code matches how domain experts speak — eliminating the translation friction that produces 90% of "but my code does the wrong thing" bugs.
- **When** — The domain is complex and central to the business (banking, insurance, logistics, healthcare); two teams use the same word for different things; you're integrating with a legacy mess.
- **Where** — Pair with `dot-net/docs/ddd/` (.NET treatment of tactical + strategic patterns), `books/clean-architecture-robert-martin.md` (the architectural shell that DDD's domain layer lives inside), and `books/building-microservices-sam-newman.md` (one bounded context per service).

---

## Table of Contents

1. [About the Book](#1-about-the-book)
2. [Evans' Central Claim](#2-evans-central-claim)
3. [Part I — Putting the Domain Model to Work](#3-part-i--putting-the-domain-model-to-work)
4. [Ubiquitous Language](#4-ubiquitous-language)
5. [Model-Driven Design](#5-model-driven-design)
6. [Part II — The Tactical Patterns](#6-part-ii--the-tactical-patterns)
7. [Layered Architecture & The Smart UI Anti-Pattern](#7-layered-architecture--the-smart-ui-anti-pattern)
8. [Entities](#8-entities)
9. [Value Objects](#9-value-objects)
10. [Domain Services](#10-domain-services)
11. [Modules](#11-modules)
12. [Aggregates](#12-aggregates)
13. [Repositories](#13-repositories)
14. [Factories](#14-factories)
15. [Domain Events (Added Later)](#15-domain-events-added-later)
16. [Part III — Refactoring Toward Deeper Insight](#16-part-iii--refactoring-toward-deeper-insight)
17. [Supple Design](#17-supple-design)
18. [Part IV — Strategic Design](#18-part-iv--strategic-design)
19. [Bounded Context](#19-bounded-context)
20. [Context Mapping — The Relationship Patterns](#20-context-mapping--the-relationship-patterns)
21. [Distillation — Core, Supporting, Generic Subdomains](#21-distillation--core-supporting-generic-subdomains)
22. [Large-Scale Structure](#22-large-scale-structure)
23. [Big-Picture Take-Aways](#23-big-picture-take-aways)
24. [What the Book Doesn't Cover (And Vernon's Companion)](#24-what-the-book-doesnt-cover-and-vernons-companion)
25. [How This Book Fits the Repo](#25-how-this-book-fits-the-repo)
26. [Reading Order Recommendations](#26-reading-order-recommendations)
27. [Common Misreadings](#27-common-misreadings)
28. [References](#28-references)

---

## 1. About the Book

* **Title:** *Domain-Driven Design: Tackling Complexity in the Heart of Software*
* **Author:** Eric Evans (consultant, founder of Domain Language)
* **Publisher:** Addison-Wesley, 2003
* **Length:** ~560 pages
* **Audience:** designers, architects, senior developers working on systems where the *domain itself* is complex — banking, insurance, healthcare, logistics, supply chain, ERP.

The book is famously hard to read end-to-end. Evans interleaves principle, narrative, code, and reflection in a way that rewards rereads. The most-cited content is split between the **tactical patterns** (Part II) and the **strategic design** (Part IV). Many engineers read these two parts and ignore the rest.

---

## 2. Evans' Central Claim

> **The heart of software is its ability to solve domain-related problems. All other features, vital though they may be, support this essential purpose.**

Evans' argument unfolds:

1. Software complexity has a source: the **domain** itself.
2. The way to manage that complexity is to develop a **deep model** of the domain.
3. The model is shared between domain experts and developers via a **Ubiquitous Language**.
4. The model must be expressed *directly* in the code — not translated into a separate "technical model" that drifts from the domain language.
5. To keep the model coherent at scale, draw explicit **Bounded Contexts**, document their relationships in a **Context Map**, and protect the most valuable parts (the **Core Domain**) with disciplined design.

Everything in the book serves these five ideas.

---

## 3. Part I — Putting the Domain Model to Work

### Crunching knowledge

Evans starts with a story: a developer working on a domain they don't understand produces code that's technically correct but useless to the business. The cure is **knowledge crunching** — relentlessly working with domain experts to expose the implicit rules behind the explicit ones.

The takeaway: **modeling is a collaboration**, not a solo activity. The model is the medium through which developers and domain experts negotiate understanding.

### The "what is a model" question

In DDD, a **domain model** is not a UML diagram, not the database schema, not the class hierarchy. It is the **shared conceptual structure** that both code and conversation refer to. A diagram captures one snapshot of it; the language is the canonical form.

---

## 4. Ubiquitous Language

The most influential idea in the book.

> A **Ubiquitous Language** is a *single, rigorous language* shared by developers, domain experts, testers, and stakeholders — used in conversations, documents, tickets, code, and tests.

Where most teams have a *jargon gap* between business and engineering, a UL eliminates it:

* The domain expert says "underwrite a policy".
* The ticket says "underwrite a policy".
* The acceptance test scenario says "underwrite a policy".
* The class is `Policy` with a method `Underwrite(...)`.
* The git commit says "fix bug in policy underwriting".

When the language is consistent across all those artifacts, **translation friction disappears**, and the model can evolve safely because everyone shares the same vocabulary.

### Practical mechanics

* Maintain a **glossary** per Bounded Context.
* **Refactor names ruthlessly** when the domain experts' vocabulary shifts. Renaming a class is cheap; talking past each other for months is not.
* **Use the language in code reviews.** "Why is this called `Container`? Domain experts call it a Shipment."
* **One language per Bounded Context, not one language for the whole company.** "Customer" in Sales and "Customer" in Support are different concepts; force them into one and you destroy the model.

---

## 5. Model-Driven Design

Evans argues for a tight binding between the **conceptual model** and the **code**. Two ways teams fail at this:

* **The model is a diagram nobody updates.** Drift accumulates; the code becomes the only source of truth, and the diagrams lie.
* **The model and code use different vocabularies.** The model says "underwrite"; the code has `update_status_3()`.

The DDD remedy:

* Code is *expressed in the language of the model*.
* Class names, method names, variable names — all match the Ubiquitous Language.
* The model evolves through refactoring; the code follows.

### Hands-On Modelers

A controversial chapter: Evans argues that **modelers must write code**, and **coders must participate in modeling**. The traditional separation — analysts produce a "design document", coders implement it — produces dead models. The team that owns the model owns the code; vice versa.

---

## 6. Part II — The Tactical Patterns

The most-cited section. These are the building blocks of a model-driven design inside one Bounded Context.

The pattern catalog:

* **Layered Architecture** — separation of concerns (UI / Application / Domain / Infrastructure).
* **Entity** — has identity that persists through change.
* **Value Object** — defined by its attributes; immutable; no identity.
* **Domain Service** — domain behavior that doesn't naturally belong to an Entity or Value Object.
* **Module** — group of related domain concepts.
* **Aggregate** — a cluster of Entities and Value Objects treated as one unit for consistency.
* **Repository** — collection-like abstraction for retrieving Aggregates.
* **Factory** — encapsulates complex creation of Aggregates.
* **Domain Event** — something that has happened in the domain (added by Evans in a later 2014 paper; routinely included in DDD now).

> Cross-reference: [`../dot-net/docs/ddd/tactical-patterns.md`](../dot-net/docs/ddd/tactical-patterns.md) for full .NET treatment with C# samples.

---

## 7. Layered Architecture & The Smart UI Anti-Pattern

The book defines a **layered architecture** with four classic layers:

| Layer              | Responsibility                                                              |
| ------------------ | --------------------------------------------------------------------------- |
| **User Interface** | Display, accept user commands                                                |
| **Application**    | Orchestrate use cases, manage transactions and security; no business rules |
| **Domain**         | Domain model — Entities, Value Objects, Services, Events                    |
| **Infrastructure** | Persistence, messaging, file I/O, external APIs                             |

Strict rule: each layer depends only on layers below.

### The Smart UI Anti-Pattern

Evans is candid: **for simple, throwaway CRUD apps, full DDD is overkill.** The "Smart UI" — putting everything (UI, business logic, persistence) in one place — is a *legitimate* choice when the project is small and the domain is shallow. But you sacrifice the ability to evolve. The book is explicit: choose Smart UI knowingly, not by accident.

> The Blue Book's most-overlooked sentence: *"Don't apply Domain-Driven Design to a problem that doesn't need it."*

---

## 8. Entities

> An **Entity** is an object that has a **distinct identity** which runs through time and different representations.

Two `Order` objects are *the same entity* if they have the same `Id`, even if their attributes differ (different statuses, line items, totals). Two `Money` objects with the same amount and currency are *equal* — they're Value Objects, not Entities.

### Implementing identity

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
    public EmailAddress Email { get; private set; }

    private Customer() { /* for EF */ }

    public Customer(Guid id, string name, EmailAddress email)
    {
        Id = id;
        Name = name;
        Email = email;
    }

    public void ChangeEmail(EmailAddress newEmail)
    {
        if (newEmail == Email) return;
        Email = newEmail;
        // Could raise a CustomerEmailChanged domain event here
    }
}
```

### Evans' rules

* **Identity is assigned at creation** and **never changes**.
* **Use natural identifiers when possible** (an ISBN, a SSN). Otherwise generate one (`Guid`, autoincrement).
* **Identity is the only attribute that matters for equality**. Two entities with identical state but different ids are *different*; two entities with the same id are *the same*.
* **Behavior belongs to the entity**, not to a service. `customer.ChangeEmail(...)`, not `customerService.ChangeCustomerEmail(customer, ...)`.

---

## 9. Value Objects

> A **Value Object** is an object that describes a characteristic — defined only by its attributes, with **no conceptual identity**.

Money, dates, addresses, colors, phone numbers, ranges, coordinates. Two `Money(100, "USD")` instances are interchangeable; there is no notion of *this* hundred dollars versus *that* hundred dollars.

### The three rules

1. **Immutable.** Once created, never changes. Operations return new instances.
2. **Equality by value.** All attributes contribute to equality and hash code.
3. **Encapsulates business rules.** Validation, formatting, arithmetic — they live here, not scattered across the codebase.

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money
    {
        if (Amount < 0) throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(Currency)) throw new ArgumentException("Currency required");
        Currency = Currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(decimal factor) => this with { Amount = Math.Round(Amount * factor, 2) };

    public static Money operator +(Money a, Money b) => a.Add(b);
}

public sealed record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email", nameof(value));
        Value = value.ToLowerInvariant();
    }

    public override string ToString() => Value;
}
```

### Why Value Objects matter so much

Without them, every method receives `string email, decimal price, string currency` and re-validates. Logic spreads everywhere. With Value Objects, the **type system itself enforces correctness**:

* A method taking `EmailAddress` cannot be called with an invalid email.
* A method taking `Money` cannot multiply a USD amount by a EUR exchange rate by mistake.

This is the cure for **primitive obsession** — one of the smells in *Refactoring* that DDD eliminates by construction.

### The Evans heuristic

> **Prefer Value Objects.** Use Entities only when you genuinely need identity. Most teams over-use Entities and end up with bloated, identity-laden objects representing data that should have been Value Objects.

---

## 10. Domain Services

When a piece of domain behavior **doesn't naturally belong to a single Entity or Value Object**, model it as a **Domain Service**.

Heuristic: if the operation involves *multiple Aggregates from different concepts* (a Customer + an Order + a Promotion), and putting it on any one of them would feel forced — it's a Domain Service.

```csharp
public sealed class OvercreditCheck
{
    public bool IsApproved(Customer customer, Order proposed, IReadOnlyList<Order> outstanding)
    {
        var outstandingTotal = outstanding.Sum(o => o.Total.Amount);
        return outstandingTotal + proposed.Total.Amount <= customer.CreditLimit.Amount;
    }
}
```

### Domain Service vs. Application Service

Important distinction Evans labors over:

| Layer                 | Service                                                                   |
| --------------------- | ------------------------------------------------------------------------- |
| **Domain Service**    | Pure domain logic. No I/O, no transactions, no framework knowledge.        |
| **Application Service** | Use-case orchestration. Loads aggregates from repositories, calls domain logic, commits via Unit of Work, publishes events. |

A `PlaceOrderHandler` (Application) calls a `PricingPolicy` (Domain Service) to compute the price, then saves the Order via its Repository. Clean separation.

---

## 11. Modules

A **Module** (called *package* or *namespace* in code) groups related domain concepts. Evans' rule: **modules should reflect the model**, not technical concerns.

Bad: `Controllers/`, `Services/`, `Models/`, `Repositories/`.

Good: `Orders/`, `Pricing/`, `Shipping/`, `Customers/`.

This is the same idea as **Screaming Architecture** (Robert Martin), but Evans pre-dates it.

---

## 12. Aggregates

The single most-misunderstood pattern in the book.

> An **Aggregate** is a cluster of associated objects (Entities and Value Objects) treated as a **unit for the purpose of data changes**. The cluster has a single **root Entity** through which the outside world interacts with it.

### Why aggregates exist

In a complex domain, business rules often span multiple objects. Without explicit boundaries, every transaction risks corrupting invariants across the model. Aggregates solve this by:

* Bounding **transactional consistency** — within one aggregate, invariants hold *immediately* after each transaction.
* Allowing **eventual consistency** *between* aggregates — different aggregates may be inconsistent for a short period.

### The rules

1. **One aggregate root per aggregate.** The root is the only externally referenced Entity in the aggregate.
2. **Outside objects reference only the root** — never the internal Entities. Internal Entities have no externally usable identity.
3. **Outside objects reference other aggregates only by ID**, not by direct object reference.
4. **A transaction modifies one aggregate.** If a use case must modify two aggregates, do them in separate transactions and coordinate via Domain Events or Sagas.
5. **The aggregate root enforces invariants** of the aggregate.

```csharp
public sealed class Order : Entity<Guid>     // ← Aggregate root
{
    private readonly List<OrderLine> _lines = new();

    public Guid CustomerId { get; }            // Reference to another aggregate by ID (not direct)
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Money Total => _lines.Aggregate(
        new Money(0m, "USD"),
        (sum, line) => sum.Add(line.LineTotal));

    private Order() { }

    public static Order Place(Guid customerId)
    {
        return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify a submitted order");
        if (quantity <= 0)
            throw new ArgumentException("Quantity must be positive");

        _lines.Add(new OrderLine(productId, quantity, unitPrice));   // internal Entity
    }

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Cannot submit empty order");
        Status = OrderStatus.Submitted;
        // Raise a domain event: OrderSubmitted
    }
}

public sealed class OrderLine : Entity<Guid>   // Internal Entity — only Order creates these
{
    public Guid ProductId { get; }
    public int Quantity { get; }
    public Money UnitPrice { get; }
    public Money LineTotal => UnitPrice.Multiply(Quantity);

    internal OrderLine(Guid productId, int quantity, Money unitPrice)
    {
        Id = Guid.NewGuid();
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }
}
```

### Aggregate design rules of thumb (often attributed to Vaughn Vernon)

* **Small aggregates.** Big aggregates cause contention and slow performance.
* **Reference other aggregates by id.** Not by navigation property.
* **One aggregate per transaction.** If you need to change two, use Domain Events.
* **Eventual consistency between aggregates.** Two aggregates briefly inconsistent is normal and acceptable.

---

## 13. Repositories

> A **Repository** provides a **collection-like abstraction** for retrieving and storing Aggregate Roots. Clients interact with it as if it were an in-memory collection.

### What a repository should look like

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> FindUnpaidByCustomerAsync(Guid customerId, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

### The rules Evans lays out

1. **One repository per aggregate root.** Not per table. `IOrderRepository`, not `IOrderLineRepository`.
2. **Methods return aggregates**, not DTOs and not `IQueryable`.
3. **Method names speak the domain language**: `FindUnpaidByCustomer`, not `FindByPredicate`.
4. **The interface lives in the domain layer**; implementations live in infrastructure.
5. **Repositories hide persistence completely.** Domain code that uses a repository should be portable across SQL Server, Postgres, MongoDB, file storage, in-memory — no changes.

### Why the Generic Repository (`IRepository<T>`) is widely considered an anti-pattern in DDD

* It exposes a CRUD-shaped interface that has no business meaning.
* Methods like `Find(Expression<Func<T, bool>>)` push query construction back into application code, violating "hide the data store".
* It often gets applied to *every* entity, including ones that aren't aggregate roots — destroying the aggregate boundary.

> Cross-reference: [`../dot-net/docs/enterprise-pattern/repository.md`](../dot-net/docs/enterprise-pattern/repository.md) has the full discussion of Generic Repository vs. proper DDD Repository.

---

## 14. Factories

When constructing an Aggregate is complex (many invariants, related objects to create together, derived values), encapsulate that complexity in a **Factory** — either a static factory method on the Aggregate Root itself, or a dedicated Factory class.

```csharp
// Static factory method on the aggregate
public static Order Place(Guid customerId)
{
    return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
}

// Or a dedicated Factory for more elaborate creation
public sealed class OrderFactory
{
    private readonly IClock _clock;
    private readonly IIdGenerator _ids;

    public OrderFactory(IClock clock, IIdGenerator ids)
    {
        _clock = clock;
        _ids = ids;
    }

    public Order CreateFromCart(Customer customer, Cart cart, IReadOnlyList<Promotion> activePromotions)
    {
        if (cart.IsEmpty)
            throw new InvalidOperationException("Cannot place an order from an empty cart");

        var order = Order.Place(customer.Id);
        foreach (var item in cart.Items)
            order.AddLine(item.ProductId, item.Quantity, item.UnitPrice);

        foreach (var promotion in activePromotions.Where(p => p.AppliesTo(order)))
            promotion.Apply(order);

        return order;
    }
}
```

The point: **construction logic belongs in one place**, not scattered across application services.

---

## 15. Domain Events (Added Later)

Evans introduced Domain Events in a 2014 paper, not in the original Blue Book — but the pattern is now considered canonical DDD.

> A **Domain Event** is something that has happened in the domain that domain experts care about.

Past tense, immutable, raised by aggregates as side effects of business operations.

```csharp
public sealed record OrderSubmitted(Guid OrderId, Guid CustomerId, Money Total, DateTime OccurredAt);
public sealed record OrderPaid(Guid OrderId, string TransactionId, DateTime OccurredAt);
public sealed record OrderShipped(Guid OrderId, string TrackingNumber, DateTime OccurredAt);
```

### How aggregates raise them

```csharp
public abstract class Entity<TId>
{
    private readonly List<INotification> _events = new();
    public IReadOnlyList<INotification> DomainEvents => _events;
    protected void Raise(INotification @event) => _events.Add(@event);
    public void ClearDomainEvents() => _events.Clear();

    public TId Id { get; protected set; } = default!;
}

public sealed class Order : Entity<Guid>
{
    // ... as above ...

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Cannot submit empty order");
        Status = OrderStatus.Submitted;
        Raise(new OrderSubmitted(Id, CustomerId, Total, DateTime.UtcNow));
    }
}
```

### Two types — domain events vs. integration events

| Type                 | Scope                          | Subscribers                            |
| -------------------- | ------------------------------ | -------------------------------------- |
| **Domain Event**     | In-process, within one bounded context | Other aggregates / read models in this context |
| **Integration Event** | Cross-service, across bounded contexts | Other services via message bus         |

Domain events are dispatched after the transaction commits. Integration events are published via the Outbox pattern to a message broker.

> Cross-reference: [`../dot-net/docs/architectural-pattern/outbox.md`](../dot-net/docs/architectural-pattern/outbox.md), [`../dot-net/docs/architectural-style/event-driven.md`](../dot-net/docs/architectural-style/event-driven.md).

---

## 16. Part III — Refactoring Toward Deeper Insight

Evans' thesis: **the first model you write is wrong**. You discover the right model only by working with the domain for months. Refactoring isn't optional — it's the *primary mechanism* by which the model improves.

### The breakthrough

Sometimes a small change reveals a much deeper insight. A concept the team had been working around (always passing the same parameter, always doing the same special case) suddenly becomes its own first-class concept, and a swathe of code simplifies. Evans calls these moments **breakthroughs**.

### Making implicit concepts explicit

The single biggest source of breakthroughs: **a concept that the domain experts mention but the code doesn't name**. Example: every order has a "shipping mode" that affects pricing, but the code threads it through as a string in twelve places. Make `ShippingMode` an explicit Value Object, and the design becomes manifest.

### Watch for the smells

Evans is specific about signals to refactor:

* **Awkward names** — methods called `process(...)` that do four things.
* **Parameter lists** with the same trio of arguments everywhere — they're a missing Value Object.
* **Conditionals based on type** — likely a missing polymorphism.
* **"Comments explaining what the code does"** — the language is wrong; refactor names so the comments aren't needed.

---

## 17. Supple Design

Chapter 10 — the densest chapter in the book, and Evans' aesthetic statement about *what good domain code looks like*.

### The principles

| Principle                                | Definition                                                                                                  |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Intention-Revealing Interfaces**       | Names communicate *what* and *why*, not *how*. `customer.MarkAsPreferred()` not `customer.UpdateStatus(3)`. |
| **Side-Effect-Free Functions**           | Queries don't mutate; commands mutate but have clear semantics. Most operations should be either or both — never sneaky. |
| **Assertions**                           | Invariants of operations are explicit. Pre- and post-conditions stated in code (`Guard.Against.Null(...)`) or in tests. |
| **Conceptual Contours**                  | The boundaries between classes match the boundaries between concepts in the domain. Resist splitting/merging concepts to fit technical constraints. |
| **Standalone Classes**                   | Reduce dependencies. A class with fewer collaborators is easier to reason about.                            |
| **Closure of Operations**                | When an operation on a type returns the same type (`Money + Money = Money`), composition becomes easy.       |
| **Declarative Design**                   | When the structure of the domain admits it, write code that *describes* the domain rules — specifications, policy objects — rather than imperative algorithms. |

Supple Design is what *expert* DDD code feels like — small, named, composable, expressive. The earlier patterns make Supple Design *possible*; this chapter is what to *aim for*.

---

## 18. Part IV — Strategic Design

The most overlooked but arguably most-valuable section of the book. It pays off even if you adopt zero tactical patterns.

### Why strategic design matters

The tactical patterns work inside *one* model. The moment you have **two teams**, **multiple services**, or **legacy systems to integrate with**, you have multiple models — and they will conflict.

Strategic design provides:

* **Bounded Context** — explicit boundaries for one model.
* **Context Map** — the diagram of how all the contexts relate.
* **Distillation** — focus the team's modeling effort on what matters most.
* **Large-Scale Structure** — patterns to keep the whole system coherent.

> Cross-reference: [`../dot-net/docs/ddd/strategic-patterns.md`](../dot-net/docs/ddd/strategic-patterns.md) for full .NET treatment.

---

## 19. Bounded Context

> A **Bounded Context** is the explicit boundary within which a particular model is valid and unambiguous.

Inside the boundary, terms have one meaning. Outside it, the same word may mean something different.

### Why this is the most important strategic pattern

In a large company, the word "Customer" appears in dozens of places:

* In **Sales**, a Customer has a `LeadScore` and `LastQuotedAt`.
* In **Billing**, a Customer has a `PaymentTerms` and `OutstandingBalance`.
* In **Support**, a Customer has `OpenTickets` and `SatisfactionScore`.
* In **Shipping**, a Customer is a delivery address.

A single "company-wide Customer" model trying to serve all of them ends up serving none of them well. **Each context owns its own Customer model.** That is correct, not a duplication.

### Each Bounded Context is

* A unit of **model integrity** — the model is consistent inside.
* A unit of **team ownership** — typically one team per context.
* A unit of **deployment** — often one microservice per context.
* A unit of **Ubiquitous Language** — terms are unambiguous inside.

### Identifying Bounded Contexts

Evans gives heuristics; later authors (Vernon, the DDD community) added more:

* **Linguistic boundaries** — where does a word change meaning?
* **Team boundaries** — which group owns this?
* **Business capability boundaries** — Sales, Billing, Shipping…
* **Subdomain boundaries** — Core, Supporting, Generic (next section).

**EventStorming** (Alberto Brandolini, ~2013) is the most popular post-Evans technique for discovering Bounded Contexts collaboratively. The Blue Book itself doesn't cover it, but every modern DDD practitioner uses it.

---

## 20. Context Mapping — The Relationship Patterns

A **Context Map** is the diagram of how Bounded Contexts relate. Evans defines several relationship types, each with implications for team coordination and integration design.

### The nine canonical relationships

| Pattern                       | Meaning                                                                                                  |
| ----------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Partnership**               | Two teams succeed or fail together; close cooperation; shared model parts.                                |
| **Shared Kernel**             | A small portion of the model is shared explicitly between two contexts. High coordination cost.          |
| **Customer / Supplier**       | Upstream provides; downstream consumes; downstream has a voice in upstream's priorities.                  |
| **Conformist**                | Downstream conforms to upstream's model as-is. No translation. Cheap, but couples downstream to upstream. |
| **Anticorruption Layer (ACL)** | Downstream translates upstream's model into its own. Protects the downstream model from the upstream's mess. |
| **Open Host Service**         | Upstream provides a well-documented protocol/API for many downstream consumers.                          |
| **Published Language**         | A well-defined shared language (often an event schema) used as the lingua franca between contexts.        |
| **Separate Ways**             | Two contexts don't integrate at all. Sometimes the right call.                                            |
| **Big Ball of Mud**           | A context with no clear model. Wrap it with an ACL when you must integrate.                              |

### Anti-Corruption Layer in code

The pattern you reach for when you must integrate with a legacy system whose model is *different* from yours.

```csharp
// Your clean domain model
public sealed record Address(string Street, string City, string PostalCode, string Country);

// Legacy SOAP API exposes this:
public class LegacyAddressDto
{
    public string? AddressLine1 { get; set; }
    public string? AddressLine2 { get; set; }   // sometimes contains city+postal mashed
    public string? CityField { get; set; }
    public string? ZipPostCode { get; set; }
    public int CountryCode { get; set; }        // numeric ISO code
}

// ACL — translates between the two worlds
public interface IAddressProvider
{
    Task<Address> GetAsync(Guid customerId, CancellationToken ct);
}

public sealed class LegacyAddressAcl : IAddressProvider
{
    private readonly LegacyCustomerClient _legacy;
    private readonly ICountryCodeMapper _countries;

    public LegacyAddressAcl(LegacyCustomerClient legacy, ICountryCodeMapper countries)
    {
        _legacy = legacy;
        _countries = countries;
    }

    public async Task<Address> GetAsync(Guid customerId, CancellationToken ct)
    {
        var dto = await _legacy.GetCustomerAddressAsync(customerId.ToString(), ct);

        var street = string.Join(", ",
            new[] { dto.AddressLine1, dto.AddressLine2 }
                .Where(s => !string.IsNullOrWhiteSpace(s)));

        return new Address(
            Street: street,
            City: dto.CityField ?? "",
            PostalCode: dto.ZipPostCode ?? "",
            Country: _countries.IsoCodeFromLegacy(dto.CountryCode));
    }
}
```

The rest of the codebase uses `Address` cleanly. The mess lives **only** in `LegacyAddressAcl`. When the legacy system changes, only this class changes.

---

## 21. Distillation — Core, Supporting, Generic Subdomains

Once you have many Bounded Contexts, where do you spend your team's expensive modeling effort? Evans answers with **subdomain classification**:

| Subdomain                | What it is                                                            | Modeling effort |
| ------------------------ | --------------------------------------------------------------------- | --------------- |
| **Core Domain**          | The part of the business that makes you different from competitors    | **Maximum** — invest heavily |
| **Supporting Subdomain** | Important to the business but not differentiating                     | Moderate — build appropriately |
| **Generic Subdomain**    | Solved problem; off-the-shelf is fine                                  | Minimal — buy, integrate via ACL |

Examples for an e-commerce company:

* **Core**: pricing, recommendation, fraud detection. The bits that make them better than competitors.
* **Supporting**: order management, shipping coordination. Custom but not differentiating.
* **Generic**: identity, billing, payments. Use off-the-shelf (Auth0, Stripe).

### Why distillation matters

A finite team has finite modeling capacity. Putting your best modelers on a **Generic Subdomain** is a tragic waste. Reserve them for the **Core Domain** — the part where deep modeling produces business advantage.

### The Domain Vision Statement

Evans recommends writing a **short statement** (a paragraph or two) that articulates the Core Domain's purpose. Reread it when in doubt. It keeps the team's attention on what matters.

---

## 22. Large-Scale Structure

The final strategic chapter. When the system is huge, individual Bounded Contexts are not enough — you need patterns to **organize many contexts coherently**:

* **System Metaphor** — a simple analogy that helps everyone understand the system. (XP's idea; Evans endorses.)
* **Responsibility Layers** — partitioning by horizontal responsibility within or across contexts.
* **Knowledge Level** — separating instance behavior from type behavior.
* **Pluggable Component Framework** — when many contexts plug into a common platform.

These are the most-aspirational and least-applied patterns in the book. Most modern systems use simpler organizing principles (microservices + bounded contexts + a context map).

---

## 23. Big-Picture Take-Aways

1. **DDD is a methodology, not a pattern.** It is a way of working that places the **domain model** at the center of design.
2. **Ubiquitous Language is the most important idea in the book.** Adopt it even if you adopt nothing else.
3. **Aggregates bound transactional consistency.** Modify one aggregate per transaction; use Domain Events between them.
4. **Value Objects are the cure for primitive obsession.** Prefer them; use Entities only when identity is needed.
5. **Bounded Contexts give each model integrity.** One model per company is a fantasy that destroys clarity.
6. **The Context Map is the most-valuable single diagram in a large system.** Draw it; label every relationship; revisit it.
7. **Distill the Core Domain.** Invest modeling effort where the business differentiates itself.
8. **Don't apply DDD to a CRUD app.** The book is explicit: choose Smart UI knowingly when the domain is shallow.
9. **Refactoring is the engine.** The first model is wrong; the right model emerges through cycles of deepening insight.
10. **Pair DDD with Clean / Hexagonal Architecture.** They are made for each other; Evans was an early influence on both.

---

## 24. What the Book Doesn't Cover (And Vernon's Companion)

The Blue Book is 2003. Several now-canonical DDD ideas were added later:

* **Domain Events** — Evans, 2014 paper.
* **EventStorming** — Alberto Brandolini, ~2013.
* **Bounded Context = microservice** — Sam Newman, 2015 (*Building Microservices*).
* **Event Sourcing / CQRS pairing with DDD** — Greg Young, ~2010.
* **Practical implementation patterns** — Vaughn Vernon, *Implementing Domain-Driven Design* ("Red Book"), 2013.
* **DDD-Lite vs. Full DDD** — community evolution.

> **The standard recommendation:** read the Blue Book for *strategy* and *philosophy*; read Vernon's *Implementing DDD* for *tactics* and *implementation*. Together they give you the full picture.

---

## 25. How This Book Fits the Repo

| Idea in the book                            | Repo doc                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Ubiquitous Language & Bounded Context       | [`../dot-net/docs/ddd/strategic-patterns.md`](../dot-net/docs/ddd/strategic-patterns.md)              |
| Entity / Value Object / Aggregate / Service / Factory / Repository | [`../dot-net/docs/ddd/tactical-patterns.md`](../dot-net/docs/ddd/tactical-patterns.md) |
| Repository (and why Generic Repo is anti-DDD) | [`../dot-net/docs/enterprise-pattern/repository.md`](../dot-net/docs/enterprise-pattern/repository.md) |
| Layered Architecture & Clean Architecture   | [`clean-architecture-robert-martin.md`](./clean-architecture-robert-martin.md), [`../dot-net/docs/architectural-style/clean-architecture.md`](../dot-net/docs/architectural-style/clean-architecture.md) |
| Domain Events & Integration Events          | [`../dot-net/docs/architectural-style/event-driven.md`](../dot-net/docs/architectural-style/event-driven.md), [`../dot-net/docs/architectural-pattern/outbox.md`](../dot-net/docs/architectural-pattern/outbox.md) |
| Bounded Context = microservice               | [`building-microservices-sam-newman.md`](./building-microservices-sam-newman.md)                       |
| Sagas (cross-aggregate workflows)            | [`../dot-net/docs/architectural-pattern/saga.md`](../dot-net/docs/architectural-pattern/saga.md)      |

---

## 26. Reading Order Recommendations

The Blue Book is famously hard to read sequentially. A common modern path:

1. **Part I (Putting the Domain Model to Work)** — short, sets the philosophy.
2. **Chapter 4 (Layered Architecture)** — quick.
3. **Part IV first, before Part II** (controversial; Vernon recommends it). Strategic context makes tactical patterns make more sense.
4. **Part II (Tactical Patterns)** — Entity, Value Object, Aggregate, Repository, Factory, Service.
5. **Chapter 10 (Supple Design)** — read after you've practiced the tactical patterns; it makes much more sense then.
6. **Part III (Refactoring Toward Deeper Insight)** — short and rewards rereading once you're using DDD.

Companion books to read in parallel:
* Vaughn Vernon — *Implementing Domain-Driven Design* (the practical implementation).
* Vlad Khononov — *Learning Domain-Driven Design* (2021, modern, accessible).

---

## 27. Common Misreadings

| Misreading                                                          | Reality                                                                                                  |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| "DDD means Entity / Value Object / Aggregate / Repository."         | Those are *tactical patterns* — the easy half. The strategic half (Bounded Context, Context Map) is more important. |
| "Every project should use DDD."                                     | No. Evans is explicit: small CRUD apps don't need it. Apply DDD to **complex** domains.                  |
| "DDD requires microservices."                                       | No. DDD predates microservices by a decade. A modular monolith with Bounded Contexts is excellent DDD.    |
| "Aggregates can reference each other directly."                     | No. Reference other aggregates only by **ID**. Use Domain Events between aggregates.                     |
| "Generic Repository is fine in DDD."                                | Widely considered an anti-pattern. Use specific repositories per aggregate root, with method names in the Ubiquitous Language. |
| "Bounded Contexts are about size."                                  | They're about *model integrity*, not size. Some contexts are big, some small.                            |
| "DDD = Clean Architecture."                                         | They pair well, but solve different problems. DDD is about modeling the domain; Clean is about layering the system. |
| "Anti-Corruption Layer is just Adapter."                            | Structurally similar, conceptually different. ACL specifically protects model integrity across context boundaries. |

---

## 28. References

* Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software*, Addison-Wesley, 2003.
* Eric Evans — *Domain-Driven Design Reference* (free PDF summary by Evans himself, 2015). [domainlanguage.com](https://www.domainlanguage.com/ddd/reference/).
* Vaughn Vernon — *Implementing Domain-Driven Design*, Addison-Wesley, 2013 (the "Red Book").
* Vaughn Vernon — *Domain-Driven Design Distilled*, Addison-Wesley, 2016 (short version).
* Vlad Khononov — *Learning Domain-Driven Design*, O'Reilly, 2021 (modern, accessible).
* Alberto Brandolini — [*Introducing EventStorming*](https://www.eventstorming.com/) (workshop format for discovering Bounded Contexts).
* Martin Fowler — [*Bounded Context*](https://martinfowler.com/bliki/BoundedContext.html), [*Ubiquitous Language*](https://martinfowler.com/bliki/UbiquitousLanguage.html).
* DDD Crew — [github.com/ddd-crew](https://github.com/ddd-crew) (community templates for Context Maps, Bounded Context Canvas, EventStorming).
* Cross-reference in this repo: [`../dot-net/docs/ddd/`](../dot-net/docs/ddd/), [`../dot-net/docs/architectural-pattern/`](../dot-net/docs/architectural-pattern/), [`../dot-net/docs/enterprise-pattern/repository.md`](../dot-net/docs/enterprise-pattern/repository.md).
