# Event Sourcing

> Một **architectural pattern** trong đó **các event đã thay đổi state** là source of truth — không phải state hiện tại. State hiện tại được *suy ra* bằng cách replay các event.

> 🇻🇳 Phiên bản tiếng Việt. English: [`event-sourcing.md`](./event-sourcing.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Persist **event** đã change state như source of truth; rebuild state hiện tại bằng replay event. Snapshot tối ưu replay; projection expose shape read.
- **Tại sao** — Audit log hoàn hảo built-in; query time-travel; projection read mới từ data lịch sử; debug bằng replay sequence user thật; tự nhiên publish domain event.
- **Khi nào** — Domain inherently event-driven (banking, trading, audit-heavy workflow, gaming); auditability regulatory required; cần nhiều shape read khác của cùng fact.
- **Ở đâu** — Bên trong Bounded Context, ở **write side** của CQRS split. Pair với EventStoreDB / Marten / custom Postgres event store, cộng projection trong SQL hoặc Redis cho read side.

---

## Persistence truyền thống vs. Event Sourcing

### Truyền thống (state-based)

```
[Orders]
 id   | status   | total
 ---  | -------- | -----
 1    | Shipped  | 100
```

Bạn thấy **state hiện tại**. Bạn **mất** lịch sử nó đến đó như thế nào. *Tại sao* status là `Shipped`? Khi nào nó là `Paid`? Phải có audit log riêng — và audit log luôn drift khỏi thực tế.

### Event-Sourced

```
[OrderEvents]
 OrderId | Sequence | EventType        | Payload                    | OccurredAt
 ------- | -------- | ---------------- | -------------------------- | -----------
 1       | 1        | OrderPlaced      | {customer,items,total:100} | 2026-05-10
 1       | 2        | PaymentCharged   | {txId:abc}                 | 2026-05-10
 1       | 3        | OrderShipped     | {trackingNo:Z42}           | 2026-05-11
```

State hiện tại của order #1 được *tính ra* bằng cách replay 3 event này. Event là immutable.

```csharp
var state = events.Aggregate(new OrderState(), (s, e) => s.Apply(e));
```

---

## Tại sao mạnh

* **Audit log hoàn hảo** — có sẵn, không thể drift, không thể bị sửa lén.
* **Time travel** — replay tới bất kỳ thời điểm: *"order này lúc 09:00 trông thế nào?"*
* **Tạo projection mới từ data cũ** — nghĩ ra read model mới sau nhiều năm, replay event để populate.
* **Debug** — reproduce bug bằng cách replay đúng chuỗi event trong env test.
* **Tự nhiên publish domain event** — chúng đã là format lưu trữ rồi.

## Tại sao tốn

* **Versioning event** — một khi đã ship, không bao giờ thay đổi schema của event. Chỉ có thể thêm event type mới.
* **Query state hiện tại khó** — cần projection cho mỗi shape read.
* **Snapshot cần thiết ở scale** — replay 100K event cho một aggregate quá chậm nếu không có snapshot.
* **Đổi mindset** — phần lớn dev nghĩ theo state hiện tại, không nghĩ theo event stream.

---

## Các building block cốt lõi

### 1. Event — fact bất biến

```csharp
public abstract record OrderEvent;
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total) : OrderEvent;
public sealed record PaymentCharged(Guid OrderId, string TxId) : OrderEvent;
public sealed record OrderShipped(Guid OrderId, string TrackingNumber) : OrderEvent;
public sealed record OrderCancelled(Guid OrderId, string Reason) : OrderEvent;
```

Quy ước đặt tên: **thì quá khứ**. Điều gì đó *đã xảy ra*.

### 2. Aggregate — apply event để tính state

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

    // Replay constructor — rebuild state từ history
    public static Order Load(IEnumerable<OrderEvent> history)
    {
        var o = new Order();
        foreach (var e in history) o.Apply(e);
        return o;
    }

    // Operation business — emit event, không trực tiếp mutate state
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

### 3. Event Store — persistence chỉ append

```csharp
public interface IEventStore
{
    Task AppendAsync(Guid streamId, IEnumerable<OrderEvent> events, long expectedVersion, CancellationToken ct);
    IAsyncEnumerable<OrderEvent> LoadAsync(Guid streamId, CancellationToken ct);
}
```

Trong .NET có nhiều lựa chọn vững chắc:

| Tool                         | Ghi chú                                                          |
| ---------------------------- | ---------------------------------------------------------------- |
| **EventStoreDB**             | Event store chuyên dụng. Subscription native.                     |
| **Marten**                   | Backed bởi Postgres; document DB + event store; ergonomics .NET tốt. |
| **Axon Server**              | Tốt nhưng phổ biến hơn ở Java.                                    |
| **Kurrent / SqlStreamStore** | Nhẹ hơn, đơn giản hơn.                                            |
| **Tự build trên Postgres**   | Table append-only; OK ở scale thấp; kết hợp `LISTEN/NOTIFY`.       |

### 4. Projection — build read model

Projector subscribe event stream và update view denormalized:

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

Đây tự nhiên là **CQRS** — event là write model, projection là read model.

---

## Snapshot

Replay 50,000 event cho một aggregate mỗi lần load quá chậm. Persist **snapshot** của state mỗi N event:

```
events 1..1000 → SNAPSHOT @ v1000 → events 1001..1042
loading: snapshot mới nhất + event sau nó
```

Đây là tối ưu, không phải source of truth riêng — event vẫn là chính.

---

## Khi nào dùng Event Sourcing

✅ **Dùng khi:**

* Domain **bản chất event-driven** (banking, trading, audit-heavy, workflow, gaming).
* Auditability là yêu cầu pháp lý.
* Cần project cùng fact thành nhiều shape read.

❌ **Không dùng khi:**

* Data dạng CRUD không có lịch sử inherent.
* Team mới làm DDD / CQRS — học cái đó trước.
* Cần report / ad-hoc query mạnh trên state hiện tại, không kiên nhẫn được với projection lag.

**Cảnh báo của Greg Young:** *Event Sourcing nên là lựa chọn có ý thức cho một bounded context cụ thể — không phải style toàn hệ thống.*

---

## Lỗi thường gặp

* **Thay đổi schema event** — versioning là bắt buộc. Dùng upcaster hoặc type có version.
* **Quên idempotent** — projection phải xử lý được replay cùng event mà không áp dụng đúp.
* **Trộn lẫn command và event** — event mô tả *điều đã xảy ra*, không phải *điều nên làm*. `ChargeCard` là command; `CardCharged` là event.
* **Lạm dụng ES** — áp dụng ES cho mọi bounded context là cái bẫy over-engineering phổ biến.

---

## Tham khảo

* Greg Young — *CQRS and Event Sourcing* talks (2014). [YouTube](https://www.youtube.com/watch?v=JHGkaShoyNs).
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013), chapter on Domain Events.
* Marten docs: [martendb.io/events](https://martendb.io/events/).
