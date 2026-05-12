# Event-Driven Architecture (EDA / EDD)

> An **architectural style** where components communicate primarily by **producing and consuming events**, rather than by direct request/response calls. The producer does not know who consumes (loose coupling). Consumers react asynchronously.

---

## Is EDA a Design Pattern?

**No.** Event-Driven Architecture is an **architectural style** — it's at the same level as Layered, Hexagonal, REST, or Microservices. It is **not** a GoF design pattern.

* **EDA** = Event-Driven Architecture (the system shape).
* **EDD** = Event-Driven Design / Development (the practice of designing with events).

The pattern building blocks **inside** EDA include: pub/sub, **Outbox**, **Inbox**, event sourcing, **Saga**, CQRS, message broker, projector. Each of *those* is a pattern.

---

## Core Idea

```
[Order Service]
       │
       │ publishes  OrderPlaced
       ▼
   ┌────────────┐
   │  Broker    │   (Kafka, RabbitMQ, Azure Service Bus, Redis Streams)
   └────────────┘
       │
       ├──▶  [Inventory Service]  → reserves stock
       ├──▶  [Email Service]      → sends confirmation
       ├──▶  [Analytics Service]  → records metric
       └──▶  [Fraud Service]      → scores risk
```

The Order Service has **no idea** who reacts to `OrderPlaced`. New subscribers can be added without touching it. This is the headline advantage of EDA: **decoupling in time and identity**.

---

## Three Types of Events

A frequent source of confusion. **Pick the right type for the right job.**

### 1. Domain Events — "something happened in *my* bounded context"

* In-process (or within one service).
* Carry rich domain meaning.
* Subscribers live in the **same service**.
* Often raised by aggregates as side effects of business operations.

```csharp
public sealed record OrderPlaced(Guid OrderId, decimal Total, DateTime OccurredAt);
```

Implemented in .NET with **MediatR `INotification`**, **in-process event bus**, or an aggregate's `_pendingEvents` list dispatched on save.

### 2. Integration Events — "something happened that *other services* need to know"

* Cross-service.
* Versioned. Public contracts.
* Travel through a **message broker**.
* Smaller, stable payloads.

```csharp
public sealed record OrderPlacedV1(Guid OrderId, Guid CustomerId, decimal Total, DateTime OccurredAt);
```

Implemented with **MassTransit / Wolverine / NServiceBus** over RabbitMQ / Kafka / Azure Service Bus.

### 3. Event-Carried State Transfer — "here's the latest state"

* The event payload contains **all data the consumer needs** so they don't have to call back.
* Trade-off: bigger events; risk of stale data.

```csharp
// Includes a full snapshot — consumer doesn't have to query Order service
public sealed record OrderStateChangedV1(
    Guid OrderId,
    OrderStatus Status,
    decimal Total,
    Guid CustomerId,
    DateTime OccurredAt);
```

---

## C# — In-Process Domain Events with MediatR

```csharp
// Aggregate raises events as side effects
public sealed class Order
{
    private readonly List<INotification> _events = new();
    public IReadOnlyList<INotification> DomainEvents => _events;
    public void ClearDomainEvents() => _events.Clear();

    public void Place(...) {
        // ... mutate state ...
        _events.Add(new OrderPlaced(Id, Total));
    }
}

// Dispatch events on SaveChanges
public sealed class AppDbContext : DbContext
{
    private readonly IMediator _mediator;
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var aggregates = ChangeTracker.Entries<IHasDomainEvents>()
            .Select(e => e.Entity).Where(a => a.DomainEvents.Count > 0).ToList();

        var result = await base.SaveChangesAsync(ct);

        foreach (var agg in aggregates)
        {
            foreach (var ev in agg.DomainEvents) await _mediator.Publish(ev, ct);
            agg.ClearDomainEvents();
        }
        return result;
    }
}

// Subscribers in the same service
public sealed class SendOrderEmail : INotificationHandler<OrderPlaced> { ... }
public sealed class ReserveInventory : INotificationHandler<OrderPlaced> { ... }
```

---

## C# — Integration Events with MassTransit

```csharp
// Publish to broker
public sealed class OrderService
{
    private readonly IPublishEndpoint _bus;

    public async Task PlaceAsync(...) {
        // ... save to DB via outbox ...
        await _bus.Publish(new OrderPlacedV1(order.Id, order.CustomerId, order.Total, DateTime.UtcNow));
    }
}

// Consumer in a different service
public sealed class OrderPlacedConsumer : IConsumer<OrderPlacedV1>
{
    public async Task Consume(ConsumeContext<OrderPlacedV1> ctx)
    {
        await _email.SendOrderConfirmationAsync(ctx.Message.OrderId);
    }
}

// Program.cs
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderPlacedConsumer>();
    x.UsingRabbitMq((ctx, cfg) => cfg.ConfigureEndpoints(ctx));
});
```

---

## Trade-offs You Sign Up For

✅ **Wins:**

* **Decoupling** — producers know nothing about consumers.
* **Extensibility** — new behavior = new subscriber, no change to existing code.
* **Scalability** — each consumer scales independently.
* **Resilience** — broker buffers when a consumer is down.

⚠️ **Costs:**

* **Eventual consistency** — the world is not in sync; UIs must accept this.
* **Hard to trace** — *what triggered this side effect?* Distributed tracing (OpenTelemetry) becomes essential.
* **Event schema evolution** — once an event is consumed in production, you can't change it freely. Version it.
* **Out-of-order delivery** — most brokers don't guarantee order. Idempotency + sequence numbers if order matters.
* **Duplicate delivery** — at-least-once means you'll see duplicates. Pair with the **Inbox pattern** (see [`../architectural-pattern/outbox.md`](../architectural-pattern/outbox.md)).

---

## EDA vs. Request/Response

| Aspect                 | Request/Response (REST, gRPC)            | EDA (events)                                  |
| ---------------------- | ---------------------------------------- | --------------------------------------------- |
| Coupling               | Caller knows callee                      | Producer doesn't know consumers               |
| Timing                 | Synchronous                              | Asynchronous                                  |
| Consistency            | Strong (per request)                     | Eventual                                      |
| Failure handling       | Caller sees errors immediately           | Retries, DLQ, redrives                        |
| Adding new behavior    | Modify caller or service                 | Add new subscriber                            |
| Best for…              | Queries, user-facing latency             | Workflows, side effects, notifications, async |

Most real systems use **both** — REST for queries, EDA for side effects.

---

## EDA Topology Patterns

* **Broker topology** — services publish to a central broker (Kafka, RabbitMQ). Most common.
* **Mediator topology** — a central orchestrator routes events. Centralizes workflow logic. (Often a **Saga orchestrator**.)
* **Choreography** — services emit events that trigger others, no central brain. Simple but workflow logic is implicit.

---

## When to Choose EDA

✅ **Use when:**

* Multiple subsystems need to react to one business fact.
* Workflows are long-running or cross many services.
* You want to add new functionality without touching existing services.
* You're already on microservices.

❌ **Don't use when:**

* The system is a small monolith with simple flows.
* Strong, immediate consistency is required across the workflow.
* The team has no operational maturity for brokers, retries, idempotency, tracing.

---

## References

* Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003). The canonical reference.
* Martin Fowler — [*What do you mean by "Event-Driven"?*](https://martinfowler.com/articles/201701-event-driven.html).
* Chris Richardson — *Microservices Patterns* (2018). Chapters on messaging.
* Mark Richards — *Software Architecture Patterns* (O'Reilly, free).
