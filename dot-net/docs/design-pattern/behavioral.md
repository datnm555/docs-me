# Behavioral Design Patterns in .NET

> The 11 GoF patterns concerned with **communication and responsibility** between objects — how they interact, who handles what, and how algorithms vary. Each pattern below is shown with a **real-world scenario** you would actually meet in production .NET code.

---

## Table of Contents

1. [Chain of Responsibility](#1-chain-of-responsibility)
2. [Command](#2-command)
3. [Interpreter](#3-interpreter)
4. [Iterator](#4-iterator)
5. [Mediator](#5-mediator)
6. [Memento](#6-memento)
7. [Observer](#7-observer)
8. [State](#8-state)
9. [Strategy](#9-strategy)
10. [Template Method](#10-template-method)
11. [Visitor](#11-visitor)

---

## 1. Chain of Responsibility

**Intent:** Pass a request along a chain of handlers; each handler decides to process it or pass it on.

**Real-world need:** An HTTP request needs to flow through logging, authentication, rate-limiting, caching, and finally hit the controller. Each step can short-circuit.

### Real project sample — ASP.NET Core middleware (the canonical CoR in .NET)

```csharp
// Each middleware is a handler in the chain
public sealed class RateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimiter _limiter;

    public RateLimitMiddleware(RequestDelegate next, IRateLimiter limiter)
    { _next = next; _limiter = limiter; }

    public async Task InvokeAsync(HttpContext ctx)
    {
        if (!await _limiter.AllowAsync(ctx.User.Identity?.Name ?? ctx.Connection.RemoteIpAddress!.ToString()))
        {
            ctx.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            return; // short-circuits the chain
        }
        await _next(ctx); // pass to next handler
    }
}

// Program.cs — assembles the chain
app.UseSerilogRequestLogging();
app.UseAuthentication();
app.UseMiddleware<RateLimitMiddleware>();
app.UseAuthorization();
app.UseResponseCaching();
app.MapControllers();
```

**Other real-world uses:** `HttpClient` `DelegatingHandler` pipeline, validation pipelines (FluentValidation chains), event filters in CQRS, exception-handling fallbacks.

---

## 2. Command

**Intent:** Encapsulate a request as an object — allowing queueing, logging, undo, and parameterization.

**Real-world need:** You want every state-changing operation to be auditable, retryable, and triggered from queues. Just calling service methods is too coupled.

### Real project sample — CQRS commands with MediatR

```csharp
// The command — a record carrying the request data
public record CreateOrderCommand(Guid CustomerId, IReadOnlyList<OrderItemDto> Items, string Currency)
    : IRequest<Guid>;

// The handler — single responsibility, easy to test
public sealed class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IPaymentGateway _payments;

    public CreateOrderHandler(IOrderRepository orders, IPaymentGateway payments)
    { _orders = orders; _payments = payments; }

    public async Task<Guid> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.New(cmd.CustomerId, cmd.Items, cmd.Currency);
        await _orders.AddAsync(order, ct);
        await _payments.ChargeAsync(order.Total, cmd.Currency, cmd.CustomerId.ToString(), ct);
        return order.Id;
    }
}

// Controller dispatches via mediator
[HttpPost]
public Task<Guid> Create([FromBody] CreateOrderCommand cmd, [FromServices] IMediator mediator, CancellationToken ct)
    => mediator.Send(cmd, ct);
```

Commands enable: replay from a queue, idempotency keys, audit logging, command/query separation, and **undo** via paired `UndoCommand` objects (rare in web apps; common in editors).

---

## 3. Interpreter

**Intent:** Define a representation for a grammar and an interpreter that uses it to interpret sentences.

**Real-world need:** Users supply a filter like `"price > 100 AND category = 'books'"`. You need to parse and execute it safely (without `eval`).

### Real project sample — building a LINQ expression from a tiny rule DSL

```csharp
public interface IRule<T>
{
    bool Evaluate(T item);
}

public sealed class GreaterThan<T> : IRule<T>
{
    private readonly Func<T, decimal> _selector;
    private readonly decimal _value;
    public GreaterThan(Func<T, decimal> sel, decimal v) { _selector = sel; _value = v; }
    public bool Evaluate(T item) => _selector(item) > _value;
}

public sealed class Equals<T, TVal> : IRule<T>
{
    private readonly Func<T, TVal> _selector;
    private readonly TVal _value;
    public Equals(Func<T, TVal> sel, TVal v) { _selector = sel; _value = v; }
    public bool Evaluate(T item) => EqualityComparer<TVal>.Default.Equals(_selector(item), _value);
}

public sealed class And<T> : IRule<T>
{
    private readonly IRule<T> _left, _right;
    public And(IRule<T> l, IRule<T> r) { _left = l; _right = r; }
    public bool Evaluate(T item) => _left.Evaluate(item) && _right.Evaluate(item);
}

// Usage — composed from a parser, applied to a stream of items
IRule<Product> rule = new And<Product>(
    new GreaterThan<Product>(p => p.Price, 100m),
    new Equals<Product, string>(p => p.Category, "books"));

var matches = products.Where(rule.Evaluate);
```

**Real codebase appearances:** LINQ expression trees, Roslyn, FluentValidation rule builders, business rule engines (e.g. NRules), regex engines, XPath/JSONPath, search query parsers (OData, GraphQL where-clauses).

---

## 4. Iterator

**Intent:** Provide a way to access the elements of an aggregate sequentially without exposing its underlying representation.

**Real-world need:** Read 100M database rows page-by-page; stream a large CSV file without loading it into memory.

### Real project sample — streaming pages from EF Core with `IAsyncEnumerable`

```csharp
public sealed class OrderRepository
{
    private readonly AppDbContext _db;
    public OrderRepository(AppDbContext db) => _db = db;

    public async IAsyncEnumerable<Order> StreamSinceAsync(DateTime since, [EnumeratorCancellation] CancellationToken ct = default)
    {
        const int pageSize = 500;
        var lastId = Guid.Empty;
        while (true)
        {
            var page = await _db.Orders
                .Where(o => o.CreatedAt >= since && o.Id.CompareTo(lastId) > 0)
                .OrderBy(o => o.Id)
                .Take(pageSize)
                .AsNoTracking()
                .ToListAsync(ct);

            if (page.Count == 0) yield break;

            foreach (var order in page) yield return order;
            lastId = page[^1].Id;
        }
    }
}

// Consumer — no idea about paging
await foreach (var order in repo.StreamSinceAsync(DateTime.UtcNow.AddDays(-30), ct))
{
    await _exporter.WriteRowAsync(order, ct);
}
```

**Built-in .NET:** every `foreach` uses `IEnumerator<T>`; `IAsyncEnumerable<T>` is its async sibling; `yield return` is a compiler-generated iterator.

---

## 5. Mediator

**Intent:** Define an object that encapsulates how a set of objects interact, promoting loose coupling.

**Real-world need:** Many handlers (validators, command processors, event subscribers) need to be wired together without each one knowing the others. A central dispatcher decouples them.

### Real project sample — MediatR with a pipeline behavior

```csharp
// A "behavior" wraps every request — perfect for cross-cutting concerns
public sealed class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var ctx = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(ctx, ct))))
                       .SelectMany(r => r.Errors)
                       .Where(f => f is not null)
                       .ToList();
        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}

// Wiring
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<Program>();
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

**Mediator vs. Facade:** Facade exposes a simpler interface over a subsystem; Mediator coordinates peers that all talk through it.

---

## 6. Memento

**Intent:** Capture and restore an object's internal state without violating encapsulation.

**Real-world need:** Undo/redo in an editor; "draft" snapshots of a form; transactional rollback when an in-memory operation fails partway.

### Real project sample — undoable document editor

```csharp
public sealed class TextDocument
{
    private string _text = "";

    public string Text => _text;
    public void Type(string s) => _text += s;

    // Memento — opaque snapshot
    public sealed record Snapshot(string Text);

    public Snapshot Save() => new(_text);
    public void Restore(Snapshot s) => _text = s.Text;
}

// Caretaker — owns the history but cannot peek inside
public sealed class UndoHistory
{
    private readonly Stack<TextDocument.Snapshot> _history = new();
    private readonly TextDocument _doc;
    public UndoHistory(TextDocument doc) => _doc = doc;

    public void Checkpoint() => _history.Push(_doc.Save());
    public void Undo() { if (_history.Count > 0) _doc.Restore(_history.Pop()); }
}

// Usage
var doc = new TextDocument();
var history = new UndoHistory(doc);

history.Checkpoint();
doc.Type("Hello ");
history.Checkpoint();
doc.Type("World");
history.Undo(); // back to "Hello "
```

---

## 7. Observer

**Intent:** Define a one-to-many dependency so that when one object changes state, all dependents are notified automatically.

**Real-world need:** When an order is placed, multiple subsystems must react: email, inventory, analytics, fraud check. The order code shouldn't know about any of them.

### Real project sample — domain events with MediatR notifications

```csharp
// The event (the "subject" emits this)
public sealed record OrderPlaced(Guid OrderId, decimal Total) : INotification;

// The publisher
public sealed class OrderService
{
    private readonly IMediator _mediator;
    public OrderService(IMediator mediator) => _mediator = mediator;

    public async Task PlaceAsync(Order order, CancellationToken ct)
    {
        // ... persist ...
        await _mediator.Publish(new OrderPlaced(order.Id, order.Total), ct);
    }
}

// Multiple, independent observers (subscribers)
public sealed class SendOrderEmail : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced ev, CancellationToken ct) { /* send email */ return Task.CompletedTask; }
}

public sealed class UpdateInventory : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced ev, CancellationToken ct) { /* decrement stock */ return Task.CompletedTask; }
}

public sealed class RecordAnalytics : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced ev, CancellationToken ct) { /* push to analytics */ return Task.CompletedTask; }
}
```

**Built-in .NET observer mechanisms:**

* `event` keyword (`EventHandler<T>`).
* `IObservable<T>` / `IObserver<T>` (Reactive Extensions).
* `INotifyPropertyChanged` in WPF/MAUI/Blazor.
* `IHostedService` reacting to host lifetime events.

---

## 8. State

**Intent:** Allow an object to alter its behavior when its internal state changes. The object appears to change its class.

**Real-world need:** An `Order` behaves very differently as `Draft`, `Submitted`, `Paid`, `Shipped`, `Refunded`. A pile of `if (status == X)` blocks gets messy fast.

### Real project sample — order state machine with the `Stateless` library

```csharp
public enum OrderState { Draft, Submitted, Paid, Shipped, Cancelled }
public enum OrderTrigger { Submit, Pay, Ship, Cancel }

public sealed class Order
{
    private readonly StateMachine<OrderState, OrderTrigger> _sm;

    public OrderState State => _sm.State;

    public Order()
    {
        _sm = new StateMachine<OrderState, OrderTrigger>(OrderState.Draft);

        _sm.Configure(OrderState.Draft)
           .Permit(OrderTrigger.Submit, OrderState.Submitted)
           .Permit(OrderTrigger.Cancel, OrderState.Cancelled);

        _sm.Configure(OrderState.Submitted)
           .Permit(OrderTrigger.Pay, OrderState.Paid)
           .Permit(OrderTrigger.Cancel, OrderState.Cancelled);

        _sm.Configure(OrderState.Paid)
           .Permit(OrderTrigger.Ship, OrderState.Shipped);

        _sm.Configure(OrderState.Shipped)
           .OnEntry(() => Console.WriteLine("Order shipped! Notify customer."));
    }

    public void Submit() => _sm.Fire(OrderTrigger.Submit);
    public void Pay()    => _sm.Fire(OrderTrigger.Pay);
    public void Ship()   => _sm.Fire(OrderTrigger.Ship);
    public void Cancel() => _sm.Fire(OrderTrigger.Cancel);
}
```

Without a state library, you'd implement each state as a class implementing `IOrderState` with the same operations — the textbook GoF approach. In practice, **Stateless** or workflow engines (Workflow Core, Elsa) cover this.

---

## 9. Strategy

**Intent:** Define a family of interchangeable algorithms, encapsulate each, and let the client choose at runtime.

**Real-world need:** Shipping cost depends on carrier (UPS / FedEx / DHL). Tax calculation depends on country. Discount rules depend on promo type.

### Real project sample — shipping cost strategies registered in DI

```csharp
public interface IShippingCostStrategy
{
    string CarrierCode { get; }
    decimal Calculate(Parcel parcel, Address dest);
}

public sealed class UpsStrategy : IShippingCostStrategy
{
    public string CarrierCode => "UPS";
    public decimal Calculate(Parcel p, Address d) => 5m + p.WeightKg * 1.20m;
}

public sealed class FedexStrategy : IShippingCostStrategy
{
    public string CarrierCode => "FEDEX";
    public decimal Calculate(Parcel p, Address d) => 6.50m + p.WeightKg * 1.10m;
}

public sealed class DhlStrategy : IShippingCostStrategy
{
    public string CarrierCode => "DHL";
    public decimal Calculate(Parcel p, Address d) => 7m + p.WeightKg * 0.95m;
}

// Resolver — picks a strategy by name
public sealed class ShippingCostResolver
{
    private readonly IReadOnlyDictionary<string, IShippingCostStrategy> _strategies;
    public ShippingCostResolver(IEnumerable<IShippingCostStrategy> strategies)
        => _strategies = strategies.ToDictionary(s => s.CarrierCode, StringComparer.OrdinalIgnoreCase);

    public decimal Calculate(string carrier, Parcel p, Address d) => _strategies[carrier].Calculate(p, d);
}

// DI — registering many implementations of one interface IS the .NET way to do Strategy
builder.Services.AddScoped<IShippingCostStrategy, UpsStrategy>();
builder.Services.AddScoped<IShippingCostStrategy, FedexStrategy>();
builder.Services.AddScoped<IShippingCostStrategy, DhlStrategy>();
builder.Services.AddScoped<ShippingCostResolver>();
```

**Strategy vs. State:** State changes itself; Strategy is chosen by the client.

---

## 10. Template Method

**Intent:** Define the skeleton of an algorithm in a base class; let subclasses override specific steps without changing the structure.

**Real-world need:** All ETL jobs follow the same skeleton — *extract → transform → load → notify* — but the actual queries and transformations differ.

### Real project sample — base background ETL job

```csharp
public abstract class EtlJob<TRow>
{
    protected readonly ILogger Log;
    protected EtlJob(ILogger log) => Log = log;

    // Template method — fixed skeleton
    public async Task RunAsync(CancellationToken ct)
    {
        Log.LogInformation("ETL starting: {Job}", GetType().Name);
        var rows = await ExtractAsync(ct);
        var transformed = Transform(rows);
        await LoadAsync(transformed, ct);
        await NotifyAsync(rows.Count, ct);
        Log.LogInformation("ETL finished: {Job}, {Count} rows", GetType().Name, rows.Count);
    }

    protected abstract Task<IReadOnlyList<TRow>> ExtractAsync(CancellationToken ct);
    protected abstract IReadOnlyList<TRow> Transform(IReadOnlyList<TRow> rows);
    protected abstract Task LoadAsync(IReadOnlyList<TRow> rows, CancellationToken ct);

    // Optional hook with default
    protected virtual Task NotifyAsync(int count, CancellationToken ct) => Task.CompletedTask;
}

// Concrete job fills in the steps
public sealed class DailyOrdersEtl : EtlJob<OrderRow>
{
    public DailyOrdersEtl(ILogger<DailyOrdersEtl> log) : base(log) {}

    protected override Task<IReadOnlyList<OrderRow>> ExtractAsync(CancellationToken ct) { /* SQL */ ... }
    protected override IReadOnlyList<OrderRow> Transform(IReadOnlyList<OrderRow> rows) { /* normalize */ ... }
    protected override Task LoadAsync(IReadOnlyList<OrderRow> rows, CancellationToken ct) { /* push to warehouse */ ... }
}
```

**Built-in .NET examples:** `BackgroundService.ExecuteAsync` is a template method; ASP.NET Core controllers inherit a fixed pipeline; xUnit `TestClass` lifecycle.

---

## 11. Visitor

**Intent:** Represent an operation to be performed on the elements of an object structure — letting you define a new operation without changing the classes of the elements.

**Real-world need:** You have a closed set of node types (an AST, a JSON tree, a tax-document hierarchy) and need to add many operations (validate, print, compile, optimize) without modifying the node classes each time.

### Real project sample — AST traversal with pattern-matched visitors (modern C#)

```csharp
public abstract record Expr;
public sealed record Number(double Value) : Expr;
public sealed record Add(Expr Left, Expr Right) : Expr;
public sealed record Multiply(Expr Left, Expr Right) : Expr;

public interface IExprVisitor<TResult>
{
    TResult Visit(Number n);
    TResult Visit(Add a);
    TResult Visit(Multiply m);
}

// Visitor 1 — evaluate the expression
public sealed class EvaluateVisitor : IExprVisitor<double>
{
    public double Visit(Number n)   => n.Value;
    public double Visit(Add a)      => Walk(a.Left) + Walk(a.Right);
    public double Visit(Multiply m) => Walk(m.Left) * Walk(m.Right);

    private double Walk(Expr e) => e switch
    {
        Number x   => Visit(x),
        Add x      => Visit(x),
        Multiply x => Visit(x),
        _          => throw new ArgumentOutOfRangeException()
    };
}

// Visitor 2 — pretty-print
public sealed class PrintVisitor : IExprVisitor<string>
{
    public string Visit(Number n)   => n.Value.ToString();
    public string Visit(Add a)      => $"({Walk(a.Left)} + {Walk(a.Right)})";
    public string Visit(Multiply m) => $"({Walk(m.Left)} * {Walk(m.Right)})";

    private string Walk(Expr e) => e switch
    {
        Number x   => Visit(x),
        Add x      => Visit(x),
        Multiply x => Visit(x),
        _          => throw new ArgumentOutOfRangeException()
    };
}

// Usage
var expr = new Add(new Number(2), new Multiply(new Number(3), new Number(4)));
Console.WriteLine(new EvaluateVisitor().Visit((dynamic)expr)); // 14
Console.WriteLine(new PrintVisitor().Visit((dynamic)expr));    // (2 + (3 * 4))
```

**Real codebase appearances:** Roslyn's `CSharpSyntaxWalker` / `SyntaxVisitor<T>`, expression-tree visitors in LINQ providers (`ExpressionVisitor`), serializers traversing a model.

---

## Summary Table

| Pattern                  | One-line intent                                      | Most common real use in .NET                              |
| ------------------------ | ---------------------------------------------------- | --------------------------------------------------------- |
| Chain of Responsibility  | Pass a request along handlers                        | ASP.NET middleware, `DelegatingHandler` pipeline           |
| Command                  | Encapsulate a request as an object                   | MediatR / CQRS commands, message-queue payloads            |
| Interpreter              | Evaluate a grammar                                   | LINQ expression trees, Roslyn, rule engines                |
| Iterator                 | Sequential traversal                                 | `IEnumerable<T>` / `IAsyncEnumerable<T>`, `yield return`   |
| Mediator                 | Coordinate peers through one object                  | MediatR with pipeline behaviors                            |
| Memento                  | Snapshot + restore state                             | Undo/redo, transactional in-memory rollback                |
| Observer                 | One-to-many notifications                            | `event`, `IObservable<T>`, domain events                   |
| State                    | Behavior changes with internal state                 | Order/booking workflows with `Stateless` or Workflow Core  |
| Strategy                 | Pluggable algorithms                                 | DI'd interface implementations, pricing/shipping rules     |
| Template Method          | Fixed algorithm skeleton, variable steps             | `BackgroundService`, controller base classes, base jobs    |
| Visitor                  | New operations on a fixed type hierarchy             | Roslyn `SyntaxVisitor`, `ExpressionVisitor`                |
