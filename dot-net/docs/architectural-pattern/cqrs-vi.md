# CQRS — Command Query Responsibility Segregation

> Một **architectural pattern** tách model dùng để **thay đổi** data (commands) khỏi model dùng để **đọc** data (queries).

> 🇻🇳 Phiên bản tiếng Việt. English: [`cqrs.md`](./cqrs.md)

---

## Ý tưởng cốt lõi

Một domain model duy nhất phục vụ cả write và read sẽ phục vụ không tốt bên nào:

* **Write** muốn model normalized, consistent về transaction, có behavior phong phú.
* **Read** muốn data denormalized, pre-joined, tối ưu cho view, query nhanh.

CQRS nói: ngưng bắt một model làm cả hai việc.

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

## Các cấp độ CQRS

CQRS là một **dải lựa chọn (spectrum)**, không phải binary. Chọn cấp phù hợp với bài toán.

### Cấp 1 — In-Process CQRS (cùng database)

* Tách handler **command** và **query** trong code.
* Cùng database, nhưng **command dùng domain model** (EF Core entity) và **query dùng Dapper hoặc projected DTO** (không load entity).
* Phần lớn app ".NET CQRS với MediatR" làm đến cấp này.

### Cấp 2 — Read model nằm ở store riêng

* Write đi Postgres (consistent).
* Read đi view denormalized trong Redis / Elastic / Postgres schema khác, sync qua event.

### Cấp 3 — Service write và service read tách hẳn

* Service khác nhau, DB khác nhau, scaling khác nhau.
* Sync qua event stream (thường đi kèm **Event Sourcing**).
* Độ phức tạp cao — chỉ justify cho hệ thống scale lớn.

---

## C# — Cấp 1 CQRS với MediatR

### Phía Command — domain model phong phú

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

        var order = customer.PlaceOrder(cmd.Items, cmd.Currency); // business rules ở trong domain
        _db.Orders.Add(order);

        await _payments.ChargeAsync(order.Total, cmd.Currency, customer.Id.ToString(), ct);
        await _db.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

### Phía Query — flat DTO, raw SQL (Dapper) — không có domain model

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

Phía write không bao giờ thấy Dapper; phía read không bao giờ thấy EF Core entity. Mỗi phía **tối ưu cho việc riêng**.

---

## C# — Cấp 2 CQRS với Projection

Phía write commit Postgres; background projector listen domain event và update view backed by Redis.

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

Read view có **eventually consistent** — query ngay sau write có thể vẫn thấy data cũ trong vài mili-giây. Đây là trade-off của việc tách store.

---

## Khi nào dùng CQRS

✅ **Dùng khi:**

* Workload read và write có **shape rất khác nhau** (query phức tạp vs write đơn giản, hoặc ngược lại).
* Bạn muốn write logic trong domain model phong phú nhưng read path nhanh, đơn giản.
* Cần scale read và write độc lập.

❌ **Không dùng khi:**

* App CRUD đơn giản. CQRS thêm code mà không có lợi ích.
* Team chưa quen với eventual consistency.
* "Ngân sách phức tạp" đã dùng hết chỗ khác.

**Greg Young (người đặt tên CQRS):** *"CQRS không phải là kiến trúc cấp cao nhất. Nó là pattern áp dụng cho một Bounded Context."* Áp dụng nơi nào nó có lợi; đừng áp dụng cho toàn hệ thống mặc định.

---

## CQRS vs. CQS

* **CQS (Command-Query Separation)** — nguyên lý của Bertrand Meyer: một method hoặc thay đổi state HOẶC trả về data, không bao giờ cả hai. Mức class.
* **CQRS** — mở rộng CQS thành **tách model / class / pipeline**.

CQS luôn tốt. CQRS thì tuỳ trường hợp.

---

## CQRS và Event Sourcing

CQRS **thường đi đôi** nhưng **độc lập** với [Event Sourcing](./event-sourcing-vi.md):

| Kết hợp                   | Phổ biến? | Ghi chú                                                            |
| ------------------------- | -------- | ------------------------------------------------------------------ |
| CQRS only                 | Rất      | Trường hợp 80%. Mediator + Dapper reads + EF writes.               |
| Event Sourcing only       | Hiếm     | Có thể có nhưng lạ; ES tự nhiên dẫn tới read model tách biệt.      |
| **CQRS + Event Sourcing** | Phổ biến | Event là write model; projection là read model.                     |
| Cả hai đều không          | Mặc định | Hầu hết các app CRUD đơn giản.                                      |

---

## Tham khảo

* Greg Young — *CQRS Documents* (2010). [PDF](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
* Martin Fowler — [*CQRS*](https://martinfowler.com/bliki/CQRS.html).
* Microsoft — *Cloud Design Patterns: CQRS*.
