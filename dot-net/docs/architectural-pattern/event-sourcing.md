# Event Sourcing

> An **architectural pattern** where the **events that changed the state** are the source of truth — not the current state itself. Current state is *derived* by replaying events.

---

## Quick Reference (What · Why · When · Where)

- **What** — Persist the **events** that changed state as the source of truth; rebuild current state by replaying events. Snapshots optimize replay; projections expose read shapes.
- **Why** — Perfect audit log built-in; time-travel queries; new read projections from historical data; debugging by replaying actual user sequences; naturally publishes domain events.
- **When** — Domain is inherently event-driven (banking, trading, audit-heavy workflows, gaming); regulatory auditability is required; you need many different read shapes of the same facts.
- **Where** — Inside a Bounded Context, on the **write side** of a CQRS split. Pair with EventStoreDB / Marten / a custom Postgres event store, plus projections in SQL or Redis for the read side.

---

## Traditional Persistence vs. Event Sourcing

### Traditional (state-based)

```
[Orders]
 id   | status   | total
 ---  | -------- | -----
 1    | Shipped  | 100
```

You see the **current** state. You **lose** how it got there. *Why* is status `Shipped`? When was it `Paid`? You'd need a separate audit log — and audit logs always drift from reality.

### Event-Sourced

```
[OrderEvents]
 OrderId | Sequence | EventType        | Payload                    | OccurredAt
 ------- | -------- | ---------------- | -------------------------- | -----------
 1       | 1        | OrderPlaced      | {customer,items,total:100} | 2026-05-10
 1       | 2        | PaymentCharged   | {txId:abc}                 | 2026-05-10
 1       | 3        | OrderShipped     | {trackingNo:Z42}           | 2026-05-11
```

The current state of order #1 is *computed* by replaying these three events. The events themselves are immutable.

```csharp
var state = events.Aggregate(new OrderState(), (s, e) => s.Apply(e));
```

---

## Why It's Powerful

* **Perfect audit log** — built in, can't drift, can't be tampered with silently.
* **Time travel** — replay to any point: *"what did this order look like at 09:00?"*
* **New projections from old data** — invent a new read model years later, replay events to populate it.
* **Debugging** — reproduce a bug by replaying the exact event sequence in a test env.
* **Naturally publishes domain events** — they're already the storage format.

## Why It's Costly

* **Versioning events** — once shipped, you can never change an event's schema. You can only add new event types.
* **Querying current state is hard** — you need projections for every read shape.
* **Snapshots required at scale** — replaying 100K events for one aggregate is too slow without snapshots.
* **Mental model shift** — most developers think in terms of current state, not event streams.

---

## Core Building Blocks

### 1. Events — immutable facts

```csharp
public abstract record OrderEvent;
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total) : OrderEvent;
public sealed record PaymentCharged(Guid OrderId, string TxId) : OrderEvent;
public sealed record OrderShipped(Guid OrderId, string TrackingNumber) : OrderEvent;
public sealed record OrderCancelled(Guid OrderId, string Reason) : OrderEvent;
```

Naming convention: **past tense**. Something *that happened*.

### 2. Aggregate — applies events to compute state

```csharp
public enum OrderStatus { Placed, Paid, Shipped, Cancelled }

public sealed class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; private set; }
    public string? TrackingNumber { get; private set; }

    private readonly List<OrderEvent> _pending = new();
    public IReadOnlyList<OrderEvent> PendingEvents => _pending;

    // Replay constructor — rebuilds state from history
    public static Order Load(IEnumerable<OrderEvent> history)
    {
        var o = new Order();
        foreach (var e in history) o.Apply(e);
        return o;
    }

    // Business operation — emits an event, doesn't mutate state directly
    public static Order Place(Guid customerId, decimal total)
    {
        var o = new Order();
        var ev = new OrderPlaced(Guid.NewGuid(), customerId, total);
        o.Apply(ev);
        o._pending.Add(ev);
        return o;
    }

    public void Pay(string txId)
    {
        if (Status != OrderStatus.Placed) throw new InvalidOperationException();
        Raise(new PaymentCharged(Id, txId));
    }

    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Paid) throw new InvalidOperationException();
        Raise(new OrderShipped(Id, trackingNumber));
    }

    private void Raise(OrderEvent ev) { Apply(ev); _pending.Add(ev); }

    private void Apply(OrderEvent ev)
    {
        switch (ev)
        {
            case OrderPlaced p:    Id = p.OrderId; Total = p.Total; Status = OrderStatus.Placed; break;
            case PaymentCharged:   Status = OrderStatus.Paid; break;
            case OrderShipped s:   Status = OrderStatus.Shipped; TrackingNumber = s.TrackingNumber; break;
            case OrderCancelled:   Status = OrderStatus.Cancelled; break;
        }
    }
}
```

### 3. Event Store — append-only persistence

```csharp
public interface IEventStore
{
    Task AppendAsync(Guid streamId, IEnumerable<OrderEvent> events, long expectedVersion, CancellationToken ct);
    IAsyncEnumerable<OrderEvent> LoadAsync(Guid streamId, CancellationToken ct);
}
```

In .NET you have several solid options:

| Tool                         | Notes                                                            |
| ---------------------------- | ---------------------------------------------------------------- |
| **EventStoreDB**             | Purpose-built event store. Native subscriptions.                  |
| **Marten**                   | Postgres-backed; document DB + event store; great .NET ergonomics. |
| **Axon Server**              | Strong choice but more popular in Java land.                      |
| **Kurrent / SqlStreamStore** | Lighter, simpler.                                                 |
| **Roll your own on Postgres**| Append-only table; OK for low scale; pair with `LISTEN/NOTIFY`.   |

### 4. Projections — build read models

A projector subscribes to the event stream and updates a denormalized view:

```csharp
public sealed class OrderListProjector
{
    private readonly IDbConnection _conn;

    public async Task Handle(OrderEvent ev, CancellationToken ct) => ev switch
    {
        OrderPlaced p => await _conn.ExecuteAsync(
            "INSERT INTO order_list (id, total, status) VALUES (@id, @total, 'Placed')",
            new { id = p.OrderId, total = p.Total }),
        PaymentCharged c => await _conn.ExecuteAsync(
            "UPDATE order_list SET status = 'Paid' WHERE id = @id", new { id = c.OrderId }),
        OrderShipped s => await _conn.ExecuteAsync(
            "UPDATE order_list SET status = 'Shipped', tracking = @t WHERE id = @id",
            new { id = s.OrderId, t = s.TrackingNumber }),
        _ => 0
    };
}
```

This is naturally **CQRS** — events are the write model, projections are the read model.

---

## Snapshots

Replaying 50,000 events for one aggregate on every load is too slow. Persist a **snapshot** of state every N events:

```
events 1..1000 → SNAPSHOT @ v1000 → events 1001..1042
loading: latest snapshot + events after it
```

This is an optimization, not a separate source of truth — the events remain authoritative.

---

## When to Use Event Sourcing

✅ **Use when:**

* The domain is **inherently event-driven** (banking, trading, audit-heavy domains, workflows, gaming).
* Auditability is a regulatory requirement.
* You need to project the same facts into many read shapes.

❌ **Don't use when:**

* The data is CRUD-shaped with no inherent history.
* The team is new to DDD / CQRS — start there first.
* Strong reporting / ad-hoc query needs against current state, with no patience for projection lag.

**Greg Young's warning:** *Event Sourcing should be a deliberate choice for a specific bounded context — not a system-wide style.*

---

## Common Pitfalls

* **Event schema changes** — versioning is non-negotiable. Use upcasters or versioned types.
* **Forgetting idempotency** — projections must handle a replay of the same event without double-applying.
* **Mixing commands and events** — events describe *what happened*, not *what should happen*. `ChargeCard` is a command; `CardCharged` is an event.
* **Over-using ES** — applying ES to every bounded context is a common over-engineering trap.

---

## References

* Greg Young — *CQRS and Event Sourcing* talks (2014). [YouTube](https://www.youtube.com/watch?v=JHGkaShoyNs).
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013), chapter on Domain Events.
* Marten docs: [martendb.io/events](https://martendb.io/events/).
