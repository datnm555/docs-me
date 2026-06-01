# Outbox / Inbox Patterns

> Hai **architectural pattern** kết hợp với nhau để giải bài toán **dual-write** trong distributed system: làm sao để atomic (1) update database VÀ (2) publish message — khi mỗi cái có thể fail độc lập.

> 🇻🇳 Phiên bản tiếng Việt. English: [`outbox.md`](./outbox.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — **Outbox**: write message vào outbox table trong cùng DB transaction với business data; process background poll và publish. **Inbox**: record ID message đã xử lý để dedupe redelivery. Cùng nhau chúng làm publish + handle effectively exactly-once.
- **Tại sao** — Không có nó, vấn đề dual-write ("save DB VÀ publish broker") có thể để bạn với row saved không event, hoặc event không row, ở bất kỳ failure mode nào.
- **Khi nào** — Bất cứ lúc nào service emit integration event từ business operation, đặc biệt trong Saga; bắt buộc nếu at-least-once delivery là default (Kafka, RabbitMQ, Azure Service Bus).
- **Ở đâu** — Producer side: trong cùng `DbContext` với aggregate. Consumer side: inbox table trong DB consumer, transactional với side effect. Pair với saga và domain event.

---

## Bài toán Dual-Write

Code ngây thơ:

```csharp
await _db.SaveChangesAsync(ct);          // 1. commit DB
await _bus.PublishAsync(orderCreated);    // 2. publish lên broker
```

Các trường hợp fail:

| Cái nào fail             | Hệ quả                                          |
| ------------------------ | ----------------------------------------------- |
| Step 1 fail              | Không xảy ra gì. OK.                            |
| Step 1 OK, step 2 fail   | DB đã update, **message không được publish**. **Inconsistency.** |
| Cả hai OK                | OK.                                             |

Đảo thứ tự cũng tệ không kém:

```csharp
await _bus.PublishAsync(orderCreated);    // 1. publish
await _db.SaveChangesAsync(ct);           // 2. commit
```

Bây giờ ta có thể publish event cho order mà **chưa bao giờ được save**.

**Không có cách nào** làm cho hai write này atomic mà không cần distributed transaction (2PC) — cái mà ta không muốn dùng.

---

## Outbox Pattern

**Ý tưởng:** ghi message vào **một outbox table trong cùng database**, trong **cùng local transaction** với business data. Một process riêng poll outbox và publish lên broker.

```
┌─────────────────────────────────────────┐
│      Database của Service A             │
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

Nếu broker down, message vẫn nằm trong outbox; publisher retry tới khi success. Business write **không bị block** bởi broker availability.

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

### C# — Save vào Outbox trong cùng Transaction

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

        // Ghi integration event VÀO CÙNG DbContext — same transaction
        _db.Outbox.Add(new OutboxMessage
        {
            Type    = nameof(OrderPlaced),
            Content = JsonSerializer.Serialize(new OrderPlaced(order.Id, order.Total)),
        });

        await _db.SaveChangesAsync(ct); // atomic: order + outbox row commit cùng nhau
    }
}
```

### C# — Publisher (background worker)

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

Để throughput cao hơn, thay polling bằng **Change Data Capture (CDC)** — ví dụ Debezium đọc WAL/binlog. MassTransit và Wolverine đều ship Outbox implementation production-grade.

---

## Inbox Pattern (de-duplicate phía consumer)

Broker giao message **at-least-once**. Một blip mạng làm cùng một message bị giao 2 lần. Không có bảo vệ, consumer xử lý 2 lần → duplicate order, double refund.

**Ý tưởng:** consumer giữ một **Inbox table** chứa ID các message đã xử lý. Trước khi xử lý, check inbox; sau khi xử lý, insert ID — cả hai trong cùng transaction với side effect.

### Inbox Schema

```sql
CREATE TABLE InboxMessages (
    Id              UUID PRIMARY KEY,   -- message id từ broker
    HandledOnUtc    TIMESTAMP NOT NULL
);
```

### C# — Idempotent Consumer dùng Inbox

```csharp
public sealed class OrderPlacedHandler
{
    private readonly AppDbContext _db;
    private readonly IInventoryService _inventory;

    public async Task HandleAsync(string messageId, OrderPlaced ev, CancellationToken ct)
    {
        await using var tx = await _db.Database.BeginTransactionAsync(ct);

        // 1. Đã xử lý chưa?
        var alreadyHandled = await _db.Inbox
            .AsNoTracking()
            .AnyAsync(i => i.Id == Guid.Parse(messageId), ct);
        if (alreadyHandled) return;

        // 2. Thực thi
        await _inventory.ReserveAsync(ev.OrderId, ct);

        // 3. Ghi nhận đã xử lý — same TX với side effect
        _db.Inbox.Add(new InboxMessage { Id = Guid.Parse(messageId), HandledOnUtc = DateTime.UtcNow });

        await _db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
}
```

---

## Outbox + Inbox kết hợp

```
Producer Service                                  Consumer Service
┌─────────────────────┐                          ┌──────────────────────┐
│ Business TX:        │                          │ Business TX:         │
│   • Save order      │                          │   • Reserve stock     │
│   • Save outbox row │  →  Broker (at-least)  → │   • Insert inbox row  │
└─────────────────────┘                          └──────────────────────┘
       Publish reliable                                  Handle idempotent
```

Đây là **gold standard** cho messaging vừa reliable vừa exactly-once-effective, không cần distributed transaction. Đây cũng là nền tảng giúp [Saga](./saga-vi.md) an toàn.

---

## Công cụ trong .NET

* **[MassTransit](https://masstransit.io/)** — extension `EntityFrameworkOutbox` / `EntityFrameworkInbox`.
* **[Wolverine](https://wolverine.netlify.app/)** — transactional outbox được tích hợp sẵn vào framework.
* **[NServiceBus](https://particular.net/nservicebus)** — feature `Outbox`, có phí enterprise.
* **Debezium + Kafka Connect** — outbox dựa trên CDC cho throughput cao.
* **DotNetCore.CAP** — thư viện open-source làm outbox/inbox.

---

## Các lỗi thường gặp

* **Quên outbox trong test** — integration event không bao giờ được publish trong env test, che giấu bug.
* **Outbox tồn đọng dài** — message fail làm tắc nghẽn table. Cần max-retry + DLQ pattern.
* **Out-of-order delivery** — outbox publisher parallel; nếu order quan trọng, dùng partition đơn / publisher single-threaded theo từng aggregate.
* **Migration schema cho outbox** — khi message cũ đã tồn tại với schema cũ, không thể break được. Version event payload của mình.

---

## Tham khảo

* Chris Richardson — *Microservices Patterns* (2018). Chapter 3: Inter-process communication; Chapter 4: Sagas.
* [microservices.io/patterns/data/transactional-outbox.html](https://microservices.io/patterns/data/transactional-outbox.html)
* Gunnar Morling — [*Reliable Microservices Data Exchange With the Outbox Pattern*](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/).
