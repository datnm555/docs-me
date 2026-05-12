# Repository Pattern (and the Generic Repository Anti-Pattern)

> **Repository** is an **Enterprise Pattern** (Fowler PoEAA, 2002) and a **Tactical DDD pattern** (Evans, 2003). It mediates between the **domain** and **data mapping layers**, behaving like an **in-memory collection** of domain objects.

It is **NOT** a GoF design pattern.

---

## What Repository Is

A Repository:

1. Exposes the domain in **collection-like terms** (`Add`, `Get`, `Remove`).
2. **Hides** the data store (SQL, NoSQL, file, web API).
3. Is defined as an **interface in the domain** and **implemented in infrastructure**.
4. In DDD: **one repository per Aggregate Root** — not one per table.

```csharp
// In the domain — abstraction
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetUnpaidByCustomerAsync(Guid customerId, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// In the infrastructure — implementation
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        _db.Orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task<IReadOnlyList<Order>> GetUnpaidByCustomerAsync(Guid customerId, CancellationToken ct) =>
        _db.Orders.Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Placed)
                  .ToListAsync(ct)
                  .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);

    public Task AddAsync(Order order, CancellationToken ct)
    {
        _db.Orders.Add(order);
        return Task.CompletedTask; // commit via Unit of Work (SaveChangesAsync)
    }
}
```

Notice:

* The methods are **business-meaningful** — `GetUnpaidByCustomerAsync`, not `GetByPredicate(x => ...)`.
* The interface knows nothing about EF Core; the implementation knows everything.
* There's **no `Update` method** — EF Core tracks changes; you mutate the entity, then `SaveChangesAsync`.

This is the **proper Repository pattern** as defined by Fowler and Evans.

---

## The Generic Repository — and Why Many DDD Authors Call It an Anti-Pattern

A **Generic Repository** is the temptation:

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct);
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task AddAsync(T entity, CancellationToken ct);
    void Update(T entity);
    void Remove(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly AppDbContext _db;
    public Repository(AppDbContext db) => _db = db;
    public Task<T?> GetByIdAsync(object id, CancellationToken ct) =>
        _db.Set<T>().FindAsync([id], ct).AsTask();
    public Task<IEnumerable<T>> GetAllAsync(CancellationToken ct) =>
        _db.Set<T>().ToListAsync(ct).ContinueWith<IEnumerable<T>>(t => t.Result, ct);
    public Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct) =>
        _db.Set<T>().Where(predicate).ToListAsync(ct).ContinueWith<IEnumerable<T>>(t => t.Result, ct);
    public Task AddAsync(T entity, CancellationToken ct) { _db.Set<T>().Add(entity); return Task.CompletedTask; }
    public void Update(T entity) => _db.Set<T>().Update(entity);
    public void Remove(T entity) => _db.Set<T>().Remove(entity);
}

// Registered as: services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```

### Why this is widely considered an **anti-pattern** in DDD / Clean architectures

1. **It already exists.** EF Core's `DbContext` + `DbSet<T>` *is* a generic repository. Wrapping it in another generic repository **adds a layer with no behavior** — pure ceremony.

2. **It leaks persistence concerns into the domain.** The signature `Find(Expression<Func<T, bool>>)` means **the caller writes queries**. That's the opposite of "hide the data store." Every controller that calls `repo.Find(o => o.Status == OrderStatus.Placed)` is doing data access.

3. **It encourages anemic, intent-less interfaces.** The repository becomes a faceless CRUD hatch — no `GetUnpaidByCustomer`, no `GetOverdueInvoices`. You lose the *domain vocabulary* that's the whole point of a repository.

4. **It pretends to be testable but isn't.** People mock `IRepository<T>` in tests, but those mocks accept any `Expression<Func<T,bool>>` — they don't enforce that the expression actually returns the right thing. The tests pass; the SQL is wrong.

5. **It complicates real queries.** Joins, projections, paging, includes — none fit `IRepository<T>` cleanly. You end up exposing `IQueryable<T>` from the repository, which is just `_db.Set<T>()` again.

6. **It violates the "one repository per aggregate root" rule.** You get `IRepository<OrderItem>`, `IRepository<Address>`, etc. — but `OrderItem` is **not an aggregate root**; it should only be modified through `Order`. The generic repo lets clients bypass that rule.

### The famous quotes

> *"Don't use a Generic Repository. It is the wrong abstraction."* — **Tim McCarthy**, *.NET Domain-Driven Design with C#*.
>
> *"A repository should not be a thin wrapper around your data access technology. If it is, it has no reason to exist."* — **Vaughn Vernon**, *Implementing Domain-Driven Design*.
>
> *"The generic repository … doesn't actually abstract anything. It just adds a layer of indirection."* — **Jimmy Bogard** (creator of MediatR / AutoMapper).

---

## What Generic Repository Actually Is

It's **not** a named GoF, PoEAA, or DDD pattern. It is an **idiom / implementation technique** in C# using generics. Some call it:

* "Generic Repository (over an ORM)"
* "Repository over IQueryable"
* "Type-parameterized DAO"
* In Java land: similar to **Spring Data's `JpaRepository<T, ID>`** — but Spring Data *generates implementations* from method names, which is materially different.

**In short:** *Generic Repository* is the name people use, but it isn't in any canonical pattern catalog. Calling it a "design pattern" is generous.

---

## When IS a Generic Repository Acceptable?

You can defend it in **narrow** situations:

* **Simple CRUD admin apps** — you really do want `Add/Update/Remove/GetById` for every table and nothing more.
* **A reusable foundation** — you ship a library used by many tiny services, all CRUD. Generic repo saves boilerplate.
* **A typed wrapper over a non-LINQ store** (e.g. raw MongoDB or DynamoDB) where you want a uniform shape.

But in a serious DDD codebase: **don't**. Write specific repositories that speak the domain language.

---

## C# — A Good DDD-Style Repository

```csharp
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<Customer?> GetByEmailAsync(Email email, CancellationToken ct);
    Task<IReadOnlyList<Customer>> FindHighValueAsync(Money threshold, CancellationToken ct);
    Task AddAsync(Customer customer, CancellationToken ct);
}
```

Every method name is a **noun-verb from the business**, not "find with predicate." The interface defines the **questions the domain wants answered**, not how to answer them.

---

## Repository vs. Service vs. Query Object

| If you need…                                            | Use…                                       |
| ------------------------------------------------------- | ------------------------------------------ |
| Load / persist an **aggregate root**                    | **Repository** (one per aggregate)          |
| Run a complex **read** (joins, projections, paging)     | **Query object** (Dapper, raw SQL) — *not* a repo |
| Orchestrate domain + infrastructure for a use case      | **Application Service** / MediatR handler   |
| Encapsulate a **composable predicate**                  | **Specification pattern**                    |

CQRS makes this very clean: **commands use repositories**, **queries skip the domain entirely** (Dapper straight to DTO).

---

## Common Variations

* **Read-only repository** — only `GetById` / queries. For aggregates that don't change.
* **Specification + Repository** — a `GetBySpec(ISpecification<T>)` method that composes expressions. Cleaner than raw predicates if used carefully.
* **In-memory repository for tests** — implement the same interface backed by a `List<T>`.

---

## TL;DR

* **Repository** — yes, a real pattern (Fowler PoEAA + DDD tactical). Use it as a domain-language interface, one per aggregate root.
* **Generic Repository** — not a named pattern; in DDD circles widely considered an anti-pattern over EF Core because EF already does the job and the abstraction leaks data-access concerns into the domain.
* **What to call it instead** — if you want a name, "Generic Repository idiom" or "Repository over `IQueryable<T>`" is honest.

---

## References

* Eric Evans — *Domain-Driven Design* (2003). Chapter on Repositories.
* Martin Fowler — *PoEAA* (2002). [Repository entry](https://martinfowler.com/eaaCatalog/repository.html).
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Chapter 12.
* Jimmy Bogard — [*Repository is the new Singleton*](https://lostechies.com/jimmybogard/2009/05/15/repositories-and-the-laws-of-thermodynamics/).
* Mark Seemann — [*IQueryable is Tight Coupling*](https://blog.ploeh.dk/2012/03/26/IQueryableTightCouplingPart1/).
