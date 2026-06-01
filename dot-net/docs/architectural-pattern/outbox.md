# Outbox / Inbox Patterns

> Two **architectural patterns** that together solve the **dual-write problem** in distributed systems: how to atomically (1) update a database AND (2) publish a message — when each can fail independently.

---

## Quick Reference (What · Why · When · Where)

- **What** — **Outbox**: write the message to an outbox table in the same DB transaction as the business data; a background process polls and publishes. **Inbox**: record processed message IDs to deduplicate redeliveries. Together they make publish + handle effectively exactly-once.
- **Why** — Without it, the dual-write problem ("save to DB AND publish to broker") can leave you with a saved row and no event, or an event and no row, in any failure mode.
- **When** — Any time a service emits an integration event from a business operation, especially in a Saga; mandatory if at-least-once delivery is your default (Kafka, RabbitMQ, Azure Service Bus).
- **Where** — Producer side: in the same `DbContext` as the aggregate. Consumer side: an inbox table in the consumer's DB, transactionally with side effects. Pair with sagas and domain events.

---

## The Dual-Write Problem

Naive code:

```csharp
await _db.SaveChangesAsync(ct);          // 1. commit to DB
await _bus.PublishAsync(orderCreated);    // 2. publish to broker
```

Failure modes:

| What fails               | Result                                         |
| ------------------------ | ---------------------------------------------- |
| Step 1 fails             | Nothing happens. Fine.                          |
| Step 1 succeeds, step 2 fails | DB updated, **no message published**. **Inconsistency.** |
| Both succeed             | Fine.                                          |

The reverse order is just as bad:

```csharp
await _bus.PublishAsync(orderCreated);    // 1. publish
await _db.SaveChangesAsync(ct);           // 2. commit
```

Now you can publish an event for an order that was **never saved**.

There is **no way to make these two writes atomic** without a distributed transaction (2PC), which you don't want.

---

## The Outbox Pattern

**Idea:** write the message to an **outbox table in the same database**, in the **same local transaction** as the business data. A separate process polls the outbox and publishes to the broker.

```
┌─────────────────────────────────────────┐
│      Service A's Database               │
│                                         │
│  [Orders]     +-->   [Outbox]            │
│   ↑                    │                 │
│   │ same TX            │                 │
│  Commit                ▼                 │
└────────────────┐    ┌──────────────────┐ │
                 │    │ Outbox Publisher │ │
                 │    │   (poller)       │ │
                 │    └────────┬─────────┘ │
                 │             │           │
                 └─────────────┼───────────┘
                               ▼
                         ┌──────────┐
                         │  Broker  │
                         └──────────┘
```

If the broker is down, the message stays in the outbox; the publisher retries until success. The business write is **not blocked** by broker availability.

### Outbox Schema

```sql
CREATE TABLE OutboxMessages (
    Id              UUID PRIMARY KEY,
    OccurredOnUtc   TIMESTAMP NOT NULL,
    Type            VARCHAR(255) NOT NULL,
    Content         JSONB NOT NULL,
    ProcessedOnUtc  TIMESTAMP NULL,
    Error           TEXT NULL
);

CREATE INDEX ix_outbox_unprocessed
    ON OutboxMessages (OccurredOnUtc)
    WHERE ProcessedOnUtc IS NULL;
```

### C# — Saving to Outbox in the Same Transaction

```csharp
public sealed class OutboxMessage
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime OccurredOnUtc { get; init; } = DateTime.UtcNow;
    public string Type { get; init; } = "";
    public string Content { get; init; } = "";
    public DateTime? ProcessedOnUtc { get; set; }
    public string? Error { get; set; }
}

public sealed class OrderService
{
    private readonly AppDbContext _db;

    public async Task PlaceAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.New(cmd);
        _db.Orders.Add(order);

        // Write the integration event INTO THE SAME DbContext — same transaction
        _db.Outbox.Add(new OutboxMessage
        {
            Type    = nameof(OrderPlaced),
            Content = JsonSerializer.Serialize(new OrderPlaced(order.Id, order.Total)),
        });

        await _db.SaveChangesAsync(ct); // atomic: order + outbox row commit together
    }
}
```

### C# — The Publisher (background worker)

```csharp
public sealed class OutboxPublisher : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IEventBus _bus;
    private readonly ILogger<OutboxPublisher> _log;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var scope = _scopeFactory.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            var batch = await db.Outbox
                .Where(m => m.ProcessedOnUtc == null)
                .OrderBy(m => m.OccurredOnUtc)
                .Take(100)
                .ToListAsync(stoppingToken);

            foreach (var msg in batch)
            {
                try
                {
                    await _bus.PublishAsync(msg.Type, msg.Content, stoppingToken);
                    msg.ProcessedOnUtc = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    msg.Error = ex.Message;
                    _log.LogError(ex, "Outbox publish failed for {Id}", msg.Id);
                }
            }

            await db.SaveChangesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}
```

For higher throughput, replace polling with **Change Data Capture (CDC)** — e.g. Debezium reading the WAL/binlog. MassTransit and Wolverine both ship production-grade Outbox implementations.

---

## The Inbox Pattern (de-duplication on the consumer side)

The broker delivers **at-least-once**. A network blip causes the same message to be redelivered. Without protection, the consumer processes it twice → duplicate orders, double refunds.

**Idea:** the consumer maintains an **Inbox table** of message IDs it has already handled. Before processing, check the inbox; after processing, insert the ID — both in the same transaction as the side effects.

### Inbox Schema

```sql
CREATE TABLE InboxMessages (
    Id              UUID PRIMARY KEY,   -- the broker's message id
    HandledOnUtc    TIMESTAMP NOT NULL
);
```

### C# — Idempotent Consumer Using Inbox

```csharp
public sealed class OrderPlacedHandler
{
    private readonly AppDbContext _db;
    private readonly IInventoryService _inventory;

    public async Task HandleAsync(string messageId, OrderPlaced ev, CancellationToken ct)
    {
        await using var tx = await _db.Database.BeginTransactionAsync(ct);

        // 1. Have we handled this before?
        var alreadyHandled = await _db.Inbox
            .AsNoTracking()
            .AnyAsync(i => i.Id == Guid.Parse(messageId), ct);
        if (alreadyHandled) return;

        // 2. Do the work
        await _inventory.ReserveAsync(ev.OrderId, ct);

        // 3. Record that we handled it — same TX as the side effect
        _db.Inbox.Add(new InboxMessage { Id = Guid.Parse(messageId), HandledOnUtc = DateTime.UtcNow });

        await _db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
}
```

---

## Outbox + Inbox Together

```
Producer Service                                  Consumer Service
┌─────────────────────┐                          ┌──────────────────────┐
│ Business TX:        │                          │ Business TX:         │
│   • Save order      │                          │   • Reserve stock     │
│   • Save outbox row │  →  Broker (at-least)  → │   • Insert inbox row  │
└─────────────────────┘                          └──────────────────────┘
       Reliable publish                                  Idempotent handle
```

This is the **gold standard** for reliable, exactly-once-effective messaging without distributed transactions. It's the foundation that makes [Sagas](./saga.md) safe.

---

## Tooling in .NET

* **[MassTransit](https://masstransit.io/)** — `EntityFrameworkOutbox` / `EntityFrameworkInbox` extensions.
* **[Wolverine](https://wolverine.netlify.app/)** — first-class transactional outbox built into the framework.
* **[NServiceBus](https://particular.net/nservicebus)** — `Outbox` feature, enterprise pricing.
* **Debezium + Kafka Connect** — CDC-based outbox for high throughput.
* **DotNetCore.CAP** — open-source library that does outbox/inbox.

---

## Common Pitfalls

* **Forgetting the outbox in tests** — the integration event is never published in test envs, hiding bugs.
* **Long outbox tail** — failing messages clog the table. Add a max-retry + DLQ pattern.
* **Out-of-order delivery** — outbox publishers parallelize; if order matters, use a single partition / single-threaded publisher per aggregate.
* **Schema migrations on the outbox** — once messages exist with old schemas, you can't break them. Version your event payloads.

---

## References

* Chris Richardson — *Microservices Patterns* (2018). Chapter 3: Inter-process communication; Chapter 4: Sagas.
* [microservices.io/patterns/data/transactional-outbox.html](https://microservices.io/patterns/data/transactional-outbox.html)
* Gunnar Morling — [*Reliable Microservices Data Exchange With the Outbox Pattern*](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/).
