# Unit of Work Pattern

> An **Enterprise Pattern** (Fowler PoEAA, 2002): *"Maintains a list of objects affected by a business transaction and coordinates the writing out of changes and the resolution of concurrency problems."*

In short: a Unit of Work **tracks what changed** during a business operation and **commits them all as one transaction** — atomically — at the end.

---

## The Problem It Solves

Without a Unit of Work, the application code is responsible for tracking every change and saving each one explicitly:

```csharp
// Without UoW — fragile
await _orderRepo.SaveAsync(order);
await _customerRepo.SaveAsync(customer);
await _inventoryRepo.SaveAsync(inventory);
// If the 3rd save fails, the first two are already committed. Inconsistency.
```

A Unit of Work changes the conversation:

```csharp
// With UoW
_orderRepo.Add(order);
customer.IncrementOrderCount();   // tracked automatically
inventory.Decrement(quantity);    // tracked automatically
await _uow.SaveChangesAsync(ct);  // one transaction, all-or-nothing
```

---

## EF Core's DbContext **is** a Unit of Work

This is the most important sentence in this file:

> **`DbContext` is the Unit of Work. `DbSet<T>` is the Repository.**

Microsoft's docs confirm this explicitly. EF Core implements the **Identity Map**, **Unit of Work**, and **(generic) Repository** patterns out of the box:

```csharp
public sealed class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
}

// This IS a Unit of Work
public async Task PlaceOrderAsync(...)
{
    var customer = await _db.Customers.FindAsync(customerId);
    var order = customer.PlaceOrder(items);

    _db.Orders.Add(order);
    customer.IncrementOrderCount();

    await _db.SaveChangesAsync(ct); // single transaction commits both changes
}
```

You **do not need** to wrap `DbContext` in your own `IUnitOfWork` interface to "follow the pattern". You're already following it.

---

## When People Wrap DbContext in an `IUnitOfWork`

You'll often see this in DDD-leaning codebases:

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

public sealed class AppDbContext : DbContext, IUnitOfWork { /* ... */ }
```

### Why people do this

* The **application layer** must not reference EF Core directly (Clean Architecture rule).
* So they expose `IUnitOfWork` from the domain/application layer, with EF Core hidden behind it in infrastructure.
* This is the **only legitimate reason** to introduce an explicit `IUnitOfWork`.

### When NOT to do this

* If your architecture already lets the application call `_db.SaveChangesAsync(ct)` directly (common in pragmatic monoliths), an extra `IUnitOfWork` interface adds nothing.
* If you're combining multiple `DbContext` instances under one `IUnitOfWork` — careful. EF Core's transaction scope is per `DbContext`. You'd need `TransactionScope` or `IDbContextTransaction`.

---

## A Domain-Friendly Unit of Work Interface

```csharp
// Domain / Application layer (no EF reference)
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// Infrastructure layer
public sealed class AppDbContext : DbContext, IUnitOfWork
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    // SaveChangesAsync is already on DbContext — interface fulfilled
}

// Wire-up
services.AddDbContext<AppDbContext>(opts => opts.UseNpgsql(connectionString));
services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
services.AddScoped<IOrderRepository, EfOrderRepository>();

// Application-layer use case
public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    ICustomerRepository customers,
    IUnitOfWork uow) : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await customers.GetByIdAsync(cmd.CustomerId, ct)
                       ?? throw new NotFoundException();
        var order = customer.PlaceOrder(cmd.Items);
        await orders.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct); // commits both the new order and customer mutations
        return order.Id;
    }
}
```

This keeps the use-case code **EF-free** while still using EF's built-in Unit of Work under the hood.

---

## Unit of Work vs. Database Transaction

These are **not** the same.

| Concept              | Scope                                            | Mechanism                                |
| -------------------- | ------------------------------------------------ | ---------------------------------------- |
| **Database Transaction** | A `BEGIN`/`COMMIT` in the DB                  | `BeginTransaction()` / SQL                |
| **Unit of Work**     | Application-layer change tracker                  | `DbContext` (tracks changes in memory)    |

`SaveChangesAsync` opens **one DB transaction**, applies all tracked changes, and commits. So in practice they coincide — but conceptually a Unit of Work could batch and retry; a transaction is just the durable persistence step.

For **multi-DbContext** or **multi-resource** scenarios:

```csharp
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await _ordersDb.SaveChangesAsync(ct);
await _inventoryDb.SaveChangesAsync(ct);
scope.Complete();
```

(Promotes to a distributed transaction — usually a sign you should be using a [Saga](../architectural-pattern/saga.md) instead.)

---

## Anti-Patterns

* **Custom `UnitOfWork` that just wraps `DbContext`** with `Begin`/`Commit`/`Rollback` methods. EF Core already gives you `SaveChanges` atomicity.
* **Repository methods that call `SaveChanges` internally.** This forces each write into its own transaction — breaking atomicity across multiple repositories. Commit only at the use-case level.
* **Long-lived UoW.** A UoW should match a single use case / HTTP request. EF Core's scoped `DbContext` enforces this naturally.

---

## TL;DR

* **Unit of Work** is a real Enterprise Pattern.
* **EF Core's `DbContext` is already a Unit of Work**.
* Wrap it in `IUnitOfWork` **only** to keep your application layer free of EF dependencies.
* Commit **once per use case**, never inside individual repositories.

---

## References

* Martin Fowler — *PoEAA* (2002). [Unit of Work entry](https://martinfowler.com/eaaCatalog/unitOfWork.html).
* Microsoft Docs — *Implement the infrastructure persistence layer with Entity Framework Core*. ([learn.microsoft.com](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core)).
