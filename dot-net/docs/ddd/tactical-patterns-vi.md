# Tactical DDD Patterns

> Các **building block ở cấp code** của Domain-Driven Design. Dùng *bên trong* một Bounded Context để model domain phong phú và bảo vệ invariant. Mỗi block có ý nghĩa chính xác — đừng nhập nhằng.

> 🇻🇳 Phiên bản tiếng Việt. English: [`tactical-patterns.md`](./tactical-patterns.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Building block cấp code của DDD: **Entity** (identity qua thời gian), **Value Object** (immutable, bằng theo value), **Aggregate** (boundary consistency với một root), **Domain Service** (behavior không fit một entity), **Domain Event** (cái gì đã xảy ra), **Factory** (construction phức tạp), **Repository** (truy cập collection-like per aggregate root), **Module** (group theo concept domain).
- **Tại sao** — Các pattern này enforce domain invariant ở mức type system, loại bỏ primitive obsession, và giữ business logic ngoài service / controller.
- **Khi nào** — Bên trong Bounded Context nơi domain có behavior có ý nghĩa (không chỉ CRUD). Skip cho admin tool trivial — ceremony cost hơn benefit ở đó.
- **Ở đâu** — Sống trong **Domain layer** của kiến trúc Clean/Hexagonal/Onion. Pair với strategic pattern (Bounded Context) và domain event flow ra qua Outbox pattern tới context khác.

---

## Index

1. [Entity](#1-entity)
2. [Value Object](#2-value-object)
3. [Aggregate & Aggregate Root](#3-aggregate--aggregate-root)
4. [Domain Event](#4-domain-event)
5. [Domain Service](#5-domain-service)
6. [Factory](#6-factory)
7. [Repository](#7-repository)
8. [Tổng hợp](#8-tổng-hợp)

---

## 1. Entity

**Entity** là object có **identity tồn tại theo thời gian**. Hai entity bằng nhau nếu **ID** bằng nhau — dù mọi field khác giống hệt.

* `Order #42` vẫn là cùng order dù mọi line item thay đổi.
* `Customer { Name = "Quan" }` và `Customer { Name = "Quan" }` là **hai customer khác nhau** nếu ID khác nhau.

```csharp
public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;

    public override bool Equals(object? obj) =>
        obj is Entity<TId> other && EqualityComparer<TId>.Default.Equals(Id, other.Id);

    public override int GetHashCode() => Id?.GetHashCode() ?? 0;
}

public sealed class Customer : Entity<Guid>
{
    public string Name { get; private set; }
    public Email Email { get; private set; }

    public Customer(Guid id, string name, Email email)
    {
        Id = id;
        Name = name;
        Email = email;
    }
}
```

**Quy tắc:**

* Identity được xác lập khi tạo và **không bao giờ thay đổi**.
* Behavior nằm **trên entity** (không phải trên class "Service" riêng) khi có thể.
* Dùng setter private; mutate state qua method enforce invariant.

---

## 2. Value Object

**Value Object** **không có identity**. Nó được định nghĩa hoàn toàn bằng **giá trị**. Hai value object cùng data là tương đương.

* `Money { Amount = 100, Currency = "USD" }` bằng bất kỳ `Money { 100, USD }` nào khác.
* Bạn không nói *"$100 này"* vs *"$100 kia"* — chúng giống nhau.

### Tính chất

* **Immutable** — tạo xong không thay đổi.
* **Equality theo value** (mọi property).
* **Operation trả về instance mới**, không có side effect.
* **Đóng gói business rule** (validation, computation).

### C# — Value Object với `record`

```csharp
public sealed record Email
{
    public string Value { get; }

    public Email(string value)
    {
        if (!Regex.IsMatch(value, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
            throw new ArgumentException("Invalid email", nameof(value));
        Value = value.ToLowerInvariant();
    }

    public override string ToString() => Value;
}

public sealed record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentException("Amount cannot be negative");
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }

    public static Money operator +(Money a, Money b) => a.Add(b);
}
```

### Tại sao Value Object quan trọng

Không có chúng, mọi method nhận primitive `string email, decimal amount, string currency` — và mỗi method phải validate. **Primitive obsession** lan validation khắp nơi.

Với Value Object, *bản thân type system* enforce tính đúng: không thể construct `Email` invalid. Method nhận `Money` không thể bị gọi với số âm.

---

## 3. Aggregate & Aggregate Root

Khái niệm DDD quan trọng nhất và hay bị hiểu sai nhất.

### Định nghĩa

* **Aggregate** là một cluster entity + value object liên quan, được treat như **một đơn vị** cho data change và enforce invariant.
* **Aggregate Root** là *entity duy nhất* mà thế giới bên ngoài tương tác để vào aggregate. Entity bên trong chỉ truy cập *qua* root.
* Aggregate có **boundary invariant**: các rule phải đúng cho aggregate phải enforce được trong **một local transaction**.

### Ví dụ

```
Order (Aggregate Root)
 ├── OrderLine (Entity bên trong aggregate, không có reference từ ngoài)
 ├── ShippingAddress (Value Object)
 └── TotalAmount (Value Object — derived)

Customer (Aggregate Root khác)
```

Code bên ngoài có thể `orders.GetById(id)`, rồi `order.AddLine(...)`. Không thể load `OrderLine` trực tiếp — `OrderLine` chỉ tồn tại trong ngữ cảnh `Order`.

### C# Sample

```csharp
public sealed class Order : Entity<Guid>
{
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total => new(_lines.Sum(l => l.LineTotal.Amount), _lines.FirstOrDefault()?.LineTotal.Currency ?? "USD");

    private Order() { } // EF

    public static Order Place(Guid customerId)
    {
        return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Cannot modify a submitted order");
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive");
        _lines.Add(new OrderLine(productId, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_lines.Count == 0) throw new InvalidOperationException("Cannot submit an empty order");
        Status = OrderStatus.Submitted;
    }
}

public sealed class OrderLine : Entity<Guid> // entity bên trong aggregate
{
    public Guid ProductId { get; }
    public int Quantity { get; }
    public Money UnitPrice { get; }
    public Money LineTotal => new(UnitPrice.Amount * Quantity, UnitPrice.Currency);

    internal OrderLine(Guid productId, int quantity, Money unitPrice)
    {
        Id = Guid.NewGuid();
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }
}
```

Lưu ý:

* `_lines` private; chỉ có thể đọc.
* Mọi mutation qua method `Order` enforce invariant.
* `OrderLine.ctor` là `internal` — code ngoài không thể bypass `Order.AddLine`.

### Quy tắc thiết kế Aggregate

* **Aggregate nhỏ**. Vaughn Vernon: *"thiết kế aggregate nhỏ."* Aggregate to thành bottleneck dưới concurrency.
* **Một aggregate cho mỗi transaction**. Nếu một use case modify hai aggregate, quy tắc là: mỗi cái transaction riêng; điều phối qua **Domain Event** (in-process) hoặc **Saga** (cross-service).
* **Reference aggregate khác bằng ID, không phải navigation property.** `Order.CustomerId : Guid`, không phải `Order.Customer : Customer`. Tránh accidentally load graph nhiều aggregate.
* **Một Repository cho mỗi Aggregate Root** — không bao giờ có repository cho entity bên trong.

---

## 4. Domain Event

**Domain Event** là điều gì đó domain quan tâm **đã xảy ra** — thì quá khứ.

```csharp
public sealed record OrderPlaced(Guid OrderId, Guid CustomerId, decimal Total, DateTime OccurredAt);
public sealed record OrderShipped(Guid OrderId, string TrackingNumber, DateTime OccurredAt);
public sealed record CustomerEmailChanged(Guid CustomerId, Email OldEmail, Email NewEmail);
```

**Tính chất:**

* **Thì quá khứ**, immutable.
* Được raise bởi một **Aggregate** như side effect của business operation.
* Dispatch **sau khi** aggregate được persist (commit-then-publish).
* Được consume bởi aggregate khác (cùng context) hoặc service khác (integration event).

### C# — Raise Domain Event từ Aggregate

```csharp
public interface IHasDomainEvents
{
    IReadOnlyList<INotification> DomainEvents { get; }
    void ClearDomainEvents();
}

public abstract class Entity<TId> : IHasDomainEvents
{
    public TId Id { get; protected set; } = default!;
    private readonly List<INotification> _events = new();
    public IReadOnlyList<INotification> DomainEvents => _events;
    protected void Raise(INotification ev) => _events.Add(ev);
    public void ClearDomainEvents() => _events.Clear();
}

public sealed class Order : Entity<Guid>
{
    public void Submit()
    {
        // ... check + thay đổi state ...
        Status = OrderStatus.Submitted;
        Raise(new OrderPlaced(Id, CustomerId, Total.Amount, DateTime.UtcNow));
    }
}

// Dispatch khi SaveChangesAsync (trong AppDbContext) — xem architectural-style/event-driven-vi.md
```

Xem [`../architectural-style/event-driven-vi.md`](../architectural-style/event-driven-vi.md) cho khác biệt giữa **domain event** (in-process) và **integration event** (cross-service).

---

## 5. Domain Service

Khi một mảng behavior domain **không tự nhiên fit vào entity hay value object nào**, model nó thành **Domain Service**.

Heuristic: nếu bạn thấy mình thêm method vào entity mà method đó nhận *entity từ aggregate khác*, đó thường là Domain Service.

```csharp
// Tệ — Order không nên biết cách look-up credit profile của customer
public class Order
{
    public void ApplyDiscount(Customer customer) { /* ... */ }
}

// Tốt — domain service điều phối hai aggregate
public sealed class PricingService
{
    public Money CalculateDiscountedTotal(Order order, Customer customer, IReadOnlyList<Promotion> active)
    {
        // logic liên quan cả ba
    }
}
```

**Domain Service vs. Application Service:**

* **Domain Service** sống ở domain layer. Không biết gì về HTTP, EF, MediatR. Thuần domain logic.
* **Application Service** sống ở application layer. Điều phối use case (load aggregate từ repo, gọi method domain, save, publish event). Biết về repository và unit of work.

---

## 6. Factory

Khi construct một aggregate phức tạp (nhiều invariant, nhiều phần, value tính toán), đóng gói trong **Factory**:

```csharp
public static class OrderFactory
{
    public static Order CreateFromCart(Customer customer, Cart cart, IReadOnlyList<DiscountRule> rules)
    {
        if (cart.IsEmpty) throw new InvalidOperationException("Cannot place an order with an empty cart");

        var order = Order.Place(customer.Id);
        foreach (var item in cart.Items)
            order.AddLine(item.ProductId, item.Quantity, item.UnitPrice);

        foreach (var rule in rules.Where(r => r.AppliesTo(order)))
            rule.Apply(order);

        return order;
    }
}
```

Một static factory method trên chính aggregate (`Order.Place(...)`) cũng tính là Factory và thường được ưa dùng hơn.

---

## 7. Repository

Xem thảo luận đầy đủ trong [`../enterprise-pattern/repository-vi.md`](../enterprise-pattern/repository-vi.md).

Theo tactical DDD:

* **Một repository cho mỗi aggregate root.** `IOrderRepository`, không phải `IOrderLineRepository`.
* Method nói **ngôn ngữ domain**: `GetUnpaidByCustomerAsync`, không phải `Find(predicate)`.
* Repository thuộc **domain layer** (interface) và **infrastructure layer** (implementation).

---

## 8. Tổng hợp

Một slice .NET DDD điển hình cho "place an order" trông như sau:

```
[ASP.NET Controller]
    └─▶ MediatR.Send(new PlaceOrderCommand(...))
            └─▶ PlaceOrderHandler                        ← Application Service
                  ├─▶ ICustomerRepository.GetById(...)   ← Repository (interface ở Domain)
                  ├─▶ OrderFactory.CreateFromCart(...)   ← Factory
                  │       └─▶ Order (Aggregate Root)     ← Tactical pattern đang hoạt động
                  │              ├─ OrderLine (Entity)
                  │              ├─ Money (Value Object)
                  │              └─ raise OrderPlaced (Domain Event)
                  ├─▶ IOrderRepository.AddAsync(order)
                  └─▶ IUnitOfWork.SaveChangesAsync()      ← commit
                          └─▶ Domain event dispatch (in-process) hoặc ghi vào outbox (integration)
```

Mỗi tactical pattern có một việc. Cùng nhau chúng bảo vệ invariant domain và giữ business logic tập trung ở domain layer.

---

## Anti-Pattern thường gặp

* **Anemic Domain Model** — entity chỉ có getter/setter, mọi logic trong service. Bạn có cấu trúc DDD nhưng không có substance.
* **Aggregate to bao trùm mọi thứ** — một aggregate root chứa cả thế giới. Concurrency chết.
* **Reference cross-aggregate qua navigation property** — `Order.Customer` thay vì `Order.CustomerId`. Load bùng nổ; transaction mở rộng.
* **Repository có method `Update`** — EF Core track thay đổi; bạn không cần "update". Mutate aggregate rồi save.
* **Domain event leak ra như integration event** — domain event là in-process; integration event xuyên service boundary. Đừng trộn.

---

## Tham khảo

* Eric Evans — *Domain-Driven Design* (2003). Part II.
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Chương 5–11.
* Vaughn Vernon — *Effective Aggregate Design* (3 phần, miễn phí). [vaughnvernon.com](https://www.dddcommunity.org/library/vernon_2011/).
