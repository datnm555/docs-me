# Hexagonal Architecture (Ports & Adapters) — Giải thích, Sample, Best Approach & Key Point

> Giới thiệu bởi **Alistair Cockburn** (~2005). Đổi tên từ "Hexagonal Architecture" sang **"Ports and Adapters"** để intent rõ ràng: application có *port* (interface) và *adapter* (implementation) rõ ràng mọi phía, hexagon chỉ là quy ước vẽ.

> **Companion doc:**
> * [`clean-architecture-vi.md`](./clean-architecture-vi.md)
> * [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md)
> * [`vertical-slice-architecture-vi.md`](./vertical-slice-architecture-vi.md)
> * [`architecture-comparison-vi.md`](./architecture-comparison-vi.md)

> 🇻🇳 Phiên bản tiếng Việt. English: [`hexagonal-architecture.md`](./hexagonal-architecture.md)

---

## Mục lục

1. [Hexagonal Architecture là gì?](#1-hexagonal-architecture-là-gì)
2. [Tại sao Hexagon? (Spoiler: không quan trọng)](#2-tại-sao-hexagon-spoiler-không-quan-trọng)
3. [Mục tiêu — Cô lập đối xứng Application Core](#3-mục-tiêu--cô-lập-đối-xứng-application-core)
4. [Khái niệm cốt lõi: Port & Adapter](#4-khái-niệm-cốt-lõi-port--adapter)
5. [Primary vs. Secondary (Driving vs. Driven)](#5-primary-vs-secondary-driving-vs-driven)
6. [Diagram Hexagon](#6-diagram-hexagon)
7. [Hexagonal trong .NET — Solution Layout](#7-hexagonal-trong-net--solution-layout)
8. [Sample End-to-End trong .NET](#8-sample-end-to-end-trong-net)
9. [Best Approach & Practice khuyến nghị](#9-best-approach--practice-khuyến-nghị)
10. [Ưu điểm](#10-ưu-điểm)
11. [Nhược điểm](#11-nhược-điểm)
12. [Cạm bẫy phổ biến](#12-cạm-bẫy-phổ-biến)
13. [Chiến lược Test — Nơi Hexagonal thật sự toả sáng](#13-chiến-lược-test--nơi-hexagonal-thật-sự-toả-sáng)
14. [Hexagonal vs. Onion vs. Clean vs. N-Layer](#14-hexagonal-vs-onion-vs-clean-vs-n-layer)
15. [Hexagonal vs. Microservices](#15-hexagonal-vs-microservices)
16. [Khi nào dùng (và không)](#16-khi-nào-dùng-và-không)
17. [Key Point — Danh sách ngắn](#17-key-point--danh-sách-ngắn)
18. [Checklist](#18-checklist)
19. [Tham khảo](#19-tham-khảo)

---

## 1. Hexagonal Architecture là gì?

> *"Cho phép application được drive đồng đều bởi user, program, test tự động, hoặc batch script, và được develop và test cô lập khỏi device và database runtime cuối cùng."* — Alistair Cockburn, paper gốc

Hexagonal Architecture cô lập **business logic của application** ("core") đằng sau **interface technology-agnostic** gọi là **port**. Mọi technology bên ngoài (HTTP API, CLI, database, message queue, SMTP, S3, third-party API) plug in qua **adapter** implement (hoặc call) một port.

Kết quả: **core không biết gì** về thế giới bên ngoài. Bạn có thể swap web framework, database, hay message bus mà không đụng một dòng business logic.

Nó là **tiền thân khái niệm** của Clean Architecture và Onion Architecture — chúng refine idea nhưng giữ essence: **dependency trỏ vào trong, technology ở rìa**.

---

## 2. Tại sao Hexagon? (Spoiler: không quan trọng)

Hình hexagon **không có ý nghĩa đặc biệt**. Cockburn vẽ hexagon vì:

- Hình chữ nhật gợi layer (mà diagram **không phải**).
- Hexagon có nhiều cạnh phẳng — tiện để vẽ nhiều adapter quanh core.
- Số sáu tuỳ ý; kiến trúc work với 2, 3, hay 12 cạnh.

Bản thân Cockburn sau thừa nhận nên gọi là **"Ports and Adapters"** từ ngày một. Cả hai tên chỉ cùng pattern.

---

## 3. Mục tiêu — Cô lập đối xứng Application Core

Kiến trúc layered tách **trên** (UI) khỏi **dưới** (DB). Hexagonal làm cái tinh tế hơn:

> Mọi hệ thống bên ngoài — **inbound hoặc outbound** — được treat **đối xứng**: nó "bên ngoài" core, và core nói chuyện với nó qua một port.

```
   user request  ▶ ────────┐
                            │ ──► primary port ──► CORE ──► secondary port ──► database
   scheduled job ▶ ─────────┘                                         └─────► message bus
   HTTP API ▶ ──────────────┘                                         └─────► email server
```

Không có "trên" và không có "dưới" — chỉ **driving side** (thứ call application) và **driven side** (thứ application call).

---

## 4. Khái niệm cốt lõi: Port & Adapter

### Port

**Port** là interface mô tả capability mà core hoặc *expose* hoặc *cần*. Port thuộc về core, định nghĩa trong ngôn ngữ của core, và **technology-agnostic**.

Hai loại:

- **Primary / Driving port** — application offer (vd., `IPlaceOrderUseCase`).
- **Secondary / Driven port** — application cần (vd., `IOrderRepository`, `IEmailSender`).

### Adapter

**Adapter** là mảnh code cụ thể **bắc cầu** giữa port và technology cụ thể.

- **Primary / Driving adapter** — invoke primary port từ bên ngoài (vd., ASP.NET Core controller).
- **Secondary / Driven adapter** — implement secondary port dùng tech cụ thể (vd., `EfOrderRepository`, `SendGridEmailSender`).

> **Quy tắc chính:** core phụ thuộc **port** thôi. Adapter phụ thuộc **port của core** cộng technology cụ thể. **Không bao giờ ngược lại.**

---

## 5. Primary vs. Secondary (Driving vs. Driven)

| Loại           | Đồng nghĩa                    | Vai trò                                      | Hướng              | Ví dụ                                                |
| -------------- | ----------------------------- | -------------------------------------------- | ------------------ | ---------------------------------------------------- |
| **Primary**    | Driving, inbound, "left-side" | Thế giới drive application                   | **In →** tới core  | `OrdersController` call `IPlaceOrderUseCase`         |
| **Secondary**  | Driven, outbound, "right-side"| Application drive thế giới                   | Core → **out**     | Use case call `IOrderRepository` → `EfRepository`    |

Cockburn vẽ driving adapter bên **trái** hexagon và driven adapter bên **phải**, nhưng hình lại là incidental.

---

## 6. Diagram Hexagon

```
                      Driving adapter                             Driven adapter
                      (primary side)                              (secondary side)
                            │                                            │
                            ▼                                            ▼
   ┌──────────┐       ┌─────────────────────────────────────────┐     ┌──────────────┐
   │ ASP.NET  │──port─►                                         ◄─port│   EF Core    │
   │ Controller│      │              APPLICATION CORE          │     │ Repository   │
   └──────────┘       │  ┌────────────────────────────────────┐│     └──────────────┘
                      │  │   Use Case / Domain Service         ││
   ┌──────────┐       │  │   Entity, Value Object, Aggregate   ││     ┌──────────────┐
   │   CLI    │──port─►  │                                     ◄─port│ SendGrid     │
   └──────────┘       │  └────────────────────────────────────┘│     │ Email Adapter│
                      │                                         │     └──────────────┘
   ┌──────────┐       │                                         │     ┌──────────────┐
   │ Test     │──port─►                                         ◄─port│ Service Bus  │
   │ Harness  │       │                                         │     │ Adapter      │
   └──────────┘       └─────────────────────────────────────────┘     └──────────────┘

                  Dependency luôn trỏ VÀO hexagon  ─────►
```

Lưu ý **test harness chỉ là driving adapter khác**. Sự đối xứng đó là điểm chính: test, controller, và CLI đều drive cùng primary port.

---

## 7. Hexagonal trong .NET — Solution Layout

```
src/
├── Shop.Core/                     ← Core application (hexagon)
│   ├── Domain/
│   │   ├── Order.cs
│   │   ├── OrderLine.cs
│   │   └── ValueObjects/
│   ├── UseCases/
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderRequest.cs
│   │   │   ├── PlaceOrderResponse.cs
│   │   │   ├── IPlaceOrderUseCase.cs     ← PRIMARY PORT
│   │   │   └── PlaceOrderUseCase.cs      ← Implementation Core
│   │   └── CancelOrder/…
│   └── Ports/                            ← SECONDARY PORT
│       ├── IOrderRepository.cs
│       ├── IEmailSender.cs
│       ├── IClock.cs
│       └── IEventPublisher.cs
│
├── Shop.Adapters.Web/             ← Driving adapter (HTTP)
│   ├── Controllers/OrdersController.cs
│   └── Program.cs
│
├── Shop.Adapters.Cli/             ← Driving adapter (CLI)
│
├── Shop.Adapters.Persistence.Ef/  ← Driven adapter (EF Core)
│   ├── AppDbContext.cs
│   └── EfOrderRepository.cs
│
├── Shop.Adapters.Email.SendGrid/  ← Driven adapter (SendGrid)
│   └── SendGridEmailSender.cs
│
└── Shop.Adapters.Messaging.AzureServiceBus/
    └── AzureServiceBusPublisher.cs

tests/
├── Shop.Core.UnitTests/           ← In-memory fake adapter
├── Shop.Adapters.Persistence.IntegrationTests/
└── Shop.Web.FunctionalTests/
```

### Project Reference Graph

```
Adapters.Web                  → Core
Adapters.Cli                  → Core
Adapters.Persistence.Ef       → Core
Adapters.Email.SendGrid       → Core
Adapters.Messaging.ServiceBus → Core

Core                          → (không gì)
```

> **Critical:** `Core` **không có** reference tới ASP.NET, EF Core, hay library third-party nào ngoài helper thuần. Nếu remove mọi project adapter, `Core` vẫn compile.

---

## 8. Sample End-to-End trong .NET

### 8.1 Domain (bên trong Core)

```csharp
// Shop.Core/Domain/Order.cs
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; }
    public Guid CustomerId { get; }
    public OrderStatus Status { get; private set; }

    public Order(Guid customerId)
    {
        Id = Guid.NewGuid();
        CustomerId = customerId;
        Status = OrderStatus.Draft;
    }

    public void AddLine(Guid productId, int qty, decimal price)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Order is not editable.");
        _lines.Add(new OrderLine(productId, qty, price));
    }

    public void Place()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Order is empty.");
        Status = OrderStatus.Placed;
    }
}
```

### 8.2 Primary port (Core)

```csharp
// Shop.Core/UseCases/PlaceOrder/IPlaceOrderUseCase.cs
public interface IPlaceOrderUseCase
{
    Task<PlaceOrderResponse> ExecuteAsync(PlaceOrderRequest request, CancellationToken ct);
}

public sealed record PlaceOrderRequest(Guid CustomerId, IReadOnlyList<PlaceOrderLine> Lines);
public sealed record PlaceOrderLine(Guid ProductId, int Quantity, decimal Price);
public sealed record PlaceOrderResponse(Guid OrderId);
```

### 8.3 Secondary port (Core)

```csharp
// Shop.Core/Ports/IOrderRepository.cs
public interface IOrderRepository
{
    Task AddAsync(Order order, CancellationToken ct);
    Task SaveChangesAsync(CancellationToken ct);
}

// Shop.Core/Ports/IEmailSender.cs
public interface IEmailSender
{
    Task SendOrderConfirmationAsync(Guid orderId, CancellationToken ct);
}
```

### 8.4 Implementation use case (Core)

```csharp
// Shop.Core/UseCases/PlaceOrder/PlaceOrderUseCase.cs
public sealed class PlaceOrderUseCase : IPlaceOrderUseCase
{
    private readonly IOrderRepository _orders;
    private readonly IEmailSender _email;

    public PlaceOrderUseCase(IOrderRepository orders, IEmailSender email)
    {
        _orders = orders;
        _email = email;
    }

    public async Task<PlaceOrderResponse> ExecuteAsync(
        PlaceOrderRequest req, CancellationToken ct)
    {
        var order = new Order(req.CustomerId);
        foreach (var line in req.Lines)
            order.AddLine(line.ProductId, line.Quantity, line.Price);

        order.Place();
        await _orders.AddAsync(order, ct);
        await _orders.SaveChangesAsync(ct);
        await _email.SendOrderConfirmationAsync(order.Id, ct);

        return new PlaceOrderResponse(order.Id);
    }
}
```

### 8.5 Driving adapter — ASP.NET Core controller

```csharp
// Shop.Adapters.Web/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly IPlaceOrderUseCase _placeOrder;
    public OrdersController(IPlaceOrderUseCase placeOrder) => _placeOrder = placeOrder;

    [HttpPost]
    public async Task<IActionResult> Place(PlaceOrderRequest req, CancellationToken ct)
    {
        var res = await _placeOrder.ExecuteAsync(req, ct);
        return CreatedAtAction(nameof(Place), new { id = res.OrderId }, res);
    }
}
```

### 8.6 Driven adapter — EF Core repository

```csharp
// Shop.Adapters.Persistence.Ef/EfOrderRepository.cs
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public async Task AddAsync(Order order, CancellationToken ct)
        => await _db.Orders.AddAsync(order, ct);

    public Task SaveChangesAsync(CancellationToken ct)
        => _db.SaveChangesAsync(ct);
}
```

### 8.7 Driven adapter — SendGrid email

```csharp
// Shop.Adapters.Email.SendGrid/SendGridEmailSender.cs
public sealed class SendGridEmailSender : IEmailSender
{
    private readonly ISendGridClient _client;
    public SendGridEmailSender(ISendGridClient client) => _client = client;

    public Task SendOrderConfirmationAsync(Guid orderId, CancellationToken ct)
    {
        var msg = MailHelper.CreateSingleEmail(/* … */);
        return _client.SendEmailAsync(msg, ct);
    }
}
```

### 8.8 Composition root — `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Core registration
builder.Services.AddScoped<IPlaceOrderUseCase, PlaceOrderUseCase>();

// Driven adapter registration
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();

builder.Services.AddSendGrid(builder.Configuration["SendGrid:ApiKey"]);
builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();

var app = builder.Build();
app.MapControllers();
app.Run();
```

> **Composition root** là nơi duy nhất biết về cả Core và adapter. Mọi nơi khác, dependency flow qua port.

---

## 9. Best Approach & Practice khuyến nghị

1. **Core có zero dependency ngoài** — không NuGet reference ngoài helper thuần.
2. **Port sống với Core**, đặt tên business term (`IOrderRepository`), không tech term (`ISqlServerOrderDao`).
3. **Một project per adapter.** Ép boundary reference cứng.
4. **Một composition root** (thường là web host). Mọi wiring ở đó.
5. **DTO/request trong Core** cho primary port; adapter dịch từ HTTP/JSON sang DTO này.
6. **Tránh leak adapter type** (`DbContext`, `HttpClient`, `IBus`) vào Core — kể cả như parameter type.
7. **Adapter có thể có model internal riêng**, nhưng phải map đến/từ Core type ở boundary.
8. **Architecture test** trong CI để enforce: *Core không reference assembly adapter nào*.
9. **Dùng `Result<T>`** cho fail business dự đoán; cho exception cross chỉ cho programmer error.
10. **Cancellation token** flow qua mọi method port.
11. **Làm port nhỏ và tập trung** (Interface Segregation Principle). Một port per capability thật.
12. **Test Core cô lập** dùng **in-memory fake adapter**, không cần mocking framework.
13. **Migration & schema** sống với adapter persistence — không bao giờ trong Core.
14. **Async mặc định** trên mọi method port.
15. **Không service locator bên trong Core.** Chỉ constructor injection.
16. **Đừng đặt cross-cutting concern bên trong Core.** Logging và caching thuộc adapter hoặc decorator.
17. **Version primary port** khi có consumer ngoài (CLI, public API). Adapter có thể absorb compatibility shim.

---

## 10. Ưu điểm

✅ **Cô lập đối xứng** — Core được decouple khỏi **mọi** hệ thống ngoài, không chỉ database.
✅ **Testability tối đa** — Test Core không cần web server, DB, mocking framework.
✅ **Swap technology** — đổi web framework, ORM, message bus, hay email provider bằng viết adapter mới.
✅ **Nhiều driving adapter** — cùng Core có thể phục vụ HTTP API, CLI, gRPC, và test đồng thời.
✅ **Boundary explicit** — port declare chính xác cái Core cần và offer.
✅ **Fit DDD hoàn hảo** — hexagon = bounded context.
✅ **Khuyến khích adapter mỏng** — phần lớn "framework code" giảm xuống thành glue.
✅ **Ép dependency trung thực** — NuGet package adapter-only không thể lén vào Core.

---

## 11. Nhược điểm

❌ **Nhiều project, nhiều file** — overhead cho app nhỏ.
❌ **Learning curve** — dev phải internalize "port thuộc về core".
❌ **Mapping nhiều** — translation type adapter ↔ type Core khắp nơi.
❌ **Indirection** — call `IOrderRepository` thay vì `_db.Orders` có thể cảm giác ceremony ở scale nhỏ.
❌ **Dễ làm sai** — nhiều codebase "hexagonal" bí mật reference EF Core trong domain.
❌ **Rủi ro over-abstraction** — định nghĩa port cho mọi dependency trivial làm nghẽn design.
❌ **Không prescribe cấu trúc internal** — Hexagonal không nói gì về cách use case hay entity được tổ chức bên trong Core; team phải layer hoặc slice tự.

---

## 12. Cạm bẫy phổ biến

| Cạm bẫy                                                  | Gì sai                                                                                  |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **EF Core type trong Core**                              | Phá đối xứng; không thể swap ORM; test cần DB.                                          |
| **Port định nghĩa trong project adapter**                | Đảo mũi tên dependency — Core phụ thuộc adapter.                                        |
| **Generic `IRepository<T>`**                             | Ẩn intent; thường wrapper mỏng quanh EF — đánh bại mục đích.                            |
| **Return `IQueryable` từ port**                          | Leak semantic LINQ-to-EF vào Core.                                                       |
| **Anemic Core**                                          | Use case thành như script; domain logic drift vào adapter.                              |
| **Một project "Adapter" giữ mọi thứ**                    | Mất isolation per-technology motivate kiến trúc.                                         |
| **Service locator bên trong Core**                       | Dependency ẩn; không thể unit-test sạch.                                                 |
| **Cho middleware leak vào Core**                         | Vd., `HttpContext`, `ClaimsPrincipal`. Pass DTO nhỏ như `CurrentUserDto` thay vào.       |
| **Call cross-adapter**                                   | Một adapter call adapter khác bypass Core; reintroduce coupling.                         |

---

## 13. Chiến lược Test — Nơi Hexagonal thật sự toả sáng

Test Core dùng **in-memory fake adapter** thay vì mock. Đây là lý do toàn bộ kiến trúc tồn tại.

```csharp
// Shop.Core.UnitTests/Fakes/InMemoryOrderRepository.cs
internal sealed class InMemoryOrderRepository : IOrderRepository
{
    public readonly List<Order> Items = new();
    public Task AddAsync(Order o, CancellationToken ct) { Items.Add(o); return Task.CompletedTask; }
    public Task SaveChangesAsync(CancellationToken ct) => Task.CompletedTask;
}

// Shop.Core.UnitTests/PlaceOrderUseCaseTests.cs
public class PlaceOrderUseCaseTests
{
    [Fact]
    public async Task Places_order_and_sends_email()
    {
        var orders = new InMemoryOrderRepository();
        var email = new RecordingEmailSender();
        var sut = new PlaceOrderUseCase(orders, email);

        var res = await sut.ExecuteAsync(new PlaceOrderRequest(
            Guid.NewGuid(),
            new[] { new PlaceOrderLine(Guid.NewGuid(), 1, 10m) }), default);

        orders.Items.Should().HaveCount(1);
        email.Sent.Should().ContainSingle(e => e.OrderId == res.OrderId);
    }
}
```

| Layer test            | Mục tiêu                                  | Tool                                                 |
| --------------------- | ----------------------------------------- | ---------------------------------------------------- |
| **Core unit**         | Use case + domain                         | xUnit + FluentAssertions + in-memory fake            |
| **Adapter integration** | Per-adapter, tech thật (DB, queue, v.v.) | xUnit + Testcontainers / WireMock                  |
| **End-to-end functional** | Hexagon đầy đủ với mọi adapter         | `WebApplicationFactory<Program>` + Testcontainers   |
| **Architecture**      | "Core không reference adapter"            | NetArchTest / ArchUnitNET                            |

```csharp
[Fact]
public void Core_must_not_reference_any_adapter()
{
    var result = Types.InAssembly(typeof(PlaceOrderUseCase).Assembly)
        .Should()
        .NotHaveDependencyOnAny(
            "Shop.Adapters.Web",
            "Shop.Adapters.Persistence.Ef",
            "Shop.Adapters.Email.SendGrid",
            "Microsoft.EntityFrameworkCore",
            "Microsoft.AspNetCore.Mvc")
        .GetResult();
    Assert.True(result.IsSuccessful);
}
```

---

## 14. Hexagonal vs. Onion vs. Clean vs. N-Layer

| Khía cạnh           | **N-Layer**                | **Hexagonal**                                       | **Onion**                          | **Clean**                                       |
| ------------------- | -------------------------- | --------------------------------------------------- | ---------------------------------- | ----------------------------------------------- |
| Năm                 | 1990s                      | ~2005                                               | 2008                               | 2012/2017                                       |
| Tác giả             | Quy ước ngành               | Alistair Cockburn                                   | Jeffrey Palermo                    | Robert C. Martin                                |
| Hình                | Stack                      | Hexagon (tuỳ ý)                                     | Vòng đồng tâm                      | Vòng đồng tâm                                   |
| Idea cốt lõi        | Dependency top-down         | Port + adapter, cô lập đối xứng                     | Domain ở trung tâm; dependency vào trong | Tổng hợp Onion + Hexagonal + Use-Case design |
| Quy định gì         | Trách nhiệm layer           | Interface boundary & adapter                        | Sắp xếp layer                      | Layer, Use Case, Dependency Rule, boundary      |
| Cấu trúc internal   | Layer                       | **Không quy định**                                  | Domain → Service → App             | Entity → Use Case → Interface Adapter → Framework |
| Cách tiếp cận test  | Mock layer                  | Fake adapter; test harness đối xứng                 | Mock vòng ngoài                    | Test domain thuần + integration boundary        |
| Quan hệ             | Cũ hơn, rộng hơn            | Tiền thân khái niệm của Onion & Clean               | Tinh chỉnh Hexagonal               | Tổng hợp prescriptive nhất của ba                |

> **Cả ba Hexagonal, Onion, và Clean share cùng essence: dependency phải trỏ vào trong, và domain độc lập framework.** Khác chủ yếu ở vocabulary và mức độ nghiêm ngặt prescribe cấu trúc internal.

---

## 15. Hexagonal vs. Microservices

Hexagonal Architecture là **per-service**. Nền tảng microservices thường có nhiều hexagon — mỗi service có Core, port, và adapter riêng.

| Khía cạnh          | Hexagonal               | Microservices                                          |
| ------------------ | ----------------------- | ------------------------------------------------------ |
| Phạm vi            | Bên trong một application | Qua nhiều application                                  |
| Boundary           | Internal process (port) | External network (HTTP, gRPC, queue)                   |
| Couple với         | Adapter                 | Contract của service khác                              |
| Substitution       | Swap adapter            | Thay service                                           |

Hexagonal và microservices là **trực giao**. Monolith có thể hexagonal; microservice nên hexagonal.

---

## 16. Khi nào dùng (và không)

### ✅ Phù hợp mạnh

- App với **nhiều kênh driving** (HTTP API + CLI + scheduled job + test).
- App nơi **domain là tài sản** (banking, insurance, healthcare, trading).
- Hệ thống sống lâu nơi **technology sẽ outlive framework hiện tại**.
- Team DDD-oriented đã dùng bounded context.
- App với **integration nặng** với hệ thống ngoài (Stripe, SAP, SOAP legacy, S3).

### ⚠️ Xem xét lại khi

- App CRUD nhỏ — ceremony lấn lợi ích.
- Prototype sống ngắn — tốc độ > flexibility.
- Team brand new với .NET và DI — khởi đầu N-Layer rồi graduate Hexagonal.

---

## 17. Key Point — Danh sách ngắn

* **Ports & Adapters** là tên thật; hình hexagon là incidental.
* **Core không phụ thuộc gì** bên ngoài — adapter phụ thuộc port của Core.
* **Driving adapter** call vào Core; **driven adapter** được Core call.
* **Một composition root** wire Core + adapter; không nơi khác biết cả hai.
* **Test Core với fake**, không mock. Test chạy instant với zero infrastructure.
* **Architecture test** enforce Core không reference adapter.
* Hexagonal là **tiền thân khái niệm** của Onion và Clean — cùng essence, vocabulary khác.
* Pair với **DDD**, **CQRS**, và **Vertical Slice** (bên trong Core) cho stack hiện đại đầy đủ.

---

## 18. Checklist

- [ ] Project `Core` reference **không adapter assembly** và **không framework NuGet** (ngoài helper thuần).
- [ ] Mọi capability ngoài Core cần có **interface port trong Core**.
- [ ] Mỗi adapter sống trong **project riêng** đặt tên cho technology (`Adapters.Persistence.Ef`).
- [ ] Project adapter reference **chỉ** Core (và library tech của chúng).
- [ ] **Composition root** trong host app là **nơi duy nhất** biết cả Core và adapter.
- [ ] Không EF Core entity, `IQueryable`, `HttpContext`, hay type framework khác xuất hiện trong Core.
- [ ] Unit test trong `Core.UnitTests` dùng **in-memory fake** — không cần mocking framework.
- [ ] Integration test adapter dùng **tech thật** qua Testcontainers.
- [ ] Architecture test trong CI verify boundary dependency Core.
- [ ] Port theo ISP — nhỏ, tập trung, intention-revealing.
- [ ] Mọi method port async accept `CancellationToken`.
- [ ] Không adapter call adapter khác trực tiếp — mọi flow đi qua Core.

---

## 19. Tham khảo

### Nguồn gốc
* [Alistair Cockburn — Hexagonal Architecture (gốc)](https://alistair.cockburn.us/hexagonal-architecture)
* [Alistair Cockburn — Hexagonal Architecture Explained (paper 2023 update)](https://alistaircockburn.com/Hexagonal%20Budapest%2023-05-18.pdf)
* [Wikipedia — Hexagonal architecture (software)](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))

### Practical Guide
* [Ports and Adapters (Hexagonal Architecture) — DEV](https://dev.to/rafaeljcamara/ports-and-adapters-hexagonal-architecture-547c)
* [Understanding Hexagonal Architecture — DEV](https://dev.to/xoubaman/understanding-hexagonal-architecture-3gk)
* [AWS Prescriptive Guidance — Hexagonal Architecture Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html)
* [hexagonalarchitecture.org](https://www.hexagonalarchitecture.org/)
