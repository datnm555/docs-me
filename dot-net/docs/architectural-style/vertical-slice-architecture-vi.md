# Vertical Slice Architecture — Giải thích, Sample, Best Approach & Key Point

> Tổ chức code theo **feature**, không phải **layer technical**. Mỗi feature sở hữu mọi thứ nó cần — từ endpoint HTTP đến database — trong một folder.

> Đặt tên và phổ biến trong .NET bởi **Jimmy Bogard** (tác giả MediatR và AutoMapper).

> **Companion doc:**
> * [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md)
> * [`clean-architecture-vi.md`](./clean-architecture-vi.md)
> * [`architecture-comparison-vi.md`](./architecture-comparison-vi.md)

> 🇻🇳 Phiên bản tiếng Việt. English: [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)

---

## Mục lục

1. [Vertical Slice Architecture là gì?](#1-vertical-slice-architecture-là-gì)
2. [Tại sao tồn tại — Pain nó giải](#2-tại-sao-tồn-tại--pain-nó-giải)
3. [Horizontal Layer vs. Vertical Slice](#3-horizontal-layer-vs-vertical-slice)
4. [Nguyên lý cốt lõi](#4-nguyên-lý-cốt-lõi)
5. [Cấu trúc Folder](#5-cấu-trúc-folder)
6. [Sample End-to-End trong .NET](#6-sample-end-to-end-trong-net)
7. [Vai trò của MediatR](#7-vai-trò-của-mediatr)
8. [Pipeline Behavior — Cross-Cutting không cần "Cross-Cutting Layer"](#8-pipeline-behavior--cross-cutting-không-cần-cross-cutting-layer)
9. [Best Approach & Practice khuyến nghị](#9-best-approach--practice-khuyến-nghị)
10. [Ưu điểm](#10-ưu-điểm)
11. [Nhược điểm](#11-nhược-điểm)
12. [Cạm bẫy phổ biến](#12-cạm-bẫy-phổ-biến)
13. [Chiến lược Test](#13-chiến-lược-test)
14. [Vertical Slice vs. Clean / N-Layer](#14-vertical-slice-vs-clean--n-layer)
15. [Hybrid: Clean + Vertical Slice ("Clean Slice")](#15-hybrid-clean--vertical-slice-clean-slice)
16. [Khi nào dùng (và không)](#16-khi-nào-dùng-và-không)
17. [Key Point — Danh sách ngắn](#17-key-point--danh-sách-ngắn)
18. [Checklist](#18-checklist)
19. [Tham khảo](#19-tham-khảo)

---

## 1. Vertical Slice Architecture là gì?

**Vertical slice** là mảnh functionality mỏng, end-to-end cắt qua mọi "layer" của hệ thống — request, validation, business logic, persistence, response — và sống **cùng nhau trong một folder**.

> *"Giảm thiểu coupling giữa slice, và tối đa coupling bên trong slice."* — Jimmy Bogard

Thay vì hỏi *"controller / service / repository sống ở đâu?"*, bạn hỏi *"feature `PlaceOrder` sống ở đâu?"* — câu trả lời: trong một folder tên `PlaceOrder/`.

Kiến trúc là **feature-centric**, không phải technology-centric.

---

## 2. Tại sao tồn tại — Pain nó giải

Kiến trúc N-Layer truyền thống ép một feature đơn lẻ touch nhiều file rải qua nhiều folder:

```
Thêm "PlaceOrder" trong N-Layer:
  Controllers/OrdersController.cs           ← sửa
  Services/IOrderService.cs                 ← sửa
  Services/OrderService.cs                  ← sửa
  Repositories/IOrderRepository.cs          ← sửa
  Repositories/OrderRepository.cs           ← sửa
  Models/OrderDto.cs                        ← sửa
  Validators/OrderValidator.cs              ← sửa
  → 7 file, 4 folder, chỉ để thêm MỘT feature
```

Vấn đề:
- **Cognitive load** — dev nhảy giữa folder xa để hiểu một feature.
- **Refactor risk** — touch service share cho một feature có thể phá feature không liên quan.
- **God service / God controller** — chúng tích luỹ hàng chục method qua thời gian.
- **Abstraction lạm dụng** — mọi feature nhận cùng repository, kể cả khi 90% dùng query duy nhất một lần.

Vertical Slice flip vấn đề: đưa mọi thứ cho `PlaceOrder` vào một folder, chấp nhận chút trùng lặp, và **giảm coupling giữa feature**.

---

## 3. Horizontal Layer vs. Vertical Slice

```
HORIZONTAL (N-Layer):                       VERTICAL (Slice):

┌──────────────────────────────┐            ┌──────┬──────┬──────┬──────┐
│      Presentation            │            │Place │Cancel│ Get  │ List │
├──────────────────────────────┤            │Order │Order │Order │Order │
│      Application             │            │ ───  │ ───  │ ───  │ ───  │
├──────────────────────────────┤            │ Req  │ Req  │ Req  │ Req  │
│      Domain                  │            │ Val  │ Val  │ Val  │ Val  │
├──────────────────────────────┤            │ Hdl  │ Hdl  │ Hdl  │ Hdl  │
│      Data Access             │            │ Res  │ Res  │ Res  │ Res  │
└──────────────────────────────┘            └──────┴──────┴──────┴──────┘
   một feature = nhiều folder                một feature = một folder
```

Trong codebase vertical-slice, xoá một feature là thao tác single-folder. Thêm feature touch gần như không code share.

---

## 4. Nguyên lý cốt lõi

1. **Group theo feature, không layer.** Một folder per use case.
2. **Tối thiểu coupling giữa slice.** Slice không nên call lẫn nhau trực tiếp.
3. **Tối đa cohesion bên trong slice.** Mọi thứ feature cần sống cùng nhau.
4. **Tolerate trùng lặp.** Một số mapping hoặc query trùng lặp rẻ hơn abstraction sai.
5. **Dùng tool đơn giản nhất giải slice.** Một slice có thể dùng Dapper, slice khác EF Core, slice khác raw SQL — bất cứ gì fit.
6. **Cross-cutting qua pipeline behavior**, không phải base class hay service wrapper.
7. **Refactor về abstraction *sau* instance thứ ba, không trước.**

---

## 5. Cấu trúc Folder

```
src/
├── Shop.Api/
│   ├── Program.cs
│   └── Features/
│       ├── Orders/
│       │   ├── PlaceOrder/
│       │   │   ├── PlaceOrderCommand.cs
│       │   │   ├── PlaceOrderHandler.cs
│       │   │   ├── PlaceOrderValidator.cs
│       │   │   ├── PlaceOrderEndpoint.cs
│       │   │   └── PlaceOrderResponse.cs
│       │   ├── CancelOrder/
│       │   ├── GetOrder/
│       │   └── ListOrders/
│       ├── Customers/
│       └── Products/
│
├── Shop.Domain/                ← (tuỳ chọn) entity/value object share
│   ├── Orders/
│   ├── Customers/
│   └── ValueObjects/
│
└── Shop.Infrastructure/         ← (tuỳ chọn) infra cross-cutting share
    ├── Persistence/
    │   └── AppDbContext.cs
    └── Behaviors/               ← MediatR pipeline behavior
        ├── ValidationBehavior.cs
        ├── LoggingBehavior.cs
        └── TransactionBehavior.cs
```

> Lưu ý: trong setup *strict* vertical slice, kể cả `Shop.Domain` là tuỳ chọn — mỗi slice có thể định nghĩa model riêng. Trong thực tế, phần lớn team giữ `Domain` share mỏng để host entity dùng bởi nhiều slice (vd., `Order`, `Customer`).

---

## 6. Sample End-to-End trong .NET

Dưới đây là slice feature đơn — **`PlaceOrder`** — implement trong một folder.

### 6.1 Command

```csharp
// Features/Orders/PlaceOrder/PlaceOrderCommand.cs
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<PlaceOrderLine> Lines
) : IRequest<Result<Guid>>;

public sealed record PlaceOrderLine(Guid ProductId, int Quantity, decimal UnitPrice);
```

### 6.2 Validator

```csharp
// Features/Orders/PlaceOrder/PlaceOrderValidator.cs
public sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.ProductId).NotEmpty();
            line.RuleFor(l => l.Quantity).GreaterThan(0);
            line.RuleFor(l => l.UnitPrice).GreaterThanOrEqualTo(0);
        });
    }
}
```

### 6.3 Handler

```csharp
// Features/Orders/PlaceOrder/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler
    : IRequestHandler<PlaceOrderCommand, Result<Guid>>
{
    private readonly AppDbContext _db;

    public PlaceOrderHandler(AppDbContext db) => _db = db;

    public async Task<Result<Guid>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await _db.Customers.FindAsync([cmd.CustomerId], ct);
        if (customer is null) return Result.Fail<Guid>("Customer not found.");

        var order = new Order(cmd.CustomerId);
        foreach (var line in cmd.Lines)
            order.AddLine(line.ProductId, line.Quantity, line.UnitPrice);

        order.Place();

        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        return Result.Ok(order.Id);
    }
}
```

> Chú ý: không interface repository, không service layer, không class mapping riêng. Handler là slice — làm chính xác cái `PlaceOrder` cần và không hơn. Nếu slice khác cần behavior phong phú hơn trên `Order`, entity grow; nếu slice khác cần query khác, có handler riêng.

### 6.4 Endpoint (minimal API)

```csharp
// Features/Orders/PlaceOrder/PlaceOrderEndpoint.cs
public static class PlaceOrderEndpoint
{
    public static IEndpointRouteBuilder MapPlaceOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (
            PlaceOrderCommand cmd, ISender mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value}", result.Value)
                : Results.BadRequest(result.Error);
        });
        return app;
    }
}
```

### 6.5 Composition trong Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);

builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));

var app = builder.Build();

app.MapPlaceOrder();
app.MapCancelOrder();
app.MapGetOrder();
app.MapListOrders();

app.Run();
```

Thêm feature mới = một folder mới + một call `MapXxx()`. Đó là tất cả.

---

## 7. Vai trò của MediatR

MediatR cung cấp **mediator in-process** dispatch request tới handler của nó. Trong Vertical Slice, nó là glue:

```
Endpoint ──Send(command)──► MediatR ──► PlaceOrderHandler
```

Tại sao fit tốt:

- **Endpoint giữ mỏng** — không reference handler trực tiếp; chỉ send `Command`.
- **Một handler per feature** = một slice. Không reuse handler cross-feature.
- **Pipeline behavior** bọc mọi handler với cross-cutting concern (logging, validation, transaction, caching) mà không thêm layer.
- **CQRS miễn phí** — command và query chỉ là `IRequest<T>` type khác nhau.

> MediatR không bắt buộc. Cùng pattern work với raw handler resolution từ DI container. Nhưng MediatR + Vertical Slice là combination canonical trong .NET.

---

## 8. Pipeline Behavior — Cross-Cutting không cần "Cross-Cutting Layer"

Pipeline behavior là MediatR middleware. Nó chạy quanh mọi handler.

```csharp
// Infrastructure/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(context, ct))))
                .SelectMany(r => r.Errors)
                .Where(f => f is not null)
                .ToList();
            if (failures.Count > 0) throw new ValidationException(failures);
        }
        return await next();
    }
}
```

Behavior phổ biến: validation, logging, transaction, caching, performance, audit, retry.

---

## 9. Best Approach & Practice khuyến nghị

1. **Một folder per use case.** Tên folder = tên feature.
2. **Đặt mọi thứ feature cần trong folder:** command, handler, validator, endpoint, response, mapping riêng nếu cần.
3. **Dùng MediatR** cho dispatch — endpoint mỏng, handler tập trung.
4. **Pipeline behavior** cho cross-cutting — validation, logging, transaction.
5. **DbContext trực tiếp trong handler** — không repository giả tạo. Mỗi handler tự quyết định cách truy cập data.
6. **Tolerate trùng lặp** — không refactor về abstraction quá sớm.
7. **Per-slice persistence choice** — slice này EF, slice kia Dapper, slice khác raw SQL — fit-for-purpose.
8. **DTO/Command/Response trong slice** — không chia sẻ qua slice.
9. **Result<T>** cho fail business dự đoán; exception cho cái thật exceptional.
10. **CancellationToken end-to-end.**
11. **Architecture test:** một slice không nên reference slice khác (chỉ Domain share và Infrastructure).
12. **Domain shared mỏng:** chỉ entity và value object dùng bởi nhiều slice.
13. **Endpoint extension method** đặt tên `MapXxx()` đăng ký gần command.
14. **Source generator** (Mapperly) hơn AutoMapper khi cần map nhiều.

---

## 10. Ưu điểm

✅ **High cohesion bên trong feature** — mọi thứ liên quan ở một chỗ.
✅ **Low coupling giữa feature** — thay đổi một slice ít rủi ro phá slice khác.
✅ **Dễ xoá** — xoá folder, xoá feature.
✅ **Dễ onboard** — dev mới tìm feature một chỗ, hiểu nhanh.
✅ **No god class** — không có service hoặc controller tích luỹ trách nhiệm.
✅ **Tech-flexible per slice** — chọn ORM/storage tối ưu cho từng slice.
✅ **Pair tốt với CQRS và MediatR.**
✅ **Scale well với team** — feature team tránh đụng nhau.

---

## 11. Nhược điểm

❌ **Trùng lặp** — một số code lặp qua slice (mapping, query base).
❌ **Yếu enforce cross-cutting rule** — không có cấu trúc layer cứng.
❌ **Khó tìm "domain logic chung"** khi nó thực sự cần share.
❌ **Vẫn cần Domain layer share** cho entity dùng bởi nhiều slice.
❌ **Phụ thuộc nặng vào MediatR** (hoặc tương đương).
❌ **Có thể grow vô tổ chức** nếu không có quy ước đặt tên feature.

---

## 12. Cạm bẫy phổ biến

| Cạm bẫy                                                  | Tại sao tệ                                                                              |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Slice call slice khác trực tiếp**                       | Reintroduce coupling. Dùng domain event hoặc shared domain.                              |
| **God folder** — slice 30 file                            | Slice nên ngắn. Tách thành nhiều slice nếu phức tạp.                                     |
| **Refactor abstraction sớm**                              | Trừ khi có 3+ instance, để code trùng lặp.                                                |
| **Domain anemic** — handler làm hết logic                 | Push behavior vào entity; handler là orchestrator, không doer.                            |
| **DbContext leak vào endpoint**                           | Pass command, không entity. Giữ endpoint mỏng.                                            |
| **Quên ValidationBehavior**                              | Mỗi handler tự validate — trùng lặp. Đặt behavior trong pipeline.                         |

---

## 13. Chiến lược Test

| Layer test            | Mục tiêu                                  | Tool                                                 |
| --------------------- | ----------------------------------------- | ---------------------------------------------------- |
| **Handler unit**      | Logic của handler                          | xUnit + in-memory DbContext hoặc fake                |
| **Handler integration** | Handler + DB thật                        | xUnit + Testcontainers                              |
| **Endpoint functional** | HTTP đầy đủ qua `WebApplicationFactory`   | xUnit + Testcontainers                              |
| **Validator**         | Rule validation                           | xUnit                                                |

Test một slice = test một feature. Đơn giản, tập trung.

---

## 14. Vertical Slice vs. Clean / N-Layer

| Khía cạnh           | **N-Layer**                | **Clean**                                          | **Vertical Slice**                                |
| ------------------- | -------------------------- | -------------------------------------------------- | ------------------------------------------------- |
| Tổ chức code         | Theo layer technical       | Theo layer concentric                              | Theo feature                                       |
| Couple giữa feature  | Cao (qua service share)    | Trung bình (qua use case + abstraction)            | Thấp                                                |
| Cohesion trong feature | Thấp (rải nhiều folder)  | Trung bình                                          | Cao (một folder)                                   |
| Số file per feature  | Nhiều                      | Nhiều                                              | Ít                                                 |
| Tolerate trùng lặp    | Không                      | Không                                              | Có                                                 |
| Phù hợp với          | App CRUD ổn định           | Domain phức tạp, longevity cao                     | App với nhiều feature độc lập, team feature-based |

---

## 15. Hybrid: Clean + Vertical Slice ("Clean Slice")

Layout phổ biến: Clean Architecture's *outer* layer (Web, Infrastructure, Domain) + Vertical Slice *bên trong* Application layer.

```
MyApp.sln
│
├── MyApp.Domain/             ← Entity, Value Object, Domain Service
│
├── MyApp.Application/        ← Vertical slice bên trong
│   ├── Abstractions/
│   └── Features/
│       ├── PlaceOrder/
│       ├── CancelOrder/
│       └── GetOrder/
│
├── MyApp.Infrastructure/     ← EF Core, external API
│
└── MyApp.Web/                ← ASP.NET Core API
```

Nhận được:
- **Boundary cứng** từ Clean (Domain không reference Infrastructure).
- **Cohesion cao** từ Vertical Slice (mỗi feature một folder bên trong Application).
- Pair tốt với DDD: Domain Layer giữ entity share, Application Layer organize per-feature.

Đây thường được khuyến nghị làm default cho project .NET hiện đại (~2023+).

Xem [`clean-slice-architecture-vi.md`](./clean-slice-architecture-vi.md) cho deep dive.

---

## 16. Khi nào dùng (và không)

### ✅ Phù hợp

* App với **nhiều feature độc lập** (CRUD app, admin tool có nhiều screen).
* **Team feature-based** — mỗi team owns feature folder.
* App có **lifecycle feature thường xuyên** (add, delete, modify).
* Microservice xử lý **một bounded context với nhiều operation**.

### ⚠️ Xem xét lại khi

* Domain phong phú với business rule phức tạp cross-feature → Clean + DDD.
* App nhỏ với 1-2 feature → N-Layer là đủ.
* Cần rule cross-cutting strict không thoả hiệp → Clean với layer.

---

## 17. Key Point — Danh sách ngắn

* **Một folder per feature** — high cohesion, low coupling.
* **MediatR + pipeline behavior** = canonical .NET setup.
* **Tolerate trùng lặp** — đừng refactor về abstraction sớm.
* **DbContext trực tiếp** trong handler — không repository giả.
* **Dễ xoá** = feature dễ thay đổi.
* **Pair với Clean** thành "Clean Slice" cho default hiện đại.

---

## 18. Checklist

- [ ] Mỗi feature có folder riêng với command/handler/validator/endpoint.
- [ ] Slice không call slice khác trực tiếp.
- [ ] Cross-cutting concern qua pipeline behavior.
- [ ] DbContext access trực tiếp trong handler (không repository giả).
- [ ] Result<T> cho fail business; exception cho exceptional.
- [ ] CancellationToken end-to-end.
- [ ] Architecture test verify slice không reference slice khác.
- [ ] Tên folder mô tả feature, không technology.
- [ ] Endpoint extension method `MapXxx()` cùng folder với command.
- [ ] Test handler unit và endpoint functional.

---

## 19. Tham khảo

* [Jimmy Bogard — Vertical Slice Architecture (2018 blog post)](https://www.jimmybogard.com/vertical-slice-architecture/)
* [Microsoft Learn — CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
* [GitHub — Jimmy Bogard's ContosoUniversity demo](https://github.com/jbogard/ContosoUniversityCore)
* [MediatR documentation](https://github.com/jbogard/MediatR)
* [antondevtips — N-Layered vs Clean vs Vertical Slice](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
