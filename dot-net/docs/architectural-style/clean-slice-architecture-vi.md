# Clean Slice Architecture — Best Practice, Best Approach, Cấu trúc & Key Point

> Hybrid thực dụng: **Clean Architecture bên ngoài, Vertical Slice bên trong.**
> Clean enforce *Dependency Rule* qua boundary project; Vertical Slice tổ chức layer *Application* theo feature.

> Layout .NET hiện đại được khuyến nghị nhất 2025/2026.

> **Companion doc:**
> * [`clean-architecture-vi.md`](./clean-architecture-vi.md)
> * [`vertical-slice-architecture-vi.md`](./vertical-slice-architecture-vi.md)
> * [`five-architectures-comparison-and-mixing-vi.md`](./five-architectures-comparison-and-mixing-vi.md)

> 🇻🇳 Phiên bản tiếng Việt. English: [`clean-slice-architecture.md`](./clean-slice-architecture.md)

---

## Mục lục

1. [Clean Slice là gì?](#1-clean-slice-là-gì)
2. [Tại sao tồn tại — Pain mỗi bên giải](#2-tại-sao-tồn-tại--pain-mỗi-bên-giải)
3. [Nguyên lý cốt lõi](#3-nguyên-lý-cốt-lõi)
4. [Diagram Architectural](#4-diagram-architectural)
5. [Cấu trúc Solution](#5-cấu-trúc-solution)
6. [Project Reference Map](#6-project-reference-map)
7. [Anatomy của một Slice](#7-anatomy-của-một-slice)
8. [Code Sample End-to-End](#8-code-sample-end-to-end)
9. [Pipeline Behavior](#9-pipeline-behavior)
10. [Wire Dependency Injection](#10-wire-dependency-injection)
11. [CQRS bên trong Slice](#11-cqrs-bên-trong-slice)
12. [Domain Layer Discipline](#12-domain-layer-discipline)
13. [Infrastructure Layer Discipline](#13-infrastructure-layer-discipline)
14. [Best Practice](#14-best-practice)
15. [Best Approach — Quyết định quan trọng](#15-best-approach--quyết-định-quan-trọng)
16. [Cạm bẫy phổ biến](#16-cạm-bẫy-phổ-biến)
17. [Chiến lược Test](#17-chiến-lược-test)
18. [Architecture Test](#18-architecture-test)
19. [Khi nào dùng (và không)](#19-khi-nào-dùng-và-không)
20. [Migration Path](#20-migration-path)
21. [Key Point — Danh sách ngắn](#21-key-point--danh-sách-ngắn)
22. [Checklist](#22-checklist)
23. [Tham khảo](#23-tham-khảo)

---

## 1. Clean Slice là gì?

**Clean Slice Architecture** là hybrid thực dụng kết hợp:

- **Clean Architecture** cho *cấu trúc ngoài* — project, layer, dependency rule.
- **Vertical Slice Architecture** cho *tổ chức nội bộ của Application layer* — một folder per use case.

Tóm gọn:

> **Clean định nghĩa ai được phép phụ thuộc ai.**
> **Vertical Slice định nghĩa feature tổ chức bên trong ra sao.**

Bạn nhận được:
- **Dependency Rule** enforce bởi project reference (Clean).
- **Cohesion feature** bên trong `Application/Features/` (Vertical Slice).
- **Độc lập framework** của Domain (Clean).
- **Coupling cross-feature thấp** giữa use case (Vertical Slice).

Pair tự nhiên với **MediatR + FluentValidation + Pipeline behavior + CQRS** cho dispatch layer use-case.

---

## 2. Tại sao tồn tại — Pain mỗi bên giải

### Pain của Vertical Slice thuần

- Không enforce layer — slice có thể accidentally phụ thuộc EF Core trực tiếp.
- Domain logic drift vào handler; entity thành anemic.
- Rule cross-cutting (vd., "mọi command publish event") khó enforce toàn cục.
- Không nhà rõ cho entity và value object share domain.

### Pain của Clean Architecture thuần

- Mỗi feature touch nhiều folder (`Domain`, `Application`, `Infrastructure`, `Web`).
- Boilerplate: file interface riêng, class service, class mapping cho mỗi feature.
- Trở thành "god Application project" khi feature nhân.
- Refactor một feature ripple qua nhiều file.

### Cái gì Clean Slice fix

- **Vòng ngoài Clean** giữ domain thuần và dependency rule enforce.
- **Folder feature Vertical Slice** giữ mỗi use case cohesive và dễ tìm/xoá.
- Dev mới đọc một folder để hiểu một feature.
- Cross-cutting concern đi qua pipeline behavior thay vì layer ad-hoc.

---

## 3. Nguyên lý cốt lõi

1. **Dependency Rule không thương lượng.** Chỉ vào trong: `Web → Infrastructure → Application → Domain`.
2. **Domain framework-free.** Không EF Core, không ASP.NET, không MediatR — C# thuần.
3. **Application layer tổ chức theo feature**, không technical type (không folder `Services/`, không `Repositories/` ở mức Application).
4. **Mỗi slice sở hữu command, validator, handler, DTO response, và (tuỳ chọn) endpoint.**
5. **Cross-cutting concern sống trong MediatR pipeline behavior**, không base class hoặc service wrapper.
6. **Tolerate trùng lặp giữa slice.** Refactor bằng extract về Domain khi pattern ổn định (rule of three).
7. **Infrastructure thay thế được.** EF Core, SendGrid, Redis là detail đằng sau interface.
8. **Endpoint humble.** Dịch HTTP ↔ Command/Query và không gì hơn.
9. **Slice không call slice khác.** Share **Domain** (entity, value object) hoặc giao tiếp qua **event**.
10. **Per-slice technology freedom** — read query nặng có thể dùng Dapper; write command có thể dùng EF Core. Cả hai trong `Application`, nhưng lựa chọn local cho slice.

---

## 4. Diagram Architectural

```
                      ┌──────────────────────────────────────────────────┐
                      │                    Web                           │
                      │   (ASP.NET Core endpoint — một per feature)      │
                      └─────────────────────────┬────────────────────────┘
                                                │ phụ thuộc
                                                ▼
   ┌────────────────────────────────────────────────────────────────────────────┐
   │                              Application                                    │
   │                                                                             │
   │  ┌─────────────────────────────────────────────────────────────────────┐    │
   │  │                            Features/                               │    │
   │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │    │
   │  │  │ Orders/      │  │ Customers/   │  │ Catalog/     │   …          │    │
   │  │  │ ┌──────────┐ │  │              │  │              │              │    │
   │  │  │ │PlaceOrder│ │  │              │  │              │              │    │
   │  │  │ │ Cmd/Val/H│ │  │              │  │              │              │    │
   │  │  │ └──────────┘ │  │              │  │              │              │    │
   │  │  └──────────────┘  └──────────────┘  └──────────────┘              │    │
   │  └─────────────────────────────────────────────────────────────────────┘    │
   │                                                                             │
   │  Abstractions/  ← interface consume bởi slice (IUnitOfWork, IClock…)      │
   │  Behaviors/     ← MediatR pipeline behavior                                │
   └─────────────────────────┬───────────────────────────────────────────────┘
                             │ phụ thuộc
                             ▼
                      ┌──────────────────────────────────────────────────┐
                      │                    Domain                        │
                      │   Entity • Value Object • Domain Event           │
                      │   (C# thuần — không dep framework)               │
                      └──────────────────────────────────────────────────┘
                             ▲
                             │ implement abstraction
                      ┌──────┴──────────────────────────────────────────┐
                      │                Infrastructure                    │
                      │  EF Core • Cache • Email • External API         │
                      └──────────────────────────────────────────────────┘
```

> Application layer reference Domain. Infrastructure reference Application + Domain để implement abstraction Application declare. Web là composition root.

---

## 5. Cấu trúc Solution

```
src/
├── Shop.Domain/                              ← Domain thuần (trong cùng)
│   ├── Orders/
│   │   ├── Order.cs
│   │   ├── OrderLine.cs
│   │   ├── OrderStatus.cs
│   │   └── Events/OrderPlaced.cs
│   ├── Customers/
│   │   └── Customer.cs
│   ├── ValueObjects/
│   │   ├── Money.cs
│   │   └── EmailAddress.cs
│   ├── Exceptions/
│   │   └── DomainException.cs
│   └── Abstractions/
│       └── IClock.cs                         ← port thuần định nghĩa bởi Domain
│
├── Shop.Application/                         ← Slice use-case sống ở đây
│   ├── Features/
│   │   ├── Orders/
│   │   │   ├── PlaceOrder/
│   │   │   │   ├── PlaceOrderCommand.cs
│   │   │   │   ├── PlaceOrderValidator.cs
│   │   │   │   ├── PlaceOrderHandler.cs
│   │   │   │   └── PlaceOrderResponse.cs
│   │   │   ├── CancelOrder/
│   │   │   ├── GetOrder/
│   │   │   └── ListOrders/
│   │   ├── Customers/
│   │   │   ├── RegisterCustomer/
│   │   │   └── GetCustomer/
│   │   └── Catalog/
│   │       └── ListProducts/
│   │
│   ├── Abstractions/                         ← Port định nghĩa bởi Application
│   │   ├── IOrderRepository.cs
│   │   ├── ICustomerRepository.cs
│   │   ├── IUnitOfWork.cs
│   │   ├── IEmailSender.cs
│   │   ├── ICurrentUser.cs
│   │   └── IEventPublisher.cs
│   │
│   ├── Behaviors/                            ← MediatR pipeline behavior
│   │   ├── ValidationBehavior.cs
│   │   ├── LoggingBehavior.cs
│   │   ├── TransactionBehavior.cs
│   │   ├── PerformanceBehavior.cs
│   │   └── CachingBehavior.cs
│   │
│   ├── Common/                               ← Result<T>, error, DTO chung
│   │   ├── Result.cs
│   │   └── Errors.cs
│   │
│   └── DependencyInjection.cs
│
├── Shop.Infrastructure/                      ← Adapter (concern ngoài cùng)
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   ├── Configurations/
│   │   │   ├── OrderConfiguration.cs
│   │   │   └── CustomerConfiguration.cs
│   │   ├── Migrations/
│   │   └── Repositories/
│   │       ├── EfOrderRepository.cs
│   │       └── EfCustomerRepository.cs
│   ├── Email/SendGridEmailSender.cs
│   ├── Messaging/AzureServiceBusPublisher.cs
│   ├── Identity/CurrentUser.cs
│   ├── Time/SystemClock.cs
│   └── DependencyInjection.cs
│
└── Shop.Web/                                 ← ASP.NET Core (composition root)
    ├── Endpoints/
    │   ├── Orders/
    │   │   ├── PlaceOrderEndpoint.cs
    │   │   ├── CancelOrderEndpoint.cs
    │   │   ├── GetOrderEndpoint.cs
    │   │   └── ListOrdersEndpoint.cs
    │   └── Customers/…
    ├── Middleware/ExceptionMiddleware.cs
    ├── Program.cs
    └── appsettings.json

tests/
├── Shop.Domain.UnitTests/
├── Shop.Application.UnitTests/
├── Shop.Infrastructure.IntegrationTests/
└── Shop.Web.FunctionalTests/
```

> Endpoint cũng có thể sống **bên trong folder slice** (vd., `Application/Features/Orders/PlaceOrder/PlaceOrderEndpoint.cs`) nếu muốn co-location thật. Cả hai layout hợp lệ; chọn một và nhất quán.

---

## 6. Project Reference Map

```
Shop.Web            → Shop.Application + Shop.Infrastructure + Shop.Domain
Shop.Infrastructure → Shop.Application + Shop.Domain
Shop.Application    → Shop.Domain
Shop.Domain         → (không gì)
```

Domain không reference gì. Application chỉ reference Domain. Infrastructure implement abstraction Application declare. Web là composition root, reference mọi thứ.

---

## 7. Anatomy của một Slice

Một slice tiêu biểu có:
- **Command/Query** — record IRequest<T>.
- **Validator** — FluentValidation, kiểm tra input.
- **Handler** — IRequestHandler<TReq, TRes>. Orchestrate domain, gọi abstraction.
- **Response** — DTO trả về.
- **Endpoint** — extension method MapXxx() đăng ký HTTP route.

Mỗi file gần nhau trong folder slice. Slice tự đủ ngữ cảnh.

---

## 8. Code Sample End-to-End

### Command

```csharp
// Application/Features/Orders/PlaceOrder/PlaceOrderCommand.cs
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<PlaceOrderLine> Lines
) : IRequest<Result<Guid>>;

public sealed record PlaceOrderLine(Guid ProductId, int Quantity, decimal UnitPrice);
```

### Validator

```csharp
// Application/Features/Orders/PlaceOrder/PlaceOrderValidator.cs
public sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.Quantity).GreaterThan(0);
            line.RuleFor(l => l.UnitPrice).GreaterThanOrEqualTo(0);
        });
    }
}
```

### Handler

```csharp
// Application/Features/Orders/PlaceOrder/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Result<Guid>>
{
    private readonly IOrderRepository _orders;
    private readonly ICustomerRepository _customers;
    private readonly IUnitOfWork _uow;

    public PlaceOrderHandler(IOrderRepository orders, ICustomerRepository customers, IUnitOfWork uow)
    {
        _orders = orders; _customers = customers; _uow = uow;
    }

    public async Task<Result<Guid>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await _customers.GetAsync(cmd.CustomerId, ct);
        if (customer is null) return Result.Fail<Guid>("Customer not found.");

        var order = new Order(cmd.CustomerId);
        foreach (var line in cmd.Lines)
            order.AddLine(line.ProductId, line.Quantity, line.UnitPrice);
        order.Place();

        await _orders.AddAsync(order, ct);
        await _uow.SaveChangesAsync(ct);
        return Result.Ok(order.Id);
    }
}
```

### Endpoint

```csharp
// Web/Endpoints/Orders/PlaceOrderEndpoint.cs (hoặc cùng folder slice)
public static class PlaceOrderEndpoint
{
    public static IEndpointRouteBuilder MapPlaceOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (PlaceOrderCommand cmd, ISender mediator, CancellationToken ct) =>
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

### Composition Root

```csharp
// Web/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);

var app = builder.Build();
app.UseExceptionHandling();
app.MapPlaceOrder();
app.MapCancelOrder();
// … map các endpoint khác
app.Run();
```

---

## 9. Pipeline Behavior

Pipeline behavior bọc mọi handler. Chúng đặt cross-cutting concern ở một chỗ thay vì rải qua handler.

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors).Where(f => f is not null).ToList();
        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}
```

Behavior phổ biến: Validation, Logging, Transaction, Performance, Caching, Audit, Retry.

---

## 10. Wire Dependency Injection

```csharp
// Application/DependencyInjection.cs
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(IApplicationMarker).Assembly));
    services.AddValidatorsFromAssembly(typeof(IApplicationMarker).Assembly);
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
    return services;
}

// Infrastructure/DependencyInjection.cs
public static IServiceCollection AddInfrastructure(this IServiceCollection s, IConfiguration cfg)
{
    s.AddDbContext<AppDbContext>(o => o.UseSqlServer(cfg.GetConnectionString("Default")));
    s.AddScoped<IOrderRepository, EfOrderRepository>();
    s.AddScoped<ICustomerRepository, EfCustomerRepository>();
    s.AddScoped<IUnitOfWork, EfUnitOfWork>();
    s.AddScoped<IEmailSender, SendGridEmailSender>();
    s.AddSingleton<IClock, SystemClock>();
    return s;
}
```

---

## 11. CQRS bên trong Slice

Command và query đều dùng `IRequest<T>`. Sự khác biệt:

**Command (write):**
- Đi qua domain model — load aggregate, mutate, save.
- Wrap trong transaction (TransactionBehavior).
- Có thể raise domain event.

**Query (read):**
- Bypass domain — project trực tiếp từ DB sang DTO.
- Có thể dùng Dapper hoặc EF projection cho performance.
- Không transaction (read-only).

```csharp
// Query slice: Application/Features/Orders/GetOrder/
public record GetOrderQuery(Guid Id) : IRequest<OrderDto?>;

public sealed class GetOrderHandler(AppDbContext db) : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public Task<OrderDto?> Handle(GetOrderQuery q, CancellationToken ct) =>
        db.Orders.Where(o => o.Id == q.Id)
            .Select(o => new OrderDto(o.Id, o.Status.ToString(), o.Total))
            .FirstOrDefaultAsync(ct);
}
```

Query handler có thể dùng `DbContext` trực tiếp — không cần repository abstraction.

---

## 12. Domain Layer Discipline

Domain phải:

- **C# thuần** — không NuGet reference framework.
- **Entity với behavior** — method enforce invariant.
- **Value object immutable** — equality theo value.
- **Domain event** — raise khi business fact xảy ra.
- **Domain service** chỉ khi behavior không tự nhiên fit entity.
- **Không leak persistence** — không EF attribute, không lazy-load proxy.

Test Domain với pure xUnit, không mock.

---

## 13. Infrastructure Layer Discipline

Infrastructure phải:

- **Implement abstraction** define bởi Application.
- **Một file per concern external** — `EfOrderRepository`, `SendGridEmailSender`, `AzureServiceBusPublisher`.
- **Internal classes** trừ method `AddInfrastructure(...)` extension.
- **EF Core configuration** trong folder Configurations.
- **Migration** trong project Infrastructure; chạy ở startup (dev) hoặc one-shot job (prod).
- **Không business logic** — chỉ glue tới technology bên ngoài.

---

## 14. Best Practice

1. **Folder per slice.** Tên = tên use case.
2. **Endpoint co-location.** Đặt MapXxx() gần command, hoặc trong folder `Web/Endpoints/` rõ ràng.
3. **Pipeline behavior** cho cross-cutting — không base class, không service wrapper.
4. **Result<T>** cho fail business; exception cho exceptional thật.
5. **CancellationToken** end-to-end.
6. **Architecture test** enforce Dependency Rule.
7. **Domain event** cho cross-aggregate side effect.
8. **Outbox pattern** cho integration event (DB + message bus atomic).
9. **Per-slice technology choice** — EF Core write, Dapper read, raw SQL bulk.
10. **Slice không call slice khác** — share Domain hoặc event.
11. **Tolerate trùng lặp** — rule of three trước refactor về abstraction.
12. **Source generator** (Mapperly) hơn AutoMapper.
13. **Health check** + global exception middleware + structured logging.
14. **Configuration validate ở startup** với `IOptions<T>` + `ValidateOnStart()`.

---

## 15. Best Approach — Quyết định quan trọng

- **Endpoint trong Web/ hay Application/?** Phần lớn team đặt trong Web/Endpoints/ để Application không reference ASP.NET. Nếu chấp nhận Application reference `Microsoft.AspNetCore.Http` (cho `IResult`), có thể co-location trong slice.
- **Repository hay DbContext trực tiếp trong handler?** Cho write, repository (tách Domain). Cho read, DbContext trực tiếp OK (CQRS phân kỳ).
- **Một DbContext hay nhiều?** Một cho phần lớn app. Nhiều nếu có module với schema/connection riêng.
- **AutoMapper hay Mapperly?** Mapperly (source generator) cho AOT-ready và performance. AutoMapper khi mapping động cần thiết.
- **MediatR hay không?** MediatR cho pipeline behavior tự động. Không MediatR nếu app nhỏ và không cần cross-cutting.

---

## 16. Cạm bẫy phổ biến

| Cạm bẫy                                                  | Tại sao tệ                                                                              |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **EF entity trong Domain layer**                         | Phá purity của Domain. Dùng separate persistence model nếu cần.                          |
| **Generic repository everywhere**                        | Defeat boundary feature. Ưu tiên repository specific per aggregate.                      |
| **Slice call slice trực tiếp**                           | Reintroduce coupling. Share Domain hoặc event.                                            |
| **Endpoint chứa logic**                                  | Endpoint humble. Move logic vào handler.                                                  |
| **Cross-cutting trong handler thay vì pipeline behavior**| Trùng lặp khắp slice. Move vào behavior.                                                  |
| **Anemic domain — handler làm hết logic**               | Push behavior vào entity. Handler orchestrate, không doer.                                |
| **God Application project**                              | Quá nhiều feature trong một project. Split theo bounded context.                          |
| **Quên architecture test**                               | Dependency Rule drift theo thời gian không kiểm soát.                                     |

---

## 17. Chiến lược Test

| Layer test            | Mục tiêu                                  | Tool                                                 |
| --------------------- | ----------------------------------------- | ---------------------------------------------------- |
| **Domain unit**       | Entity, value object, domain logic         | xUnit + FluentAssertions, không mock                |
| **Application unit**  | Handler, validator                        | xUnit + NSubstitute mock abstraction                 |
| **Infrastructure integration** | Repository thật + DB              | xUnit + Testcontainers                              |
| **Web functional**    | API đầy đủ qua `WebApplicationFactory`    | xUnit + Testcontainers                              |
| **Architecture**      | Dependency Rule                            | NetArchTest / ArchUnitNET                            |

---

## 18. Architecture Test

```csharp
[Fact]
public void Domain_must_not_reference_anything()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .Should()
        .NotHaveDependencyOnAny(
            "Shop.Application", "Shop.Infrastructure", "Shop.Web",
            "Microsoft.EntityFrameworkCore", "MediatR")
        .GetResult();
    Assert.True(result.IsSuccessful);
}

[Fact]
public void Application_must_not_reference_Infrastructure_or_Web()
{
    var result = Types.InAssembly(typeof(IApplicationMarker).Assembly)
        .Should()
        .NotHaveDependencyOnAny("Shop.Infrastructure", "Shop.Web", "Microsoft.EntityFrameworkCore")
        .GetResult();
    Assert.True(result.IsSuccessful);
}
```

---

## 19. Khi nào dùng (và không)

### ✅ Phù hợp

* App .NET hiện đại với nhiều use case.
* Domain phong phú (DDD) + workflow CRUD-heavy mix.
* Team thoải mái với MediatR + CQRS + Result<T>.
* Hệ thống sống lâu cần testability và maintainability.

### ⚠️ Xem xét lại khi

* App rất nhỏ (5 endpoint) — N-Layer đủ.
* Team chưa biết MediatR — khởi đầu Clean Architecture thuần.
* App không có domain logic phức tạp — Vertical Slice thuần đủ.

---

## 20. Migration Path

### N-Layer → Clean Slice

1. Extract Domain ra project riêng, không reference EF.
2. Tạo Application project, move command/handler/validator/response vào Features/.
3. Define abstraction trong Application/Abstractions/.
4. Move EF Core, external client vào Infrastructure.
5. Convert controller thành minimal API endpoint mỏng.
6. Thêm MediatR + pipeline behavior.
7. Thêm architecture test.

### Vertical Slice → Clean Slice

1. Extract Domain (entity + value object) vào project riêng.
2. Move EF Core, external client từ slice ra Infrastructure project.
3. Define abstraction trong Application/Abstractions/.
4. Slice giờ phụ thuộc abstraction, không tech concrete.

### Clean Architecture → Clean Slice

1. Reorganize Application bằng folder feature thay vì folder technical type.
2. Move command/handler/validator/response của mỗi feature vào folder slice.
3. Endpoint thành extension method MapXxx() per feature.

---

## 21. Key Point — Danh sách ngắn

* **Clean ngoài, Vertical Slice trong.**
* **Dependency Rule enforce** bởi project reference.
* **Feature cohesion** trong `Application/Features/`.
* **Domain framework-free** — pure C#.
* **MediatR + pipeline behavior** cho dispatch và cross-cutting.
* **Per-slice tech choice** — EF Core / Dapper / raw SQL.
* **Architecture test** enforce boundary trong CI.
* **Default khuyến nghị cho .NET hiện đại** (2025+).

---

## 22. Checklist

- [ ] Project Domain không reference framework.
- [ ] Project Application chỉ reference Domain.
- [ ] Project Infrastructure implement abstraction Application.
- [ ] Mỗi slice ở folder riêng với cmd/val/handler/response.
- [ ] Endpoint mỏng — chỉ map command tới HTTP.
- [ ] Cross-cutting qua pipeline behavior.
- [ ] Domain event raise trong entity, dispatch ở UoW.
- [ ] Result<T> cho fail business; exception cho exceptional.
- [ ] CancellationToken end-to-end.
- [ ] Architecture test trong CI.
- [ ] Test domain pure, application với mock, infrastructure với DB thật, web functional.
- [ ] Health check + global exception middleware + structured logging.
- [ ] Configuration validate startup.

---

## 23. Tham khảo

* Robert C. Martin — *Clean Architecture* (2017).
* Jimmy Bogard — [*Vertical Slice Architecture*](https://www.jimmybogard.com/vertical-slice-architecture/).
* Jason Taylor — [Clean Architecture Solution Template](https://github.com/jasontaylordev/CleanArchitecture).
* Milan Jovanović — [*Vertical Slice Architecture* (blog)](https://www.milanjovanovic.tech/blog/vertical-slice-architecture).
* Amichai Mantinband — [Clean Architecture Template](https://github.com/amantinband/clean-architecture).
* Microsoft — [eShopOnWeb](https://github.com/dotnet-architecture/eShopOnWeb).
* MediatR documentation: [github.com/jbogard/MediatR](https://github.com/jbogard/MediatR).
* FluentValidation: [docs.fluentvalidation.net](https://docs.fluentvalidation.net/).
