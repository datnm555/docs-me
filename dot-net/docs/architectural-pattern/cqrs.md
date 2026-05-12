# CQRS — Command Query Responsibility Segregation

> An **architectural pattern** that separates the model used to **change** data (commands) from the model used to **read** data (queries).

---

## The Core Idea

A single domain model that supports both writes and reads ends up serving neither well:

* **Writes** want a normalized, transactionally-consistent model with rich behavior.
* **Reads** want denormalized, pre-joined, view-optimized data for fast queries.

CQRS says: stop forcing one model to do both jobs.

```
                Client
                  │
       ┌──────────┴──────────┐
       │                     │
   Commands               Queries
       │                     │
       ▼                     ▼
 [Write Model]         [Read Model]
 Domain + rules        Flat DTOs
 (DDD Aggregates)      (denormalized views)
       │                     ▲
       └──── projections ────┘
```

---

## Levels of CQRS

CQRS is a **spectrum**, not a binary choice. Pick the level that matches your problem.

### Level 1 — In-Process CQRS (same database)

* Separate command and query **handlers** in code.
* Same database, but **commands use the domain model** (EF Core entities) and **queries use Dapper or projected DTOs** (no entity loading).
* This is what most "MediatR CQRS" .NET apps actually do.

### Level 2 — Read model in a separate store

* Writes go to PostgreSQL (consistent).
* Reads go to a denormalized view in Redis / Elastic / a separate Postgres schema, kept in sync via events.

### Level 3 — Fully separate write and read **services**

* Different services, different databases, different scaling strategies.
* Synced by event stream (often paired with **Event Sourcing**).
* High complexity — only justified for high-scale systems.

---

## C# — Level 1 CQRS with MediatR

### Command side — rich domain model

```csharp
public record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<LineItemDto> Items, string Currency)
    : IRequest<Guid>;

public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly AppDbContext _db;        // EF Core, full domain model
    private readonly IPaymentGateway _payments;

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await _db.Customers.FindAsync([cmd.CustomerId], ct)
                       ?? throw new NotFoundException();

        var order = customer.PlaceOrder(cmd.Items, cmd.Currency); // business rules live in the domain
        _db.Orders.Add(order);

        await _payments.ChargeAsync(order.Total, cmd.Currency, customer.Id.ToString(), ct);
        await _db.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

### Query side — flat DTO, raw SQL (Dapper) — no domain model

```csharp
public record GetOrderSummary(Guid OrderId) : IRequest<OrderSummaryDto>;

public record OrderSummaryDto(
    Guid OrderId,
    string CustomerName,
    string Status,
    decimal Total,
    int ItemCount,
    DateTime PlacedAt);

public sealed class GetOrderSummaryHandler : IRequestHandler<GetOrderSummary, OrderSummaryDto>
{
    private readonly IDbConnection _conn; // Dapper

    public async Task<OrderSummaryDto> Handle(GetOrderSummary q, CancellationToken ct)
    {
        const string sql = """
            SELECT  o.Id          AS OrderId,
                    c.Name        AS CustomerName,
                    o.Status,
                    o.Total,
                    (SELECT COUNT(*) FROM OrderItems oi WHERE oi.OrderId = o.Id) AS ItemCount,
                    o.PlacedAt
            FROM    Orders o
            JOIN    Customers c ON c.Id = o.CustomerId
            WHERE   o.Id = @id
            """;
        return await _conn.QuerySingleAsync<OrderSummaryDto>(sql, new { id = q.OrderId });
    }
}
```

The write side never sees Dapper; the read side never sees EF Core entities. Each side is **optimal for its job**.

---

## C# — Level 2 CQRS with a Projection

Write side commits to Postgres; a background projector listens to domain events and updates a Redis-backed read view.

```csharp
public sealed class OrderViewProjector : INotificationHandler<OrderPlaced>
{
    private readonly IConnectionMultiplexer _redis;

    public async Task Handle(OrderPlaced ev, CancellationToken ct)
    {
        var view = new OrderViewDto(ev.OrderId, ev.CustomerName, "Placed", ev.Total, ev.PlacedAt);
        var db = _redis.GetDatabase();
        await db.StringSetAsync($"order-view:{ev.OrderId}", JsonSerializer.Serialize(view), TimeSpan.FromHours(24));
    }
}

public sealed class GetOrderViewHandler : IRequestHandler<GetOrderView, OrderViewDto?>
{
    private readonly IConnectionMultiplexer _redis;

    public async Task<OrderViewDto?> Handle(GetOrderView q, CancellationToken ct)
    {
        var raw = await _redis.GetDatabase().StringGetAsync($"order-view:{q.OrderId}");
        return raw.IsNullOrEmpty ? null : JsonSerializer.Deserialize<OrderViewDto>(raw!);
    }
}
```

The read view is **eventually consistent** — a query right after a write might still show the old data for milliseconds. This trade-off is the cost of separate stores.

---

## When to Use CQRS

✅ **Use when:**

* Read and write workloads have **very different shapes** (complex queries vs simple writes, or vice versa).
* You want write logic in a rich domain model but read paths to stay fast and simple.
* Scaling reads and writes independently matters.

❌ **Don't use when:**

* CRUD app with simple read/write patterns. CQRS adds code with no payoff.
* Team is uncomfortable with eventual consistency.
* The "complexity budget" is already spent elsewhere.

**Greg Young (who coined CQRS):** *"CQRS is not a top-level architecture. It is a pattern applied to a Bounded Context."* Apply it where it pays; don't apply it system-wide by default.

---

## CQRS vs. CQS

* **CQS (Command-Query Separation)** — Bertrand Meyer's principle: a method either changes state OR returns data, never both. Class-level.
* **CQRS** — extends CQS to **separate models / classes / pipelines**.

CQS is always good. CQRS is sometimes good.

---

## CQRS and Event Sourcing

CQRS is **commonly paired with** but **independent from** [Event Sourcing](./event-sourcing.md):

| Combination               | Common?  | Notes                                                              |
| ------------------------- | -------- | ------------------------------------------------------------------ |
| CQRS only                 | Very     | The 80% case. Mediator + Dapper reads + EF writes.                 |
| Event Sourcing only       | Rare     | Possible but unusual; ES naturally drives a separate read model.    |
| **CQRS + Event Sourcing** | Common   | Events are the write model; projections are the read model.        |
| Neither                   | Default  | Most simple CRUD apps.                                              |

---

## References

* Greg Young — *CQRS Documents* (2010). [PDF](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
* Martin Fowler — [*CQRS*](https://martinfowler.com/bliki/CQRS.html).
* Microsoft — *Cloud Design Patterns: CQRS*.
