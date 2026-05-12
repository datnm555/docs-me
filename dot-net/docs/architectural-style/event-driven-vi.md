# Event-Driven Architecture (EDA / EDD)

> Một **architectural style** trong đó các component giao tiếp chủ yếu bằng cách **publish và consume event**, thay vì gọi trực tiếp request/response. Producer không biết ai consume (loose coupling). Consumer phản ứng asynchronous.

> 🇻🇳 Phiên bản tiếng Việt. English: [`event-driven.md`](./event-driven.md)

---

## EDA có phải Design Pattern không?

**Không.** Event-Driven Architecture là **architectural style** — cùng cấp với Layered, Hexagonal, REST, hay Microservices. **Không phải** GoF design pattern.

* **EDA** = Event-Driven Architecture (hình dạng hệ thống).
* **EDD** = Event-Driven Design / Development (practice thiết kế theo event).

Các building block pattern **bên trong** EDA gồm: pub/sub, **Outbox**, **Inbox**, event sourcing, **Saga**, CQRS, message broker, projector. Mỗi cái *đó* là một pattern.

---

## Ý tưởng cốt lõi

```
[Order Service]
       │
       │ publish OrderPlaced
       ▼
   ┌────────────┐
   │  Broker    │   (Kafka, RabbitMQ, Azure Service Bus, Redis Streams)
   └────────────┘
       │
       ├──▶  [Inventory Service]  → reserve stock
       ├──▶  [Email Service]      → gửi confirmation
       ├──▶  [Analytics Service]  → ghi metric
       └──▶  [Fraud Service]      → score rủi ro
```

Order Service **không biết** ai phản ứng với `OrderPlaced`. Có thể thêm subscriber mới mà không sửa nó. Đây là lợi thế chính của EDA: **decoupling cả về thời gian và danh tính**.

---

## Ba loại Event

Nguồn gây nhầm lẫn rất phổ biến. **Chọn đúng loại cho đúng việc.**

### 1. Domain Event — "có gì đó xảy ra trong bounded context *của tôi*"

* In-process (hoặc trong một service).
* Mang ý nghĩa domain phong phú.
* Subscriber nằm trong **cùng service**.
* Thường được raise bởi aggregate như side effect của business operation.

```csharp
public sealed record OrderPlaced(Guid OrderId, decimal Total, DateTime OccurredAt);
```

Implement trong .NET bằng **MediatR `INotification`**, **in-process event bus**, hoặc danh sách `_pendingEvents` của aggregate được dispatch khi save.

### 2. Integration Event — "có gì đó xảy ra mà *service khác* cần biết"

* Cross-service.
* Có version. Contract public.
* Đi qua **message broker**.
* Payload nhỏ, ổn định hơn.

```csharp
public sealed record OrderPlacedV1(Guid OrderId, Guid CustomerId, decimal Total, DateTime OccurredAt);
```

Implement bằng **MassTransit / Wolverine / NServiceBus** trên RabbitMQ / Kafka / Azure Service Bus.

### 3. Event-Carried State Transfer — "đây là state mới nhất"

* Payload của event chứa **tất cả data consumer cần** để không phải gọi ngược lại.
* Trade-off: event to hơn; nguy cơ data stale.

```csharp
// Bao gồm snapshot đầy đủ — consumer không phải query Order service
public sealed record OrderStateChangedV1(
    Guid OrderId,
    OrderStatus Status,
    decimal Total,
    Guid CustomerId,
    DateTime OccurredAt);
```

---

## C# — In-Process Domain Event với MediatR

```csharp
// Aggregate raise event như side effect
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

// Dispatch event khi SaveChanges
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

// Subscriber trong cùng service
public sealed class SendOrderEmail : INotificationHandler<OrderPlaced> { ... }
public sealed class ReserveInventory : INotificationHandler<OrderPlaced> { ... }
```

---

## C# — Integration Event với MassTransit

```csharp
// Publish lên broker
public sealed class OrderService
{
    private readonly IPublishEndpoint _bus;

    public async Task PlaceAsync(...) {
        // ... save vào DB qua outbox ...
        await _bus.Publish(new OrderPlacedV1(order.Id, order.CustomerId, order.Total, DateTime.UtcNow));
    }
}

// Consumer ở service khác
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

## Trade-off bạn đăng ký theo

✅ **Lợi:**

* **Decoupling** — producer không biết gì về consumer.
* **Mở rộng** — thêm behavior = thêm subscriber, không sửa code cũ.
* **Scalability** — mỗi consumer scale độc lập.
* **Resilience** — broker buffer khi consumer down.

⚠️ **Chi phí:**

* **Eventually consistent** — thế giới không sync ngay; UI phải chấp nhận.
* **Khó trace** — *cái gì trigger side effect này?* Distributed tracing (OpenTelemetry) thiết yếu.
* **Tiến hoá schema** — một khi event đã được consume trong prod, không thể thay đổi tuỳ tiện. Phải version.
* **Out-of-order delivery** — phần lớn broker không đảm bảo thứ tự. Idempotent + sequence number nếu thứ tự quan trọng.
* **Duplicate delivery** — at-least-once nghĩa là sẽ có duplicate. Phải kết hợp **Inbox pattern** (xem [`../architectural-pattern/outbox-vi.md`](../architectural-pattern/outbox-vi.md)).

---

## EDA vs. Request/Response

| Khía cạnh              | Request/Response (REST, gRPC)            | EDA (event)                                   |
| ---------------------- | ---------------------------------------- | --------------------------------------------- |
| Coupling               | Caller biết callee                       | Producer không biết consumer                  |
| Timing                 | Synchronous                              | Asynchronous                                  |
| Consistency            | Strong (mỗi request)                     | Eventual                                      |
| Xử lý fail             | Caller thấy error ngay                   | Retry, DLQ, redrive                           |
| Thêm behavior mới      | Sửa caller hoặc service                  | Thêm subscriber mới                           |
| Phù hợp cho…           | Query, latency user-facing               | Workflow, side effect, notification, async    |

Hầu hết hệ thống thật dùng **cả hai** — REST cho query, EDA cho side effect.

---

## Các topology EDA

* **Broker topology** — service publish lên broker trung tâm (Kafka, RabbitMQ). Phổ biến nhất.
* **Mediator topology** — một orchestrator trung tâm route event. Tập trung logic workflow. (Thường là **Saga orchestrator**.)
* **Choreography** — service emit event trigger các service khác, không có brain trung tâm. Đơn giản nhưng workflow logic implicit.

---

## Khi nào chọn EDA

✅ **Dùng khi:**

* Nhiều subsystem cần phản ứng với một business fact.
* Workflow chạy dài hoặc xuyên nhiều service.
* Bạn muốn thêm chức năng mới mà không đụng service hiện có.
* Bạn đã ở trên microservices.

❌ **Không dùng khi:**

* Hệ thống là monolith nhỏ với flow đơn giản.
* Cần strong, immediate consistency xuyên workflow.
* Team chưa đủ trưởng thành về vận hành broker, retry, idempotent, tracing.

---

## Tham khảo

* Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003). Tham chiếu chuẩn.
* Martin Fowler — [*What do you mean by "Event-Driven"?*](https://martinfowler.com/articles/201701-event-driven.html).
* Chris Richardson — *Microservices Patterns* (2018). Chương về messaging.
* Mark Richards — *Software Architecture Patterns* (O'Reilly, miễn phí).
