# Other Application Patterns

> The everyday patterns that live above the data layer: **Specification**, **CQRS**, **Mediator**, **Result**, **Options**, **Decorator pipeline**, **Factory**, **Strategy at the seam**. Each one solves a specific application-level pain — don't reach for any of them without that pain.

---

## Table of Contents

1. [Specification](#1-specification)
2. [CQRS — Command/Query Responsibility Segregation](#2-cqrs--commandquery-responsibility-segregation)
3. [Mediator (in-process bus)](#3-mediator-in-process-bus)
4. [Result / Either Pattern](#4-result--either-pattern)
5. [Options Pattern](#5-options-pattern)
6. [Decorator Pipeline (Cross-Cutting Concerns)](#6-decorator-pipeline-cross-cutting-concerns)
7. [Factory](#7-factory)
8. [Strategy at the Seam](#8-strategy-at-the-seam)
9. [How to Choose](#9-how-to-choose)

---

## 1. Specification

> Encapsulate a business rule as an object you can **combine**, **reuse**, and **test in isolation**.

Useful when a predicate is used in multiple places — a query, a domain assertion, a validation step.

### 1.1 The Pattern

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T candidate);
    Expression<Func<T, bool>> ToExpression();
}

public abstract class Specification<T> : ISpecification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    public bool IsSatisfiedBy(T candidate) => ToExpression().Compile()(candidate);

    public Specification<T> And(Specification<T> other) => new AndSpec<T>(this, other);
    public Specification<T> Or(Specification<T> other)  => new OrSpec<T>(this, other);
    public Specification<T> Not()                       => new NotSpec<T>(this);
}
```

### 1.2 Concrete Specs

```csharp
public class IsActiveCustomer : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression()
        => c => c.Status == CustomerStatus.Active;
}

public class HasMinOrders(int n) : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression()
        => c => c.Orders.Count >= n;
}

// Compose:
var loyal = new IsActiveCustomer().And(new HasMinOrders(5));

// Use as a query:
var query = db.Customers.Where(loyal.ToExpression());

// Or as an in-memory check:
if (loyal.IsSatisfiedBy(currentCustomer)) ...
```

### 1.3 When to Use

* The same predicate appears in **multiple** places.
* You want to **name** business rules in the language of the domain.
* You need to compose rules dynamically (filters from a UI, for example).

### 1.4 When *Not* to Use

* The predicate is one-line, used once. Just write a lambda.
* Compile-time `Expression<>` composition becomes a pain for non-trivial rules. Consider [LinqKit](https://github.com/scottksmith95/LINQKit) or domain-specific composition helpers.

---

## 2. CQRS — Command/Query Responsibility Segregation

> Use a **different model** to write (commands) than to read (queries).

### 2.1 The Idea

A single domain model trying to serve both reads and writes ends up bad at both — reads want denormalized, fast, shape-specific data; writes want consistent, validated aggregates.

CQRS separates them:

```
┌──────────────┐    Command    ┌──────────────────┐
│   Client     │ ────────────► │ Command Handler  │ ──► Domain ──► DB (write model)
└──────────────┘                └──────────────────┘
       │
       │ Query
       ▼
┌──────────────────┐
│ Query Handler    │ ──► DB / cache / read model
└──────────────────┘
```

### 2.2 Minimal Sketch (with or without MediatR)

```csharp
public record PlaceOrderCommand(Guid CustomerId, IReadOnlyList<LineItemDto> Items);

public class PlaceOrderHandler(IOrderRepository repo, IUnitOfWork uow)
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Start(cmd.CustomerId);
        // ... build order, apply rules ...
        await repo.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);
        return order.Id;
    }
}

public record GetOrderSummaryQuery(Guid OrderId);
public record OrderSummary(Guid Id, decimal Total, string Status, int ItemCount);

public class GetOrderSummaryHandler(ShopDbContext db)
{
    public Task<OrderSummary?> Handle(GetOrderSummaryQuery q, CancellationToken ct)
        => db.Orders
             .Where(o => o.Id == q.OrderId)
             .Select(o => new OrderSummary(o.Id, o.Total, o.Status.ToString(), o.Items.Count))
             .FirstOrDefaultAsync(ct);
}
```

Note: the **query** doesn't go through the domain model. It hits the database directly and projects into a read shape.

### 2.3 Variants — From Light to Heavy

| Flavor                     | Storage                                | Complexity |
| -------------------------- | -------------------------------------- | ---------- |
| **Light CQRS**             | Same DB; commands use domain, queries project DTOs | ★ |
| **CQRS + Read Models**     | Materialized views / cache for reads   | ★★         |
| **CQRS + Event Sourcing**  | Append-only event store; projections   | ★★★★       |

### 2.4 When to Use

* Read and write models have **genuinely different shapes**.
* Read scale **dwarfs** write scale.
* You want to introduce **caching, projections, search indexes** for reads.

### 2.5 When *Not* to Use

* Simple CRUD app.
* Read and write shapes are identical and small.
* Team is unfamiliar — the cost of two paths exceeds the benefit.

> **Anti-pattern:** *Anemic CQRS* — two paths but identical models. You doubled your code for no win.

---

## 3. Mediator (in-process bus)

> Decouple senders from handlers via an in-process **message bus**.

In .NET, the most popular implementation is **MediatR**.

### 3.1 The Idea

```csharp
public record PlaceOrderCommand(Guid CustomerId, ...) : IRequest<Guid>;

public class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    public Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct) { ... }
}

// Caller:
public class CheckoutController(IMediator mediator)
{
    [HttpPost]
    public async Task<Guid> Place(PlaceOrderCommand cmd) => await mediator.Send(cmd);
}
```

The controller doesn't know `PlaceOrderHandler` exists. MediatR routes the request to the right handler.

### 3.2 Pipeline Behaviors

MediatR can wrap every request with cross-cutting concerns — logging, validation, transactions, caching:

```csharp
public class LoggingBehavior<TRequest, TResponse>(ILogger<TRequest> log) : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        log.LogInformation("Handling {Request}", typeof(TRequest).Name);
        var response = await next();
        log.LogInformation("Handled {Request}", typeof(TRequest).Name);
        return response;
    }
}
```

This is **Decorator** + **Chain of Responsibility** at the framework level.

### 3.3 When to Use

* You're doing **CQRS** and want a clean dispatch mechanism.
* You want **uniform cross-cutting** (validation, logging, transactions) per request.
* You want **plug-in handlers** discoverable by convention.

### 3.4 When *Not* to Use

* Simple controllers calling simple services. MediatR adds indirection for no payoff.
* Tiny CRUD app. Just inject the service.

> **Anti-pattern:** *Mediator-everywhere*. If 100% of requests go through MediatR even when there's only one obvious handler, you've added a layer that buys nothing.

---

## 4. Result / Either Pattern

> Return success-or-failure as a **value**, not via exceptions.

### 4.1 The Idea

Exceptions are for **exceptional** cases (truly broken invariants, infra failure). For expected failures — validation errors, business rule violations — return a result.

```csharp
public abstract record Result<TOk, TErr>
{
    public sealed record Ok(TOk Value)   : Result<TOk, TErr>;
    public sealed record Err(TErr Error) : Result<TOk, TErr>;
}

public record OrderError(string Code, string Message);

public class PlaceOrderHandler
{
    public Result<Guid, OrderError> Handle(PlaceOrderCommand cmd)
    {
        if (cmd.Items.Count == 0)
            return new Result<Guid, OrderError>.Err(new("EMPTY_CART", "Cart is empty"));

        // ... happy path ...
        return new Result<Guid, OrderError>.Ok(orderId);
    }
}

// Caller:
return handler.Handle(cmd) switch
{
    Result<Guid, OrderError>.Ok ok    => Ok(ok.Value),
    Result<Guid, OrderError>.Err err  => BadRequest(err.Error),
    _ => throw new UnreachableException()
};
```

### 4.2 Benefits

* **Explicit** — the signature shows that failure is possible.
* **Cheaper than exceptions** — no stack walks.
* **Composable** — chain `.Map / .Bind` for success-only pipelines.
* **Testable** — assert on result values, not on exception types.

### 4.3 Caveat

Don't replace **every** exception. Reserve Result for **expected, recoverable** failures. A null database connection is still an exception.

### 4.4 In Practice

Libraries: `OneOf`, `LanguageExt`, `FluentResults`. Or write your own — it's 30 lines.

---

## 5. Options Pattern

> Strongly-typed, validated, bound configuration.

### 5.1 Bad — Stringly-Typed Config

```csharp
var key = configuration["Stripe:ApiKey"];   // typo silently returns null
```

### 5.2 Good — Options

```csharp
public class StripeOptions
{
    [Required, MinLength(8)] public required string ApiKey { get; init; }
    [Range(1, 60)]            public int RetryAttempts { get; init; } = 3;
}

builder.Services
    .AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

public class StripeGateway(IOptions<StripeOptions> opts)
{
    private readonly StripeOptions _o = opts.Value;
}
```

### 5.3 Three Flavors

| Interface             | When config changes                        |
| --------------------- | ------------------------------------------ |
| `IOptions<T>`         | At startup only (singleton)                |
| `IOptionsSnapshot<T>` | Per request (scoped)                       |
| `IOptionsMonitor<T>`  | At any time; subscribe to changes (singleton with callbacks) |

### 5.4 Benefits

* Fails **at startup** if config is bad — not at 3 a.m. when the feature first runs.
* IntelliSense, type safety, refactor-friendly.
* Cleanly testable by passing `Options.Create(new StripeOptions { ... })`.

---

## 6. Decorator Pipeline (Cross-Cutting Concerns)

> Wrap a service with layers of behavior (logging, retry, caching) without modifying it.

### 6.1 The Pattern

```csharp
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument instrument, CancellationToken ct);
}

public class StripePaymentGateway : IPaymentGateway { ... }

public class LoggingPaymentGateway(IPaymentGateway inner, ILogger<LoggingPaymentGateway> log) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument inst, CancellationToken ct)
    {
        log.LogInformation("Charging {Amount}", amount);
        var result = await inner.ChargeAsync(amount, inst, ct);
        log.LogInformation("Charge result: {Result}", result);
        return result;
    }
}

public class RetryingPaymentGateway(IPaymentGateway inner) : IPaymentGateway
{
    public async Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument inst, CancellationToken ct)
    {
        for (var attempt = 0; attempt < 3; attempt++)
        {
            var r = await inner.ChargeAsync(amount, inst, ct);
            if (r is PaymentResult.Success) return r;
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
        }
        return new PaymentResult.Declined("Out of retries");
    }
}
```

### 6.2 Wiring

```csharp
services.AddScoped<StripePaymentGateway>();
services.AddScoped<IPaymentGateway>(sp =>
    new LoggingPaymentGateway(
        new RetryingPaymentGateway(
            sp.GetRequiredService<StripePaymentGateway>()),
        sp.GetRequiredService<ILogger<LoggingPaymentGateway>>()));
```

Or use a library like **Scrutor** that supports `services.Decorate<IPaymentGateway, LoggingPaymentGateway>();`.

### 6.3 Why Decorators Beat Mixed-In Code

* The core class **stays focused** on its real responsibility.
* Each concern is **independently testable**.
* You can **toggle** concerns via configuration (e.g., disable caching in tests).

> The **MediatR pipeline behaviors** above are decorators applied uniformly to every request. The HTTP middleware pipeline is the same idea at the request boundary.

---

## 7. Factory

> Centralize **complex construction** logic when `new T(...)` isn't enough.

### 7.1 When You Need a Factory

* Construction requires **runtime decisions** ("which payment gateway? Stripe or PayPal?").
* Construction is **expensive** and you want caching.
* The constructor takes **dependencies the caller shouldn't know about**.

### 7.2 Typed Factory

```csharp
public interface IPaymentGatewayFactory
{
    IPaymentGateway Create(PaymentProvider provider);
}

public class PaymentGatewayFactory(
    IServiceProvider sp) : IPaymentGatewayFactory
{
    public IPaymentGateway Create(PaymentProvider provider) => provider switch
    {
        PaymentProvider.Stripe  => sp.GetRequiredService<StripePaymentGateway>(),
        PaymentProvider.PayPal  => sp.GetRequiredService<PayPalPaymentGateway>(),
        PaymentProvider.ApplePay=> sp.GetRequiredService<ApplePayPaymentGateway>(),
        _ => throw new NotSupportedException()
    };
}
```

Typed factories are **acceptable** Service Locator usage — they have a small, intentional surface and live near the composition root.

### 7.3 EF Core's `IDbContextFactory<T>`

For long-running operations (background workers, batch jobs) the per-request `DbContext` lifetime is wrong. `IDbContextFactory<T>` creates a fresh context on demand.

---

## 8. Strategy at the Seam

> The simplest, most-used "pattern" of all: **inject an interface**.

It's the Gang-of-Four **Strategy** pattern, but in a modern DI world you rarely *call* it that. You just inject an `IDiscountRule`, an `ITaxPolicy`, an `IEmailSender`.

Every place you reach for `if/else`-by-type, or where you anticipate adding variants, lean on this:

```csharp
public class CheckoutUseCase(IDiscountRule discount, ITaxPolicy tax, IPaymentGateway payment)
{
    // The "strategies" are wired by DI; this class never knows which one it has.
}
```

This single move enables **OCP** (Open/Closed), **DIP** (Dependency Inversion), and **Protected Variations** (GRASP) — all at once.

---

## 9. How to Choose

Before adopting any pattern in this folder, ask:

| Question                                        | If yes, consider…              |
| ----------------------------------------------- | ------------------------------ |
| Will this rule be **reused** across queries and validations? | Specification     |
| Are read and write models **genuinely different**?          | CQRS              |
| Do you want **uniform pipelines** across handlers?          | Mediator + behaviors |
| Are validation / business failures **frequent and expected**?| Result/Either    |
| Do you need **typed, validated configuration**?              | Options          |
| Do you need **cross-cutting concerns** without touching core code? | Decorator   |
| Is construction **complex or runtime-conditional**?          | Factory          |
| Will behavior need to **vary or grow** in the future?        | Strategy via DI  |

**If no clear answer:** skip the pattern. You can introduce it later when the pain is real.

---

## Cross-Cutting Reading

* [`ioc-di.md`](./ioc-di.md) — every pattern here depends on DI.
* [`data-access.md`](./data-access.md) — Repository / Unit of Work / Lazy / Eager.
* [`../../../principles/solid.md`](../../../principles/solid.md) — the principles every pattern serves.
* [`../../../oop/real-project.md`](../../../oop/real-project.md) — many of these patterns shown in a real walkthrough.
* [`../design-pattern/`](../design-pattern/) — GoF patterns for the class-level toolbox.

---

> **Bottom line:** these patterns are **answers to specific pains**. Apply them when you feel the pain — not because they appeared on a slide deck.
