# Saga Pattern

> **Saga** là một **architectural pattern** để quản lý **business transaction phân tán, chạy dài** xuyên nhiều service — mà không cần dùng global ACID transaction (2PC).

> 🇻🇳 Phiên bản tiếng Việt. English: [`saga.md`](./saga.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Chuỗi local transaction qua nhiều service, mỗi cái với compensating action semantic-undo step của nó khi cái gì downstream fail — orchestrated (coordinator trung tâm) hoặc choreographed (event-driven).
- **Tại sao** — Distributed ACID transaction (2PC) fragile vận hành qua microservices. Saga trade atomicity strict cho atomicity *eventual* bằng cách make mọi step idempotent và compensable.
- **Khi nào** — Workflow business span 2+ service, mỗi cái own data riêng, và không thể dùng single DB transaction. Ví dụ: place order → charge payment → reserve inventory → ship.
- **Ở đâu** — Trong hệ thống microservices; pair với Outbox pattern (cho publish reliable), Inbox (cho receive idempotent), và Domain Event (trong aggregate nguồn). Thường sống cạnh service own workflow.

---

## Bài toán

Trong monolith, "place order → charge card → reserve inventory → ship" chạy trong **một DB transaction**. Nếu có gì fail, ta `ROLLBACK`.

Trong microservices, mỗi step nằm ở **DB khác nhau**, thuộc **service khác nhau**. Không thể bọc tất cả trong một transaction. Nhưng workflow business vẫn cần atomic *về tinh thần*: hoặc tất cả thành công, hoặc mọi step đã thành công đều phải **được undo**.

```
[OrderService]  → CreateOrder      ✓
[PaymentService] → ChargeCard      ✓
[InventorySvc]   → ReserveStock    ✗ FAILED
[ShippingSvc]    → không bao giờ chạy
```

Cái gì rollback order và payment?

---

## Giải pháp

Một **Saga** là một chuỗi **local transaction**. Mỗi step:

1. Update database của một service, *hoặc*
2. Publish một message để trigger step kế tiếp.

Nếu một step fail, saga thực thi **compensating action** (undo về mặt nghiệp vụ) cho mọi step đã thành công trước đó.

| Forward step       | Compensation (Bù trừ)        |
| ------------------ | ---------------------------- |
| CreateOrder        | CancelOrder                  |
| ChargePayment      | RefundPayment                |
| ReserveInventory   | ReleaseInventory             |
| ShipPackage        | RecallShipment               |

Compensation là **undo ở mức business** — không thể "un-charge" thẻ tín dụng, ta phải refund. Không thể "un-ship", ta phải recall hàng.

---

## Hai phong cách Saga

### 1. Choreography — service phản ứng theo event

Không có coordinator trung tâm. Mỗi service listen event và emit event của riêng nó.

```
OrderService    ──OrderCreated──▶
                                  PaymentService ──PaymentCharged──▶
                                                                     InventoryService
                                                                       │
                                                  ◀──InventoryFailed──┘
PaymentService  ◀──InventoryFailed── (refund)
OrderService    ◀──PaymentRefunded── (cancel)
```

* **Ưu:** đơn giản, không có SPOF, các service decoupled.
* **Nhược:** logic workflow *bị tản mác* — khó nhìn toàn cảnh; dễ tạo dependency vòng tròn.
* **Khi nào dùng:** saga có 2–4 step và ít thay đổi.

### 2. Orchestration — có coordinator điều phối các step

Một service chuyên trách ("Saga Orchestrator") gửi command và xử lý reply.

```
            ┌─────────────────────┐
            │ OrderSagaOrchestrator│
            └──┬───────┬───────┬──┘
               │       │       │
       ChargeCmd  ReserveCmd  ShipCmd
               │       │       │
            Payment  Inventory  Shipping
```

* **Ưu:** workflow *rõ ràng*, dễ visualize, dễ thêm step.
* **Nhược:** orchestrator dễ thành god service; thêm một deployable phải vận hành.
* **Khi nào dùng:** saga có 5+ step, có branch, có retry, hoặc có bước người duyệt.

---

## C# — Orchestration với MassTransit

[MassTransit](https://masstransit.io/) ship sẵn `Saga State Machine` dựa trên Automatonymous.

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

State của saga được persist trong DB nên survive khi restart. MassTransit xử lý correlation, retry, timeout.

---

## C# — Choreography với event handler đơn giản

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

// OrderService listen PaymentFailed và tự cancel
public class PaymentFailedHandler : INotificationHandler<PaymentFailed>
{
    public Task Handle(PaymentFailed e, CancellationToken ct) => _orders.CancelAsync(e.OrderId, ct);
}
```

Không có state trung tâm — mỗi service chỉ biết phần của mình trong workflow.

---

## Khi nào dùng Saga

✅ **Dùng khi:**

* Workflow xuyên service cần **eventually consistent**.
* Mỗi step **idempotent** hoặc có thể làm thành idempotent (thiết yếu).
* Có thể định nghĩa **compensation cấp business** cho mọi step.

❌ **Không dùng khi:**

* Tất cả operation nằm trong **một database** — dùng ACID transaction là đủ.
* Có step không thể compensate và cần atomicity thật sự.
* Workflow chỉ có một step (không cần saga).

---

## Các lỗi thường gặp

* **Quên compensation** — saga không có compensation chỉ là chuỗi event thông thường.
* **Handler không idempotent** — duplicate từ at-least-once delivery sẽ corrupt state. Saga phải đi kèm **Inbox pattern** (xem [`outbox-vi.md`](./outbox-vi.md)).
* **Message bị mất** — service phát message phải dùng **Outbox pattern** để publish reliable.
* **Timeout dài** — phải set explicit timeout và alert cho "saga bị kẹt"; nếu không saga sẽ tồn đọng.
* **Thiếu observability** — log correlation ID end-to-end; treat saga như entity hạng nhất trong tracing/dashboard.

---

## Saga vs. 2PC vs. TCC

| Cách tiếp cận         | Atomicity        | Performance | Hỗ trợ cross-tech | Còn dùng không?            |
| --------------------- | ---------------- | ----------- | ------------------ | -------------------------- |
| **2-Phase Commit**    | Strict ACID      | Chậm, block | Hạn chế            | Chỉ legacy enterprise      |
| **TCC (Try-Confirm-Cancel)** | Strict       | Trung bình  | Hạn chế            | Niche                      |
| **Saga**              | Eventual         | Nhanh       | Rất tốt            | **Industry standard**      |

---

## Tham khảo

* Hector Garcia-Molina & Kenneth Salem — *Sagas* (1987). [Paper gốc.](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)
* Chris Richardson — *Microservices Patterns* (2018). Chapter 4: Managing Transactions with Sagas.
* [microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html)
* MassTransit Sagas: [masstransit.io/documentation/patterns/saga](https://masstransit.io/documentation/patterns/saga)
