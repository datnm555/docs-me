# Clean Slice Architecture — Best Practices, Best Approach, Structure & Key Points

> The pragmatic hybrid: **Clean Architecture on the outside, Vertical Slice on the inside.**
> Clean enforces the *Dependency Rule* across project boundaries; Vertical Slice organizes the *Application* layer by feature.

> The most recommended modern .NET layout in 2025/2026.

> **Companion docs:**
> * [`clean-architecture.md`](./clean-architecture.md)
> * [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)
> * [`five-architectures-comparison-and-mixing.md`](./five-architectures-comparison-and-mixing.md)

---

## Quick Reference (What · Why · When · Where)

- **What** — A pragmatic hybrid: **Clean Architecture on the outside, Vertical Slice on the inside**. Clean enforces the Dependency Rule across project boundaries (Domain ← Application ← Infrastructure ← Web); Vertical Slice organizes the *Application* layer into one folder per use case.
- **Why** — Get the dependency-rule discipline of Clean (framework-free Domain, replaceable Infrastructure) *plus* the feature cohesion of Vertical Slice (one folder per use case, no god services).
- **When** — Modern .NET greenfield projects (2025+); evolving from N-Layer or Clean toward feature-first organization; team mature enough for MediatR + Pipeline Behaviors + FluentValidation.
- **Where** — The current default recommendation for new .NET projects. Implements the Dependency Rule with project references; organizes Application by feature; pairs naturally with CQRS, DDD aggregates, and 3-tier deployment.

---

## Table of Contents

1. [What Is Clean Slice?](#1-what-is-clean-slice)
2. [Why It Exists — The Pain Each Side Solves](#2-why-it-exists--the-pain-each-side-solves)
3. [Core Principles](#3-core-principles)
4. [Architectural Diagram](#4-architectural-diagram)
5. [Solution Structure](#5-solution-structure)
6. [Project Reference Map](#6-project-reference-map)
7. [Anatomy of a Single Slice](#7-anatomy-of-a-single-slice)
8. [End-to-End Code Sample](#8-end-to-end-code-sample)
9. [Pipeline Behaviors](#9-pipeline-behaviors)
10. [Dependency Injection Wiring](#10-dependency-injection-wiring)
11. [CQRS Inside a Slice](#11-cqrs-inside-a-slice)
12. [Domain Layer Discipline](#12-domain-layer-discipline)
13. [Infrastructure Layer Discipline](#13-infrastructure-layer-discipline)
14. [Best Practices](#14-best-practices)
15. [Best Approach — Decisions That Matter](#15-best-approach--decisions-that-matter)
16. [Common Pitfalls](#16-common-pitfalls)
17. [Testing Strategy](#17-testing-strategy)
18. [Architecture Tests](#18-architecture-tests)
19. [When to Use (and When Not To)](#19-when-to-use-and-when-not-to)
20. [Migration Paths](#20-migration-paths)
21. [Key Points — The Short List](#21-key-points--the-short-list)
22. [Checklist](#22-checklist)
23. [References](#23-references)

---

## 1. What Is Clean Slice?

**Clean Slice Architecture** is a pragmatic hybrid that combines:

- **Clean Architecture** for the *outer structure* — projects, layers, dependency rule.
- **Vertical Slice Architecture** for the *internal organization of the Application layer* — one folder per use case.

In short:

> **Clean defines who is allowed to depend on whom.**
> **Vertical Slice defines how features are organized inside.**

You get:
- The **Dependency Rule** enforced by project references (Clean).
- **Feature cohesion** inside `Application/Features/` (Vertical Slice).
- **Framework independence** of the Domain (Clean).
- **Low cross-feature coupling** between use cases (Vertical Slice).

This pairs naturally with **MediatR + FluentValidation + Pipeline behaviors + CQRS** for the use-case dispatch layer.

---

## 2. Why It Exists — The Pain Each Side Solves

### Pain of pure Vertical Slice

- No enforced layering — a slice can accidentally depend on EF Core directly.
- Domain logic drifts into handlers; entities become anemic.
- Cross-cutting rules (e.g., "every command publishes an event") hard to enforce globally.
- No clear home for shared domain entities and value objects.

### Pain of pure Clean Architecture

- Every feature touches many folders (`Domain`, `Application`, `Infrastructure`, `Web`).
- Boilerplate: separate interface files, service classes, mapping classes for each feature.
- Becomes a "god Application project" once features multiply.
- Refactoring one feature ripples across many files.

### What Clean Slice fixes

- **Clean's outer ring** keeps the domain pure and the dependency rule enforced.
- **Vertical Slice's feature folders** keep each use case cohesive and easy to find/delete.
- New devs read one folder to understand one feature.
- Cross-cutting concerns go through pipeline behaviors instead of ad-hoc layers.

---

## 3. Core Principles

1. **The Dependency Rule is non-negotiable.** Inward only: `Web → Infrastructure → Application → Domain`.
2. **The Domain is framework-free.** No EF Core, no ASP.NET, no MediatR — pure C#.
3. **The Application layer is organized by feature**, not by technical type (no `Services/`, no `Repositories/` folders at the Application level).
4. **Each slice owns its command, validator, handler, response DTO, and (optionally) endpoint.**
5. **Cross-cutting concerns live in MediatR pipeline behaviors**, not in base classes or service wrappers.
6. **Tolerate duplication between slices.** Refactor by extracting to Domain when the pattern is stable (rule of three).
7. **Infrastructure is replaceable.** EF Core, SendGrid, Redis are details behind interfaces.
8. **Endpoints are humble.** They translate HTTP ↔ Command/Query and nothing more.
9. **Slices do not call other slices.** They share **Domain** (entities, value objects) or communicate via **events**.
10. **Per-slice technology freedom** — a heavy read query can use Dapper; a write command can use EF Core. Both are inside `Application`, but the choice is local to the slice.

---

## 4. Architectural Diagram

```
                      ┌──────────────────────────────────────────────────┐
                      │                    Web                           │
                      │   (ASP.NET Core endpoints — one per feature)     │
                      └─────────────────────────┬────────────────────────┘
                                                │ depends on
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
   │  Abstractions/  ← interfaces consumed by slices (IUnitOfWork, IClock…)    │
   │  Behaviors/     ← MediatR pipeline behaviors                              │
   └─────────────────────────┬───────────────────────────────────────────────┘
                             │ depends on
                             ▼
                      ┌──────────────────────────────────────────────────┐
                      │                    Domain                        │
                      │   Entities • Value Objects • Domain Events       │
                      │   (pure C# — no framework deps)                  │
                      └──────────────────────────────────────────────────┘
                             ▲
                             │ implements abstractions
                      ┌──────┴──────────────────────────────────────────┐
                      │                Infrastructure                    │
                      │  EF Core • Cache • Email • External APIs        │
                      └──────────────────────────────────────────────────┘
```

> The Application layer references Domain. Infrastructure references Application + Domain to implement the abstractions Application declares. Web is the composition root.

---

## 5. Solution Structure

```
src/
├── Shop.Domain/                              ← Pure domain (innermost)
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
│       └── IClock.cs                         ← pure ports defined by Domain
│
├── Shop.Application/                         ← Use-case slices live here
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
│   ├── Abstractions/                         ← Ports defined by Application
│   │   ├── IOrderRepository.cs
│   │   ├── ICustomerRepository.cs
│   │   ├── IUnitOfWork.cs
│   │   ├── IEmailSender.cs
│   │   ├── ICurrentUser.cs
│   │   └── IEventPublisher.cs
│   │
│   ├── Behaviors/                            ← MediatR pipeline behaviors
│   │   ├── ValidationBehavior.cs
│   │   ├── LoggingBehavior.cs
│   │   ├── TransactionBehavior.cs
│   │   ├── PerformanceBehavior.cs
│   │   └── CachingBehavior.cs
│   │
│   ├── Common/                               ← Result<T>, errors, common DTOs
│   │   ├── Result.cs
│   │   └── Errors.cs
│   │
│   └── DependencyInjection.cs
│
├── Shop.Infrastructure/                      ← Adapters (outermost concerns)
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

> Endpoints can also live **inside the slice folder** (e.g., `Application/Features/Orders/PlaceOrder/PlaceOrderEndpoint.cs`) if you want true co-location. Both layouts are valid; pick one and stay consistent.

---

## 6. Project Reference Map

```
Shop.Web            → Shop.Application + Shop.Infrastructure + Shop.Domain
Shop.Application    → Shop.Domain
Shop.Infrastructure → Shop.Application + Shop.Domain
Shop.Domain         → (nothing)
```

Critical invariants:
- **`Shop.Domain` references nothing** (no NuGet beyond pure helpers like `Ardalis.GuardClauses`).
- **`Shop.Application` does not reference `Shop.Infrastructure`** — only declares interfaces in `Abstractions/`.
- **`Shop.Web`** is the *composition root* and the *only* project that references everything.

If you cannot delete `Shop.Infrastructure` and replace it with a fake without touching Application or Domain, the boundary is broken.

---

## 7. Anatomy of a Single Slice

Every slice has the same predictable shape:

```
Features/Orders/PlaceOrder/
├── PlaceOrderCommand.cs           ← MediatR IRequest<T>
├── PlaceOrderValidator.cs         ← FluentValidation
├── PlaceOrderHandler.cs           ← IRequestHandler<TReq, TRes>
└── PlaceOrderResponse.cs          ← DTO returned to the caller
```

Optionally (if endpoints live next to slices):

```
└── PlaceOrderEndpoint.cs          ← MapPost("/api/orders", …)
```

> A slice answers exactly one user intent. If you find yourself naming a slice `OrderOperations`, split it.

---

## 8. End-to-End Code Sample

### 8.1 Domain — entity with behavior

```csharp
// Shop.Domain/Orders/Order.cs
public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; }
    public Guid CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public Money Total => Money.Sum(_lines.Select(l => l.Subtotal));
    public IReadOnlyList<OrderLine> Lines => _lines;

    private Order(Guid id, Guid customerId)
    {
        Id = id;
        CustomerId = customerId;
        Status = OrderStatus.Draft;
    }

    public static Order Create(Guid customerId, IClock clock)
        => new(Guid.NewGuid(), customerId);

    public void AddLine(Guid productId, int qty, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a placed order.");
        if (qty <= 0)
            throw new DomainException("Quantity must be positive.");
        _lines.Add(new OrderLine(productId, qty, unitPrice));
    }

    public OrderPlaced Place()
    {
        if (_lines.Count == 0)
            throw new DomainException("Order must have at least one line.");
        Status = OrderStatus.Placed;
        return new OrderPlaced(Id, CustomerId, Total);
    }
}
```

### 8.2 Application — Abstractions

```csharp
// Shop.Application/Abstractions/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Shop.Application/Abstractions/IUnitOfWork.cs
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// Shop.Application/Abstractions/IEventPublisher.cs
public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct);
}
```

### 8.3 Slice — `PlaceOrder`

#### Command

```csharp
// Shop.Application/Features/Orders/PlaceOrder/PlaceOrderCommand.cs
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<PlaceOrderLine> Lines
) : IRequest<Result<PlaceOrderResponse>>;

public sealed record PlaceOrderLine(Guid ProductId, int Quantity, decimal UnitPrice);
```

#### Validator

```csharp
// Shop.Application/Features/Orders/PlaceOrder/PlaceOrderValidator.cs
public sealed class PlaceOrderValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty();
        RuleForEach(x => x.Lines).ChildRules(l =>
        {
            l.RuleFor(x => x.ProductId).NotEmpty();
            l.RuleFor(x => x.Quantity).GreaterThan(0);
            l.RuleFor(x => x.UnitPrice).GreaterThanOrEqualTo(0);
        });
    }
}
```

#### Handler

```csharp
// Shop.Application/Features/Orders/PlaceOrder/PlaceOrderHandler.cs
public sealed class PlaceOrderHandler
    : IRequestHandler<PlaceOrderCommand, Result<PlaceOrderResponse>>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;
    private readonly IEventPublisher _events;
    private readonly IClock _clock;

    public PlaceOrderHandler(
        IOrderRepository orders,
        IUnitOfWork uow,
        IEventPublisher events,
        IClock clock)
    {
        _orders = orders;
        _uow = uow;
        _events = events;
        _clock = clock;
    }

    public async Task<Result<PlaceOrderResponse>> Handle(
        PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, _clock);
        foreach (var l in cmd.Lines)
            order.AddLine(l.ProductId, l.Quantity, new Money(l.UnitPrice));

        var evt = order.Place();

        await _orders.AddAsync(order, ct);
        await _uow.SaveChangesAsync(ct);
        await _events.PublishAsync(evt, ct);

        return Result.Success(new PlaceOrderResponse(order.Id, order.Total.Amount));
    }
}
```

#### Response

```csharp
// Shop.Application/Features/Orders/PlaceOrder/PlaceOrderResponse.cs
public sealed record PlaceOrderResponse(Guid OrderId, decimal Total);
```

### 8.4 Infrastructure — EF repository

```csharp
// Shop.Infrastructure/Persistence/Repositories/EfOrderRepository.cs
internal sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetAsync(Guid id, CancellationToken ct) =>
        _db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await _db.Orders.AddAsync(order, ct);
}
```

### 8.5 Web — Endpoint

```csharp
// Shop.Web/Endpoints/Orders/PlaceOrderEndpoint.cs
public static class PlaceOrderEndpoint
{
    public static IEndpointRouteBuilder MapPlaceOrder(this IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (
            PlaceOrderCommand cmd,
            ISender mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value.OrderId}", result.Value)
                : Results.BadRequest(result.Error);
        });
        return app;
    }
}
```

---

## 9. Pipeline Behaviors

Pipeline behaviors are MediatR middlewares that wrap every handler. They replace the "service base class" or "AOP filter" approach found in pure N-Layer.

```csharp
// Shop.Application/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var ctx = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(ctx, ct))))
            .SelectMany(r => r.Errors).Where(f => f is not null).ToList();

        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}
```

Common behaviors and the order they should run:

| Order | Behavior              | Purpose                                                |
| :---: | --------------------- | ------------------------------------------------------ |
| 1     | `LoggingBehavior`     | First in / last out — wraps everything.                |
| 2     | `PerformanceBehavior` | Alert on slow handlers (> 500ms threshold).            |
| 3     | `ValidationBehavior`  | Reject invalid commands before any work happens.       |
| 4     | `AuthorizationBehavior`| Check permissions per request.                        |
| 5     | `CachingBehavior`     | Return cached response for queries.                    |
| 6     | `TransactionBehavior` | Wrap command in a DB transaction (commands only).      |
| 7     | (handler)             | Your slice handler.                                    |

---

## 10. Dependency Injection Wiring

Each project exposes one `AddXxx` extension. `Program.cs` composes them.

```csharp
// Shop.Application/DependencyInjection.cs
public static class ApplicationServiceCollectionExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection s)
    {
        s.AddMediatR(c => c.RegisterServicesFromAssembly(typeof(ApplicationServiceCollectionExtensions).Assembly));
        s.AddValidatorsFromAssembly(typeof(ApplicationServiceCollectionExtensions).Assembly);

        s.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
        s.AddTransient(typeof(IPipelineBehavior<,>), typeof(PerformanceBehavior<,>));
        s.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        s.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));

        return s;
    }
}
```

```csharp
// Shop.Infrastructure/DependencyInjection.cs
public static class InfrastructureServiceCollectionExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection s, IConfiguration cfg)
    {
        s.AddDbContext<AppDbContext>(o =>
            o.UseSqlServer(cfg.GetConnectionString("Default"),
                sql => sql.EnableRetryOnFailure()));

        s.AddScoped<IOrderRepository, EfOrderRepository>();
        s.AddScoped<ICustomerRepository, EfCustomerRepository>();
        s.AddScoped<IUnitOfWork, EfUnitOfWork>();

        s.AddSingleton<IClock, SystemClock>();
        s.AddScoped<IEmailSender, SendGridEmailSender>();
        s.AddScoped<IEventPublisher, AzureServiceBusPublisher>();

        return s;
    }
}
```

```csharp
// Shop.Web/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration);

builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();
app.MapPlaceOrder();
app.MapCancelOrder();
app.MapGetOrder();
app.MapListOrders();
app.MapRegisterCustomer();

app.Run();
```

---

## 11. CQRS Inside a Slice

Commands and queries are different `IRequest<T>` types — that's all CQRS means here.

| Aspect              | Command                                      | Query                                        |
| ------------------- | -------------------------------------------- | -------------------------------------------- |
| Intent              | Change state                                 | Read state                                   |
| Returns             | `Result<T>` (or void)                        | DTO (read model)                             |
| Goes through        | Domain (entities + invariants)               | Often bypasses entities; direct projection   |
| Persistence         | EF Core (change tracking)                    | Dapper / EF Core / raw SQL — whatever's fast |
| Transaction         | `TransactionBehavior` wraps it               | Usually no transaction                       |

A read slice can look very different from a write slice:

```csharp
// Shop.Application/Features/Orders/ListOrders/ListOrdersHandler.cs
public sealed class ListOrdersHandler
    : IRequestHandler<ListOrdersQuery, IReadOnlyList<OrderSummaryDto>>
{
    private readonly IDbConnection _db;
    public ListOrdersHandler(IDbConnection db) => _db = db;

    public async Task<IReadOnlyList<OrderSummaryDto>> Handle(
        ListOrdersQuery q, CancellationToken ct)
    {
        const string sql = @"
            SELECT Id, CustomerId, Total, Status
            FROM Orders
            WHERE CustomerId = @CustomerId
            ORDER BY CreatedAt DESC
            OFFSET @Skip ROWS FETCH NEXT @Take ROWS ONLY;";

        var rows = await _db.QueryAsync<OrderSummaryDto>(
            new CommandDefinition(sql, new { q.CustomerId, q.Skip, q.Take }, cancellationToken: ct));
        return rows.AsList();
    }
}
```

Per-slice technology freedom is fine here — that's a feature, not a bug.

---

## 12. Domain Layer Discipline

The Domain is the heart of Clean Slice. Get this wrong and the rest unravels.

- **Pure C#.** No NuGet references except very pure helpers (`Ardalis.GuardClauses`).
- **No EF Core attributes** in entities. Use EF Core Fluent API in `Infrastructure/Persistence/Configurations/`.
- **Rich entities** — methods enforce invariants. No public setters.
- **Value objects** for things with no identity (`Money`, `EmailAddress`, `Address`).
- **Domain events** for things the domain notices that other parts may care about.
- **Domain services** only when behavior doesn't belong to a single entity.
- **No `IRepository<T>`** generic abstraction. Repositories are per-aggregate and return aggregates, not `IQueryable`.

---

## 13. Infrastructure Layer Discipline

- **`internal`** classes by default. The only public surface is `AddInfrastructure(...)`.
- **EF Core configurations** in `Persistence/Configurations/` — keep the Domain free of EF attributes.
- **Repositories implement** the interfaces declared in `Application/Abstractions/`.
- **External SDKs** wrapped behind small interfaces (`IEmailSender`, `IPaymentGateway`).
- **Migrations** live here; run them in CI or at app startup in dev only.
- **Resilience policies** (Polly / `Microsoft.Extensions.Resilience`) applied to outbound calls.
- **Outbox pattern** for "DB write + message publish" combinations.

---

## 14. Best Practices

1. **One folder per use case** inside `Application/Features/<Bounded Context>/<UseCase>/`.
2. **Domain has zero infrastructure dependencies.** Verify via architecture tests.
3. **All slice-consumed interfaces** live in `Application/Abstractions/`.
4. **No EF entities returned to the Web layer.** Always return slice-specific response DTOs.
5. **Use `Result<T>` for expected failures**; reserve exceptions for truly exceptional cases.
6. **One handler ≤ ~80 lines.** If it grows, push logic into entities or extract a domain service.
7. **`CancellationToken` flows through every async call** — repositories, behaviors, handlers, endpoints.
8. **Endpoints ≤ ~15 lines.** They translate HTTP and dispatch to MediatR. Period.
9. **Use minimal APIs over controllers** for new code in .NET 8/9 — less ceremony.
10. **Group endpoints with extension methods** (`MapPlaceOrder()`, `MapCancelOrder()`).
11. **MediatR pipeline behaviors handle cross-cutting**: logging, validation, transactions, performance, caching.
12. **Each slice is independent.** Slices share **Domain** (entities, value objects) and **events**, never **services**.
13. **Per-slice tech freedom** — Dapper for heavy read queries, EF Core for writes, raw SQL where it pays.
14. **Domain events + Outbox pattern** for side effects that must survive the use case's transaction.
15. **Architecture tests in CI** to enforce the Dependency Rule mechanically.
16. **Unit tests** focus on Domain (zero infra) and slice handlers (mock abstractions).
17. **Integration tests** focus on Infrastructure (real DB via Testcontainers).
18. **Functional tests** focus on full slices via `WebApplicationFactory<Program>`.
19. **`internal` everywhere it makes sense** — especially in Infrastructure.
20. **Source generators** (`System.Text.Json`, `Mapperly`) over reflection for AOT readiness.
21. **`IOptions<T>` + `ValidateOnStart()`** for all configuration; fail fast on misconfiguration.
22. **Don't add a "Shared.Services" project** between Application and Infrastructure. That re-creates N-Layer pain.

---

## 15. Best Approach — Decisions That Matter

These are the choices that make or break a Clean Slice codebase. Get them right early.

### Choice 1 — Where do endpoints live?

- **Option A (recommended):** in `Shop.Web/Endpoints/<BoundedContext>/`. Keeps web/HTTP concerns isolated.
- **Option B:** co-located inside the slice folder under `Shop.Application/Features/`. Maximum cohesion, but couples Application to ASP.NET extension methods.

Either is valid. Pick one and apply it everywhere.

### Choice 2 — How are interfaces named and grouped?

- **Per-aggregate repositories**: `IOrderRepository`, `ICustomerRepository`. ✅
- **Generic `IRepository<T>`**: avoid. ❌ — leaks `IQueryable` and discourages aggregate design.

### Choice 3 — Where do shared interfaces live?

- **Domain `Abstractions/`** for pure ports (e.g., `IClock`, `IIdGenerator`) — used by entities.
- **Application `Abstractions/`** for ports the use cases need (e.g., `IOrderRepository`, `IEmailSender`).

### Choice 4 — Validation strategy

- **FluentValidation in the slice** for input shape (`PlaceOrderValidator`).
- **Domain invariants in entities** for rules (`Order.Place()` throws if empty).
- **Don't duplicate** — input validation catches malformed requests; domain catches business violations.

### Choice 5 — Result type vs. exceptions

- **`Result<T>`** for expected failures returned to the caller (insufficient stock, customer not found).
- **Exceptions** for programmer errors and truly exceptional cases (corrupt data, infrastructure outage).
- **Global exception middleware** translates uncaught exceptions to `ProblemDetails` responses.

### Choice 6 — Transactions

- Default: **`TransactionBehavior` wraps every command** in a DB transaction.
- Read queries: **no transaction**.
- External calls (email, message bus): **outside the transaction** — use the Outbox pattern.

### Choice 7 — Mapping strategy

- **Explicit constructors / `FromDomain()` static methods** on response DTOs. ✅
- **AutoMapper / Mapster** everywhere — fine for some teams, but adds hidden runtime cost and reflection surprises.
- **Mapperly source generator** — best of both worlds for performance-sensitive paths.

### Choice 8 — Domain events delivery

- **In-memory dispatch** via MediatR `INotification` — fine for small apps.
- **Outbox + external bus** (Azure Service Bus / RabbitMQ / Kafka) — for cross-service or durable delivery.

---

## 16. Common Pitfalls

| Pitfall                                                              | What goes wrong                                                                |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Domain references EF Core**                                         | Breaks the Dependency Rule; Clean is broken even if folders look right.        |
| **Generic `IRepository<T>`**                                          | Becomes a thin EF wrapper; encourages anemic domain.                            |
| **Returning `IQueryable` from a repository**                          | Caller composes queries → encapsulation breaks; magic deferred execution bugs. |
| **Fat handlers (200+ lines)**                                         | Domain logic in the handler. Push it into entities or domain services.         |
| **Returning EF entities to the Web layer**                            | Couples the API contract to the DB schema; breaks on every migration.          |
| **Cross-slice handler calls**                                         | Slices coupled directly. Use events or shared domain instead.                  |
| **"Shared service" projects between Application and Infrastructure**  | Re-creates layered coupling; defeats Clean Slice.                              |
| **Treating MediatR as the architecture**                              | MediatR is plumbing. The architecture is **structure + rules**.                |
| **Skipping FluentValidation**                                         | Validation drifts into handlers inconsistently across slices.                  |
| **Skipping architecture tests**                                       | Boundary violations accumulate over time, invisible until they cause pain.     |
| **Mixed minimal APIs and controllers** without rationale              | Confuses readers. Pick one style per slice or per bounded context.             |
| **AutoMapper everywhere**                                             | Hidden mapping bugs; runtime errors. Prefer explicit mapping for clarity.      |
| **Sync-over-async (`.Result`, `.Wait()`)**                            | Thread-pool starvation; deadlocks under load.                                  |
| **No CancellationToken propagation**                                  | Hung downstream calls take down the API.                                       |
| **No outbox for "DB + publish" pairs**                                | Lost events when the publish step fails after the DB commits.                  |

---

## 17. Testing Strategy

Clean Slice gives you a pyramid that maps directly to your code structure.

| Test Layer            | Targets                                          | Tools                                                          | Speed       |
| --------------------- | ------------------------------------------------ | -------------------------------------------------------------- | ----------- |
| **Domain unit**       | Entities, value objects, domain services         | xUnit + FluentAssertions, **no mocks**                         | Very fast   |
| **Application unit**  | Slice handlers (mock `Abstractions/`)            | xUnit + NSubstitute / Moq                                      | Fast        |
| **Infrastructure integration** | Repositories, external clients (real DB)| xUnit + Testcontainers                                         | Medium      |
| **Functional / E2E**  | Full slice through HTTP → DB                     | `WebApplicationFactory<Program>` + Testcontainers              | Slower      |
| **Architecture**      | Layer rules                                      | NetArchTest.Rules / ArchUnitNET                                | Very fast   |

### Slice handler test example

```csharp
public class PlaceOrderHandlerTests
{
    [Fact]
    public async Task Places_order_and_publishes_event()
    {
        var orders = Substitute.For<IOrderRepository>();
        var uow = Substitute.For<IUnitOfWork>();
        var events = Substitute.For<IEventPublisher>();
        var clock = Substitute.For<IClock>();
        var sut = new PlaceOrderHandler(orders, uow, events, clock);

        var result = await sut.Handle(new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Lines: new[] { new PlaceOrderLine(Guid.NewGuid(), 2, 10m) }), default);

        result.IsSuccess.Should().BeTrue();
        await orders.Received(1).AddAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
        await uow.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
        await events.Received(1).PublishAsync(Arg.Any<OrderPlaced>(), Arg.Any<CancellationToken>());
    }
}
```

---

## 18. Architecture Tests

These run in CI and fail the build if the Dependency Rule is violated.

```csharp
public class ArchitectureTests
{
    [Fact]
    public void Domain_must_not_reference_Application_or_Infrastructure()
    {
        var result = Types.InAssembly(typeof(Order).Assembly)
            .Should()
            .NotHaveDependencyOnAny(
                "Shop.Application",
                "Shop.Infrastructure",
                "Microsoft.EntityFrameworkCore",
                "Microsoft.AspNetCore")
            .GetResult();
        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Application_must_not_reference_Infrastructure_or_Web()
    {
        var result = Types.InAssembly(typeof(PlaceOrderHandler).Assembly)
            .Should()
            .NotHaveDependencyOnAny("Shop.Infrastructure", "Shop.Web")
            .GetResult();
        Assert.True(result.IsSuccessful);
    }

    [Fact]
    public void Handlers_must_end_with_Handler_suffix()
    {
        var result = Types.InAssembly(typeof(PlaceOrderHandler).Assembly)
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .Should().HaveNameEndingWith("Handler")
            .GetResult();
        Assert.True(result.IsSuccessful);
    }
}
```

---

## 19. When to Use (and When Not To)

### ✅ Strong fit

- New or growing **SaaS / line-of-business** apps in .NET.
- **Many independent use cases** with shared domain entities.
- Long-lived systems where **maintainability beats short-term velocity**.
- Teams that practice **CI / trunk-based development / feature flags**.
- Apps needing **CQRS read/write split** for performance.

### ⚠️ Reconsider when

- The system is a **tiny CRUD admin tool** → plain N-Layer is fine.
- The domain is **trivial** → Clean Slice ceremony costs more than it saves.
- The team is **brand new to .NET** → start with N-Layer; migrate to Clean Slice later.
- A **legacy codebase** would require months of rewrites → use the Strangler Fig pattern.

---

## 20. Migration Paths

### From plain N-Layer

1. Add `Shop.Domain` project; move entities and value objects into it.
2. Move business rules from services *into* entities; entities become rich.
3. Add `Shop.Application` project; move services into use-case handlers (one folder per use case).
4. Move interfaces from `Domain` (if any) into `Application/Abstractions/`.
5. Stop returning EF entities from controllers; introduce slice-specific response DTOs.
6. Add MediatR + FluentValidation + pipeline behaviors.
7. Add architecture tests to lock in the boundaries.

### From pure Vertical Slice (no Clean shell)

1. Extract entities and value objects into a `Domain` project.
2. Move framework-coupled types (EF Core configurations, SendGrid client, etc.) into `Infrastructure`.
3. Add `Application/Abstractions/` with the interfaces slices need.
4. Add architecture tests for the Dependency Rule.

### From pure Clean Architecture

1. Inside `Application/`, replace `Services/`, `Commands/`, `Queries/` folders with `Features/<BoundedContext>/<UseCase>/`.
2. Co-locate command, validator, handler, response per use case.
3. Drop service classes that wrap a single handler — the handler IS the use case.
4. Optionally move endpoints next to slices (Choice 1 above).

---

## 21. Key Points — The Short List

- **Clean outside, Slice inside.** Clean enforces who can depend on whom; Slice organizes features.
- **The Dependency Rule is non-negotiable** — Domain ← Application ← Infrastructure / Web.
- **One folder per use case** with command + validator + handler + response.
- **Domain is pure C#** — no frameworks, no EF Core, no MediatR.
- **Application is framework-light** but uses MediatR + FluentValidation.
- **Infrastructure is replaceable** — EF, SendGrid, Redis behind interfaces.
- **Endpoints are humble** — HTTP translation only.
- **Cross-cutting via MediatR pipeline behaviors**, not base classes.
- **Per-slice tech freedom** — Dapper for reads, EF Core for writes.
- **Architecture tests in CI** enforce boundaries.
- **Tolerate duplication** between slices; refactor via rule of three.
- **Result\<T\>** for expected failures; exceptions for the truly exceptional.
- **Outbox pattern** for any "DB write + publish" combination.

---

## 22. Checklist

Before merging a slice:

- [ ] Slice lives in `Application/Features/<Context>/<UseCase>/`.
- [ ] Folder contains: command, validator, handler, response.
- [ ] Handler ≤ ~80 lines; domain logic lives in entities/value objects.
- [ ] No EF Core / ASP.NET types in Domain.
- [ ] No `Shop.Infrastructure` reference from `Shop.Application`.
- [ ] Endpoint ≤ ~15 lines, pure HTTP translation.
- [ ] All `Application/Abstractions/` interfaces have at least one Infrastructure implementation.
- [ ] `CancellationToken` flows through every async call.
- [ ] FluentValidation validator exists for every command.
- [ ] Returns `Result<T>` for expected failure paths.
- [ ] Unit tests cover happy path + at least one failure path.
- [ ] Integration test exists if the slice touches Infrastructure.
- [ ] Functional test for any slice exposed via HTTP.
- [ ] Architecture tests still pass.
- [ ] No new "shared service" abstraction added without the rule-of-three justification.
- [ ] Endpoint registered in `Program.cs` (or via convention if you use a `MapEndpoints()` discoverer).
- [ ] Commands are idempotent (or accept an idempotency key).
- [ ] If the slice publishes events, the outbox pattern is wired up.
- [ ] Configuration (if any) is `IOptions<T>` with `ValidateOnStart()`.

---

## 23. References

- [`clean-architecture.md`](./clean-architecture.md)
- [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)
- [`five-architectures-comparison-and-mixing.md`](./five-architectures-comparison-and-mixing.md)
- Robert C. Martin — *Clean Architecture* (2017)
- [Jimmy Bogard — Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
- [Milan Jovanović — Vertical Slice Architecture](https://www.milanjovanovic.tech/blog/vertical-slice-architecture)
- [Jason Taylor — Clean Architecture Solution Template](https://github.com/jasontaylordev/CleanArchitecture)
- [Amichai Mantinband — Clean Architecture Template](https://github.com/amantinband/clean-architecture)
- [Microsoft ISE Devblog — Next-Level Clean Architecture Boilerplate](https://devblogs.microsoft.com/ise/next-level-clean-architecture-boilerplate/)
- [Ardalis — Clean Architecture template](https://github.com/ardalis/CleanArchitecture)
- [NetArchTest](https://github.com/BenMorris/NetArchTest)
- [ArchUnitNET](https://github.com/TNG/ArchUnitNET)
