# Hexagonal Architecture (Ports & Adapters) — Explanation, Sample, Best Approach & Key Points

> Introduced by **Alistair Cockburn** (~2005). Renamed from "Hexagonal Architecture" to **"Ports and Adapters"** to make the intent unmistakable: the application has clear *ports* (interfaces) and *adapters* (implementations) on every side, and the hexagon is just a drawing convention.

> **Companion docs:**
> * [`clean-architecture.md`](./clean-architecture.md) — Uncle Bob's evolution of these ideas.
> * [`n-layer-architecture.md`](./n-layer-architecture.md)
> * [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)
> * [`architecture-comparison.md`](./architecture-comparison.md)

---

## Table of Contents

1. [What Is Hexagonal Architecture?](#1-what-is-hexagonal-architecture)
2. [Why a Hexagon? (Spoiler: It Doesn't Matter)](#2-why-a-hexagon-spoiler-it-doesnt-matter)
3. [Goal — Symmetric Isolation of the Application Core](#3-goal--symmetric-isolation-of-the-application-core)
4. [Core Concepts: Ports & Adapters](#4-core-concepts-ports--adapters)
5. [Primary vs. Secondary (Driving vs. Driven)](#5-primary-vs-secondary-driving-vs-driven)
6. [The Hexagon Diagram](#6-the-hexagon-diagram)
7. [Hexagonal in .NET — Solution Layout](#7-hexagonal-in-net--solution-layout)
8. [End-to-End Sample in .NET](#8-end-to-end-sample-in-net)
9. [Best Approach & Recommended Practices](#9-best-approach--recommended-practices)
10. [Advantages](#10-advantages)
11. [Disadvantages](#11-disadvantages)
12. [Common Pitfalls](#12-common-pitfalls)
13. [Testing Strategy — Where Hexagonal Truly Shines](#13-testing-strategy--where-hexagonal-truly-shines)
14. [Hexagonal vs. Onion vs. Clean vs. N-Layer](#14-hexagonal-vs-onion-vs-clean-vs-n-layer)
15. [Hexagonal vs. Microservices](#15-hexagonal-vs-microservices)
16. [When to Use (and When Not To)](#16-when-to-use-and-when-not-to)
17. [Key Points — The Short List](#17-key-points--the-short-list)
18. [Checklist](#18-checklist)
19. [References](#19-references)

---

## 1. What Is Hexagonal Architecture?

> *"Allow an application to equally be driven by users, programs, automated tests, or batch scripts, and to be developed and tested in isolation from its eventual run-time devices and databases."* — Alistair Cockburn, original paper

Hexagonal Architecture isolates the **application's business logic** ("the core") behind **technology-agnostic interfaces** called **ports**. Every external technology (HTTP API, CLI, database, message queue, SMTP, S3, third-party API) plugs in via an **adapter** that implements (or calls) a port.

The result: the **core knows nothing** about the outside world. You can swap the web framework, the database, or the message bus without touching a single line of business logic.

It is the **conceptual predecessor** of Clean Architecture and Onion Architecture — they refined the idea but kept its essence: **dependencies point inward, technology stays at the edges**.

---

## 2. Why a Hexagon? (Spoiler: It Doesn't Matter)

The hexagon shape has **no special meaning**. Cockburn drew a hexagon because:

- A rectangle suggested layers (which the diagram is **not**).
- A hexagon has multiple flat sides — handy for drawing many adapters around the core.
- The number six is arbitrary; the architecture works with 2, 3, or 12 sides.

Cockburn himself later admitted he should have called it **"Ports and Adapters"** from day one. Both names refer to the same pattern.

---

## 3. Goal — Symmetric Isolation of the Application Core

Layered architectures separate **top** (UI) from **bottom** (DB). Hexagonal does something subtler:

> Every external system — **inbound or outbound** — is treated **symmetrically**: it's "outside" the core, and the core talks to it through a port.

```
   user request  ▶ ────────┐
                            │ ──► primary port ──► CORE ──► secondary port ──► database
   scheduled job ▶ ─────────┘                                         └─────► message bus
   HTTP API ▶ ──────────────┘                                         └─────► email server
```

There is no "top" and no "bottom" — only **driving sides** (things calling the application) and **driven sides** (things the application calls).

---

## 4. Core Concepts: Ports & Adapters

### Port

A **port** is an interface that describes a capability the core either *exposes* or *requires*. Ports are owned by the core, defined in the core's language, and **technology-agnostic**.

Two kinds:

- **Primary / Driving port** — what the application offers (e.g., `IPlaceOrderUseCase`).
- **Secondary / Driven port** — what the application needs (e.g., `IOrderRepository`, `IEmailSender`).

### Adapter

An **adapter** is a concrete piece of code that **bridges** a port and a specific technology.

- **Primary / Driving adapter** — invokes a primary port from outside (e.g., an ASP.NET Core controller).
- **Secondary / Driven adapter** — implements a secondary port using a specific tech (e.g., `EfOrderRepository`, `SendGridEmailSender`).

> **Key rule:** the core depends on **ports** only. Adapters depend on **the core's ports** plus their specific technology. **Never the reverse.**

---

## 5. Primary vs. Secondary (Driving vs. Driven)

| Type           | Synonyms                      | Role                                         | Direction          | Example                                              |
| -------------- | ----------------------------- | -------------------------------------------- | ------------------ | ---------------------------------------------------- |
| **Primary**    | Driving, inbound, "left-side" | The world drives the application             | **In →** to core   | `OrdersController` calls `IPlaceOrderUseCase`        |
| **Secondary**  | Driven, outbound, "right-side"| The application drives the world             | Core → **out**     | Use case calls `IOrderRepository` → `EfRepository`   |

Cockburn drew driving adapters on the **left** of the hexagon and driven adapters on the **right**, but again the shape is incidental.

---

## 6. The Hexagon Diagram

```
                      Driving adapters                            Driven adapters
                      (primary side)                              (secondary side)
                            │                                            │
                            ▼                                            ▼
   ┌──────────┐       ┌─────────────────────────────────────────┐     ┌──────────────┐
   │ ASP.NET  │──port─►                                         ◄─port│   EF Core    │
   │ Controller│      │              APPLICATION CORE          │     │ Repository   │
   └──────────┘       │  ┌────────────────────────────────────┐│     └──────────────┘
                      │  │   Use Cases / Domain Services       ││
   ┌──────────┐       │  │   Entities, Value Objects, Aggreg.  ││     ┌──────────────┐
   │   CLI    │──port─►  │                                     ◄─port│ SendGrid     │
   └──────────┘       │  └────────────────────────────────────┘│     │ Email Adapter│
                      │                                         │     └──────────────┘
   ┌──────────┐       │                                         │     ┌──────────────┐
   │ Test     │──port─►                                         ◄─port│ Service Bus  │
   │ Harness  │       │                                         │     │ Adapter      │
   └──────────┘       └─────────────────────────────────────────┘     └──────────────┘

                  Dependencies always point INTO the hexagon  ─────►
```

Note that the **test harness is just another driving adapter**. That symmetry is the whole point: a test, a controller, and a CLI all drive the same primary ports.

---

## 7. Hexagonal in .NET — Solution Layout

```
src/
├── Shop.Core/                     ← The application core (hexagon)
│   ├── Domain/
│   │   ├── Order.cs
│   │   ├── OrderLine.cs
│   │   └── ValueObjects/
│   ├── UseCases/
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderRequest.cs
│   │   │   ├── PlaceOrderResponse.cs
│   │   │   ├── IPlaceOrderUseCase.cs     ← PRIMARY PORT
│   │   │   └── PlaceOrderUseCase.cs      ← Core implementation
│   │   └── CancelOrder/…
│   └── Ports/                            ← SECONDARY PORTS
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
├── Shop.Core.UnitTests/           ← In-memory fake adapters
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

Core                          → (nothing)
```

> **Critical:** `Core` has **no** references to ASP.NET, EF Core, or any third-party library beyond pure helpers. If you remove every adapter project, `Core` still compiles.

---

## 8. End-to-End Sample in .NET

### 8.1 Domain (inside Core)

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

### 8.3 Secondary ports (Core)

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

### 8.4 Use case implementation (Core)

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

// Core registrations
builder.Services.AddScoped<IPlaceOrderUseCase, PlaceOrderUseCase>();

// Driven adapter registrations
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();

builder.Services.AddSendGrid(builder.Configuration["SendGrid:ApiKey"]);
builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();

var app = builder.Build();
app.MapControllers();
app.Run();
```

> The **composition root** is the only place that knows about both Core and adapters. Everywhere else, dependencies flow through ports.

---

## 9. Best Approach & Recommended Practices

1. **Core has zero external dependencies** — no NuGet references beyond pure helpers.
2. **Ports live with the Core**, named in business terms (`IOrderRepository`), not tech terms (`ISqlServerOrderDao`).
3. **One project per adapter.** Forces hard reference boundaries.
4. **One composition root** (typically the web host). All wiring happens there.
5. **DTOs/requests in the Core** for primary ports; adapters translate from HTTP/JSON to these DTOs.
6. **Avoid leaking adapter types** (`DbContext`, `HttpClient`, `IBus`) into the Core — even as parameter types.
7. **Adapters can have their own internal models**, but they must map to/from Core types at the boundary.
8. **Architecture tests** in CI to enforce: *Core must not reference any adapter assembly*.
9. **Use `Result<T>`** for expected business failures; let exceptions cross only for programmer errors.
10. **Cancellation tokens** flow through every port method.
11. **Make ports small and focused** (Interface Segregation Principle). One port per real capability.
12. **Test the Core in isolation** using **in-memory fake adapters**, no mocking framework needed.
13. **Migrations & schema** live with the persistence adapter — never in the Core.
14. **Async by default** on every port method.
15. **No service locators inside the Core.** Constructor injection only.
16. **Don't put cross-cutting concerns inside the Core.** Logging and caching belong in adapters or decorators.
17. **Version primary ports** when you have external consumers (CLI, public API). Adapters can absorb compatibility shims.

---

## 10. Advantages

✅ **Symmetric isolation** — the Core is decoupled from **every** external system, not just the database.
✅ **Maximum testability** — Core tests need no web server, no DB, no mocking framework.
✅ **Technology swappability** — change web framework, ORM, message bus, or email provider by writing a new adapter.
✅ **Multiple driving adapters** — same Core can serve HTTP API, CLI, gRPC, and tests simultaneously.
✅ **Explicit boundaries** — ports declare exactly what the Core needs and offers.
✅ **Fits DDD perfectly** — the hexagon = the bounded context.
✅ **Encourages thin adapters** — most "framework code" reduces to glue.
✅ **Forces honest dependencies** — adapter-only NuGet packages can't sneak into the Core.

---

## 11. Disadvantages

❌ **More projects, more files** — overhead for small apps.
❌ **Learning curve** — devs must internalize "ports are owned by the core".
❌ **More mapping** — adapter types ↔ Core types translation everywhere.
❌ **Indirection** — calling `IOrderRepository` instead of `_db.Orders` can feel like ceremony at small scale.
❌ **Easy to get wrong** — many "hexagonal" codebases secretly reference EF Core in their domain.
❌ **Risk of over-abstraction** — defining a port for every trivial dependency clogs the design.
❌ **No prescribed internal structure** — Hexagonal says nothing about how use cases or entities are organized inside the Core; teams must layer or slice on their own.

---

## 12. Common Pitfalls

| Pitfall                                                  | What goes wrong                                                                         |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **EF Core types in the Core**                            | Breaks the symmetry; can't swap ORMs; tests need a DB.                                  |
| **Ports defined in the adapter project**                 | Inverts the dependency arrow — Core ends up depending on adapter.                       |
| **Generic `IRepository<T>`**                             | Hides intent; often a thin wrapper around EF — defeats the purpose.                     |
| **Returning `IQueryable` from a port**                   | Leaks LINQ-to-EF semantics into the Core.                                               |
| **Anemic Core**                                          | Use cases become script-like; domain logic drifts into adapters.                        |
| **One "Adapter" project holding everything**             | Loses the per-technology isolation that motivated the architecture.                     |
| **Service locator inside Core**                          | Hidden dependencies; impossible to unit-test cleanly.                                   |
| **Letting middleware leak into Core**                    | E.g., `HttpContext`, `ClaimsPrincipal`. Pass a tiny DTO like `CurrentUserDto` instead.  |
| **Cross-adapter calls**                                  | One adapter calling another bypasses the Core; reintroduces coupling.                   |

---

## 13. Testing Strategy — Where Hexagonal Truly Shines

The Core's tests use **in-memory fake adapters** instead of mocks. This is the whole reason the architecture exists.

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

| Test layer            | Targets                                  | Tools                                                |
| --------------------- | ---------------------------------------- | ---------------------------------------------------- |
| **Core unit**         | Use cases + domain                       | xUnit + FluentAssertions + in-memory fakes           |
| **Adapter integration** | Per-adapter, real tech (DB, queue, etc.) | xUnit + Testcontainers / WireMock                  |
| **End-to-end functional** | Full hexagon with all adapters         | `WebApplicationFactory<Program>` + Testcontainers   |
| **Architecture**      | "Core references no adapter"             | NetArchTest / ArchUnitNET                            |

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

| Aspect              | **N-Layer**                | **Hexagonal**                                       | **Onion**                          | **Clean**                                       |
| ------------------- | -------------------------- | --------------------------------------------------- | ---------------------------------- | ----------------------------------------------- |
| Year                | 1990s                      | ~2005                                               | 2008                               | 2012/2017                                       |
| Author              | Industry convention        | Alistair Cockburn                                   | Jeffrey Palermo                    | Robert C. Martin                                |
| Shape               | Stack                      | Hexagon (arbitrary)                                 | Concentric circles                 | Concentric circles                              |
| Core idea           | Top-down dependencies      | Ports + adapters, symmetric isolation               | Domain at center; dependencies inward | Synthesis of Onion + Hexagonal + Use-Case design |
| What it prescribes  | Layer responsibilities      | Boundary interfaces & adapters                      | Layer arrangement                  | Layers, Use Cases, Dependency Rule, boundaries  |
| Inner structure     | Layers                     | **Not prescribed**                                  | Domain → Services → App            | Entities → Use Cases → Interface Adapters → Frameworks |
| Test approach       | Mock layers                | Fake adapters; symmetrical test harness             | Mock outer rings                   | Pure-domain unit tests + boundary integration   |
| Relationship        | Older, broader             | Conceptual ancestor of Onion & Clean                | Refinement of Hexagonal             | Most prescriptive synthesis of the three        |

> **All three of Hexagonal, Onion, and Clean share the same essence: dependencies must point inward, and the domain is independent of frameworks.** They differ mostly in vocabulary and how strictly they prescribe internal structure.

---

## 15. Hexagonal vs. Microservices

Hexagonal Architecture is **per-service**. A microservices platform typically has many hexagons — each service has its own Core, ports, and adapters.

| Aspect             | Hexagonal               | Microservices                                          |
| ------------------ | ----------------------- | ------------------------------------------------------ |
| Scope              | Inside one application  | Across many applications                               |
| Boundary           | Process-internal (ports) | Network-external (HTTP, gRPC, queues)                  |
| Couples to         | Adapters                | Other services' contracts                              |
| Substitution       | Adapter swap             | Service replacement                                    |

Hexagonal and microservices are **orthogonal**. A monolith can be hexagonal; a microservice should be hexagonal.

---

## 16. When to Use (and When Not To)

### ✅ Strong fit

- Apps with **multiple driving channels** (HTTP API + CLI + scheduled jobs + tests).
- Apps where the **domain is the asset** (banking, insurance, healthcare, trading).
- Long-lived systems where **technology will outlive its current frameworks**.
- DDD-oriented teams that already use bounded contexts.
- Apps with heavy **integration** to external systems (Stripe, SAP, legacy SOAP, S3).

### ⚠️ Reconsider when

- Tiny CRUD app — the ceremony outweighs the benefit.
- Short-lived prototype — speed > flexibility.
- Team is brand new to .NET and DI — start with N-Layer and graduate to Hexagonal.

---

## 17. Key Points — The Short List

* **Ports & Adapters** is the real name; the hexagon shape is incidental.
* The **Core depends on nothing** external — adapters depend on the Core's ports.
* **Driving adapters** call into the Core; **driven adapters** are called by the Core.
* **One composition root** wires Core + adapters; nowhere else knows both.
* **Test the Core with fakes**, not mocks. Tests run instantly with zero infrastructure.
* **Architecture tests** enforce that Core references no adapter.
* Hexagonal is the **conceptual ancestor** of Onion and Clean — same essence, different vocabulary.
* Pair with **DDD**, **CQRS**, and **Vertical Slice** (inside the Core) for a complete modern stack.

---

## 18. Checklist

- [ ] `Core` project references **no adapter assembly** and **no framework NuGet** (beyond pure helpers).
- [ ] Every external capability the Core needs has a **port interface in the Core**.
- [ ] Each adapter lives in **its own project** named for the technology (`Adapters.Persistence.Ef`).
- [ ] Adapter projects reference **only** the Core (and their tech's libraries).
- [ ] **Composition root** in the host app is the **only** place that knows both Core and adapters.
- [ ] No EF Core entities, `IQueryable`, `HttpContext`, or other framework types appear in the Core.
- [ ] Unit tests in `Core.UnitTests` use **in-memory fakes** — no mocking framework needed.
- [ ] Adapter integration tests use **real tech** via Testcontainers.
- [ ] Architecture tests in CI verify Core's dependency boundary.
- [ ] Ports follow ISP — small, focused, intention-revealing names.
- [ ] All async port methods accept `CancellationToken`.
- [ ] No adapter calls another adapter directly — all flow goes through the Core.

---

## 19. References

### Primary Sources
* [Alistair Cockburn — Hexagonal Architecture (original)](https://alistair.cockburn.us/hexagonal-architecture)
* [Alistair Cockburn — Hexagonal Architecture Explained (2023 updated paper)](https://alistaircockburn.com/Hexagonal%20Budapest%2023-05-18.pdf)
* [Wikipedia — Hexagonal architecture (software)](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software))

### Practical Guides
* [Ports and Adapters (Hexagonal Architecture) — DEV](https://dev.to/rafaeljcamara/ports-and-adapters-hexagonal-architecture-547c)
* [Understanding Hexagonal Architecture — DEV](https://dev.to/xoubaman/understanding-hexagonal-architecture-3gk)
* [AWS Prescriptive Guidance — Hexagonal Architecture Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html)
* [hexagonalarchitecture.org](https://www.hexagonalarchitecture.org/)
