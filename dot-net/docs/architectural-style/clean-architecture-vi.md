# Clean Architecture — Khuyến nghị, Key Point & Best Approach

> Dựa trên *Clean Architecture: A Craftsman's Guide to Software Structure and Design* của **Robert C. Martin (Uncle Bob)**, với hướng dẫn áp dụng .NET thực tế.

> 🇻🇳 Phiên bản tiếng Việt. English: [`clean-architecture.md`](./clean-architecture.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Tổng hợp prescriptive của Robert C. Martin về Hexagonal + Onion + Use Cases. Bốn vòng đồng tâm (Entity ← Use Case ← Interface Adapter ← Frameworks & Drivers) với một rule tuyệt đối: **source-code dependency chỉ trỏ vào trong**.
- **Tại sao** — Giữ business policy giá trị cao, thay đổi chậm độc lập với technology giá trị thấp, biến động (DB, web framework, UI). Database/UI/framework là "detail" — không phải foundation.
- **Khi nào** — Hệ thống sống lâu với business rule phức tạp hoặc evolving; nhiều kênh delivery (web + mobile + worker + CLI); team practice DDD; refactor legacy về maintainability.
- **Ở đâu** — Style toàn hệ thống trên trục *principles*. Implement qua project N-Layer (Domain → Application → Infrastructure → Web). Pair tự nhiên với DDD, CQRS, MediatR, và Vertical Slice bên trong Application layer ("Clean Slice").

---

## Mục lục

1. [Idea lớn](#1-idea-lớn)
2. [Mục tiêu của Clean Architecture](#2-mục-tiêu-của-clean-architecture)
3. [SOLID — Nền tảng OO](#3-solid--nền-tảng-oo)
4. [Component Principle](#4-component-principle)
5. [Bốn Layer đồng tâm](#5-bốn-layer-đồng-tâm)
6. [Dependency Rule (Quy tắc cai trị tất cả)](#6-dependency-rule-quy-tắc-cai-trị-tất-cả)
7. [Vượt boundary — Pattern Input/Output Boundary](#7-vượt-boundary--pattern-inputoutput-boundary)
8. [Screaming Architecture](#8-screaming-architecture)
9. [Humble Object Pattern](#9-humble-object-pattern)
10. [Boundary — Partial, Local, Full](#10-boundary--partial-local-full)
11. [Database & UI là Detail](#11-database--ui-là-detail)
12. [Clean Architecture trong .NET — Solution Layout](#12-clean-architecture-trong-net--solution-layout)
13. [Code Walkthrough theo Layer (.NET)](#13-code-walkthrough-theo-layer-net)
14. [Pattern đi cùng tốt: CQRS, MediatR, Result, DDD](#14-pattern-đi-cùng-tốt-cqrs-mediatr-result-ddd)
15. [Best Approach — Practice khuyến nghị](#15-best-approach--practice-khuyến-nghị)
16. [Cạm bẫy phổ biến](#16-cạm-bẫy-phổ-biến)
17. [Chiến lược Test](#17-chiến-lược-test)
18. [Clean vs. N-Layer vs. Onion vs. Hexagonal](#18-clean-vs-n-layer-vs-onion-vs-hexagonal)
19. [Khi nào dùng (và không)](#19-khi-nào-dùng-và-không)
20. [Checklist trước khi Merge](#20-checklist-trước-khi-merge)
21. [Tham khảo](#21-tham-khảo)

---

## 1. Idea lớn

Clean Architecture **không phải framework** và **không phải folder layout**. Nó là tập *nguyên lý* tạo ra hệ thống có tính chất:

> *"Kiến trúc của một hệ thống phần mềm là hình dạng được trao cho hệ thống bởi những người xây nó … để làm hệ thống dễ hiểu, develop, maintain, và deploy. Mục tiêu là giảm thiểu chi phí lifetime của hệ thống và tối đa năng suất programmer."* — Uncle Bob

Một hệ thống sạch là:

- **Độc lập với framework** — framework là công cụ, không phải ràng buộc.
- **Testable** — business rule có thể test mà không có UI, DB, hay web server.
- **Độc lập với UI** — UI có thể thay đổi mà không đụng business rule.
- **Độc lập với database** — Oracle, SQL Server, Postgres, MongoDB, hay flat file có thể thay thế.
- **Độc lập với bất cứ external agency** — business rule không biết về thế giới bên ngoài.

Kiến trúc phải giữ **quyết định defer và có thể thay đổi càng lâu càng tốt**.

---

## 2. Mục tiêu của Clean Architecture

Uncle Bob phân biệt **policy** (rule — cái business làm) khỏi **detail** (cơ chế — cách thực hiện). Clean architecture bảo vệ **policy** có giá trị cao, thay đổi chậm khỏi **detail** biến động, giá trị thấp.

| Concern         | Policy / Detail | Sống ở đâu                           |
| --------------- | --------------- | ------------------------------------ |
| Business rule   | Policy          | Entity (trong cùng)                  |
| Use case        | Policy          | Use Cases                            |
| Web framework   | Detail          | Frameworks & Drivers (ngoài cùng)    |
| Database        | Detail          | Frameworks & Drivers (ngoài cùng)    |
| Serialization   | Detail          | Interface Adapters                   |

Thay đổi detail không bao giờ được ép thay đổi policy.

---

## 3. SOLID — Nền tảng OO

SOLID áp dụng cho *class*. Component principle (section sau) áp dụng cho *module / project*. Cùng nhau chúng định nghĩa clean architecture.

| Chữ    | Nguyên lý                       | Ý nghĩa thực tế                                                                                  |
| ------ | ------------------------------- | ------------------------------------------------------------------------------------------------ |
| **S**  | Single Responsibility Principle | Module nên có một và chỉ một **lý do để thay đổi** (vd., một actor/stakeholder).                  |
| **O**  | Open/Closed Principle           | Mở cho extension, đóng cho modification. Behavior mới đến từ thêm code.                          |
| **L**  | Liskov Substitution Principle   | Subtype phải có thể substitute base type không phá kỳ vọng.                                       |
| **I**  | Interface Segregation Principle | Đừng ép client phụ thuộc method chúng không dùng. Ưu tiên nhiều interface nhỏ.                    |
| **D**  | **Dependency Inversion**        | Phụ thuộc abstraction, không concretion. **Cornerstone của Clean Architecture.**                  |

> Uncle Bob: *"Dependency Inversion Principle bảo ta hệ thống flexible nhất là những hệ thống mà source code dependency chỉ tham chiếu abstraction, không concretion."*

---

## 4. Component Principle

Component (DLL / assembly / package) có rule SOLID-style riêng.

### Component Cohesion (cái gì đi cùng nhau?)

| Nguyên lý                                         | Nói                                                                        |
| ------------------------------------------------- | -------------------------------------------------------------------------- |
| **REP** — Reuse / Release Equivalence             | Unit của reuse là unit của release. Version component cùng nhau.            |
| **CCP** — Common Closure Principle                | Group cùng nhau class thay đổi vì cùng lý do cùng lúc.                      |
| **CRP** — Common Reuse Principle                  | Đừng ép user phụ thuộc thứ chúng không dùng.                                |

### Component Coupling (component liên hệ ra sao?)

| Nguyên lý                                         | Nói                                                                        |
| ------------------------------------------------- | -------------------------------------------------------------------------- |
| **ADP** — Acyclic Dependencies Principle          | Graph component phải là DAG. **Không vòng.**                                |
| **SDP** — Stable Dependencies Principle           | Phụ thuộc theo hướng ổn định (component ổn định có nhiều dependent).        |
| **SAP** — Stable Abstractions Principle           | Component ổn định cũng nên abstract (để extensible).                        |

**Main Sequence**: component nên là *ổn định + abstract* (như Domain) hoặc *không ổn định + concrete* (như UI). Component trong "zone of pain" (ổn định + concrete) và "zone of uselessness" (không ổn định + abstract) là red flag.

---

## 5. Bốn Layer đồng tâm

Diagram nổi tiếng (paraphrase từ blog Uncle Bob):

```
                ┌───────────────────────────────────────────┐
                │  Frameworks & Drivers                     │
                │  (Web, DB, UI, Device, External API)      │
                │  ┌─────────────────────────────────────┐  │
                │  │  Interface Adapters                 │  │
                │  │  (Controller, Gateway, Presenter)   │  │
                │  │  ┌───────────────────────────────┐  │  │
                │  │  │  Application Business Rules  │  │  │
                │  │  │  (Use Case / Interactor)      │  │  │
                │  │  │  ┌─────────────────────────┐  │  │  │
                │  │  │  │  Enterprise Rules       │  │  │  │
                │  │  │  │  (Entities)             │  │  │  │
                │  │  │  └─────────────────────────┘  │  │  │
                │  │  └───────────────────────────────┘  │  │
                │  └─────────────────────────────────────┘  │
                └───────────────────────────────────────────┘

           Dependency chỉ trỏ VÀO TRONG ─────►
```

### 5.1 Entities (Enterprise-Wide Business Rule)

* **Policy cấp cao nhất**.
* Object thuần với method đóng gói business rule critical áp dụng toàn enterprise — kể cả không có application quanh chúng.
* Không biết về database, HTTP, JSON, framework, hay layer khác.
* Ổn định. Hiếm khi thay đổi.

### 5.2 Use Cases (Application-Specific Business Rule)

* Orchestrate **entity** để hoàn thành mục tiêu application.
* Mỗi use case = một intent user (`PlaceOrder`, `RegisterCustomer`, `RefundPayment`).
* Độc lập với UI và DB.
* Định nghĩa **input boundary** (request model) và **output boundary** (response model / presenter).

### 5.3 Interface Adapter

* Convert data giữa format tiện cho use case/entity và format tiện cho external agency.
* Ví dụ: MVC controller, presenter, view model, repository (phía implement), gateway, serializer.
* Đây là nơi phần lớn code "translation" sống.

### 5.4 Frameworks & Drivers (Detail)

* Vòng ngoài cùng. ASP.NET Core, EF Core, RabbitMQ, MongoDB, Stripe SDK, browser.
* Chủ yếu *glue code*. Viết ít nhất có thể ở đây.
* Replaceable: switch SQL Server sang PostgreSQL hoặc REST sang gRPC không đụng use case.

---

## 6. Dependency Rule (Quy tắc cai trị tất cả)

> **Source code dependency chỉ trỏ vào trong.**

* Inner circle phải không biết **gì** về outer circle.
* Tên khai báo trong outer circle (controller, EF entity, JSON attribute, type framework) **không xuất hiện** trong inner circle.
* Data vượt boundary phải là **cấu trúc data đơn giản** (DTO / record), không bao giờ type framework như `HttpRequest`, `DbContext`, hay `IQueryable`.

Khi flow control tự nhiên cần đi *ra ngoài* (vd., use case phải call database), dùng **Dependency Inversion**: use case phụ thuộc interface, và layer ngoài implement.

```
Use Case  ─uses→  IUserRepository  ←implements─  EfUserRepository
(inner)            (inner)                        (outer)
```

---

## 7. Vượt boundary — Pattern Input/Output Boundary

Cho mỗi use case, định nghĩa:

* **Input Boundary** (interface) controller call.
* **Request Model** (DTO) cho input.
* **Output Boundary** (interface) use case call.
* **Response Model** (DTO) cho output.
* **Presenter** consume response model và produce view model.

Đây là pattern **Boundary–Interactor–Presenter** (aka *Clean Use Case*).

```
Controller ─► IInputBoundary (UseCase) ─► Entity ─► IOutputBoundary ─► Presenter ─► View
```

Trong .NET, `IRequest<T>` + `IRequestHandler<TReq, TRes>` của MediatR đóng vai trò này với ít ceremony hơn.

---

## 8. Screaming Architecture

> *"Kiến trúc của bạn nên thét lên intent của hệ thống."* — Uncle Bob

Khi ai đó mở solution, tên folder/project nên mô tả **hệ thống làm gì**, không phải **framework nào nó dùng**.

❌ Tệ (framework thét):
```
src/
  Controllers/
  Services/
  Repositories/
  Models/
```

✅ Tốt (domain thét):
```
src/
  Ordering/
  Billing/
  Shipping/
  Inventory/
```

Framework nên là detail implementation bạn khám phá chỉ khi đào vào feature.

---

## 9. Humble Object Pattern

Code khó test (UI, DB driver, message queue) tách thành:

* Một component **humble** chỉ làm cái khó test (cơ học, không behavior).
* Một component **testable** chứa logic.

Ví dụ trong .NET:
* Controller (humble) ↔ Use case handler (testable).
* `DbContext` (humble) ↔ Repository + Domain (testable).
* SignalR hub (humble) ↔ Notification service (testable).

---

## 10. Boundary — Partial, Local, Full

Bạn không phải áp dụng boundary mọi nơi từ ngày đầu.

* **Full boundary**: process / service riêng. Chi phí cao nhất, decouple cao nhất.
* **Local boundary**: assembly riêng, giao tiếp qua interface. Phổ biến nhất trong .NET.
* **Partial boundary**: project đơn, nhưng interface boundary và DTO tồn tại. Rẻ; dễ harden sau.

Defer chi phí. Khởi đầu partial boundary; promote khi pain xuất hiện.

---

## 11. Database & UI là Detail

Clean architecture treat **database**, **web framework**, và **UI** là *detail* plug-in. Business không nên quan tâm data store trong SQL Server, Cosmos DB, hay file. Dùng **gateway interface** trong use case layer; implementation cụ thể sống trong Infrastructure.

Đây là lý do Clean Architecture pair tự nhiên với **CQRS** (tách model command/query) và **event sourcing** (database thành stream of fact, không phải source of truth).

---

## 12. Clean Architecture trong .NET — Solution Layout

```
MyApp.sln
│
├── src/
│   ├── MyApp.Domain/                  ← Entity + Domain Service (trong cùng)
│   │   ├── Customers/
│   │   ├── Orders/
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   └── Abstractions/              ← IClock, IIdGenerator (thuần)
│   │
│   ├── MyApp.Application/             ← Use Case (chỉ phụ thuộc Domain)
│   │   ├── Abstractions/              ← IOrderRepository, IEmailSender, IUnitOfWork
│   │   ├── Orders/
│   │   │   ├── PlaceOrder/
│   │   │   │   ├── PlaceOrderCommand.cs
│   │   │   │   ├── PlaceOrderHandler.cs
│   │   │   │   ├── PlaceOrderValidator.cs
│   │   │   │   └── PlaceOrderResponse.cs
│   │   │   └── GetOrder/
│   │   └── DependencyInjection.cs
│   │
│   ├── MyApp.Infrastructure/          ← EF Core, SMTP, Storage, External API
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/
│   │   │   ├── Migrations/
│   │   │   └── Repositories/
│   │   ├── Email/
│   │   ├── Storage/
│   │   └── DependencyInjection.cs
│   │
│   ├── MyApp.Web/                     ← ASP.NET Core API / Blazor (ngoài cùng)
│   │   ├── Endpoints/  (or Controllers/)
│   │   ├── Middleware/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   └── MyApp.SharedKernel/            ← (Tuỳ chọn) Result<T>, GuardClauses, base type
│
└── tests/
    ├── MyApp.Domain.UnitTests/
    ├── MyApp.Application.UnitTests/
    ├── MyApp.Infrastructure.IntegrationTests/
    └── MyApp.Web.FunctionalTests/
```

### Project Reference Graph

```
Web           → Application, Infrastructure, SharedKernel
Application   → Domain, SharedKernel
Domain        → SharedKernel
Infrastructure → Application, Domain, SharedKernel
```

> **Invariant chính:** `Domain` không reference gì ngoài `SharedKernel`. Không EF Core. Không HTTP. Không ASP.NET. Nếu không thể port `Domain` sang console app hoặc framework khác as-is, boundary đã vỡ.

---

## 13. Code Walkthrough theo Layer (.NET)

### 13.1 Domain — Entity với Behavior

```csharp
// MyApp.Domain/Orders/Order.cs
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public OrderId Id { get; }
    public CustomerId CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public Money Total => Money.Sum(_lines.Select(l => l.Subtotal));
    public IReadOnlyList<OrderLine> Lines => _lines;

    private Order(OrderId id, CustomerId customer)
    {
        Id = id;
        CustomerId = customer;
        Status = OrderStatus.Draft;
    }

    public static Order Create(CustomerId customer, IIdGenerator ids)
        => new(new OrderId(ids.NewGuid()), customer);

    public void AddLine(ProductId product, int qty, Money unitPrice)
    {
        Guard.Against.NotInStatus(Status, OrderStatus.Draft);
        Guard.Against.NonPositive(qty);
        _lines.Add(new OrderLine(product, qty, unitPrice));
    }

    public OrderPlaced Place()
    {
        Guard.Against.EmptyCollection(_lines);
        Status = OrderStatus.Placed;
        return new OrderPlaced(Id, CustomerId, Total);
    }
}
```

### 13.2 Application — Abstraction + Use Case

```csharp
// MyApp.Application/Abstractions/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

```csharp
// MyApp.Application/Orders/PlaceOrder/PlaceOrderCommand.cs
public sealed record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<PlaceOrderLine> Lines)
    : IRequest<Result<Guid>>;

public sealed record PlaceOrderLine(Guid ProductId, int Quantity, decimal UnitPrice);
```

```csharp
// MyApp.Application/Orders/PlaceOrder/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Result<Guid>>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly IIdGenerator _ids;

    public PlaceOrderHandler(IOrderRepository orders, IUnitOfWork uow, IIdGenerator ids)
    {
        _orders = orders; _uow = uow; _ids = ids;
    }

    public async Task<Result<Guid>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(new CustomerId(cmd.CustomerId), _ids);
        foreach (var l in cmd.Lines)
            order.AddLine(new ProductId(l.ProductId), l.Quantity, new Money(l.UnitPrice));

        var evt = order.Place();
        await _orders.AddAsync(order, ct);
        await _uow.SaveChangesAsync(ct);
        return Result.Success(order.Id.Value);
    }
}
```

### 13.3 Infrastructure — Implementation EF

```csharp
// MyApp.Infrastructure/Persistence/Repositories/EfOrderRepository.cs
internal sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetAsync(OrderId id, CancellationToken ct) =>
        _db.Orders.Include(o => o.Lines)
                  .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await _db.Orders.AddAsync(order, ct);
}
```

### 13.4 Web — Endpoint mỏng

```csharp
// MyApp.Web/Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (PlaceOrderCommand cmd, ISender mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value}", result.Value)
                : Results.BadRequest(result.Error);
        });
    }
}
```

> Controller / endpoint là **humble**. Việc duy nhất của nó là dịch giữa HTTP và use case.

---

## 14. Pattern đi cùng tốt: CQRS, MediatR, Result, DDD

| Pattern              | Vai trò trong Clean .NET app                                                       |
| -------------------- | --------------------------------------------------------------------------------- |
| **CQRS**             | Tách command khỏi query. Command đi qua handler; query có thể bypass entity và read trực tiếp với projection tối ưu. |
| **MediatR**          | Implement input boundary (`IRequest`/`IRequestHandler`). Pipeline behavior cho logging, validation, transaction miễn phí. |
| **Result\<T\>**      | Tránh exception cho fail business dự đoán. Mang value HOẶC error.                |
| **FluentValidation** | Validation input use-case sống trong Application — độc lập ASP.NET.              |
| **DDD tactical**     | Entity, value object, aggregate, domain event. Fit Domain layer tự nhiên.        |
| **Specification**    | Đóng gói query criteria thành object testable.                                    |
| **Domain Event + Outbox** | Decouple side effect (email, integration event) khỏi transaction use case. |

---

## 15. Best Approach — Practice khuyến nghị

1. **Khởi đầu với Domain.** Model entity và value object *trước* DB schema hay API contract.
2. **Một use case per file.** Một folder per use case với command, handler, validator, response.
3. **Dùng folder `Application/Abstractions`** cho mọi interface consume bởi use case.
4. **Làm `Domain` framework-free.** Không NuGet reference trừ package rất thuần (vd., `Ardalis.GuardClauses`).
5. **Dùng `internal` aggressively** trong Infrastructure. Chỉ expose method `AddInfrastructure(...)` registration publicly.
6. **DTO ở mọi boundary.** Đừng để EF entity reach controller.
7. **MediatR pipeline behavior** cho logging, validation, transaction, caching, performance.
8. **`Result<T>` cho fail dự đoán**, exception chỉ cho tình huống thật exceptional hoặc programmer-error.
9. **Cancel mọi nơi.** Pass `CancellationToken` qua mọi async chain.
10. **Configuration qua `IOptions<T>` + `ValidateOnStart()`.** Fail fast cho misconfiguration.
11. **Migration & seeding** sống trong Infrastructure. Project web trigger chúng chỉ lúc startup dev / qua one-shot job prod.
12. **Architecture test** (NetArchTest / ArchUnitNET) trong CI để enforce Dependency Rule cơ học.
13. **Feature folder** trong Application — group theo business capability, không technical type.
14. **Top-level thét** — tên folder nên reveal business intent (`Ordering`, `Billing`), không framework (`Controllers`, `Services`).
15. **Giữ handler ~50 dòng hoặc ít.** To hơn hint khái niệm domain bị thiếu.
16. **Tránh pattern Generic Repository.** Ưu tiên method repository use-case-specific return aggregate.
17. **Đừng return `IQueryable` từ repository.** Encapsulation vỡ khi caller thêm `.Where()`.
18. **Dùng Source Generator** (System.Text.Json, Mapperly) hơn reflection cho AOT ready trong .NET 8/9.
19. **Áp dụng Strangler Fig** khi migrate legacy code — bọc legacy trong adapter; cho phần Clean phát triển.
20. **Defer quyết định.** ORM, message broker, cloud vendor — chọn muộn; thiết kế như thể bất cứ cái nào có thể swap.

---

## 16. Cạm bẫy phổ biến

| Cạm bẫy                                                | Tại sao tệ                                                                                    |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| **Anemic domain** — entity chỉ có property              | Use case bloat với logic thuộc về entity.                                                     |
| **Domain reference EF Core**                            | Phá Dependency Rule; không thể reuse domain không có EF.                                       |
| **Một mega-`Application` project cho mọi thứ**          | Dùng feature folder hoặc project per-bounded-context.                                          |
| **Generic `IRepository<T>` mọi nơi**                    | Thường degenerate thành wrapper mỏng quanh EF và khuyến khích thiết kế aggregate kém.          |
| **Map DTO ↔ entity với AutoMapper khắp nơi**            | Behavior ẩn, error runtime. Ưu tiên constructor explicit / method `static FromEntity()`.       |
| **Bypass use case cho read "nhanh"**                    | Nếu phải, route qua query handler — giữ một path rõ ràng.                                      |
| **Binding hai chiều giữa layer**                        | Vòng. Refactor bằng introduce interface trong layer trong.                                      |
| **Lạm dụng exception cho control flow**                 | Chậm hơn, khó test hơn, leak qua boundary. Dùng `Result<T>` cho fail dự đoán.                 |
| **Over-engineer** cho app nhỏ                           | Clean Architecture có overhead; CRUD admin tool có thể không cần 4 project.                    |
| **Treat MediatR như magic**                             | Chỉ là dispatcher in-process. Đừng ẩn flow critical trong handler không follow được.            |

---

## 17. Chiến lược Test

| Layer Test            | Mục tiêu                                                 | Tool                                                 |
| --------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| **Domain unit**       | Entity, value object, domain service                     | xUnit + FluentAssertions, **không mock**             |
| **Application unit**  | Handler, validator                                       | xUnit + NSubstitute / Moq cho `Application/Abstractions` |
| **Infrastructure integration** | DB thật qua Testcontainers                      | xUnit + Testcontainers                               |
| **Functional / E2E**  | API đầy đủ qua `WebApplicationFactory<Program>`           | xUnit + Testcontainers                               |
| **Architecture**      | Quy tắc layer                                            | NetArchTest.Rules / ArchUnitNET                      |

Ví dụ test architecture:

```csharp
[Fact]
public void Domain_must_not_reference_Application_or_Infrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .Should()
        .NotHaveDependencyOnAny("MyApp.Application", "MyApp.Infrastructure", "Microsoft.EntityFrameworkCore")
        .GetResult();
    Assert.True(result.IsSuccessful);
}
```

---

## 18. Clean vs. N-Layer vs. Onion vs. Hexagonal

| Style          | Diagram          | Idea cốt lõi                                          | Điểm mạnh                              | Điểm yếu                                  |
| -------------- | ---------------- | ----------------------------------------------------- | -------------------------------------- | ----------------------------------------- |
| **N-Layer**    | Stack            | Layer phụ thuộc top-down                              | Quen thuộc, nhanh khởi đầu             | Cứng, dễ "skip" layer                     |
| **Onion**      | Vòng đồng tâm    | Domain ở trung tâm; dependency vào trong              | Decouple domain khỏi infra             | Nhiều project nhỏ; ceremony               |
| **Hexagonal**  | Hex với port     | Port & adapter quanh core                             | Adapter pluggable; I/O đối xứng        | Learning curve                            |
| **Clean**      | Vòng đồng tâm    | Kết hợp Onion + Hexagonal + Use-Case-driven design     | Prescriptive nhất; tuyệt cho DDD-style app | Nặng hơn; có thể over-engineer app đơn giản |
| **Vertical Slice** | Feature folder | Per-feature mini-layering                            | Coupling thấp giữa feature             | Một số trùng lặp; rule share yếu hơn      |

> **Trong thực tế:** Layer ngoài Clean Architecture (Web, Infrastructure) cộng folder feature kiểu Vertical-Slice *bên trong* Application là hybrid phổ biến trong .NET hiện đại (đôi khi gọi "Clean Slice").

---

## 19. Khi nào dùng (và không)

### ✅ Phù hợp

* Hệ thống sống lâu với business rule phức tạp (banking, ERP, healthcare, logistics).
* Nhiều kênh delivery (web + mobile + worker + CLI).
* Thay đổi infrastructure thường xuyên (đổi DB, di chuyển cloud).
* Team practice DDD và TDD.

### ⚠️ Xem xét lại khi

* CRUD thuần với rule đơn giản → N-Layer thường là đủ.
* Microservice nhỏ làm một việc → 1–2 project.
* Prototype sống ngắn → tốc độ thắng cấu trúc.
* Reporting read-heavy → ưu tiên read model CQRS hoặc SQL trực tiếp.

> Quy tắc: trả chi phí cấu trúc khi *chi phí thay đổi* trong domain cao. Nếu không, discipline là overhead.

---

## 20. Checklist trước khi Merge

- [ ] Project `Domain` có **zero** reference infrastructure hoặc web.
- [ ] Mọi interface use case phụ thuộc sống trong `Application/Abstractions`.
- [ ] Không EF entity hoặc `IQueryable` expose trên Infrastructure.
- [ ] Mỗi use case ở folder riêng với command/handler/validator/response.
- [ ] Endpoint/controller < 15 dòng.
- [ ] Mọi async path flow `CancellationToken`.
- [ ] Lifetime `DbContext` là `Scoped`; không capture vào singleton.
- [ ] Architecture test pass trong CI.
- [ ] Unit test cover entity và handler; integration test cover path DB thật.
- [ ] Cấu trúc folder "thét" business, không framework.
- [ ] Configuration validate lúc startup; secret qua secret store hoặc env var.
- [ ] Logging + global exception handler + health check sẵn sàng.

---

## 21. Tham khảo

### Nguồn gốc
* **Martin, Robert C.** *Clean Architecture: A Craftsman's Guide to Software Structure and Design.* Prentice Hall, 2017.

### Bài viết Uncle Bob
* [The Clean Architecture (blog post)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
* [Screaming Architecture (blog post)](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)

### Practical Guide .NET
* [Clean Architecture in .NET — Code Maze](https://code-maze.com/dotnet-clean-architecture/)
* [Microsoft eShopOnWeb Reference App](https://github.com/dotnet-architecture/eShopOnWeb)
* [Jason Taylor — Clean Architecture Solution Template](https://github.com/jasontaylordev/CleanArchitecture)
* [Amichai Mantinband — Clean Architecture Template](https://github.com/amantinband/clean-architecture)
* [Milan Jovanović — Building Your First Use Case With Clean Architecture](https://www.milanjovanovic.tech/blog/building-your-first-use-case-with-clean-architecture)
