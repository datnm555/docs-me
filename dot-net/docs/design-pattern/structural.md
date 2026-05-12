# Structural Design Patterns in .NET

> The 7 GoF patterns concerned with **object composition** — how classes and objects are assembled into larger structures while remaining flexible and efficient. Each pattern below is shown with a **real-world scenario** you would actually meet in production .NET code.

---

## Table of Contents

1. [Adapter](#1-adapter)
2. [Bridge](#2-bridge)
3. [Composite](#3-composite)
4. [Decorator](#4-decorator)
5. [Facade](#5-facade)
6. [Flyweight](#6-flyweight)
7. [Proxy](#7-proxy)

---

## 1. Adapter

**Intent:** Convert the interface of a class into another interface clients expect — a translator between incompatible APIs.

**Real-world need:** Your codebase defines `IPaymentGateway`, but the vendor SDK exposes `LegacyStripeClient` with a completely different shape. You don't want vendor types leaking into your domain.

### Real project sample — wrapping a 3rd-party SDK

```csharp
// Your domain contract (stable, owned by your team)
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(decimal amount, string currency, string customerId, CancellationToken ct);
}

public record PaymentResult(bool Success, string? TransactionId, string? Error);

// 3rd-party SDK (you don't control its shape)
public class LegacyStripeClient
{
    public StripeResponse Charge(StripeChargeRequest req) { /* sync, throws */ ... }
}

// Adapter — bridges legacy SDK to your interface
public sealed class StripeGatewayAdapter : IPaymentGateway
{
    private readonly LegacyStripeClient _stripe;

    public StripeGatewayAdapter(LegacyStripeClient stripe) => _stripe = stripe;

    public Task<PaymentResult> ChargeAsync(decimal amount, string currency, string customerId, CancellationToken ct)
    {
        try
        {
            var resp = _stripe.Charge(new StripeChargeRequest
            {
                AmountCents = (long)(amount * 100),
                Currency    = currency.ToLowerInvariant(),
                Customer    = customerId,
            });
            return Task.FromResult(new PaymentResult(true, resp.Id, null));
        }
        catch (StripeException ex)
        {
            return Task.FromResult(new PaymentResult(false, null, ex.Message));
        }
    }
}

// DI registration
builder.Services.AddSingleton<LegacyStripeClient>();
builder.Services.AddSingleton<IPaymentGateway, StripeGatewayAdapter>();
```

**Why it matters in real projects:** swapping Stripe for Adyen later only requires writing a new adapter — your `CheckoutService` never changes.

---

## 2. Bridge

**Intent:** Decouple an abstraction from its implementation so that the two can vary independently.

**Real-world need:** You need to send notifications. Notifications can be **Email**, **SMS**, **Push**; messages can be **Marketing**, **Transactional**, **Alert**. Without Bridge, you end up with `MarketingEmail`, `MarketingSms`, `AlertEmail`, `AlertSms` … an explosion.

### Real project sample — multi-channel notifications

```csharp
// Implementation hierarchy: HOW to send
public interface IMessageChannel
{
    Task SendAsync(string recipient, string subject, string body, CancellationToken ct);
}

public sealed class EmailChannel : IMessageChannel { /* SMTP / SendGrid */ }
public sealed class SmsChannel   : IMessageChannel { /* Twilio */ }
public sealed class PushChannel  : IMessageChannel { /* FCM */ }

// Abstraction hierarchy: WHAT the message is
public abstract class Notification
{
    protected readonly IMessageChannel Channel;
    protected Notification(IMessageChannel channel) => Channel = channel;
    public abstract Task NotifyAsync(string recipient, CancellationToken ct);
}

public sealed class OrderShippedNotification : Notification
{
    private readonly Order _order;
    public OrderShippedNotification(IMessageChannel channel, Order order) : base(channel) => _order = order;

    public override Task NotifyAsync(string recipient, CancellationToken ct) =>
        Channel.SendAsync(recipient,
            subject: $"Your order #{_order.Id} has shipped",
            body:    $"Track it at {_order.TrackingUrl}",
            ct);
}

public sealed class PasswordResetNotification : Notification
{
    private readonly string _resetLink;
    public PasswordResetNotification(IMessageChannel channel, string link) : base(channel) => _resetLink = link;

    public override Task NotifyAsync(string recipient, CancellationToken ct) =>
        Channel.SendAsync(recipient, "Reset your password", $"Click: {_resetLink}", ct);
}

// Mix any message with any channel at runtime
var sms = new SmsChannel();
var notif = new OrderShippedNotification(sms, order);
await notif.NotifyAsync("+84901234567", ct);
```

**Bridge vs. Strategy:** Strategy is one axis of variation (algorithm). Bridge is **two** independent axes (message-type × channel) varying together.

---

## 3. Composite

**Intent:** Compose objects into tree structures and treat individual objects and compositions uniformly.

**Real-world need:** File system trees, organizational hierarchies, UI component trees, menu structures, **nested permissions / pricing rules**, GraphQL/JSON schemas.

### Real project sample — folder/file tree with size calculation

```csharp
public abstract class FileSystemNode
{
    public string Name { get; }
    protected FileSystemNode(string name) => Name = name;
    public abstract long GetSize();
}

public sealed class FileNode : FileSystemNode
{
    private readonly long _bytes;
    public FileNode(string name, long bytes) : base(name) => _bytes = bytes;
    public override long GetSize() => _bytes;
}

public sealed class FolderNode : FileSystemNode
{
    private readonly List<FileSystemNode> _children = new();
    public FolderNode(string name) : base(name) {}

    public FolderNode Add(FileSystemNode child) { _children.Add(child); return this; }

    // Same operation works on a single file or a deep folder tree
    public override long GetSize() => _children.Sum(c => c.GetSize());
}

// Usage
var root = new FolderNode("project")
    .Add(new FileNode("README.md", 2_300))
    .Add(new FolderNode("src")
        .Add(new FileNode("Program.cs", 1_200))
        .Add(new FolderNode("Services")
            .Add(new FileNode("OrderService.cs", 4_800))));

Console.WriteLine(root.GetSize()); // recurses transparently
```

**Real codebase appearances:** Razor `RenderTreeBuilder`, Roslyn syntax trees (`SyntaxNode`), Blazor component trees, HTML DOMs, expression trees.

---

## 4. Decorator

**Intent:** Attach new responsibilities to an object dynamically. A flexible alternative to subclassing for extending behavior.

**Real-world need:** You have `IOrderRepository`. You want to **cache** reads, **log** every call, and **measure latency** — without changing the implementation or the consumers.

### Real project sample — caching + logging decorators around a repository

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
}

// Real implementation
public sealed class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public SqlOrderRepository(AppDbContext db) => _db = db;
    public Task<Order?> GetAsync(Guid id, CancellationToken ct) => _db.Orders.FindAsync([id], ct).AsTask();
}

// Decorator 1 — caching
public sealed class CachingOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly IMemoryCache _cache;

    public CachingOrderRepository(IOrderRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order?> GetAsync(Guid id, CancellationToken ct)
    {
        if (_cache.TryGetValue(id, out Order? cached)) return cached;
        var order = await _inner.GetAsync(id, ct);
        if (order is not null) _cache.Set(id, order, TimeSpan.FromMinutes(5));
        return order;
    }
}

// Decorator 2 — logging
public sealed class LoggingOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly ILogger<LoggingOrderRepository> _log;

    public LoggingOrderRepository(IOrderRepository inner, ILogger<LoggingOrderRepository> log)
    { _inner = inner; _log = log; }

    public async Task<Order?> GetAsync(Guid id, CancellationToken ct)
    {
        _log.LogInformation("Loading order {OrderId}", id);
        var sw = System.Diagnostics.Stopwatch.StartNew();
        var result = await _inner.GetAsync(id, ct);
        _log.LogInformation("Loaded order {OrderId} in {Ms}ms (hit={Hit})", id, sw.ElapsedMilliseconds, result is not null);
        return result;
    }
}

// Compose at startup — order matters: log around cache around SQL
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddMemoryCache();
builder.Services.AddScoped<SqlOrderRepository>();
builder.Services.AddScoped<IOrderRepository>(sp =>
    new LoggingOrderRepository(
        new CachingOrderRepository(
            sp.GetRequiredService<SqlOrderRepository>(),
            sp.GetRequiredService<IMemoryCache>()),
        sp.GetRequiredService<ILogger<LoggingOrderRepository>>()));
```

### Cleaner with **Scrutor** library

```csharp
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.Decorate<IOrderRepository, CachingOrderRepository>();
builder.Services.Decorate<IOrderRepository, LoggingOrderRepository>();
```

**Real codebase appearances:** ASP.NET Core middleware *is* the Decorator pattern applied to `HttpContext`; `DelegatingHandler` in `HttpClient`.

---

## 5. Facade

**Intent:** Provide a unified, simpler interface to a set of interfaces in a subsystem.

**Real-world need:** Placing an order involves: validating the cart, reserving inventory, charging payment, creating the order record, sending a confirmation email, and emitting a domain event. Controllers shouldn't orchestrate six dependencies.

### Real project sample — Checkout facade in an e-commerce app

```csharp
public interface ICheckoutFacade
{
    Task<CheckoutResult> PlaceOrderAsync(PlaceOrderRequest req, CancellationToken ct);
}

public sealed class CheckoutFacade : ICheckoutFacade
{
    private readonly ICartValidator _cart;
    private readonly IInventoryService _inventory;
    private readonly IPaymentGateway _payments;
    private readonly IOrderRepository _orders;
    private readonly IEmailService _email;
    private readonly IEventBus _bus;

    public CheckoutFacade(ICartValidator c, IInventoryService i, IPaymentGateway p,
                          IOrderRepository o, IEmailService e, IEventBus b)
    { _cart = c; _inventory = i; _payments = p; _orders = o; _email = e; _bus = b; }

    public async Task<CheckoutResult> PlaceOrderAsync(PlaceOrderRequest req, CancellationToken ct)
    {
        await _cart.ValidateAsync(req.Cart, ct);
        await _inventory.ReserveAsync(req.Cart.Items, ct);

        var charge = await _payments.ChargeAsync(req.Cart.Total, req.Currency, req.CustomerId, ct);
        if (!charge.Success) return CheckoutResult.PaymentFailed(charge.Error!);

        var order = await _orders.CreateAsync(req, charge.TransactionId!, ct);
        await _email.SendOrderConfirmationAsync(order, ct);
        await _bus.PublishAsync(new OrderPlaced(order.Id), ct);

        return CheckoutResult.Ok(order.Id);
    }
}

// Controller stays thin
[ApiController, Route("checkout")]
public class CheckoutController(ICheckoutFacade checkout) : ControllerBase
{
    [HttpPost]
    public Task<CheckoutResult> Post([FromBody] PlaceOrderRequest req, CancellationToken ct)
        => checkout.PlaceOrderAsync(req, ct);
}
```

**Facade vs. Mediator:** Facade is one-way (clients → subsystem). Mediator is multi-directional coordination between peers.

---

## 6. Flyweight

**Intent:** Share fine-grained objects efficiently to support large numbers of them.

**Real-world need:** You're rendering 100,000 markers on a map, or 1M cells in a spreadsheet, or millions of log entries. Each has a *type* (icon, font, color) that is shared, and *state* (coordinates, value) that is unique.

### Real project sample — map markers

```csharp
// Intrinsic (shared) state
public sealed class MarkerStyle
{
    public string IconUrl { get; }
    public string Color { get; }
    public int Size { get; }
    public MarkerStyle(string icon, string color, int size)
    { IconUrl = icon; Color = color; Size = size; }
}

// Flyweight factory — interns MarkerStyle instances
public static class MarkerStyleFactory
{
    private static readonly ConcurrentDictionary<string, MarkerStyle> _pool = new();

    public static MarkerStyle Get(string icon, string color, int size)
    {
        var key = $"{icon}|{color}|{size}";
        return _pool.GetOrAdd(key, _ => new MarkerStyle(icon, color, size));
    }
}

// Each marker — extrinsic state + reference to shared flyweight
public record struct Marker(double Lat, double Lng, MarkerStyle Style);

// Rendering 1M markers — but only a handful of MarkerStyle objects exist
var markers = new List<Marker>();
foreach (var poi in poisFromDb) // 1M rows
{
    var style = MarkerStyleFactory.Get(poi.Category.Icon, poi.Category.Color, 32);
    markers.Add(new Marker(poi.Lat, poi.Lng, style));
}
```

**Real codebase appearances:** `string.Intern`, `Enum` boxing avoidance, font/icon caches in UI frameworks, the `StringPool` in `Microsoft.Extensions.ObjectPool`.

---

## 7. Proxy

**Intent:** Provide a placeholder or surrogate for another object to control access to it.

**Real-world variants:**

* **Virtual Proxy** — lazy / deferred loading.
* **Protection Proxy** — authorization checks.
* **Remote Proxy** — local stand-in for a remote service (gRPC clients, Refit).
* **Caching Proxy** — see also Decorator.

### Real project sample — authorization proxy around a sensitive service

```csharp
public interface IUserService
{
    Task<User> GetAsync(Guid userId, CancellationToken ct);
    Task DeleteAsync(Guid userId, CancellationToken ct);
}

public sealed class UserService : IUserService { /* real EF Core implementation */ }

public sealed class AuthorizedUserService : IUserService
{
    private readonly IUserService _inner;
    private readonly ICurrentUser _current;

    public AuthorizedUserService(IUserService inner, ICurrentUser current)
    { _inner = inner; _current = current; }

    public Task<User> GetAsync(Guid userId, CancellationToken ct)
    {
        // Users can read their own profile; admins can read anyone's.
        if (_current.Id != userId && !_current.IsInRole("Admin"))
            throw new UnauthorizedAccessException();
        return _inner.GetAsync(userId, ct);
    }

    public Task DeleteAsync(Guid userId, CancellationToken ct)
    {
        if (!_current.IsInRole("Admin"))
            throw new UnauthorizedAccessException();
        return _inner.DeleteAsync(userId, ct);
    }
}
```

### Built-in .NET proxies

* **EF Core lazy-loading proxies** (`UseLazyLoadingProxies()`) — `Order.Customer` is a generated proxy that lazily loads the customer.
* **gRPC** and **Refit** generate **remote proxies** at compile time — local C# objects whose methods make HTTP calls.
* **Castle DynamicProxy** — the engine behind Moq, NSubstitute, AutoFac interceptors.

**Decorator vs. Proxy:** mechanically identical (wraps an object with the same interface). The **intent** differs:

* Decorator → **adds** behavior the client wants (logging, caching).
* Proxy → **controls** access (auth, lazy load, remote call) — client may not even know.

---

## Summary Table

| Pattern    | One-line intent                          | Most common real use in .NET                          |
| ---------- | ---------------------------------------- | ----------------------------------------------------- |
| Adapter    | Translate one interface to another       | Wrapping 3rd-party SDKs behind your domain interface  |
| Bridge     | Vary abstraction & implementation        | Multi-channel notification / report engines            |
| Composite  | Treat trees uniformly                    | Syntax trees, UI components, folder/permission trees   |
| Decorator  | Add behavior at runtime                  | Caching/logging/retry around services; ASP.NET middleware |
| Facade     | Simplify a complex subsystem             | "Use-case" services orchestrating many dependencies    |
| Flyweight  | Share intrinsic state                    | Icon/font/style pools, `string.Intern`                 |
| Proxy      | Control access to an object              | EF lazy loading, gRPC/Refit clients, authorization wrappers |
