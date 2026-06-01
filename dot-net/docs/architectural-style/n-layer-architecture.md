# N-Layer Architecture in .NET — Complete Guide

> A practical reference for designing, structuring, and maintaining N-Layer (a.k.a. N-Tier / Layered) applications in modern .NET (.NET 6 / 7 / 8 / 9).

---

## Quick Reference (What · Why · When · Where)

- **What** — A stack of logical layers (typically Presentation → Application → Domain → Infrastructure) with strict top-down dependencies, all running in one process.
- **Why** — Separation of concerns; familiar to almost every developer; minimal ceremony; tooling-friendly (Visual Studio templates default to it).
- **When** — Small-to-medium CRUD-heavy line-of-business apps; teams new to layered design; monoliths with stable, well-understood domains; the natural starting point before deciding you need Clean.
- **Where** — One *style* on the logical axis. Distinct from N-Tier (physical deployment) and from Clean (principles-driven dependency rule). Often deployed as 2–3 tiers; refactor toward Clean if the domain grows complex.

---

## Table of Contents

1. [What Is N-Layer Architecture?](#1-what-is-n-layer-architecture)
2. [Layer vs. Tier](#2-layer-vs-tier)
3. [The Classic Layers](#3-the-classic-layers)
4. [Common Variations](#4-common-variations)
5. [Dependency Flow & Rules](#5-dependency-flow--rules)
6. [Typical .NET Solution Structure](#6-typical-net-solution-structure)
7. [Code Walkthrough by Layer](#7-code-walkthrough-by-layer)
8. [Dependency Injection Wiring](#8-dependency-injection-wiring)
9. [Cross-Cutting Concerns](#9-cross-cutting-concerns)
10. [Best Practices](#10-best-practices)
11. [Anti-Patterns to Avoid](#11-anti-patterns-to-avoid)
12. [Testing Strategy](#12-testing-strategy)
13. [N-Layer vs. Clean / Onion / Hexagonal / Vertical Slice](#13-n-layer-vs-clean--onion--hexagonal--vertical-slice)
14. [When to Use (and When Not To)](#14-when-to-use-and-when-not-to)
15. [Migration & Evolution Paths](#15-migration--evolution-paths)
16. [Checklist](#16-checklist)
17. [References](#17-references)

---

## 1. What Is N-Layer Architecture?

**N-Layer architecture** is a software design pattern that organizes an application into a stack of logical layers, each with a single, well-defined responsibility. Each layer communicates only with the layer directly below it (and is consumed by the layer directly above).

The classic three layers are **Presentation → Business Logic → Data Access**, but real-world systems often expand to 4–6 layers (Application, Domain, Infrastructure, Common, etc.) — hence "N".

The goal: **separation of concerns**, **testability**, and **maintainability** through controlled coupling.

---

## 2. Layer vs. Tier

These terms are often used interchangeably, but they are not the same:

| Term      | Meaning                                                                                  |
| --------- | ---------------------------------------------------------------------------------------- |
| **Layer** | A *logical* separation of code (e.g., projects, namespaces). Runs in the same process.   |
| **Tier**  | A *physical* deployment boundary (e.g., separate servers, containers, services).         |

Example: A monolith can have 4 **layers** but be deployed on a single **tier** (one web server + one database).

---

## 3. The Classic Layers

### 3.1 Presentation Layer

* **Purpose:** Expose the application to the outside world.
* **Examples:** ASP.NET Core Web API controllers, Razor Pages, Blazor components, gRPC services, minimal APIs.
* **Responsibilities:**
  * Accept HTTP/gRPC/CLI requests.
  * Validate input shape (model binding, FluentValidation).
  * Map between transport DTOs and application contracts.
  * Authentication / authorization (ASP.NET middleware).
  * Return appropriate response codes and payloads.
* **Should NOT:** contain business rules, talk directly to the database, or know about EF Core entities.

### 3.2 Application / Service Layer

* **Purpose:** Orchestrate use cases. The "verbs" of the system.
* **Examples:** `OrderService`, `UserRegistrationService`, command/query handlers.
* **Responsibilities:**
  * Coordinate domain objects and infrastructure (repositories, email, queues).
  * Transactions, unit-of-work boundaries.
  * Authorization rules tied to use cases.
  * DTO ↔ domain mapping.
* **Should NOT:** know about HTTP, EF Core internals, or specific UI framing.

### 3.3 Domain / Business Logic Layer

* **Purpose:** The "nouns" and invariants of your business.
* **Examples:** Entities (`Order`, `Invoice`), value objects (`Money`, `Email`), domain services, domain events.
* **Responsibilities:**
  * Enforce business invariants.
  * Encapsulate behavior in entities (not just data bags).
  * Define repository **interfaces** (implemented in Infrastructure).
* **Should NOT:** reference EF Core, HTTP, or any framework. Pure C#.

### 3.4 Data Access / Infrastructure Layer

* **Purpose:** Provide concrete implementations for technical concerns.
* **Examples:** EF Core `DbContext`, Dapper repositories, file storage, SMTP/SendGrid, Redis caches, message bus clients.
* **Responsibilities:**
  * Implement repository interfaces defined by Domain.
  * Database migrations and configurations.
  * External API clients (Stripe, AWS, etc.).
* **Should NOT:** contain business rules.

### 3.5 (Optional) Common / Shared / Cross-Cutting Layer

* Logging abstractions, result wrappers (`Result<T>`), guard clauses, custom exceptions, primitive value-object types. Typically referenced by all other layers.

---

## 4. Common Variations

| Variation     | Layers                                                                       | Notes                                         |
| ------------- | ---------------------------------------------------------------------------- | --------------------------------------------- |
| **3-Layer**   | Presentation → Business → Data                                               | Smallest viable form. Common for small apps.  |
| **4-Layer**   | Presentation → Application → Domain → Infrastructure                         | Microsoft's reference (eShopOnWeb).           |
| **5-Layer**   | Above + Common/Shared                                                        | Adds cross-cutting helpers.                   |
| **Strict**    | No layer skipping allowed (Presentation cannot talk to Data directly).        | Best for large teams.                          |
| **Relaxed**   | Higher layers may bypass middle layers (e.g., direct read queries).           | Sometimes used for CQRS read sides.            |

---

## 5. Dependency Flow & Rules

### 5.1 The Golden Rule

> **Dependencies point downward only.** A layer may know about layers below it; never above.

```
┌────────────────────┐
│   Presentation     │
└────────┬───────────┘
         │ depends on
         ▼
┌────────────────────┐
│   Application      │
└────────┬───────────┘
         │ depends on
         ▼
┌────────────────────┐
│   Domain           │  ← references nothing else (pure)
└────────────────────┘
         ▲
         │ implements interfaces from Domain
┌────────┴───────────┐
│ Infrastructure     │
└────────────────────┘
```

### 5.2 Dependency Inversion

The Domain defines `IOrderRepository`; Infrastructure provides `EfOrderRepository`. Application depends on the interface. This way, Domain stays pure and Infrastructure becomes pluggable.

This is the *real* power of layered architecture: **the database is a detail, not a foundation.**

---

## 6. Typical .NET Solution Structure

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
│   │   ├── Interfaces/              ← IEmailSender, ICurrentUser, etc.
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
│   └── MyShop.Common/               ← Cross-Cutting (optional)
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

> Note: `Infrastructure` references `Application` only to implement interfaces it defines (e.g., `IEmailSender`). If purity matters, define those interfaces in `Domain` instead and remove the reference.

---

## 7. Code Walkthrough by Layer

### Domain (pure C#)

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

### Application (use cases)

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

## 8. Dependency Injection Wiring

Each layer exposes its own `AddXxx` extension method, and `Program.cs` composes them.

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

### Service Lifetimes Cheat Sheet

| Lifetime    | When to use                                                              |
| ----------- | ------------------------------------------------------------------------ |
| `Singleton` | Stateless config, caches, expensive-to-build clients (HttpClient via IHttpClientFactory). |
| `Scoped`    | Per-request services: `DbContext`, repositories, application handlers.   |
| `Transient` | Lightweight stateless helpers (validators, mappers).                     |

**Pitfall:** Never inject a `Scoped` service into a `Singleton` — you'll capture a stale instance.

---

## 9. Cross-Cutting Concerns

Cross-cutting concerns must NOT leak business logic across layers. Address them centrally:

| Concern         | Where                                                               |
| --------------- | ------------------------------------------------------------------- |
| Logging         | Microsoft.Extensions.Logging + Serilog/OpenTelemetry, DI everywhere |
| Validation      | FluentValidation in Application; model binding in Presentation      |
| Caching         | `IDistributedCache` / `HybridCache` (.NET 9), wrapped behind interfaces |
| Transactions    | `IUnitOfWork` abstraction in Application; implemented in Infra      |
| Error handling  | Global exception middleware in API + domain-specific exceptions     |
| Auth            | ASP.NET Core middleware; pass `ICurrentUser` into Application       |
| Mapping         | AutoMapper / Mapster / hand-written extension methods               |
| Resilience      | Polly / `Microsoft.Extensions.Resilience` (.NET 8+)                 |

---

## 10. Best Practices

1. **Define interfaces near their consumer.** Put `IOrderRepository` in Domain (or Application), not in Infrastructure.
2. **Domain depends on nothing.** No EF Core, no Newtonsoft, no `HttpContext`. Keep it portable.
3. **DTOs at the boundary.** Don't return EF entities from controllers; map to DTOs.
4. **One `DbContext` per request (`Scoped`).** Never `Singleton`.
5. **Async all the way.** Avoid blocking calls in I/O paths; pass `CancellationToken`.
6. **Repositories return aggregates, not `IQueryable`.** Leaking `IQueryable` breaks encapsulation.
7. **Use `Result<T>` for expected failures.** Reserve exceptions for truly exceptional cases.
8. **Configuration via `IOptions<T>`.** Strongly typed, validated at startup.
9. **Centralize migrations** in Infrastructure; the API project triggers them on startup or via CI.
10. **Keep Presentation thin.** Controller actions ≈ 5–15 lines: validate → call handler → map result.
11. **One assembly per layer.** Enforces references; prevents accidental coupling.
12. **Use architecture tests** (`NetArchTest` / `ArchUnitNET`) in CI to enforce dependency rules.
13. **Don't share entity types across bounded contexts.** Each context owns its model.
14. **Avoid the "service-of-services" smell.** If `OrderService` calls `InvoiceService` calls `PaymentService`, push logic into the domain instead.
15. **Use feature folders** inside Application (group by use case, not by technical type).
16. **Add health checks** (`/health`, `/health/ready`) for the Presentation layer.
17. **Use minimal APIs or controllers consistently.** Don't mix without a reason.
18. **Source generators over reflection** where possible (e.g., System.Text.Json, Mapperly).

---

## 11. Anti-Patterns to Avoid

| Anti-Pattern                          | Why It's Bad                                                                                       |
| ------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Anemic domain model**               | Entities are data bags; all logic ends up in services. Erodes the value of the Domain layer.       |
| **Layer skipping**                    | Controller → Repository directly. Hides business rules and bypasses use-case orchestration.        |
| **God services**                      | `OrderService` with 40 methods. Split by use case.                                                 |
| **Leaking EF entities** to API        | Tightly couples API contract to DB schema; breaks on every migration.                              |
| **Returning `IQueryable`** from repos | Caller can mutate query semantics, defeating encapsulation.                                        |
| **Static helpers calling DI**         | Service-locator pattern; impossible to test.                                                       |
| **Async void**                        | Unhandled exceptions crash the process.                                                            |
| **Catch-and-swallow** exceptions      | Hides real bugs. Let global middleware handle it.                                                  |
| **Mixed sync + async** (`.Result`, `.Wait()`) | Deadlocks in ASP.NET classic; thread-pool starvation in Kestrel.                            |
| **Circular layer references**         | Architecturally illegal. Reach for an interface in a lower layer.                                  |
| **Shared "Common.Models" project** with EF entities | Becomes a dumping ground; couples everything.                                       |

---

## 12. Testing Strategy

| Test Type             | Targets                                  | Tools                                                    |
| --------------------- | ---------------------------------------- | -------------------------------------------------------- |
| **Unit (Domain)**     | Entities, value objects, domain services | xUnit + FluentAssertions; **no mocks needed**            |
| **Unit (Application)**| Handlers, services                       | xUnit + NSubstitute/Moq for repository interfaces        |
| **Integration**       | Infrastructure (real DB)                 | xUnit + Testcontainers / SQL Server in Docker            |
| **Functional / E2E**  | API end-to-end                           | `WebApplicationFactory<Program>` + Testcontainers        |
| **Architecture**      | Layer rules                              | NetArchTest.Rules / ArchUnitNET                          |

**Example architecture test:**

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

| Style              | Core Idea                                                | Strengths                                       | Weaknesses                                                   |
| ------------------ | -------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| **N-Layer**        | Stacked layers, top-down dependencies                    | Familiar; easy onboarding; quick to start       | Cross-layer changes touch many files; can become rigid       |
| **Clean / Onion**  | Domain in center; everything else points inward          | Maximum decoupling; framework-agnostic core     | More ceremony; over-engineered for small apps                |
| **Hexagonal**      | Ports (interfaces) & adapters (impls) around the core    | Pluggable I/O; great for messaging-heavy systems | Steeper learning curve                                       |
| **Vertical Slice** | One folder per feature; layers exist *within* the slice  | Low coupling between features; easy to delete   | Some duplication; weaker enforcement of cross-cutting rules  |

> **Practical advice:** Many modern .NET projects use a *hybrid* — Clean Architecture's layer structure with Vertical Slice organization (feature folders) inside `Application/`. This is sometimes called "Clean Slice."

---

## 14. When to Use (and When Not To)

### ✅ Good fit

* CRUD-heavy line-of-business apps.
* Teams new to layered design (gentle learning curve).
* Monoliths or modular monoliths.
* Apps with stable, well-understood domains.

### ⚠️ Reconsider when

* You have many independent features with little shared logic → consider **Vertical Slice**.
* Domain is rich and complex with many invariants → consider **Clean / DDD**.
* You're building a microservice that does one thing → 1–2 layers may be enough.
* Read-heavy systems with diverse query needs → consider **CQRS** with separate read models.

---

## 15. Migration & Evolution Paths

A pragmatic growth path:

1. **Start small:** 3-layer (API → Service → Repository).
2. **Extract Domain** when business rules start cluttering services.
3. **Split Application from Domain** when use cases multiply.
4. **Adopt CQRS** when read and write models diverge.
5. **Move to Vertical Slice** when features outgrow shared services.
6. **Extract microservices** when bounded contexts have independent scaling/release needs.

Do not skip ahead. Premature complexity is the most expensive form of waste.

---

## 16. Checklist

Before merging, verify:

- [ ] Domain project references **only** Common (or nothing).
- [ ] No EF Core types leak above Infrastructure.
- [ ] Controllers are thin (< 15 lines per action).
- [ ] Every external dependency is behind an interface.
- [ ] `DbContext` lifetime is `Scoped`.
- [ ] `CancellationToken` flows through async chains.
- [ ] DTOs are used at the API boundary (no entities returned).
- [ ] Migrations are reproducible and reviewed.
- [ ] Architecture tests pass in CI.
- [ ] Unit tests cover domain logic; integration tests cover repositories.
- [ ] Logging and exception middleware configured.
- [ ] Configuration validated at startup (`IOptions<T>` + `ValidateOnStart`).

---

## 17. References

* [Common web application architectures — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
* [eShopOnWeb reference application](https://github.com/dotnet-architecture/eShopOnWeb)
* [ASP.NET Boilerplate — N-Layer Architecture](https://aspnetboilerplate.com/Pages/Documents/NLayer-Architecture)
* [N-Layered vs Clean vs Vertical Slice (antondevtips)](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
* [Guide to Building an N-Tier Architecture for a .NET 8 Web API (Medium)](https://medium.com/@csns.giri/guide-to-building-an-n-tier-architecture-for-a-net-8-web-api-a49f4a83335e)
* [Layered (N-Tier) Architecture in .NET Core (DEV)](https://dev.to/dotnetfullstackdev/layered-n-tier-architecture-in-net-core-51ic)
* [Stackify — What is N-Tier Architecture?](https://stackify.com/n-tier-architecture/)
* [aghayeffemin/aspnetcore.ntier (GitHub sample)](https://github.com/aghayeffemin/aspnetcore.ntier)
