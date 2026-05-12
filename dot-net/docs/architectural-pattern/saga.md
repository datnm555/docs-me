# Saga Pattern

> **Saga** is an **architectural pattern** for managing **long-running, distributed business transactions** across multiple services — without using a global ACID transaction (2PC).

---

## The Problem

In a monolith, "place order → charge card → reserve inventory → ship" runs in **one database transaction**. If anything fails, you `ROLLBACK`.

In microservices, each step lives in a **different database** owned by a **different service**. You cannot wrap them in a single transaction. Yet the business workflow still needs to be atomic *in spirit*: either all steps succeed, or every successful step is **undone**.

```
[OrderService]  → CreateOrder      ✓
[PaymentService] → ChargeCard      ✓
[InventorySvc]   → ReserveStock    ✗ FAILED
[ShippingSvc]    → never runs
```

What rolls back the order and the payment?

---

## The Solution

A **Saga** is a sequence of **local transactions**. Each step:

1. Updates one service's database, *or*
2. Publishes a message to trigger the next step.

If a step fails, the saga executes **compensating actions** (semantic undo) for every step that already succeeded.

| Forward step       | Compensation                |
| ------------------ | --------------------------- |
| CreateOrder        | CancelOrder                 |
| ChargePayment      | RefundPayment               |
| ReserveInventory   | ReleaseInventory            |
| ShipPackage        | RecallShipment              |

Compensations are **business-level undo** — you can't truly "un-charge" a card, you issue a refund. You can't "un-ship", you initiate a recall.

---

## Two Flavors

### 1. Choreography — services react to events

No central coordinator. Each service listens to events and emits its own.

```
OrderService    ──OrderCreated──▶
                                  PaymentService ──PaymentCharged──▶
                                                                     InventoryService
                                                                       │
                                                  ◀──InventoryFailed──┘
PaymentService  ◀──InventoryFailed── (refunds)
OrderService    ◀──PaymentRefunded── (cancels)
```

* **Pro:** simple, no SPOF, services are decoupled.
* **Con:** workflow logic is *scattered* — hard to see the whole picture; cyclic dependencies appear easily.
* **Best when:** the saga has 2–4 steps and is unlikely to change.

### 2. Orchestration — a coordinator drives the steps

A dedicated service ("Saga Orchestrator") issues commands and reacts to replies.

```
            ┌─────────────────────┐
            │ OrderSagaOrchestrator│
            └──┬───────┬───────┬──┘
               │       │       │
       ChargeCmd  ReserveCmd  ShipCmd
               │       │       │
            Payment  Inventory  Shipping
```

* **Pro:** workflow is *explicit*, easy to visualize, easy to add steps.
* **Con:** orchestrator can become a god service; one more deployable to operate.
* **Best when:** the saga has 5+ steps, branches, retries, or human-in-the-loop approvals.

---

## C# — Orchestration with MassTransit

[MassTransit](https://masstransit.io/) ships a `Saga State Machine` built on Automatonymous.

```csharp
public class OrderSagaState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = "";
    public Guid OrderId { get; set; }
    public decimal Amount { get; set; }
}

public class OrderSagaStateMachine : MassTransitStateMachine<OrderSagaState>
{
    public State AwaitingPayment { get; private set; } = null!;
    public State AwaitingInventory { get; private set; } = null!;
    public State Completed { get; private set; } = null!;

    public Event<OrderSubmitted> OrderSubmitted { get; private set; } = null!;
    public Event<PaymentCharged> PaymentCharged { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = null!;
    public Event<InventoryReserved> InventoryReserved { get; private set; } = null!;
    public Event<InventoryFailed> InventoryFailed { get; private set; } = null!;

    public OrderSagaStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderSubmitted,     x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentCharged,     x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentFailed,      x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => InventoryReserved,  x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => InventoryFailed,    x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(OrderSubmitted)
                .Then(ctx => { ctx.Saga.OrderId = ctx.Message.OrderId; ctx.Saga.Amount = ctx.Message.Amount; })
                .Publish(ctx => new ChargePayment(ctx.Saga.OrderId, ctx.Saga.Amount))
                .TransitionTo(AwaitingPayment));

        During(AwaitingPayment,
            When(PaymentCharged)
                .Publish(ctx => new ReserveInventory(ctx.Saga.OrderId))
                .TransitionTo(AwaitingInventory),
            When(PaymentFailed)
                .Publish(ctx => new CancelOrder(ctx.Saga.OrderId))
                .Finalize());

        During(AwaitingInventory,
            When(InventoryReserved)
                .Publish(ctx => new ShipOrder(ctx.Saga.OrderId))
                .TransitionTo(Completed),
            When(InventoryFailed)
                .Publish(ctx => new RefundPayment(ctx.Saga.OrderId, ctx.Saga.Amount)) // compensation
                .Publish(ctx => new CancelOrder(ctx.Saga.OrderId))                   // compensation
                .Finalize());

        SetCompletedWhenFinalized();
    }
}

// Program.cs
builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderSagaStateMachine, OrderSagaState>()
        .EntityFrameworkRepository(r =>
        {
            r.ExistingDbContext<SagaDbContext>();
            r.UsePostgres();
        });
    x.UsingRabbitMq((ctx, cfg) => cfg.ConfigureEndpoints(ctx));
});
```

The saga state is persisted in a database so it survives restarts. MassTransit handles correlation, retry, and timeout.

---

## C# — Choreography with simple event handlers

```csharp
// PaymentService
public class OrderSubmittedHandler : INotificationHandler<OrderSubmitted>
{
    private readonly IPaymentGateway _payments;
    private readonly IEventBus _bus;

    public async Task Handle(OrderSubmitted e, CancellationToken ct)
    {
        try
        {
            var tx = await _payments.ChargeAsync(e.Amount, e.Currency, e.CustomerId, ct);
            await _bus.PublishAsync(new PaymentCharged(e.OrderId, tx.Id), ct);
        }
        catch
        {
            await _bus.PublishAsync(new PaymentFailed(e.OrderId), ct);
        }
    }
}

// OrderService listens for PaymentFailed and cancels itself
public class PaymentFailedHandler : INotificationHandler<PaymentFailed>
{
    public Task Handle(PaymentFailed e, CancellationToken ct) => _orders.CancelAsync(e.OrderId, ct);
}
```

No central state — each service knows only its piece of the workflow.

---

## When to Use Saga

✅ **Use when:**

* Cross-service workflows must be **eventually consistent**.
* Each step is **idempotent** or can be made so (essential).
* You can define a **business-level compensation** for every step.

❌ **Don't use when:**

* All operations live in **one database** — just use an ACID transaction.
* Some steps cannot be compensated and require true atomicity.
* The workflow has only one step (no need for a saga).

---

## Common Pitfalls

* **Forgetting compensations** — a saga without compensations is just an event chain.
* **Non-idempotent handlers** — duplicates from at-least-once delivery will corrupt state. Pair sagas with the **Inbox pattern** (see [`outbox.md`](./outbox.md)).
* **Lost messages** — the originating service must use the **Outbox pattern** to publish reliably.
* **Long timeouts** — wire explicit timeouts and a "stuck saga" alert; otherwise sagas accumulate.
* **No observability** — log correlation IDs end-to-end; treat sagas as first-class entities in your tracing/dashboards.

---

## Saga vs. 2PC vs. TCC

| Approach              | Atomicity        | Performance | Cross-tech support | Used today?                |
| --------------------- | ---------------- | ----------- | ------------------ | -------------------------- |
| **2-Phase Commit**    | Strict ACID      | Slow, blocks | Limited            | Legacy enterprise only     |
| **TCC (Try-Confirm-Cancel)** | Strict       | Medium      | Limited            | Niche                      |
| **Saga**              | Eventual         | Fast        | Excellent          | **Industry standard**      |

---

## References

* Hector Garcia-Molina & Kenneth Salem — *Sagas* (1987). [Original paper.](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)
* Chris Richardson — *Microservices Patterns* (2018). Chapter 4: Managing Transactions with Sagas.
* [microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html)
* MassTransit Sagas: [masstransit.io/documentation/patterns/saga](https://masstransit.io/documentation/patterns/saga)
