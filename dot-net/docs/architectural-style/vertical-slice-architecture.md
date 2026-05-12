# Vertical Slice Architecture — Explanation, Sample, Best Approach & Key Points

> Organize code by **feature**, not by **technical layer**. Each feature owns everything it needs — from HTTP endpoint to database — in one folder.

> Coined and popularized in .NET by **Jimmy Bogard** (creator of MediatR and AutoMapper).

> **Companion docs:**
> * [`n-layer-architecture.md`](./n-layer-architecture.md)
> * [`clean-architecture.md`](./clean-architecture.md)
> * [`architecture-comparison.md`](./architecture-comparison.md)

---

## Table of Contents

1. [What Is Vertical Slice Architecture?](#1-what-is-vertical-slice-architecture)
2. [Why It Exists — The Pain It Solves](#2-why-it-exists--the-pain-it-solves)
3. [Horizontal Layers vs. Vertical Slices](#3-horizontal-layers-vs-vertical-slices)
4. [Core Principles](#4-core-principles)
5. [Folder Structure](#5-folder-structure)
6. [End-to-End Sample in .NET](#6-end-to-end-sample-in-net)
7. [The Role of MediatR](#7-the-role-of-mediatr)
8. [Pipeline Behaviors — Cross-Cutting Without a "Cross-Cutting Layer"](#8-pipeline-behaviors--cross-cutting-without-a-cross-cutting-layer)
9. [Best Approach & Recommended Practices](#9-best-approach--recommended-practices)
10. [Advantages](#10-advantages)
11. [Disadvantages](#11-disadvantages)
12. [Common Pitfalls](#12-common-pitfalls)
13. [Testing Strategy](#13-testing-strategy)
14. [Vertical Slice vs. Clean / N-Layer](#14-vertical-slice-vs-clean--n-layer)
15. [Hybrid: Clean + Vertical Slice ("Clean Slice")](#15-hybrid-clean--vertical-slice-clean-slice)
16. [When to Use (and When Not To)](#16-when-to-use-and-when-not-to)
17. [Key Points — The Short List](#17-key-points--the-short-list)
18. [Checklist](#18-checklist)
19. [References](#19-references)

---

## 1. What Is Vertical Slice Architecture?

A **vertical slice** is a thin, end-to-end piece of functionality that cuts through every "layer" of the system — request, validation, business logic, persistence, response — and lives **together in one folder**.

> *"Minimize coupling between slices, and maximize coupling within a slice."* — Jimmy Bogard

Instead of asking *"where do controllers / services / repositories live?"*, you ask *"where does the `PlaceOrder` feature live?"* — answer: in one folder named `PlaceOrder/`.

The architecture is **feature-centric**, not technology-centric.

---

## 2. Why It Exists — The Pain It Solves

Traditional N-Layer architectures force any single feature to touch many files spread across many folders:

```
Adding "PlaceOrder" in N-Layer:
  Controllers/OrdersController.cs           ← edit
  Services/IOrderService.cs                 ← edit
  Services/OrderService.cs                  ← edit
  Repositories/IOrderRepository.cs          ← edit
  Repositories/OrderRepository.cs           ← edit
  Models/OrderDto.cs                        ← edit
  Validators/OrderValidator.cs              ← edit
  → 7 files, 4 folders, just to add ONE feature
```

Problems with this:
- **Cognitive load** — devs jump between distant folders to understand one feature.
- **Refactor risk** — touching shared services for one feature can break unrelated features.
- **God services / God controllers** — these accrue dozens of methods over time.
- **Abstractions over-applied** — every feature gets the same repository, even when 90% use a unique query once.

Vertical Slice flips the problem: bring everything for `PlaceOrder` into one folder, accept some duplication, and **reduce coupling between features**.

---

## 3. Horizontal Layers vs. Vertical Slices

```
HORIZONTAL (N-Layer):                       VERTICAL (Slices):

┌──────────────────────────────┐            ┌──────┬──────┬──────┬──────┐
│      Presentation            │            │Place │Cancel│ Get  │ List │
├──────────────────────────────┤            │Order │Order │Order │Order │
│      Application             │            │ ───  │ ───  │ ───  │ ───  │
├──────────────────────────────┤            │ Req  │ Req  │ Req  │ Req  │
│      Domain                  │            │ Val  │ Val  │ Val  │ Val  │
├──────────────────────────────┤            │ Hdl  │ Hdl  │ Hdl  │ Hdl  │
│      Data Access             │            │ Res  │ Res  │ Res  │ Res  │
└──────────────────────────────┘            └──────┴──────┴──────┴──────┘
   one feature = many folders                one feature = one folder
```

In a vertical-slice codebase, deleting a feature is a single-folder operation. Adding a feature touches almost no shared code.

---

## 4. Core Principles

1. **Group by feature, not by layer.** One folder per use case.
2. **Minimize coupling between slices.** Slices should not call into each other directly.
3. **Maximize cohesion within a slice.** Everything the feature needs lives together.
4. **Tolerate duplication.** Some duplicated mapping or query code is cheaper than the wrong abstraction.
5. **Use the simplest tool that solves the slice.** One slice can use Dapper, another EF Core, another raw SQL — whatever fits.
6. **Cross-cutting via pipeline behaviors**, not base classes or service wrappers.
7. **Refactor toward an abstraction *after* the third instance, not before.**

---

## 5. Folder Structure

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
│       │   │   ├── CancelOrderCommand.cs
│       │   │   ├── CancelOrderHandler.cs
│       │   │   └── CancelOrderEndpoint.cs
│       │   ├── GetOrder/
│       │   │   ├── GetOrderQuery.cs
│       │   │   ├── GetOrderHandler.cs
│       │   │   └── GetOrderEndpoint.cs
│       │   └── ListOrders/
│       │       ├── ListOrdersQuery.cs
│       │       ├── ListOrdersHandler.cs
│       │       └── ListOrdersEndpoint.cs
│       ├── Customers/
│       └── Products/
│
├── Shop.Domain/                ← (optional) shared entities/value objects
│   ├── Orders/
│   ├── Customers/
│   └── ValueObjects/
│
└── Shop.Infrastructure/         ← (optional) shared cross-cutting infra
    ├── Persistence/
    │   └── AppDbContext.cs
    └── Behaviors/               ← MediatR pipeline behaviors
        ├── ValidationBehavior.cs
        ├── LoggingBehavior.cs
        └── TransactionBehavior.cs
```

> Note: in a *strict* vertical slice setup, even `Shop.Domain` is optional — each slice can define its own model. In practice, most teams keep a thin shared `Domain` to host entities used by many slices (e.g., `Order`, `Customer`).

---

## 6. End-to-End Sample in .NET

Below is a single feature slice — **`PlaceOrder`** — implemented in one folder.

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

> Notice: no repository interface, no service layer, no separate mapping class. The handler is the slice — it does exactly what `PlaceOrder` needs and nothing more. If another slice needs richer behavior on `Order`, the entity grows; if another slice needs different querying, it has its own handler.

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

### 6.5 Composition in Program.cs

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

Adding a new feature = one new folder + one new `MapXxx()` call. That's it.

---

## 7. The Role of MediatR

MediatR provides an **in-process mediator** that dispatches a request to its handler. In Vertical Slice, it is the glue:

```
Endpoint ──Send(command)──► MediatR ──► PlaceOrderHandler
```

Why it fits so well:

- The **endpoint stays thin** — it doesn't reference the handler directly; it just sends a `Command`.
- **One handler per feature** = one slice. No cross-feature handler reuse.
- **Pipeline behaviors** wrap every handler with cross-cutting concerns (logging, validation, transactions, caching) without adding a layer.
- **CQRS for free** — commands and queries are just different `IRequest<T>` types.

> MediatR is not required. The same pattern works with raw handler resolution from the DI container. But MediatR + Vertical Slice is the canonical .NET combination.

---

## 8. Pipeline Behaviors — Cross-Cutting Without a "Cross-Cutting Layer"

A pipeline behavior is a MediatR middleware. It runs around every handler.

```csharp
// Infrastructure/Behaviors/ValidationBehavior.cs
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var ctx = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(ctx, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}
```

Common pipeline behaviors:

| Behavior                | Purpose                                                              |
| ----------------------- | -------------------------------------------------------------------- |
| `LoggingBehavior`       | Log each request/response and timing.                                |
| `ValidationBehavior`    | Run all FluentValidation validators before the handler.              |
| `TransactionBehavior`   | Wrap commands in a DB transaction.                                   |
| `CachingBehavior`       | Cache query results by request hash.                                 |
| `PerformanceBehavior`   | Alert on slow handlers (> threshold).                                |
| `AuthorizationBehavior` | Verify permissions per-request, declaratively via attributes.        |

Pipeline behaviors **replace** the role traditional architectures give to service-layer wrappers, AOP frameworks, or filters. One place to define the policy; it applies to every slice automatically.

---

## 9. Best Approach & Recommended Practices

1. **One folder per use case.** Name folders as verbs/intents (`PlaceOrder`, `CancelOrder`, `GetOrder`).
2. **Keep the slice self-contained.** Don't reach into another slice's folder.
3. **Co-locate command, validator, handler, endpoint, and response DTO.**
4. **Use minimal APIs** for endpoints — they're a natural fit for "one feature per file".
5. **Use MediatR + FluentValidation + pipeline behaviors** for cross-cutting concerns.
6. **Allow duplication** between slices until you see a real, stable pattern repeat 3+ times.
7. **Share domain entities and value objects**, but not "services" between slices.
8. **Use the right tool per slice.** A read slice can bypass entities and use Dapper/raw SQL for performance; a write slice can use EF Core for change tracking.
9. **Commands modify; queries read.** Don't mix.
10. **Endpoints are humble.** They translate HTTP ↔ command/query. No logic.
11. **Use `Result<T>`** for expected failures; reserve exceptions for truly exceptional cases.
12. **`CancellationToken` everywhere.**
13. **Test slices end-to-end** with `WebApplicationFactory<Program>` and Testcontainers — slices are small enough that this is fast.
14. **Refactor by extracting**, not by anticipating. When 3 slices need the same logic, extract; not before.
15. **Use Source Generators / Mapperly** for performant slice-local mapping.

---

## 10. Advantages

✅ **High cohesion** — everything for a feature lives in one place.
✅ **Low coupling between features** — change one slice without touching others.
✅ **Easy onboarding** — new devs read one folder to understand one feature.
✅ **Easy deletion** — delete a folder = delete a feature.
✅ **Per-slice technology freedom** — Dapper here, EF Core there, raw SQL over there.
✅ **Pairs perfectly with CQRS** — commands and queries are first-class.
✅ **Cross-cutting via pipeline** — no AOP, no inheritance hierarchies.
✅ **Plays well with feature flags / trunk-based development.**
✅ **Reduces "god services"** — there are no services to grow indefinitely.

---

## 11. Disadvantages

❌ **Duplication is real** — similar queries or mappings may appear in multiple slices.
❌ **No enforced consistency** — two devs might solve the same problem two different ways.
❌ **Architectural discipline shifts to code review** — there's no compiler to enforce slice boundaries.
❌ **Harder to enforce global rules** — e.g., "every command must publish an event."
❌ **Domain logic can scatter** if the team treats slices as scripts instead of feature modules.
❌ **Tooling assumes layers** — many templates and frameworks generate layered code by default.
❌ **Migration from N-Layer is non-trivial** — requires rethinking shared services.
❌ **Less obvious where to put truly cross-feature logic** (currency conversion, ID generation, etc.).

---

## 12. Common Pitfalls

| Pitfall                                                       | What goes wrong                                                              |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Building "shared services" early**                          | Defeats the slice; re-creates layered coupling.                              |
| **Letting slices call each other's handlers**                 | Tight cross-slice coupling. Use events or shared domain instead.             |
| **Putting all entities into one giant `Domain` project**      | Fine, but watch for becoming a god-model. Split by bounded context.          |
| **Fat handlers (200+ lines)**                                 | The handler is doing too much. Push logic into the domain or split the slice.|
| **Skipping the domain entirely**                              | Anemic procedural code. Slices still need rich entities for invariants.      |
| **Treating MediatR as the architecture**                      | MediatR is plumbing. The architecture is the **folder structure + rules**.    |
| **Ignoring the read/write split**                             | A slice that both reads and writes blurs CQRS benefits.                      |
| **Bypassing pipeline behaviors**                              | Validation / logging / transactions drift inconsistently across slices.       |

---

## 13. Testing Strategy

Vertical slices are testable end-to-end because each slice is small.

| Test Type           | Targets                                       | Tools                                   |
| ------------------- | --------------------------------------------- | --------------------------------------- |
| **Slice integration** | One slice through endpoint → DB              | `WebApplicationFactory` + Testcontainers |
| **Domain unit**     | Entities and value objects (no infra)         | xUnit + FluentAssertions                |
| **Pipeline behavior unit** | Validation/logging/transaction behaviors | xUnit                                    |
| **Contract tests**  | API shape per slice                           | OpenAPI snapshot tests / Pact           |

Because slices are independent, the test suite can run them in parallel without shared state surprises.

---

## 14. Vertical Slice vs. Clean / N-Layer

| Aspect              | **N-Layer**                       | **Clean Architecture**                    | **Vertical Slice**                         |
| ------------------- | --------------------------------- | ----------------------------------------- | ------------------------------------------ |
| Organization        | By technical layer                | By concentric layer + dependency rule     | By feature                                  |
| Dependency rule     | Top → down                        | Outer → inner                             | None across slices (slices are independent) |
| Domain location     | Middle layer                      | Center                                    | Shared (or per-slice)                       |
| Coupling between features | Often high                  | Medium                                    | **Low**                                     |
| Cohesion of one feature   | Low (split across layers)   | Medium                                    | **High** (one folder)                       |
| Boilerplate         | High                              | High                                      | **Low**                                     |
| Onboarding curve    | Easy                              | Steep                                     | Easy                                        |
| Best for            | CRUD apps                         | Complex domains                            | Modern web APIs with many independent use cases |

---

## 15. Hybrid: Clean + Vertical Slice ("Clean Slice")

Many modern .NET projects combine the **outer structure** of Clean Architecture with **feature folders** inside the Application layer:

```
src/
├── MyApp.Domain/                  ← Clean's innermost ring
├── MyApp.Application/
│   ├── Features/                  ← Vertical slices live here
│   │   ├── Orders/
│   │   │   ├── PlaceOrder/
│   │   │   └── CancelOrder/
│   │   └── Customers/
│   └── Abstractions/              ← Interfaces used by slices
├── MyApp.Infrastructure/          ← Clean's outermost ring (EF, etc.)
└── MyApp.Web/
    └── Endpoints/                 ← Thin endpoint glue, also per-feature
```

You get:
- **Dependency Rule** enforced by project references.
- **Feature cohesion** inside `Application/Features/`.
- **Pluggable infrastructure** via Clean's outer ring.

This is the **most common modern .NET layout** in 2025/2026 codebases.

---

## 16. When to Use (and When Not To)

### ✅ Use Vertical Slice when

- The app has **many independent use cases** with little shared logic.
- You're building a **modern web API** or backend service.
- The team practices **trunk-based development** with feature flags.
- You want **fast onboarding** and **easy deletion** of features.
- Read and write paths differ enough to benefit from CQRS.

### ⚠️ Reconsider when

- The system is **dominated by a single complex domain** with many invariants — Clean / DDD may serve better.
- The team is **very new** to .NET — N-Layer is more familiar.
- You need **strict, enforced layering** (e.g., compliance-heavy systems).
- The app is **tiny** — one controller class is enough.

---

## 17. Key Points — The Short List

* **Organize by feature, not by layer.**
* **One folder per use case** — command + validator + handler + endpoint + response.
* **Minimize coupling between slices; maximize cohesion within a slice.**
* **Tolerate duplication** until a stable pattern emerges (rule of three).
* **MediatR + FluentValidation + Pipeline behaviors** = the canonical .NET stack.
* **Per-slice tech freedom** — Dapper, EF Core, raw SQL — pick what fits the slice.
* **Easy to add, easy to delete features.**
* **Pairs naturally with CQRS and minimal APIs.**
* **Hybridize with Clean Architecture** for the best of both worlds.

---

## 18. Checklist

Before merging a new slice:

- [ ] Slice lives in its own folder named after the use case.
- [ ] Folder contains: command/query, validator, handler, endpoint, response DTO.
- [ ] Handler is < ~80 lines (else extract domain logic into entities).
- [ ] No direct call into another slice's folder.
- [ ] Validation goes through FluentValidation + ValidationBehavior.
- [ ] Endpoint is < 15 lines — pure HTTP-to-command translation.
- [ ] `CancellationToken` is passed through every async call.
- [ ] Commands are idempotent (or use idempotency keys).
- [ ] Tests cover happy path + at least one failure path.
- [ ] No new "service" abstraction added without justification (rule of three).

---

## 19. References

* [Jimmy Bogard — Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
* [Jimmy Bogard — Upcoming Training on Modern .NET with Vertical Slice Architecture](https://www.jimmybogard.com/upcoming-training-on-vertical-slice-architecture/)
* [Milan Jovanović — Vertical Slice Architecture](https://www.milanjovanovic.tech/blog/vertical-slice-architecture)
* [Ghyston — Architecting for maintainability through Vertical Slices](https://www.ghyston.com/insights/architecting-for-maintainability-through-vertical-slices)
* [Exploring vertical slices in dotnet core (DEV)](https://dev.to/htech/exploring-vertical-slices-in-dotnet-core-3mik)
* [Vertical Slice Architecture in .NET — From N-Tier Layers to Feature Slices (DEV)](https://dev.to/cristiansifuentes/vertical-slice-architecture-in-net-from-n-tier-layers-to-feature-slices-4iha)
* [GitHub — jbogard/ContosoUniversityDotNetCore-Pages](https://github.com/jbogard/ContosoUniversityDotNetCore-Pages)
* [N-Layered vs Clean vs Vertical Slice (antondevtips)](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
