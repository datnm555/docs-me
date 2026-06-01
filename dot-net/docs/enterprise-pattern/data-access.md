# Data-Access Patterns — Repository, UoW, Lazy & Eager Loading

> The patterns that decide how your domain talks to a database. Most come from **Martin Fowler's PoEAA (2002)** and remain the vocabulary every working developer uses with ORMs like **EF Core**, **Hibernate**, **Dapper**, **Mongoose**, etc.

---

## Table of Contents

1. [Repository](#1-repository)
2. [Unit of Work](#2-unit-of-work)
3. [Lazy Loading](#3-lazy-loading)
4. [Eager Loading](#4-eager-loading)
5. [Explicit Loading](#5-explicit-loading)
6. [Identity Map](#6-identity-map)
7. [Active Record vs Data Mapper](#7-active-record-vs-data-mapper)
8. [Query Object & Specification](#8-query-object--specification)
9. [Choosing a Strategy](#9-choosing-a-strategy)

---

## Quick Reference (What · Why · When · Where)

- **What** — The data-access patterns from PoEAA: **Repository**, **Unit of Work**, **Lazy / Eager / Explicit Loading**, **Identity Map**, **Active Record vs. Data Mapper**, **Query Object** and **Specification**.
- **Why** — They give you a precise vocabulary for talking about how domain code reaches the database. Most production bugs in data code come from mixing strategies (lazy-load when you needed eager, generic repo when you needed specific, Active Record when you needed Data Mapper).
- **When** — Whenever you're choosing how to load related data, structure a query, or wrap an ORM. The choice is often made implicitly — making it explicit is the first step toward fixing N+1 queries and Cartesian explosions.
- **Where** — **Infrastructure** layer (the implementations) and the **Application/Domain** boundary (the interfaces). EF Core already implements many of these (Identity Map, Lazy Load via proxies, Unit of Work via `DbContext`).

---

## 1. Repository

> *"A repository mediates between the domain and the data-mapping layers, **acting like an in-memory collection** of domain objects."* — Eric Evans, *DDD*

### 1.1 The Contract

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task<IReadOnlyList<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct);
}
```

* Domain talks **only** to this interface — never to SQL, never to a `DbContext`.
* Methods are **intentful** (`FindByCustomerAsync`), not generic (`Where(predicate)`).
* Returns **domain objects**, not DTOs or `IQueryable`.

### 1.2 EF Core Implementation

```csharp
public class EfOrderRepository(ShopDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(Guid id, CancellationToken ct)
        => db.Orders
             .Include(o => o.Items)
             .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct)
    {
        await db.Orders.AddAsync(order, ct);
        // SaveChanges is the Unit of Work's job
    }

    public Task<IReadOnlyList<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct)
        => db.Orders
             .Where(o => o.CustomerId == customerId)
             .ToListAsync(ct)
             .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);
}
```

### 1.3 The "Generic Repository" Debate

```csharp
public interface IRepository<T> where T : IEntity
{
    Task<T?> GetAsync(Guid id);
    IQueryable<T> Query();        // ← uh oh
    Task AddAsync(T entity);
    Task DeleteAsync(T entity);
}
```

A generic `IRepository<T>` over EF Core is **often pointless** because:

* `DbSet<T>` is already a repository.
* Exposing `IQueryable<T>` leaks persistence concerns into the caller.
* Every "find" method ends up on the same interface, violating ISP.

> **Rule of thumb:** write a **specific** repository per aggregate when you want a stable domain-side contract. Skip the abstraction and use `DbSet<T>` directly if your service is going to talk to EF anyway.

### 1.4 When Repository Earns Its Keep

* You want to **test domain logic** without spinning up a database.
* You want to **swap data stores** (EF → Dapper → in-memory) without rewriting callers.
* You're following **DDD aggregates** and want a strict boundary.

---

## 2. Unit of Work

> *"Maintains a list of objects affected by a business transaction and coordinates writing changes and the resolution of concurrency problems."* — Fowler

In short: **track changes; commit them as one atomic operation.**

### 2.1 The Pattern

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}
```

### 2.2 Where It Lives in EF Core

`DbContext` **IS** a Unit of Work. You don't need to write your own:

```csharp
public class CheckoutUseCase(IOrderRepository orders, ShopDbContext uow)
{
    public async Task PlaceAsync(Order o, CancellationToken ct)
    {
        await orders.AddAsync(o, ct);
        // ... more changes ...
        await uow.SaveChangesAsync(ct);    // one transaction
    }
}
```

### 2.3 When to Wrap It

Wrap `SaveChangesAsync` behind `IUnitOfWork` only when:

* You don't want domain / application code to know about `DbContext`.
* You want to swap data stores (rare in practice).
* You're enforcing a **per-request** transaction boundary in middleware.

Otherwise, depending on `DbContext` directly is fine — and EF Core ships first-class support for it.

### 2.4 UoW + Repository Together

```
Use case ──► Repository  (add / remove / find)
   │
   └────────► Unit of Work (SaveChangesAsync at the end)
```

The repository adds **what** changed. The UoW decides **when** to flush.

---

## 3. Lazy Loading

> Defer loading related data until it's actually accessed.

### 3.1 The Idea

```csharp
// Order entity with a lazy navigation property
public class Order
{
    public Guid Id { get; set; }
    public virtual ICollection<LineItem> Items { get; set; } = new List<LineItem>();
    //     ^^^^^^^ — EF proxies override this to load on first access
}

var order = db.Orders.Find(id);             // SQL: SELECT * FROM Orders WHERE Id = ...
var first = order.Items.First();            // SQL: SELECT * FROM LineItems WHERE OrderId = ...
//          └─ this access triggers the second query
```

### 3.2 How EF Core Enables It

```csharp
services.AddDbContext<ShopDbContext>(o => o
    .UseLazyLoadingProxies()
    .UseSqlServer(connectionString));
```

Navigation properties must be `virtual` so EF can override them with a proxy.

### 3.3 Pros & Cons

| Pros                                    | Cons                                                |
| --------------------------------------- | --------------------------------------------------- |
| Smaller initial query                   | Hidden queries — surprising at runtime              |
| Convenient — just access the property   | **N+1 problem** (see below)                         |
| Works for code that doesn't know it'll need related data | Requires open DbContext when accessed   |
|                                         | Untestable behavior without a real DB               |

### 3.4 The N+1 Problem

The most famous performance footgun in ORM history:

```csharp
var orders = db.Orders.ToList();             // 1 query
foreach (var o in orders)
    Console.WriteLine(o.Customer.Name);      // N queries — one per order
```

`1 + N` queries instead of one — death by a thousand small queries. Lazy loading makes this **invisible** until production.

> **Strong recommendation:** turn lazy loading **off** in services that handle external requests. Use **eager** or **explicit** loading.

---

## 4. Eager Loading

> Load related data **upfront**, in the same query, before anyone needs it.

### 4.1 EF Core — `Include` / `ThenInclude`

```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .Where(o => o.PlacedAt > since)
    .ToListAsync(ct);
```

* **Predictable** — one SQL statement (or a small, fixed number).
* **Visible** — the developer chose what to load.
* **Tunable** — performance can be reasoned about.

### 4.2 When to Use

* The caller **always** needs the related data.
* You're rendering a screen / API response with a known shape.
* The relationship cardinality is small (avoid eagerly loading 10k child rows you don't need).

### 4.3 Pros & Cons

| Pros                              | Cons                                            |
| --------------------------------- | ----------------------------------------------- |
| No N+1                            | Bigger initial payload                          |
| Predictable performance           | Wasted data if caller doesn't need it           |
| Explicit, easy to audit           | Verbose for deep graphs                         |

### 4.4 Cartesian Explosion Warning

`Include` translates to SQL `JOIN`s. Two unrelated `Include` collections can multiply rows:

```csharp
db.Orders
  .Include(o => o.Items)     // 1 order × 10 items
  .Include(o => o.Comments)  // 1 order × 5 comments
  // SQL row count: 1 × 10 × 5 = 50 rows for one order
```

EF Core 5+ defaults to **split queries** in some cases, and you can force it:

```csharp
db.Orders.AsSplitQuery().Include(...).Include(...);
```

---

## 5. Explicit Loading

> The caller decides exactly when to load related data — neither lazy nor eager.

### 5.1 EF Core API

```csharp
var order = await db.Orders.FindAsync(id);

// later, if needed:
await db.Entry(order).Collection(o => o.Items).LoadAsync();
await db.Entry(order).Reference(o => o.Customer).LoadAsync();
```

### 5.2 When to Use

* Conditional loading: *"if X then also fetch Y"*.
* Streaming / paginated access where you control batching.
* Large graphs you want to materialize **piecewise**.

This is the middle ground — none of lazy's magic, but more flexibility than always-eager.

---

## 6. Identity Map

> *"Ensures that each object gets loaded only once by keeping every loaded object in a map. Looks up objects using the map when referring to them."* — Fowler

A `DbContext` in EF Core (and a `Session` in Hibernate/NHibernate) is an Identity Map.

```csharp
var a = db.Orders.Find(id);
var b = db.Orders.Find(id);     // returns the SAME instance — no second query
Console.WriteLine(ReferenceEquals(a, b));    // True
```

### Why It Matters

* **Consistency** — within a unit of work, all references to "the customer with id X" point to the same object.
* **Change tracking** — modifications to one reference are seen by everyone.
* **Performance** — duplicate lookups are free.

Identity Map is **bound to the unit of work** (per request, per session). Outside that boundary, two queries can return two different in-memory objects representing the same row.

---

## 7. Active Record vs Data Mapper

Two opposing approaches for connecting domain objects to a database.

### 7.1 Active Record

> The entity *knows how to persist itself*.

```csharp
public class Order : ActiveRecord
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }

    public void Save()    { /* INSERT/UPDATE this row */ }
    public void Delete()  { /* DELETE this row */ }

    public static Order Find(Guid id) { /* SELECT ... */ }
}
```

**Examples:** Rails ActiveRecord, Laravel Eloquent, Django ORM.

| Pros                            | Cons                                          |
| ------------------------------- | --------------------------------------------- |
| Tiny boilerplate                | Couples domain to persistence                 |
| Fast to start                   | Hard to unit-test pure domain logic           |
| Great for CRUD apps             | Hard to swap data store                       |
|                                 | Bloats the entity as rules grow               |

### 7.2 Data Mapper

> The entity is **persistence-ignorant**; a separate **mapper** moves data between objects and the database.

```csharp
public class Order   // pure domain, knows nothing about SQL
{
    public Guid Id { get; }
    public Money Total { get; private set; }
    public void MarkPaid() { ... }
}

public class OrderMapper   // does all the persistence
{
    public Order Load(Guid id) { ... }
    public void  Save(Order o) { ... }
}
```

**Examples:** EF Core (with proper aggregate design), Hibernate, Doctrine.

| Pros                                   | Cons                                |
| -------------------------------------- | ----------------------------------- |
| Clean separation of concerns           | More plumbing                       |
| Domain is easily unit-tested           | ORM mapping config can grow complex |
| Data store swappable                   | Higher learning curve               |
| Plays nicely with DDD aggregates       |                                     |

### 7.3 Which to Pick?

* **Small / CRUD-heavy app, fast iteration**? Active Record is honest about your needs.
* **Rich domain, long-lived system, varied IO, testable core**? Data Mapper.
* **Hybrid?** Yes — EF Core can be either, depending on how strictly you keep persistence concerns out of your entities.

---

## 8. Query Object & Specification

When queries grow conditional, dynamic, or reused, a single ad-hoc LINQ expression becomes a maintenance burden.

### 8.1 Query Object

> Encapsulate a query as an object with an explicit API.

```csharp
public class FindHighValueOrdersQuery
{
    public DateTime Since { get; init; }
    public decimal  MinTotal { get; init; }
    public string?  Country { get; init; }

    public IQueryable<Order> Apply(IQueryable<Order> source)
    {
        var q = source.Where(o => o.PlacedAt >= Since && o.Total >= MinTotal);
        if (Country is not null) q = q.Where(o => o.Customer.Country == Country);
        return q;
    }
}
```

### 8.2 Specification

> Encapsulate a **predicate** as an object you can combine.

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T candidate);
    Expression<Func<T, bool>> ToExpression();
}

public class HighValueOrderSpec(decimal min) : ISpecification<Order>
{
    public Expression<Func<Order, bool>> ToExpression() => o => o.Total >= min;
    public bool IsSatisfiedBy(Order o) => o.Total >= min;
}

// Compose:
var spec = new HighValueOrderSpec(1000m).And(new InCountrySpec("US"));
var orders = db.Orders.Where(spec.ToExpression()).ToList();
```

Specifications are great when:

* The same predicate is used by both queries **and** in-memory validation.
* Business rules are explicit, named, testable objects.

(See [`other-enterprise.md`](./other-enterprise.md) for more on Specification.)

---

## 9. Choosing a Strategy

A practical decision tree for related-data loading:

```
                ┌─────────────────────────────────────────┐
                │ Will the caller always need this data?  │
                └──────────────┬──────────────────────────┘
                               │
                  ┌────────────┴────────────┐
                 Yes                       No
                  ▼                         ▼
            EAGER (Include)          ┌─────────────────────────┐
                                     │ Will it be needed       │
                                     │ sometimes / conditionally?│
                                     └──────────┬──────────────┘
                                                │
                                  ┌─────────────┴────────────┐
                                 Yes                        No
                                  ▼                          ▼
                        EXPLICIT (LoadAsync)            Don't load it.
                                                       (Just don't.)
```

> **Default:** disable lazy loading globally. Make every data fetch a **conscious decision**.

---

## Quick Reference

| Concept            | Owns…                        | Returns…                      |
| ------------------ | ---------------------------- | ----------------------------- |
| **Repository**     | A domain aggregate            | Domain objects                |
| **Unit of Work**   | Change tracking + transaction | Number of rows saved          |
| **Lazy Loading**   | Deferred fetch on access      | Hidden extra queries          |
| **Eager Loading**  | Upfront join                  | Predictable single query      |
| **Explicit Load**  | Caller-controlled fetch       | Exactly what was asked for    |
| **Identity Map**   | Per-context cache by id       | Same instance for same row    |
| **Active Record**  | Entity = persistence object   | CRUD on `this`                |
| **Data Mapper**    | Mapper class                  | Pure domain object            |
| **Query Object**   | A named query                 | Filtered results              |
| **Specification**  | A named predicate             | A composable rule             |

---

> **Next:** [`other-enterprise.md`](./other-enterprise.md) — Specification (deeper), CQRS, Mediator, Result, Options, Decorator pipeline.
