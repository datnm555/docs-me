# Clean Architecture — Recommendations, Key Points & Best Approaches

> Based on *Clean Architecture: A Craftsman's Guide to Software Structure and Design* by **Robert C. Martin (Uncle Bob)**, with practical .NET application guidance.

---

## Table of Contents

1. [The Big Idea](#1-the-big-idea)
2. [Goals of a Clean Architecture](#2-goals-of-a-clean-architecture)
3. [SOLID — The Object-Oriented Foundation](#3-solid--the-object-oriented-foundation)
4. [Component Principles](#4-component-principles)
5. [The Four Concentric Layers](#5-the-four-concentric-layers)
6. [The Dependency Rule (The One Rule to Rule Them All)](#6-the-dependency-rule-the-one-rule-to-rule-them-all)
7. [Crossing Boundaries — The Input/Output Boundary Pattern](#7-crossing-boundaries--the-inputoutput-boundary-pattern)
8. [Screaming Architecture](#8-screaming-architecture)
9. [Humble Object Pattern](#9-humble-object-pattern)
10. [Boundaries — Partial, Local, and Full](#10-boundaries--partial-local-and-full)
11. [The Database & UI Are Details](#11-the-database--ui-are-details)
12. [Clean Architecture in .NET — Solution Layout](#12-clean-architecture-in-net--solution-layout)
13. [Code Walkthrough by Layer (.NET)](#13-code-walkthrough-by-layer-net)
14. [Patterns That Pair Well: CQRS, MediatR, Result, DDD](#14-patterns-that-pair-well-cqrs-mediatr-result-ddd)
15. [Best Approach — Recommended Practices](#15-best-approach--recommended-practices)
16. [Common Pitfalls](#16-common-pitfalls)
17. [Testing Strategy](#17-testing-strategy)
18. [Clean vs. N-Layer vs. Onion vs. Hexagonal](#18-clean-vs-n-layer-vs-onion-vs-hexagonal)
19. [When to Use (and When Not To)](#19-when-to-use-and-when-not-to)
20. [Checklist Before You Merge](#20-checklist-before-you-merge)
21. [References](#21-references)

---

## 1. The Big Idea

Clean Architecture is **not a framework** and **not a folder layout**. It is a set of *principles* that produces systems with these properties:

> *"The architecture of a software system is the shape given to that system by those who build it … to make the system easy to understand, develop, maintain, and deploy. The goal is to minimize the lifetime cost of the system and maximize programmer productivity."* — Uncle Bob

A clean system is:

- **Independent of frameworks** — the framework is a tool, not a constraint.
- **Testable** — business rules can be tested without UI, DB, or web server.
- **Independent of UI** — the UI can change without touching business rules.
- **Independent of database** — Oracle, SQL Server, Postgres, MongoDB, or flat files are interchangeable.
- **Independent of any external agency** — business rules don't know about the outside world.

The architecture must keep **decisions deferred and changeable for as long as possible**.

---

## 2. Goals of a Clean Architecture

Uncle Bob distinguishes **policy** (the rules — what the business does) from **detail** (the mechanism — how it's accomplished). A clean architecture protects high-value, slow-changing **policy** from volatile, low-value **details**.

| Concern         | Policy / Detail | Where it lives                       |
| --------------- | --------------- | ------------------------------------ |
| Business rule   | Policy          | Entities (innermost)                 |
| Use case        | Policy          | Use Cases                            |
| Web framework   | Detail          | Frameworks & Drivers (outermost)     |
| Database        | Detail          | Frameworks & Drivers (outermost)     |
| Serialization   | Detail          | Interface Adapters                   |

A change in a detail must never force a change in a policy.

---

## 3. SOLID — The Object-Oriented Foundation

SOLID applies to *classes*. Component principles (next section) apply to *modules / projects*. Together they define a clean architecture.

| Letter | Principle                       | Practical meaning                                                                                |
| ------ | ------------------------------- | ------------------------------------------------------------------------------------------------ |
| **S**  | Single Responsibility Principle | A module should have one and only one **reason to change** (i.e., one actor/stakeholder).        |
| **O**  | Open/Closed Principle           | Open for extension, closed for modification. New behavior should come from adding code.          |
| **L**  | Liskov Substitution Principle   | Subtypes must be substitutable for their base types without breaking expectations.               |
| **I**  | Interface Segregation Principle | Don't force clients to depend on methods they don't use. Prefer many small interfaces.           |
| **D**  | **Dependency Inversion**        | Depend on abstractions, not concretions. **The cornerstone of Clean Architecture.**              |

> Uncle Bob: *"The Dependency Inversion Principle tells us that the most flexible systems are those in which source code dependencies refer only to abstractions, not to concretions."*

---

## 4. Component Principles

Components (DLLs / assemblies / packages) have their own SOLID-style rules.

### Component Cohesion (what goes together?)

| Principle                                         | Says                                                                       |
| ------------------------------------------------- | -------------------------------------------------------------------------- |
| **REP** — Reuse / Release Equivalence             | The unit of reuse is the unit of release. Version components together.     |
| **CCP** — Common Closure Principle                | Group together classes that change for the same reason at the same time.   |
| **CRP** — Common Reuse Principle                  | Don't force users to depend on things they don't use.                      |

### Component Coupling (how components relate?)

| Principle                                         | Says                                                                       |
| ------------------------------------------------- | -------------------------------------------------------------------------- |
| **ADP** — Acyclic Dependencies Principle          | The component graph must be a DAG. **No cycles.**                          |
| **SDP** — Stable Dependencies Principle           | Depend in the direction of stability (stable components have many dependents). |
| **SAP** — Stable Abstractions Principle           | Stable components should also be abstract (so they're extensible).         |

The **Main Sequence**: a component should be either *stable + abstract* (like Domain) or *unstable + concrete* (like a UI). Components in the "zone of pain" (stable + concrete) and "zone of uselessness" (unstable + abstract) are red flags.

---

## 5. The Four Concentric Layers

The famous diagram (paraphrased from Uncle Bob's blog):

```
                ┌───────────────────────────────────────────┐
                │  Frameworks & Drivers                     │
                │  (Web, DB, UI, Devices, External APIs)    │
                │  ┌─────────────────────────────────────┐  │
                │  │  Interface Adapters                 │  │
                │  │  (Controllers, Gateways, Presenters)│  │
                │  │  ┌───────────────────────────────┐  │  │
                │  │  │  Application Business Rules  │  │  │
                │  │  │  (Use Cases / Interactors)    │  │  │
                │  │  │  ┌─────────────────────────┐  │  │  │
                │  │  │  │  Enterprise Rules       │  │  │  │
                │  │  │  │  (Entities)             │  │  │  │
                │  │  │  └─────────────────────────┘  │  │  │
                │  │  └───────────────────────────────┘  │  │
                │  └─────────────────────────────────────┘  │
                └───────────────────────────────────────────┘

           Dependencies point INWARD only ─────►
```

### 5.1 Entities (Enterprise-Wide Business Rules)

* The **highest-level policy**.
* Pure objects with methods that encapsulate critical business rules that would apply across the entire enterprise — even if there were no application around them.
* Know nothing about databases, HTTP, JSON, frameworks, or other layers.
* Stable. Rarely change.

### 5.2 Use Cases (Application-Specific Business Rules)

* Orchestrate **entities** to accomplish application goals.
* Each use case = one user intent (`PlaceOrder`, `RegisterCustomer`, `RefundPayment`).
* Independent of UI and DB.
* Define **input boundaries** (request models) and **output boundaries** (response models / presenters).

### 5.3 Interface Adapters

* Convert data between the format convenient for use cases/entities and the format convenient for external agencies.
* Examples: MVC controllers, presenters, view models, repositories (the implementing side), gateways, serializers.
* This is where most of the "translation" code lives.

### 5.4 Frameworks & Drivers (Details)

* The outermost ring. ASP.NET Core, EF Core, RabbitMQ, MongoDB, Stripe SDK, browsers.
* Mostly *glue code*. You write as little here as possible.
* Replaceable: switching from SQL Server to PostgreSQL or from REST to gRPC should not touch use cases.

---

## 6. The Dependency Rule (The One Rule to Rule Them All)

> **Source code dependencies must point only inward.**

* An inner circle must know **nothing** about an outer circle.
* Names declared in outer circles (controllers, EF entities, JSON attributes, framework types) must **not appear** in inner circles.
* Data crossing boundaries should be **simple data structures** (DTOs / records), never framework types like `HttpRequest`, `DbContext`, or `IQueryable`.

When the natural flow of control needs to go *outward* (e.g., a use case must call a database), use **Dependency Inversion**: the use case depends on an interface, and the outer layer implements it.

```
Use Case  ─uses→  IUserRepository  ←implements─  EfUserRepository
(inner)            (inner)                        (outer)
```

---

## 7. Crossing Boundaries — The Input/Output Boundary Pattern

For every use case, define:

* An **Input Boundary** (interface) the controller calls.
* A **Request Model** (DTO) for inputs.
* An **Output Boundary** (interface) the use case calls.
* A **Response Model** (DTO) for outputs.
* A **Presenter** that consumes the response model and produces a view model.

This is the **Boundary–Interactor–Presenter** pattern (a.k.a. *Clean Use Case*).

```
Controller ─► IInputBoundary (UseCase) ─► Entities ─► IOutputBoundary ─► Presenter ─► View
```

In .NET, MediatR's `IRequest<T>` + `IRequestHandler<TReq, TRes>` plays this role with less ceremony.

---

## 8. Screaming Architecture

> *"Your architecture should scream the intent of the system."* — Uncle Bob

When someone opens your solution, the folder/project names should describe **what the system does**, not **what framework it uses**.

❌ Bad (frameworks scream):
```
src/
  Controllers/
  Services/
  Repositories/
  Models/
```

✅ Good (domain screams):
```
src/
  Ordering/
  Billing/
  Shipping/
  Inventory/
```

The framework should be an implementation detail you discover only when you dig into a feature.

---

## 9. Humble Object Pattern

Hard-to-test code (UI, DB drivers, message queues) is split into:

* A **humble** component that does only what's hard to test (mechanical, no behavior).
* A **testable** component that contains the logic.

Examples in .NET:
* Controller (humble) ↔ Use case handler (testable).
* `DbContext` (humble) ↔ Repository + Domain (testable).
* SignalR hub (humble) ↔ Notification service (testable).

---

## 10. Boundaries — Partial, Local, and Full

You don't have to apply boundaries everywhere from day one.

* **Full boundary**: separate process / service. Highest cost, highest decoupling.
* **Local boundary**: separate assembly, communicate through interfaces. Most common in .NET.
* **Partial boundary**: single project, but boundary interfaces and DTOs exist. Cheap; easy to harden later.

Defer the cost. Start with a partial boundary; promote when pain appears.

---

## 11. The Database & UI Are Details

A clean architecture treats the **database**, the **web framework**, and the **UI** as plug-in *details*. The business should not care whether data is stored in SQL Server, Cosmos DB, or a file. Use **gateway interfaces** in the use case layer; concrete implementations live in Infrastructure.

This is why Clean Architecture pairs naturally with **CQRS** (separate command/query models) and **event sourcing** (the database becomes a stream of facts, not the source of truth).

---

## 12. Clean Architecture in .NET — Solution Layout

```
MyApp.sln
│
├── src/
│   ├── MyApp.Domain/                  ← Entities + Domain Services (innermost)
│   │   ├── Customers/
│   │   ├── Orders/
│   │   ├── ValueObjects/
│   │   ├── Events/
│   │   └── Abstractions/              ← IClock, IIdGenerator (pure)
│   │
│   ├── MyApp.Application/             ← Use Cases (depends on Domain only)
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
│   ├── MyApp.Infrastructure/          ← EF Core, SMTP, Storage, External APIs
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/
│   │   │   ├── Migrations/
│   │   │   └── Repositories/
│   │   ├── Email/
│   │   ├── Storage/
│   │   └── DependencyInjection.cs
│   │
│   ├── MyApp.Web/                     ← ASP.NET Core API / Blazor (outermost)
│   │   ├── Endpoints/  (or Controllers/)
│   │   ├── Middleware/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   └── MyApp.SharedKernel/            ← (Optional) Result<T>, GuardClauses, base types
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

> **Key invariant:** `Domain` references nothing except `SharedKernel`. No EF Core. No HTTP. No ASP.NET. If you cannot port `Domain` to a console app or another framework as-is, the boundary is broken.

---

## 13. Code Walkthrough by Layer (.NET)

### 13.1 Domain — Entity with Behavior

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

### 13.2 Application — Abstractions + Use Case

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

### 13.3 Infrastructure — EF Implementation

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

### 13.4 Web — Thin Endpoint

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

> The controller / endpoint is **humble**. Its only job is translation between HTTP and the use case.

---

## 14. Patterns That Pair Well: CQRS, MediatR, Result, DDD

| Pattern              | Role in a Clean .NET app                                                          |
| -------------------- | --------------------------------------------------------------------------------- |
| **CQRS**             | Separate commands from queries. Commands go through handlers; queries can bypass entities and read directly with optimized projections. |
| **MediatR**          | Implements input boundaries (`IRequest`/`IRequestHandler`). Pipeline behaviors give you free logging, validation, transactions. |
| **Result\<T\>**      | Avoid exceptions for expected business failures. Carries a value OR an error.    |
| **FluentValidation** | Use-case input validation lives in Application — independent of ASP.NET.         |
| **DDD tactical**     | Entities, value objects, aggregates, domain events. Fits Domain layer naturally. |
| **Specification**    | Encapsulate query criteria into testable objects.                                 |
| **Domain Events + Outbox** | Decouple side effects (email, integration events) from the use case transaction. |

---

## 15. Best Approach — Recommended Practices

1. **Start with the Domain.** Model entities and value objects *before* DB schema or API contracts.
2. **One use case per file.** A folder per use case with command, handler, validator, response.
3. **Use the `Application/Abstractions` folder** for all interfaces consumed by use cases.
4. **Make `Domain` framework-free.** No NuGet references except for very pure packages (e.g., `Ardalis.GuardClauses`).
5. **Use `internal` aggressively** in Infrastructure. Only expose the `AddInfrastructure(...)` registration method publicly.
6. **DTOs at every boundary.** Never let EF entities reach controllers.
7. **MediatR pipeline behaviors** for logging, validation, transaction, caching, performance.
8. **`Result<T>` for expected failures**, exceptions only for truly exceptional or programmer-error situations.
9. **Cancel everywhere.** Pass `CancellationToken` through every async chain.
10. **Configuration via `IOptions<T>` + `ValidateOnStart()`.** Fail fast on misconfiguration.
11. **Migrations & seeding** live in Infrastructure. The web project triggers them only on startup in dev / via a one-shot job in prod.
12. **Architecture tests** (NetArchTest / ArchUnitNET) in CI to enforce the Dependency Rule mechanically.
13. **Feature folders** inside Application — group by business capability, not by technical type.
14. **Screaming top-level** — folder names should reveal business intent (`Ordering`, `Billing`), not framework (`Controllers`, `Services`).
15. **Keep handlers ~50 lines or less.** Bigger ones hint a missing domain concept.
16. **Avoid the Generic Repository pattern.** Prefer use-case-specific repository methods that return aggregates.
17. **Don't return `IQueryable` from repositories.** Encapsulation breaks the moment a caller adds a `.Where()`.
18. **Use Source Generators** (System.Text.Json, Mapperly) over reflection for AOT readiness in .NET 8/9.
19. **Apply the Strangler Fig** when migrating legacy code — wrap legacy in adapters; let Clean parts grow.
20. **Defer decisions.** ORMs, message brokers, cloud vendor — make those choices late; design as if any could swap.

---

## 16. Common Pitfalls

| Pitfall                                                | Why it hurts                                                                                  |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| **Anemic domain** — entities are just properties        | Use cases bloat with logic that belongs in entities.                                          |
| **Domain referencing EF Core**                          | Breaks the Dependency Rule; you can't reuse the domain without EF.                            |
| **One mega-`Application` project for everything**       | Use feature folders or per-bounded-context projects.                                          |
| **Generic `IRepository<T>` everywhere**                 | Often degenerates into a thin wrapper around EF and discourages aggregate design.             |
| **Mapping DTOs ↔ entities with AutoMapper everywhere**  | Hidden behavior, runtime errors. Prefer explicit constructors / `static FromEntity()` methods. |
| **Bypassing use cases for "quick" reads**               | If you must, route through query handlers — keep one obvious path.                            |
| **Two-way bindings between layers**                     | A cycle. Refactor by introducing an interface in the inner layer.                              |
| **Misusing exceptions for control flow**                | Slower, harder to test, leaks across boundaries. Use `Result<T>` for expected failures.       |
| **Over-engineering** for a small app                    | Clean Architecture has overhead; a CRUD admin tool may not need 4 projects.                   |
| **Treating MediatR as magic**                           | It's just an in-process dispatcher. Don't hide critical flow in handlers you can't follow.    |

---

## 17. Testing Strategy

| Test Layer            | Targets                                                  | Tools                                                |
| --------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| **Domain unit**       | Entities, value objects, domain services                 | xUnit + FluentAssertions, **no mocks**               |
| **Application unit**  | Handlers, validators                                     | xUnit + NSubstitute / Moq for `Application/Abstractions` |
| **Infrastructure integration** | Real DB via Testcontainers                       | xUnit + Testcontainers                               |
| **Functional / E2E**  | Full API through `WebApplicationFactory<Program>`        | xUnit + Testcontainers                               |
| **Architecture**      | Layer rules                                              | NetArchTest.Rules / ArchUnitNET                      |

Architecture test example:

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

| Style          | Diagram          | Core idea                                            | Strengths                              | Weaknesses                                |
| -------------- | ---------------- | ---------------------------------------------------- | -------------------------------------- | ----------------------------------------- |
| **N-Layer**    | Stack            | Layers depend top-down                               | Familiar, fast to start                | Rigid, easy to "skip" layers              |
| **Onion**      | Concentric rings | Domain at center; dependencies inward                | Decouples domain from infra            | Many small projects; ceremony             |
| **Hexagonal**  | Hex with ports   | Ports & adapters around a core                       | Pluggable adapters; symmetric I/O      | Learning curve                            |
| **Clean**      | Concentric rings | Combines Onion + Hexagonal + Use-Case-driven design  | Most prescriptive; great for DDD-style apps | Heavier; can over-engineer simple apps |
| **Vertical Slice** | Feature folders | Per-feature mini-layering                          | Low coupling between features          | Some duplication; weaker shared rules     |

> **In practice:** Clean Architecture's outer layers (Web, Infrastructure) plus Vertical-Slice-style feature folders *inside* Application is a popular hybrid in modern .NET (sometimes called "Clean Slice").

---

## 19. When to Use (and When Not To)

### ✅ Good fit

* Long-lived systems with complex business rules (banking, ERP, healthcare, logistics).
* Multiple delivery channels (web + mobile + worker + CLI).
* Frequent changes to infrastructure (DB swap, cloud move).
* Teams that practice DDD and TDD.

### ⚠️ Reconsider when

* Pure CRUD with simple rules → plain N-Layer is enough.
* Tiny microservice doing one thing → 1–2 projects.
* Short-lived prototype → speed beats structure.
* Read-heavy reporting → favor CQRS read models or direct SQL.

> Rule of thumb: pay the structural cost when *the cost of change* in your domain is high. Otherwise, the discipline is overhead.

---

## 20. Checklist Before You Merge

- [ ] `Domain` project has **zero** infrastructure or web references.
- [ ] Every interface a use case depends on lives in `Application/Abstractions`.
- [ ] No EF entities or `IQueryable` exposed above Infrastructure.
- [ ] Each use case is in its own folder with command/handler/validator/response.
- [ ] Endpoints/controllers are < 15 lines.
- [ ] All async paths flow `CancellationToken`.
- [ ] `DbContext` lifetime is `Scoped`; no captured into singletons.
- [ ] Architecture tests pass in CI.
- [ ] Unit tests cover entities and handlers; integration tests cover real DB paths.
- [ ] Folder structure "screams" the business, not the framework.
- [ ] Configuration validated at startup; secrets via secret stores or env vars.
- [ ] Logging + global exception handler + health checks in place.

---

## 21. References

### Primary Source
* **Martin, Robert C.** *Clean Architecture: A Craftsman's Guide to Software Structure and Design.* Prentice Hall, 2017.
  * [Amazon](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)
  * [O'Reilly](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/)

### Uncle Bob's Articles
* [The Clean Architecture (blog post)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
* [Screaming Architecture (blog post)](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)

### .NET Practical Guides
* [Clean Architecture in .NET — Code Maze](https://code-maze.com/dotnet-clean-architecture/)
* [Microsoft eShopOnWeb Reference App](https://github.com/dotnet-architecture/eShopOnWeb)
* [Jason Taylor — Clean Architecture Solution Template](https://github.com/jasontaylordev/CleanArchitecture)
* [Amichai Mantinband — Clean Architecture Template](https://github.com/amantinband/clean-architecture)
* [Milan Jovanović — Building Your First Use Case With Clean Architecture](https://www.milanjovanovic.tech/blog/building-your-first-use-case-with-clean-architecture)
* [Microsoft ISE Devblog — Next-Level Clean Architecture Boilerplate](https://devblogs.microsoft.com/ise/next-level-clean-architecture-boilerplate/)

### Summaries & Reviews
* [Reflectoring — Book Review](https://reflectoring.io/book-review-clean-architecture/)
* [Goodreads — Clean Architecture](https://www.goodreads.com/book/show/18043011-clean-architecture)
* [Spaceteams — A Deep Dive](https://www.spaceteams.de/en/insights/clean-architecture-a-deep-dive-into-structured-software-design)
