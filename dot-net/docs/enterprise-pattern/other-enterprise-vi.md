# Pattern Application Khác

> Các pattern hằng ngày sống trên data layer: **Specification**, **CQRS**, **Mediator**, **Result**, **Options**, **Decorator pipeline**, **Factory**, **Strategy at the seam**. Mỗi cái giải một pain cụ thể ở application level — đừng dùng nếu không có pain đó.

> 🇻🇳 Phiên bản tiếng Việt. English: [`other-enterprise.md`](./other-enterprise.md)

---

## Mục lục

1. [Specification](#1-specification)
2. [CQRS — Command/Query Responsibility Segregation](#2-cqrs--commandquery-responsibility-segregation)
3. [Mediator (in-process bus)](#3-mediator-in-process-bus)
4. [Result / Either Pattern](#4-result--either-pattern)
5. [Options Pattern](#5-options-pattern)
6. [Decorator Pipeline (Cross-Cutting Concern)](#6-decorator-pipeline-cross-cutting-concern)
7. [Factory](#7-factory)
8. [Strategy at the Seam](#8-strategy-at-the-seam)
9. [Cách chọn](#9-cách-chọn)

---

## 1. Specification

> Đóng gói business rule thành object có thể **combine**, **reuse**, và **test isolate**.

Hữu ích khi predicate dùng ở nhiều chỗ — query, domain assertion, validation step.

### 1.1 Pattern

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T candidate);
    Expression<Func<T, bool>> ToExpression();
}

public abstract class Specification<T> : ISpecification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    public bool IsSatisfiedBy(T candidate) => ToExpression().Compile()(candidate);

    public Specification<T> And(Specification<T> other) => new AndSpec<T>(this, other);
    public Specification<T> Or(Specification<T> other)  => new OrSpec<T>(this, other);
    public Specification<T> Not()                       => new NotSpec<T>(this);
}
```

### 1.2 Spec cụ thể

```csharp
public class IsActiveCustomer : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression()
        => c => c.Status == CustomerStatus.Active;
}

public class HasMinOrders(int n) : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression()
        => c => c.Orders.Count >= n;
}

// Compose:
var loyal = new IsActiveCustomer().And(new HasMinOrders(5));

// Dùng làm query:
var query = db.Customers.Where(loyal.ToExpression());

// Hoặc check in-memory:
if (loyal.IsSatisfiedBy(currentCustomer)) ...
```

### 1.3 Dùng khi

* Cùng predicate xuất hiện ở **nhiều** chỗ.
* Muốn **đặt tên** business rule trong ngôn ngữ domain.
* Cần compose rule dynamic (filter từ UI).

### 1.4 Khi *không* dùng

* Predicate một dòng, dùng một lần. Chỉ viết lambda.
* Compose `Expression<>` compile-time đau cho rule không trivial. Cân nhắc [LinqKit](https://github.com/scottksmith95/LINQKit) hoặc helper compose domain-specific.

---

## 2. CQRS — Command/Query Responsibility Segregation

> Dùng **model khác** để write (command) so với read (query).

### 2.1 Idea

Một domain model phục vụ cả read và write cuối cùng tệ cả hai — read muốn data denormalized, nhanh, shape-specific; write muốn aggregate consistent, validated.

CQRS tách chúng:

```
┌──────────────┐    Command    ┌──────────────────┐
│   Client     │ ────────────► │ Command Handler  │ ──► Domain ──► DB (write model)
└──────────────┘                └──────────────────┘
       │
       │ Query
       ▼
┌──────────────────┐
│ Query Handler    │ ──► DB / cache / read model
└──────────────────┘
```

### 2.2 Sketch tối thiểu (có hoặc không MediatR)

```csharp
public record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<LineItemDto> Items);

public class PlaceOrderHandler(IOrderRepository repo, IUnitOfWork uow)
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Start(cmd.CustomerId);
        // ... build order, áp dụng rule ...
        await repo.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);
        return order.Id;
    }
}

public record GetOrderSummaryQuery(Guid OrderId);
public record OrderSummary(Guid Id, decimal Total, string Status, int ItemCount);

public class GetOrderSummaryHandler(ShopDbContext db)
{
    public Task<OrderSummary?> Handle(GetOrderSummaryQuery q, CancellationToken ct)
        => db.Orders
             .Where(o => o.Id == q.OrderId)
             .Select(o => new OrderSummary(o.Id, o.Total, o.Status.ToString(), o.Items.Count))
             .FirstOrDefaultAsync(ct);
}
```

Lưu ý: **query** không đi qua domain model. Hit DB trực tiếp và project vào read shape.

### 2.3 Biến thể — Từ nhẹ đến nặng

| Flavor                     | Storage                                                | Độ phức tạp |
| -------------------------- | ------------------------------------------------------ | ----------- |
| **Light CQRS**             | Cùng DB; command dùng domain, query project DTO        | ★           |
| **CQRS + Read Model**      | Materialized view / cache cho read                     | ★★          |
| **CQRS + Event Sourcing**  | Event store append-only; projection                    | ★★★★        |

### 2.4 Dùng khi

* Read và write model có **shape thực sự khác**.
* Scale read **lấn át** scale write.
* Muốn introduce **caching, projection, search index** cho read.

### 2.5 Khi *không* dùng

* App CRUD đơn giản.
* Shape read và write giống hệt và nhỏ.
* Team chưa quen — chi phí 2 path vượt lợi ích.

> **Anti-pattern:** *Anemic CQRS* — hai path nhưng model giống hệt. Nhân đôi code không có lợi ích.

(Xem [`../architectural-pattern/cqrs-vi.md`](../architectural-pattern/cqrs-vi.md) cho deep dive.)

---

## 3. Mediator (in-process bus)

> Decouple sender khỏi handler qua **message bus** in-process.

Trong .NET, implementation phổ biến nhất là **MediatR**.

### 3.1 Idea

```csharp
public record PlaceOrderCommand(Guid CustomerId, ...) : IRequest<Guid>;

public class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    public Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct) { ... }
}

// Caller:
public class CheckoutController(IMediator mediator)
{
    [HttpPost]
    public async Task<Guid> Place(PlaceOrderCommand cmd) => await mediator.Send(cmd);
}
```

Controller không biết `PlaceOrderHandler` tồn tại. MediatR route request tới handler đúng.

### 3.2 Pipeline Behavior

MediatR có thể bọc mọi request với cross-cutting concern — logging, validation, transaction, caching:

```csharp
public class LoggingBehavior<TRequest, TResponse>(ILogger<TRequest> log) : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        log.LogInformation("Handling {Request}", typeof(TRequest).Name);
        var response = await next();
        log.LogInformation("Handled {Request}", typeof(TRequest).Name);
        return response;
    }
}
```

Đây là **Decorator** + **Chain of Responsibility** ở mức framework.

### 3.3 Dùng khi

* Đang làm **CQRS** và muốn cơ chế dispatch sạch.
* Muốn **cross-cutting đồng nhất** (validation, logging, transaction) per request.
* Muốn **handler plug-in** khám phá theo convention.

### 3.4 Khi *không* dùng

* Controller đơn giản call service đơn giản. MediatR thêm indirection không lợi ích.
* App CRUD nhỏ. Chỉ inject service.

> **Anti-pattern:** *Mediator-everywhere*. Nếu 100% request đi qua MediatR kể cả khi chỉ có một handler hiển nhiên, bạn thêm layer không mang lại gì.

---

## 4. Result / Either Pattern

> Trả về success-or-failure như **value**, không qua exception.

### 4.1 Idea

Exception dành cho case **exceptional** (invariant thật sự vỡ, infra fail). Cho fail dự đoán được — validation error, business rule violation — trả result.

```csharp
public abstract record Result<TOk, TErr>
{
    public sealed record Ok(TOk Value)   : Result<TOk, TErr>;
    public sealed record Err(TErr Error) : Result<TOk, TErr>;
}

public record OrderError(string Code, string Message);

public class PlaceOrderHandler
{
    public Result<Guid, OrderError> Handle(PlaceOrderCommand cmd)
    {
        if (cmd.Items.Count == 0)
            return new Result<Guid, OrderError>.Err(new("EMPTY_CART", "Cart is empty"));

        // ... happy path ...
        return new Result<Guid, OrderError>.Ok(orderId);
    }
}

// Caller:
return handler.Handle(cmd) switch
{
    Result<Guid, OrderError>.Ok ok    => Ok(ok.Value),
    Result<Guid, OrderError>.Err err  => BadRequest(err.Error),
    _ => throw new UnreachableException()
};
```

### 4.2 Lợi ích

* **Explicit** — signature cho thấy fail có thể.
* **Rẻ hơn exception** — không stack walk.
* **Composable** — chain `.Map / .Bind` cho pipeline success-only.
* **Testable** — assert trên value, không trên exception type.

### 4.3 Lưu ý

Đừng thay **mọi** exception. Reserve Result cho fail **dự đoán được, recoverable**. DB connection null vẫn là exception.

### 4.4 Trong thực tế

Library: `OneOf`, `LanguageExt`, `FluentResults`. Hoặc tự viết — 30 dòng.

---

## 5. Options Pattern

> Configuration strongly-typed, validated, bound.

### 5.1 Tệ — Config Stringly-Typed

```csharp
var key = configuration["Stripe:ApiKey"];   // typo silent trả null
```

### 5.2 Tốt — Options

```csharp
public class StripeOptions
{
    [Required, MinLength(8)] public required string ApiKey { get; init; }
    [Range(1, 60)]            public int RetryAttempts { get; init; } = 3;
}

builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

public class StripeGateway(IOptions<StripeOptions> opts)
{
    private readonly StripeOptions _o = opts.Value;
}
```

### 5.3 Ba Flavor

| Interface             | Khi config thay đổi                        |
| --------------------- | ------------------------------------------ |
| `IOptions<T>`         | Chỉ lúc startup (singleton)                |
| `IOptionsSnapshot<T>` | Per request (scoped)                       |
| `IOptionsMonitor<T>`  | Bất cứ lúc nào; subscribe thay đổi (singleton với callback) |

### 5.4 Lợi ích

* Fail **ở startup** nếu config tệ — không lúc 3 giờ sáng khi feature chạy lần đầu.
* IntelliSense, type safety, refactor-friendly.
* Sạch testable bằng pass `Options.Create(new StripeOptions { ... })`.

---

## 6. Decorator Pipeline (Cross-Cutting Concern)

> Bọc service với layer behavior (logging, retry, caching) không sửa nó.

### 6.1 Pattern

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument instrument, CancellationToken ct);
}

public class StripePaymentGateway : IPaymentGateway { ... }

public class LoggingPaymentGateway(IPaymentGateway inner, ILogger<LoggingPaymentGateway> log) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument inst, CancellationToken ct)
    {
        log.LogInformation("Charging {Amount}", amount);
        var result = await inner.ChargeAsync(amount, inst, ct);
        log.LogInformation("Charge result: {Result}", result);
        return result;
    }
}

public class RetryingPaymentGateway(IPaymentGateway inner) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument inst, CancellationToken ct)
    {
        for (var attempt = 0; attempt < 3; attempt++)
        {
            var r = await inner.ChargeAsync(amount, inst, ct);
            if (r is PaymentResult.Success) return r;
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
        }
        return new PaymentResult.Declined("Out of retries");
    }
}
```

### 6.2 Wiring

```csharp
services.AddScoped<StripePaymentGateway>();
services.AddScoped<IPaymentGateway>(sp =>
    new LoggingPaymentGateway(
        new RetryingPaymentGateway(
            sp.GetRequiredService<StripePaymentGateway>()),
        sp.GetRequiredService<ILogger<LoggingPaymentGateway>>()));
```

Hoặc dùng library như **Scrutor** hỗ trợ `services.Decorate<IPaymentGateway, LoggingPaymentGateway>();`.

### 6.3 Tại sao Decorator thắng code mixed-in

* Class lõi **tập trung** vào trách nhiệm thật.
* Mỗi concern **testable độc lập**.
* Có thể **toggle** concern qua configuration (vd. tắt caching trong test).

> **Pipeline behavior MediatR** ở trên là decorator áp dụng đồng nhất cho mọi request. ASP.NET middleware pipeline là cùng idea ở request boundary.

---

## 7. Factory

> Tập trung logic **construction phức tạp** khi `new T(...)` không đủ.

### 7.1 Khi cần Factory

* Construction cần **quyết định runtime** ("payment gateway nào? Stripe hay PayPal?").
* Construction **đắt** và muốn cache.
* Constructor nhận **dependency caller không nên biết**.

### 7.2 Typed Factory

```csharp
public interface IPaymentGatewayFactory
{
    IPaymentGateway Create(PaymentProvider provider);
}

public class PaymentGatewayFactory(
    IServiceProvider sp) : IPaymentGatewayFactory
{
    public IPaymentGateway Create(PaymentProvider provider) => provider switch
    {
        PaymentProvider.Stripe  => sp.GetRequiredService<StripePaymentGateway>(),
        PaymentProvider.PayPal  => sp.GetRequiredService<PayPalPaymentGateway>(),
        PaymentProvider.ApplePay=> sp.GetRequiredService<ApplePayPaymentGateway>(),
        _ => throw new NotSupportedException()
    };
}
```

Typed factory là dùng Service Locator **chấp nhận được** — bề mặt nhỏ, có chủ ý và sống gần composition root.

### 7.3 `IDbContextFactory<T>` của EF Core

Cho operation chạy dài (background worker, batch job) lifetime per-request `DbContext` sai. `IDbContextFactory<T>` tạo context mới theo nhu cầu.

---

## 8. Strategy at the Seam

> Pattern đơn giản nhất, dùng nhiều nhất: **inject một interface**.

Là pattern **Strategy** của Gang-of-Four, nhưng trong thế giới DI hiện đại bạn hiếm khi *gọi* nó vậy. Bạn chỉ inject `IDiscountRule`, `ITaxPolicy`, `IEmailSender`.

Mọi chỗ bạn muốn `if/else`-by-type, hoặc nơi bạn dự đoán thêm variant, dựa vào đây:

```csharp
public class CheckoutUseCase(IDiscountRule discount, ITaxPolicy tax, IPaymentGateway payment)
{
    // "Strategy" được wire bởi DI; class này không bao giờ biết cái nào nó có.
}
```

Một động tác này enable **OCP** (Open/Closed), **DIP** (Dependency Inversion), và **Protected Variations** (GRASP) — cùng lúc.

---

## 9. Cách chọn

Trước khi adopt pattern nào trong folder này, hỏi:

| Câu hỏi                                              | Nếu có, cân nhắc…              |
| ---------------------------------------------------- | ------------------------------ |
| Rule này sẽ **reuse** qua query và validation?       | Specification                  |
| Read và write model **thực sự khác**?                | CQRS                           |
| Muốn **pipeline đồng nhất** xuyên handler?           | Mediator + behavior            |
| Validation / fail business **thường xuyên dự đoán**? | Result/Either                  |
| Cần **configuration typed, validated**?              | Options                        |
| Cần **cross-cutting concern** không đụng code lõi?   | Decorator                      |
| Construction **phức tạp hoặc có điều kiện runtime**? | Factory                        |
| Behavior sẽ **vary hoặc tăng** tương lai?            | Strategy via DI                |

**Nếu không câu trả lời rõ:** bỏ qua pattern. Có thể introduce sau khi pain thực.

---

## Đọc Cross-Cutting

* [`ioc-di-vi.md`](./ioc-di-vi.md) — mọi pattern ở đây phụ thuộc DI.
* [`data-access-vi.md`](./data-access-vi.md) — Repository / Unit of Work / Lazy / Eager.
* [`../../../principles/solid-vi.md`](../../../principles/solid-vi.md) — nguyên lý mọi pattern phục vụ.
* [`../../../oop/real-project-vi.md`](../../../oop/real-project-vi.md) — nhiều pattern này thể hiện trong walkthrough thật.
* [`../design-pattern/`](../design-pattern/) — GoF pattern cho toolbox cấp class.

---

> **Bottom line:** các pattern này là **trả lời cho pain cụ thể**. Áp dụng khi cảm thấy pain — không phải vì xuất hiện trong slide deck.
