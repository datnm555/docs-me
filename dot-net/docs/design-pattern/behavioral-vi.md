# Behavioral Design Pattern trong .NET

> 11 GoF pattern liên quan đến **giao tiếp và trách nhiệm** giữa các object — cách chúng tương tác, ai xử lý gì, và thuật toán vary ra sao. Mỗi pattern dưới đây có **scenario thực tế** bạn sẽ gặp trong production .NET.

> 🇻🇳 Phiên bản tiếng Việt. English: [`behavioral.md`](./behavioral.md)

---

## Mục lục

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

**Ý định:** Truyền request qua một chuỗi handler; mỗi handler quyết định xử lý hoặc truyền tiếp.

**Nhu cầu thực tế:** Một HTTP request cần chảy qua logging, authentication, rate-limiting, caching, và cuối cùng tới controller. Mỗi step có thể short-circuit.

### Sample thực tế — ASP.NET Core middleware (CoR canonical trong .NET)

```csharp
// Mỗi middleware là một handler trong chain
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
            return; // short-circuit chain
        }
        await _next(ctx); // truyền tới handler kế tiếp
    }
}

// Program.cs — ráp chain
app.UseSerilogRequestLogging();
app.UseAuthentication();
app.UseMiddleware<RateLimitMiddleware>();
app.UseAuthorization();
app.UseResponseCaching();
app.MapControllers();
```

**Cách dùng thực tế khác:** pipeline `DelegatingHandler` của `HttpClient`, validation pipeline (chain FluentValidation), event filter trong CQRS, fallback xử lý exception.

---

## 2. Command

**Ý định:** Đóng gói request thành object — cho phép queue, log, undo, parameterize.

**Nhu cầu thực tế:** Bạn muốn mọi operation thay đổi state có thể audit, retry, và trigger từ queue. Gọi method service trực tiếp quá coupling.

### Sample thực tế — command CQRS với MediatR

```csharp
// Command — record mang data request
public record CreateOrderCommand(Guid CustomerId, IReadOnlyList<OrderItemDto> Items, string Currency)
    : IRequest<Guid>;

// Handler — trách nhiệm duy nhất, dễ test
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

// Controller dispatch qua mediator
[HttpPost]
public Task<Guid> Create([FromBody] CreateOrderCommand cmd, [FromServices] IMediator mediator, CancellationToken ct)
    => mediator.Send(cmd, ct);
```

Command cho phép: replay từ queue, idempotency key, audit logging, tách command/query, và **undo** qua object `UndoCommand` ghép cặp (hiếm trong web app; phổ biến trong editor).

---

## 3. Interpreter

**Ý định:** Định nghĩa representation cho grammar và interpreter dùng nó để diễn giải câu trong grammar.

**Nhu cầu thực tế:** User cung cấp filter như `"price > 100 AND category = 'books'"`. Bạn cần parse và thực thi an toàn (không dùng `eval`).

### Sample thực tế — build LINQ expression từ DSL rule nhỏ

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

// Usage — compose từ parser, áp dụng cho stream item
IRule<Product> rule = new And<Product>(
    new GreaterThan<Product>(p => p.Price, 100m),
    new Equals<Product, string>(p => p.Category, "books"));

var matches = products.Where(rule.Evaluate);
```

**Xuất hiện trong codebase thực:** LINQ expression tree, Roslyn, FluentValidation rule builder, business rule engine (vd. NRules), regex engine, XPath/JSONPath, search query parser (OData, GraphQL where-clause).

---

## 4. Iterator

**Ý định:** Cách truy cập element của aggregate tuần tự mà không expose representation bên trong.

**Nhu cầu thực tế:** Đọc 100M row từ DB theo từng page; stream file CSV lớn mà không load hết vào memory.

### Sample thực tế — stream từng page từ EF Core với `IAsyncEnumerable`

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

// Consumer — không biết gì về paging
await foreach (var order in repo.StreamSinceAsync(DateTime.UtcNow.AddDays(-30), ct))
{
    await _exporter.WriteRowAsync(order, ct);
}
```

**Built-in .NET:** mỗi `foreach` dùng `IEnumerator<T>`; `IAsyncEnumerable<T>` là sibling async; `yield return` là iterator compiler-generated.

---

## 5. Mediator

**Ý định:** Định nghĩa object đóng gói cách một bộ object tương tác, thúc đẩy loose coupling.

**Nhu cầu thực tế:** Nhiều handler (validator, command processor, event subscriber) cần được wire mà không cái nào biết về cái khác. Dispatcher trung tâm decouple chúng.

### Sample thực tế — MediatR với pipeline behavior

```csharp
// "Behavior" bọc mọi request — hoàn hảo cho cross-cutting concern
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

**Mediator vs. Facade:** Facade expose interface đơn giản hơn trên subsystem; Mediator điều phối các peer giao tiếp qua nó.

---

## 6. Memento

**Ý định:** Capture và restore state nội bộ của object mà không vi phạm encapsulation.

**Nhu cầu thực tế:** Undo/redo trong editor; "draft" snapshot của form; rollback transactional khi operation in-memory fail giữa chừng.

### Sample thực tế — editor document có thể undo

```csharp
public sealed class TextDocument
{
    private string _text = "";

    public string Text => _text;
    public void Type(string s) => _text += s;

    // Memento — snapshot opaque
    public sealed record Snapshot(string Text);

    public Snapshot Save() => new(_text);
    public void Restore(Snapshot s) => _text = s.Text;
}

// Caretaker — sở hữu history nhưng không peek bên trong
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
history.Undo(); // về lại "Hello "
```

---

## 7. Observer

**Ý định:** Định nghĩa dependency one-to-many để khi một object thay đổi state, mọi dependent được notify tự động.

**Nhu cầu thực tế:** Khi order được place, nhiều subsystem phải phản ứng: email, inventory, analytics, fraud check. Code order không nên biết về cái nào.

### Sample thực tế — domain event với MediatR notification

```csharp
// Event (subject emit cái này)
public sealed record OrderPlaced(Guid OrderId, decimal Total) : INotification;

// Publisher
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

// Nhiều observer (subscriber) độc lập
public sealed class SendOrderEmail : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced ev, CancellationToken ct) { /* gửi email */ return Task.CompletedTask; }
}

public sealed class UpdateInventory : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced ev, CancellationToken ct) { /* decrement stock */ return Task.CompletedTask; }
}

public sealed class RecordAnalytics : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced ev, CancellationToken ct) { /* push analytics */ return Task.CompletedTask; }
}
```

**Cơ chế observer built-in trong .NET:**

* `event` keyword (`EventHandler<T>`).
* `IObservable<T>` / `IObserver<T>` (Reactive Extensions).
* `INotifyPropertyChanged` trong WPF/MAUI/Blazor.
* `IHostedService` phản ứng với event lifetime của host.

---

## 8. State

**Ý định:** Cho object thay đổi behavior khi state nội bộ thay đổi. Object trông như thay đổi class.

**Nhu cầu thực tế:** `Order` hành xử rất khác khi là `Draft`, `Submitted`, `Paid`, `Shipped`, `Refunded`. Một mớ block `if (status == X)` nhanh chóng lộn xộn.

### Sample thực tế — order state machine với thư viện `Stateless`

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

Không có thư viện state, bạn sẽ implement mỗi state thành class implement `IOrderState` với cùng operation — đây là cách textbook GoF. Trong thực tế, **Stateless** hoặc workflow engine (Workflow Core, Elsa) bao phủ điều này.

---

## 9. Strategy

**Ý định:** Định nghĩa gia đình thuật toán có thể đổi, đóng gói mỗi cái, và cho client chọn runtime.

**Nhu cầu thực tế:** Cost shipping phụ thuộc carrier (UPS / FedEx / DHL). Tính thuế phụ thuộc quốc gia. Rule discount phụ thuộc loại promo.

### Sample thực tế — strategy cost shipping đăng ký trong DI

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

// Resolver — chọn strategy theo tên
public sealed class ShippingCostResolver
{
    private readonly IReadOnlyDictionary<string, IShippingCostStrategy> _strategies;
    public ShippingCostResolver(IEnumerable<IShippingCostStrategy> strategies)
        => _strategies = strategies.ToDictionary(s => s.CarrierCode, StringComparer.OrdinalIgnoreCase);

    public decimal Calculate(string carrier, Parcel p, Address d) => _strategies[carrier].Calculate(p, d);
}

// DI — register nhiều implementation của một interface CHÍNH LÀ cách .NET làm Strategy
builder.Services.AddScoped<IShippingCostStrategy, UpsStrategy>();
builder.Services.AddScoped<IShippingCostStrategy, FedexStrategy>();
builder.Services.AddScoped<IShippingCostStrategy, DhlStrategy>();
builder.Services.AddScoped<ShippingCostResolver>();
```

**Strategy vs. State:** State tự đổi; Strategy được client chọn.

---

## 10. Template Method

**Ý định:** Định nghĩa skeleton của thuật toán trong base class; cho subclass override step cụ thể mà không đổi cấu trúc.

**Nhu cầu thực tế:** Mọi ETL job đi theo cùng skeleton — *extract → transform → load → notify* — nhưng query và transformation thật sự khác nhau.

### Sample thực tế — base background ETL job

```csharp
public abstract class EtlJob<TRow>
{
    protected readonly ILogger Log;
    protected EtlJob(ILogger log) => Log = log;

    // Template method — skeleton cố định
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

    // Hook tuỳ chọn với default
    protected virtual Task NotifyAsync(int count, CancellationToken ct) => Task.CompletedTask;
}

// Concrete job fill các step
public sealed class DailyOrdersEtl : EtlJob<OrderRow>
{
    public DailyOrdersEtl(ILogger<DailyOrdersEtl> log) : base(log) {}

    protected override Task<IReadOnlyList<OrderRow>> ExtractAsync(CancellationToken ct) { /* SQL */ ... }
    protected override IReadOnlyList<OrderRow> Transform(IReadOnlyList<OrderRow> rows) { /* normalize */ ... }
    protected override Task LoadAsync(IReadOnlyList<OrderRow> rows, CancellationToken ct) { /* push warehouse */ ... }
}
```

**Ví dụ built-in trong .NET:** `BackgroundService.ExecuteAsync` là template method; ASP.NET Core controller thừa kế pipeline cố định; lifecycle `TestClass` của xUnit.

---

## 11. Visitor

**Ý định:** Biểu diễn operation thực hiện trên element của cấu trúc object — cho phép định nghĩa operation mới mà không thay đổi class element.

**Nhu cầu thực tế:** Bạn có một bộ kín các node type (AST, JSON tree, hierarchy tax-document) và cần thêm nhiều operation (validate, print, compile, optimize) mà không sửa class node mỗi lần.

### Sample thực tế — duyệt AST với visitor pattern-matched (C# hiện đại)

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

// Visitor 1 — evaluate expression
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

**Xuất hiện trong codebase thực:** `CSharpSyntaxWalker` / `SyntaxVisitor<T>` của Roslyn, visitor expression tree trong LINQ provider (`ExpressionVisitor`), serializer duyệt model.

---

## Bảng tổng kết

| Pattern                  | Ý định một dòng                                       | Cách dùng phổ biến nhất trong .NET                          |
| ------------------------ | ----------------------------------------------------- | ----------------------------------------------------------- |
| Chain of Responsibility  | Truyền request qua handler                            | ASP.NET middleware, pipeline `DelegatingHandler`             |
| Command                  | Đóng gói request thành object                         | Command MediatR / CQRS, payload message queue                |
| Interpreter              | Diễn giải grammar                                     | LINQ expression tree, Roslyn, rule engine                    |
| Iterator                 | Duyệt tuần tự                                         | `IEnumerable<T>` / `IAsyncEnumerable<T>`, `yield return`     |
| Mediator                 | Điều phối peer qua một object                         | MediatR với pipeline behavior                                |
| Memento                  | Snapshot + restore state                              | Undo/redo, rollback transactional in-memory                  |
| Observer                 | Notification one-to-many                              | `event`, `IObservable<T>`, domain event                      |
| State                    | Behavior thay đổi theo state nội bộ                   | Workflow order/booking với `Stateless` hoặc Workflow Core    |
| Strategy                 | Thuật toán có thể plug                                | Implementation interface qua DI, rule pricing/shipping       |
| Template Method          | Skeleton thuật toán cố định, step có thể đổi          | `BackgroundService`, base controller, base job               |
| Visitor                  | Operation mới trên hierarchy type cố định             | `SyntaxVisitor` Roslyn, `ExpressionVisitor`                  |
