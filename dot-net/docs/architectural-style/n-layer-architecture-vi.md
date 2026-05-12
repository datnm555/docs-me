# N-Layer Architecture trong .NET — Hướng dẫn đầy đủ

> Tham chiếu thực hành thiết kế, cấu trúc, và maintain N-Layer (aka N-Tier / Layered) application trong .NET hiện đại (.NET 6 / 7 / 8 / 9).

> 🇻🇳 Phiên bản tiếng Việt. English: [`n-layer-architecture.md`](./n-layer-architecture.md)

---

## Mục lục

1. [N-Layer Architecture là gì?](#1-n-layer-architecture-là-gì)
2. [Layer vs. Tier](#2-layer-vs-tier)
3. [Các Layer kinh điển](#3-các-layer-kinh-điển)
4. [Biến thể phổ biến](#4-biến-thể-phổ-biến)
5. [Dependency Flow & Quy tắc](#5-dependency-flow--quy-tắc)
6. [Cấu trúc .NET Solution điển hình](#6-cấu-trúc-net-solution-điển-hình)
7. [Code Walkthrough theo Layer](#7-code-walkthrough-theo-layer)
8. [Wire DI](#8-wire-di)
9. [Cross-Cutting Concern](#9-cross-cutting-concern)
10. [Best Practice](#10-best-practice)
11. [Anti-Pattern cần tránh](#11-anti-pattern-cần-tránh)
12. [Chiến lược Test](#12-chiến-lược-test)
13. [N-Layer vs. Clean / Onion / Hexagonal / Vertical Slice](#13-n-layer-vs-clean--onion--hexagonal--vertical-slice)
14. [Khi nào dùng (và không)](#14-khi-nào-dùng-và-không)
15. [Đường tiến hoá & Migration](#15-đường-tiến-hoá--migration)
16. [Checklist](#16-checklist)
17. [Tham khảo](#17-tham-khảo)

---

## 1. N-Layer Architecture là gì?

**N-Layer architecture** là pattern thiết kế tổ chức application thành stack các layer logic, mỗi cái có một trách nhiệm rõ ràng, đơn lẻ. Mỗi layer giao tiếp chỉ với layer ngay dưới (và được consume bởi layer ngay trên).

3 layer kinh điển là **Presentation → Business Logic → Data Access**, nhưng hệ thống thực tế thường mở rộng tới 4–6 layer (Application, Domain, Infrastructure, Common, v.v.) — vì thế "N".

Mục tiêu: **separation of concerns**, **testability**, và **maintainability** qua coupling được kiểm soát.

---

## 2. Layer vs. Tier

Hai từ này thường dùng thay thế nhau, nhưng không giống nhau:

| Từ        | Nghĩa                                                                                    |
| --------- | ---------------------------------------------------------------------------------------- |
| **Layer** | Tách *logic* code (vd., project, namespace). Chạy trong cùng process.                     |
| **Tier**  | Ranh giới deploy *vật lý* (vd., server riêng, container, service).                        |

Ví dụ: Monolith có thể có 4 **layer** nhưng deploy trên một **tier** (một web server + một database).

---

## 3. Các Layer kinh điển

### 3.1 Presentation Layer

* **Mục đích:** Expose application ra ngoài.
* **Ví dụ:** ASP.NET Core Web API controller, Razor Page, Blazor component, gRPC service, minimal API.
* **Trách nhiệm:**
  * Nhận HTTP/gRPC/CLI request.
  * Validate shape input (model binding, FluentValidation).
  * Map giữa transport DTO và application contract.
  * Authentication / authorization (ASP.NET middleware).
  * Trả response code và payload đúng.
* **KHÔNG nên:** chứa business rule, nói chuyện trực tiếp với database, biết về EF Core entity.

### 3.2 Application / Service Layer

* **Mục đích:** Orchestrate use case. "Verb" của hệ thống.
* **Ví dụ:** `OrderService`, `UserRegistrationService`, command/query handler.
* **Trách nhiệm:**
  * Điều phối domain object và infrastructure (repository, email, queue).
  * Transaction, unit-of-work boundary.
  * Authorization rule gắn với use case.
  * Map DTO ↔ domain.
* **KHÔNG nên:** biết về HTTP, internal EF Core, hoặc khung UI cụ thể.

### 3.3 Domain / Business Logic Layer

* **Mục đích:** "Noun" và invariant của business.
* **Ví dụ:** Entity (`Order`, `Invoice`), value object (`Money`, `Email`), domain service, domain event.
* **Trách nhiệm:**
  * Enforce business invariant.
  * Đóng gói behavior trong entity (không chỉ data bag).
  * Định nghĩa **interface** repository (implement trong Infrastructure).
* **KHÔNG nên:** reference EF Core, HTTP, hay framework nào. C# thuần.

### 3.4 Data Access / Infrastructure Layer

* **Mục đích:** Cung cấp implementation cụ thể cho concern kỹ thuật.
* **Ví dụ:** EF Core `DbContext`, Dapper repository, file storage, SMTP/SendGrid, Redis cache, message bus client.
* **Trách nhiệm:**
  * Implement interface repository do Domain định nghĩa.
  * Database migration và configuration.
  * Client API ngoài (Stripe, AWS, v.v.).
* **KHÔNG nên:** chứa business rule.

### 3.5 (Tuỳ chọn) Common / Shared / Cross-Cutting Layer

Abstraction logging, wrapper result (`Result<T>`), guard clause, custom exception, type primitive value-object. Thường được mọi layer khác reference.

---

## 4. Biến thể phổ biến

| Biến thể      | Layer                                                                        | Ghi chú                                       |
| ------------- | ---------------------------------------------------------------------------- | --------------------------------------------- |
| **3-Layer**   | Presentation → Business → Data                                               | Dạng nhỏ nhất khả thi. Phổ biến cho app nhỏ.  |
| **4-Layer**   | Presentation → Application → Domain → Infrastructure                         | Microsoft reference (eShopOnWeb).             |
| **5-Layer**   | Trên + Common/Shared                                                         | Thêm helper cross-cutting.                    |
| **Strict**    | Không cho phép skip layer (Presentation không nói chuyện thẳng với Data).    | Tốt nhất cho team lớn.                         |
| **Relaxed**   | Layer cao có thể bypass layer giữa (vd., read query trực tiếp).              | Đôi khi dùng cho read side CQRS.               |

---

## 5. Dependency Flow & Quy tắc

### 5.1 Quy tắc Vàng

> **Dependency chỉ trỏ xuống.** Layer có thể biết về layer dưới; không bao giờ trên.

```
┌────────────────────┐
│   Presentation     │
└────────┬───────────┘
         │ phụ thuộc
         ▼
┌────────────────────┐
│   Application      │
└────────┬───────────┘
         │ phụ thuộc
         ▼
┌────────────────────┐
│   Domain           │  ← không reference gì khác (thuần)
└────────────────────┘
         ▲
         │ implement interface từ Domain
┌────────┴───────────┐
│ Infrastructure     │
└────────────────────┘
```

### 5.2 Dependency Inversion

Domain định nghĩa `IOrderRepository`; Infrastructure cung cấp `EfOrderRepository`. Application phụ thuộc interface. Cách này, Domain giữ thuần và Infrastructure trở thành pluggable.

Đây là sức mạnh *thật* của layered architecture: **database là detail, không phải foundation.**

---

## 6. Cấu trúc .NET Solution điển hình

```
MyShop.sln
│
├── src/
│   ├── MyShop.Api/                  ← Presentation (ASP.NET Core Web API)
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   ├── MyShop.Application/          ← Application Layer (Class Library)
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   ├── Queries/
│   │   │   └── DTOs/
│   │   ├── Interfaces/              ← IEmailSender, ICurrentUser, v.v.
│   │   └── DependencyInjection.cs
│   │
│   ├── MyShop.Domain/               ← Domain Layer (Class Library)
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   ├── Exceptions/
│   │   └── Repositories/            ← IOrderRepository, IUserRepository
│   │
│   ├── MyShop.Infrastructure/       ← Infrastructure Layer (Class Library)
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/
│   │   │   ├── Migrations/
│   │   │   └── Repositories/
│   │   ├── Email/
│   │   ├── Storage/
│   │   └── DependencyInjection.cs
│   │
│   └── MyShop.Common/               ← Cross-Cutting (tuỳ chọn)
│       ├── Result.cs
│       ├── GuardClauses.cs
│       └── Logging/
│
└── tests/
    ├── MyShop.Domain.UnitTests/
    ├── MyShop.Application.UnitTests/
    ├── MyShop.Infrastructure.IntegrationTests/
    └── MyShop.Api.FunctionalTests/
```

### Project Reference Map

```
Api          → Application, Infrastructure, Common
Application  → Domain, Common
Domain       → Common
Infrastructure → Application, Domain, Common
```

> Lưu ý: `Infrastructure` reference `Application` chỉ để implement interface nó định nghĩa (vd., `IEmailSender`). Nếu purity quan trọng, định nghĩa interface đó trong `Domain` thay và xoá reference.

---

## 7. Code Walkthrough theo Layer

### Domain (C# thuần)

```csharp
// MyShop.Domain/Entities/Order.cs
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; }
    public Guid CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public Order(Guid customerId)
    {
        Id = Guid.NewGuid();
        CustomerId = customerId;
        Status = OrderStatus.Draft;
    }

    public void AddLine(Guid productId, int qty, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a submitted order.");
        _lines.Add(new OrderLine(productId, qty, unitPrice));
    }

    public void Submit()
    {
        if (!_lines.Any())
            throw new DomainException("Order must have at least one line.");
        Status = OrderStatus.Submitted;
    }
}
```

```csharp
// MyShop.Domain/Repositories/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

### Application (use case)

```csharp
// MyShop.Application/Orders/Commands/SubmitOrder.cs
public sealed record SubmitOrderCommand(Guid OrderId);

public sealed class SubmitOrderHandler
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;

    public SubmitOrderHandler(IOrderRepository orders, IUnitOfWork uow)
    {
        _orders = orders;
        _uow = uow;
    }

    public async Task Handle(SubmitOrderCommand cmd, CancellationToken ct)
    {
        var order = await _orders.GetByIdAsync(cmd.OrderId, ct)
            ?? throw new NotFoundException(nameof(Order), cmd.OrderId);

        order.Submit();
        await _uow.SaveChangesAsync(ct);
    }
}
```

### Infrastructure (EF Core)

```csharp
// MyShop.Infrastructure/Persistence/Repositories/EfOrderRepository.cs
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await _db.Orders.AddAsync(order, ct);
}
```

### Presentation (Web API)

```csharp
// MyShop.Api/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly SubmitOrderHandler _submit;

    public OrdersController(SubmitOrderHandler submit) => _submit = submit;

    [HttpPost("{id:guid}/submit")]
    public async Task<IActionResult> Submit(Guid id, CancellationToken ct)
    {
        await _submit.Handle(new SubmitOrderCommand(id), ct);
        return NoContent();
    }
}
```

---

## 8. Wire DI

Mỗi layer expose extension method `AddXxx` riêng, và `Program.cs` compose chúng.

```csharp
// MyShop.Application/DependencyInjection.cs
public static class ApplicationServiceCollectionExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<SubmitOrderHandler>();
        services.AddValidatorsFromAssemblyContaining<SubmitOrderValidator>();
        return services;
    }
}
```

```csharp
// MyShop.Infrastructure/DependencyInjection.cs
public static class InfrastructureServiceCollectionExtensions
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<AppDbContext>(opt =>
            opt.UseSqlServer(config.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, EfOrderRepository>();
        services.AddScoped<IUnitOfWork, EfUnitOfWork>();
        services.AddScoped<IEmailSender, SendGridEmailSender>();
        return services;
    }
}
```

```csharp
// MyShop.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddControllers();

builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);

var app = builder.Build();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

### Cheat Sheet Service Lifetime

| Lifetime    | Dùng khi                                                                  |
| ----------- | ------------------------------------------------------------------------- |
| `Singleton` | Config stateless, cache, client đắt build (HttpClient qua IHttpClientFactory). |
| `Scoped`    | Service per-request: `DbContext`, repository, application handler.        |
| `Transient` | Helper nhẹ stateless (validator, mapper).                                 |

**Cạm bẫy:** Đừng inject service `Scoped` vào `Singleton` — bạn sẽ capture instance stale.

---

## 9. Cross-Cutting Concern

Cross-cutting concern KHÔNG được leak business logic qua layer. Xử lý chúng tập trung:

| Concern         | Ở đâu                                                               |
| --------------- | ------------------------------------------------------------------- |
| Logging         | Microsoft.Extensions.Logging + Serilog/OpenTelemetry, DI mọi nơi    |
| Validation      | FluentValidation trong Application; model binding trong Presentation |
| Caching         | `IDistributedCache` / `HybridCache` (.NET 9), bọc đằng sau interface  |
| Transaction     | Abstraction `IUnitOfWork` trong Application; implement trong Infra  |
| Error handling  | Global exception middleware trong API + exception domain-specific   |
| Auth            | ASP.NET Core middleware; pass `ICurrentUser` vào Application        |
| Mapping         | AutoMapper / Mapster / extension method viết tay                    |
| Resilience      | Polly / `Microsoft.Extensions.Resilience` (.NET 8+)                 |

---

## 10. Best Practice

1. **Định nghĩa interface gần consumer.** Đặt `IOrderRepository` trong Domain (hoặc Application), không trong Infrastructure.
2. **Domain không phụ thuộc gì.** Không EF Core, không Newtonsoft, không `HttpContext`. Giữ portable.
3. **DTO ở boundary.** Đừng return EF entity từ controller; map thành DTO.
4. **Một `DbContext` per request (`Scoped`).** Không bao giờ `Singleton`.
5. **Async mọi nơi.** Tránh blocking call trong I/O path; pass `CancellationToken`.
6. **Repository return aggregate, không `IQueryable`.** Leak `IQueryable` phá encapsulation.
7. **Dùng `Result<T>` cho fail dự đoán.** Reserve exception cho case thật exceptional.
8. **Configuration qua `IOptions<T>`.** Strongly typed, validate ở startup.
9. **Tập trung migration** trong Infrastructure; project API trigger chúng lúc startup hoặc qua CI.
10. **Giữ Presentation mỏng.** Action controller ≈ 5–15 dòng: validate → call handler → map result.
11. **Một assembly per layer.** Enforce reference; ngăn coupling tình cờ.
12. **Dùng architecture test** (`NetArchTest` / `ArchUnitNET`) trong CI để enforce quy tắc dependency.
13. **Không share entity type qua bounded context.** Mỗi context sở hữu model riêng.
14. **Tránh smell "service-of-services".** Nếu `OrderService` call `InvoiceService` call `PaymentService`, push logic vào domain thay vào.
15. **Dùng feature folder** trong Application (group theo use case, không theo technical type).
16. **Thêm health check** (`/health`, `/health/ready`) cho Presentation layer.
17. **Dùng minimal API hoặc controller nhất quán.** Đừng trộn không lý do.
18. **Source generator hơn reflection** nơi có thể (vd., System.Text.Json, Mapperly).

---

## 11. Anti-Pattern cần tránh

| Anti-Pattern                          | Tại sao tệ                                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Anemic domain model**               | Entity là data bag; mọi logic trong service. Erode giá trị của Domain layer.                        |
| **Layer skipping**                    | Controller → Repository trực tiếp. Ẩn business rule và bypass orchestration use-case.              |
| **God service**                       | `OrderService` với 40 method. Tách theo use case.                                                  |
| **Leak EF entity** ra API             | Couple chặt contract API với DB schema; vỡ mỗi migration.                                          |
| **Return `IQueryable`** từ repo       | Caller có thể mutate semantics query, phá encapsulation.                                            |
| **Static helper call DI**             | Pattern service-locator; không thể test.                                                            |
| **Async void**                        | Exception không xử lý crash process.                                                                |
| **Catch-and-swallow** exception       | Ẩn bug thật. Để global middleware xử lý.                                                            |
| **Trộn sync + async** (`.Result`, `.Wait()`) | Deadlock trong ASP.NET classic; cạn thread-pool trong Kestrel.                              |
| **Reference layer vòng tròn**         | Architecturally illegal. Reach interface ở layer dưới.                                              |
| **Project "Common.Models" share** với EF entity | Thành thùng đổ; couple mọi thứ.                                                          |

---

## 12. Chiến lược Test

| Loại test             | Mục tiêu                                  | Tool                                                     |
| --------------------- | ----------------------------------------- | -------------------------------------------------------- |
| **Unit (Domain)**     | Entity, value object, domain service       | xUnit + FluentAssertions; **không cần mock**            |
| **Unit (Application)**| Handler, service                          | xUnit + NSubstitute/Moq cho interface repository         |
| **Integration**       | Infrastructure (DB thật)                   | xUnit + Testcontainers / SQL Server in Docker            |
| **Functional / E2E**  | API end-to-end                            | `WebApplicationFactory<Program>` + Testcontainers        |
| **Architecture**      | Quy tắc layer                             | NetArchTest.Rules / ArchUnitNET                          |

**Ví dụ architecture test:**

```csharp
[Fact]
public void Domain_should_not_reference_Infrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .Should()
        .NotHaveDependencyOn("MyShop.Infrastructure")
        .GetResult();
    Assert.True(result.IsSuccessful);
}
```

---

## 13. N-Layer vs. Clean / Onion / Hexagonal / Vertical Slice

| Style              | Idea cốt lõi                                              | Điểm mạnh                                       | Điểm yếu                                                     |
| ------------------ | ---------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| **N-Layer**        | Layer stack, dependency top-down                          | Quen thuộc; dễ onboard; nhanh khởi đầu          | Thay đổi cross-layer đụng nhiều file; có thể cứng           |
| **Clean / Onion**  | Domain ở trung tâm; mọi thứ khác trỏ vào trong            | Decouple tối đa; core framework-agnostic        | Nhiều ceremony; over-engineer cho app nhỏ                    |
| **Hexagonal**      | Port (interface) & adapter (impl) quanh core              | I/O pluggable; tốt cho hệ thống nặng messaging  | Learning curve dốc hơn                                       |
| **Vertical Slice** | Một folder per feature; layer tồn tại *bên trong* slice    | Coupling thấp giữa feature; dễ xoá              | Một số trùng lặp; enforce cross-cutting yếu hơn              |

> **Lời khuyên thực tế:** Nhiều project .NET hiện đại dùng *hybrid* — cấu trúc layer Clean Architecture với tổ chức Vertical Slice (feature folder) bên trong `Application/`. Đôi khi gọi là "Clean Slice."

---

## 14. Khi nào dùng (và không)

### ✅ Phù hợp

* App line-of-business CRUD-heavy.
* Team mới với thiết kế layer (learning curve nhẹ nhàng).
* Monolith hoặc modular monolith.
* App với domain ổn định, hiểu rõ.

### ⚠️ Xem xét lại khi

* Có nhiều feature độc lập với ít logic chia sẻ → cân nhắc **Vertical Slice**.
* Domain phong phú và phức tạp với nhiều invariant → cân nhắc **Clean / DDD**.
* Đang build microservice làm một việc → 1–2 layer có thể đủ.
* Hệ thống read-heavy với nhu cầu query đa dạng → cân nhắc **CQRS** với read model riêng.

---

## 15. Đường tiến hoá & Migration

Path tăng trưởng thực dụng:

1. **Khởi đầu nhỏ:** 3-layer (API → Service → Repository).
2. **Extract Domain** khi business rule bắt đầu lộn xộn service.
3. **Tách Application khỏi Domain** khi use case nhân lên.
4. **Adopt CQRS** khi model read và write phân kỳ.
5. **Chuyển sang Vertical Slice** khi feature vượt service share.
6. **Extract microservice** khi bounded context có nhu cầu scale/release độc lập.

Đừng nhảy bước. Phức tạp sớm là dạng lãng phí đắt nhất.

---

## 16. Checklist

Trước khi merge, verify:

- [ ] Project Domain reference **chỉ** Common (hoặc không gì).
- [ ] Không type EF Core leak lên trên Infrastructure.
- [ ] Controller mỏng (< 15 dòng per action).
- [ ] Mọi dependency ngoài đằng sau interface.
- [ ] Lifetime `DbContext` là `Scoped`.
- [ ] `CancellationToken` flow qua async chain.
- [ ] DTO dùng ở API boundary (không return entity).
- [ ] Migration reproducible và được review.
- [ ] Architecture test pass trong CI.
- [ ] Unit test cover domain logic; integration test cover repository.
- [ ] Logging và exception middleware configured.
- [ ] Configuration validate lúc startup (`IOptions<T>` + `ValidateOnStart`).

---

## 17. Tham khảo

* [Common web application architectures — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
* [eShopOnWeb reference application](https://github.com/dotnet-architecture/eShopOnWeb)
* [ASP.NET Boilerplate — N-Layer Architecture](https://aspnetboilerplate.com/Pages/Documents/NLayer-Architecture)
* [N-Layered vs Clean vs Vertical Slice (antondevtips)](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
* [Guide to Building an N-Tier Architecture for a .NET 8 Web API (Medium)](https://medium.com/@csns.giri/guide-to-building-an-n-tier-architecture-for-a-net-8-web-api-a49f4a83335e)
* [Layered (N-Tier) Architecture in .NET Core (DEV)](https://dev.to/dotnetfullstackdev/layered-n-tier-architecture-in-net-core-51ic)
* [Stackify — What is N-Tier Architecture?](https://stackify.com/n-tier-architecture/)
* [aghayeffemin/aspnetcore.ntier (GitHub sample)](https://github.com/aghayeffemin/aspnetcore.ntier)
