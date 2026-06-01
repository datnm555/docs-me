# Structural Design Pattern trong .NET

> 7 GoF pattern liên quan đến **kết hợp object** — cách class và object được lắp thành cấu trúc lớn hơn mà vẫn linh hoạt và hiệu quả. Mỗi pattern dưới đây có **scenario thực tế** bạn sẽ gặp trong production .NET.

> 🇻🇳 Phiên bản tiếng Việt. English: [`structural.md`](./structural.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — 7 GoF structural pattern: **Adapter**, **Bridge**, **Composite**, **Decorator**, **Facade**, **Flyweight**, **Proxy** — liên quan đến cách class và object được compose thành cấu trúc lớn hơn.
- **Tại sao** — Composition linh hoạt hơn inheritance. Structural pattern cho phép bridge interface không tương thích (Adapter), wrap behavior (Decorator), share object fine-grained (Flyweight), hoặc ẩn phức tạp (Facade) mà không sửa code được wrap.
- **Khi nào** — Integrating với SDK third-party mà shape không fit (Adapter); thêm cross-cutting concern như caching/logging/retry quanh service (Decorator); present API thống nhất trên subsystem (Facade); render nhiều UI element tương tự (Flyweight); control access (Proxy).
- **Ở đâu** — Mức class/object. Trong .NET: ASP.NET Core middleware *là* Decorator trên `HttpContext`; EF Core lazy-loading là Proxy; `DelegatingHandler` là Decorator flavored Chain-of-Responsibility; `Decorate<T>()` của Scrutor là form DI.

---

## Mục lục

1. [Adapter](#1-adapter)
2. [Bridge](#2-bridge)
3. [Composite](#3-composite)
4. [Decorator](#4-decorator)
5. [Facade](#5-facade)
6. [Flyweight](#6-flyweight)
7. [Proxy](#7-proxy)

---

## 1. Adapter

**Ý định:** Chuyển interface của một class thành interface client muốn — translator giữa các API không tương thích.

**Nhu cầu thực tế:** Codebase định nghĩa `IPaymentGateway`, nhưng vendor SDK expose `LegacyStripeClient` với shape khác hoàn toàn. Bạn không muốn vendor type leak vào domain.

### Sample thực tế — bọc SDK third-party

```csharp
// Contract domain của bạn (ổn định, team bạn sở hữu)
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(decimal amount, string currency, string customerId, CancellationToken ct);
}

public record PaymentResult(bool Success, string? TransactionId, string? Error);

// SDK third-party (bạn không kiểm soát shape)
public class LegacyStripeClient
{
    public StripeResponse Charge(StripeChargeRequest req) { /* sync, throw */ ... }
}

// Adapter — bắc cầu legacy SDK với interface của bạn
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

**Tại sao quan trọng trong dự án thực:** đổi Stripe sang Adyen sau này chỉ cần viết adapter mới — `CheckoutService` không bao giờ thay đổi.

---

## 2. Bridge

**Ý định:** Decouple một abstraction khỏi implementation để cả hai vary độc lập.

**Nhu cầu thực tế:** Bạn cần gửi notification. Notification có thể là **Email**, **SMS**, **Push**; message có thể là **Marketing**, **Transactional**, **Alert**. Không có Bridge, bạn end up với `MarketingEmail`, `MarketingSms`, `AlertEmail`, `AlertSms` … bùng nổ.

### Sample thực tế — notification đa kênh

```csharp
// Implementation hierarchy: GỬI BẰNG CÁCH NÀO
public interface IMessageChannel
{
    Task SendAsync(string recipient, string subject, string body, CancellationToken ct);
}

public sealed class EmailChannel : IMessageChannel { /* SMTP / SendGrid */ }
public sealed class SmsChannel   : IMessageChannel { /* Twilio */ }
public sealed class PushChannel  : IMessageChannel { /* FCM */ }

// Abstraction hierarchy: MESSAGE LÀ GÌ
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

// Mix bất kỳ message với bất kỳ channel runtime
var sms = new SmsChannel();
var notif = new OrderShippedNotification(sms, order);
await notif.NotifyAsync("+84901234567", ct);
```

**Bridge vs. Strategy:** Strategy là một trục variation (thuật toán). Bridge là **hai** trục độc lập (message-type × channel) vary cùng nhau.

---

## 3. Composite

**Ý định:** Compose object thành cấu trúc cây và treat object đơn lẻ và composition đồng nhất.

**Nhu cầu thực tế:** Cây file system, hierarchy tổ chức, cây UI component, cấu trúc menu, **nested permission / pricing rule**, GraphQL/JSON schema.

### Sample thực tế — cây folder/file với tính kích thước

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

    // Cùng operation hoạt động trên một file hoặc cây folder sâu
    public override long GetSize() => _children.Sum(c => c.GetSize());
}

// Usage
var root = new FolderNode("project")
    .Add(new FileNode("README.md", 2_300))
    .Add(new FolderNode("src")
        .Add(new FileNode("Program.cs", 1_200))
        .Add(new FolderNode("Services")
            .Add(new FileNode("OrderService.cs", 4_800))));

Console.WriteLine(root.GetSize()); // đệ quy trong suốt
```

**Xuất hiện trong codebase thực:** Razor `RenderTreeBuilder`, cây syntax Roslyn (`SyntaxNode`), cây component Blazor, DOM HTML, expression tree.

---

## 4. Decorator

**Ý định:** Gắn trách nhiệm mới vào object động. Lựa chọn linh hoạt thay cho subclassing để mở rộng behavior.

**Nhu cầu thực tế:** Bạn có `IOrderRepository`. Bạn muốn **cache** read, **log** mọi call, và **đo latency** — mà không thay đổi implementation hay consumer.

### Sample thực tế — caching + logging decorator quanh repository

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
}

// Implementation thật
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

// Compose ở startup — thứ tự quan trọng: log quanh cache quanh SQL
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

### Sạch hơn với thư viện **Scrutor**

```csharp
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.Decorate<IOrderRepository, CachingOrderRepository>();
builder.Services.Decorate<IOrderRepository, LoggingOrderRepository>();
```

**Xuất hiện trong codebase thực:** ASP.NET Core middleware *chính là* Decorator pattern áp dụng cho `HttpContext`; `DelegatingHandler` trong `HttpClient`.

---

## 5. Facade

**Ý định:** Cung cấp interface thống nhất, đơn giản hơn cho một bộ interface trong subsystem.

**Nhu cầu thực tế:** Place order gồm: validate cart, reserve inventory, charge payment, tạo order record, gửi email confirmation, emit domain event. Controller không nên orchestrate 6 dependency.

### Sample thực tế — Checkout facade trong app e-commerce

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

// Controller giữ mỏng
[ApiController, Route("checkout")]
public class CheckoutController(ICheckoutFacade checkout) : ControllerBase
{
    [HttpPost]
    public Task<CheckoutResult> Post([FromBody] PlaceOrderRequest req, CancellationToken ct)
        => checkout.PlaceOrderAsync(req, ct);
}
```

**Facade vs. Mediator:** Facade một chiều (client → subsystem). Mediator điều phối nhiều chiều giữa các peer.

---

## 6. Flyweight

**Ý định:** Share object nhỏ hiệu quả để hỗ trợ số lượng lớn.

**Nhu cầu thực tế:** Render 100,000 marker trên map, hoặc 1M cell trong spreadsheet, hoặc hàng triệu log entry. Mỗi cái có *type* (icon, font, color) được share, và *state* (toạ độ, value) là duy nhất.

### Sample thực tế — map marker

```csharp
// Intrinsic state (được share)
public sealed class MarkerStyle
{
    public string IconUrl { get; }
    public string Color { get; }
    public int Size { get; }
    public MarkerStyle(string icon, string color, int size)
    { IconUrl = icon; Color = color; Size = size; }
}

// Flyweight factory — intern instance MarkerStyle
public static class MarkerStyleFactory
{
    private static readonly ConcurrentDictionary<string, MarkerStyle> _pool = new();

    public static MarkerStyle Get(string icon, string color, int size)
    {
        var key = $"{icon}|{color}|{size}";
        return _pool.GetOrAdd(key, _ => new MarkerStyle(icon, color, size));
    }
}

// Mỗi marker — extrinsic state + reference tới flyweight share
public record struct Marker(double Lat, double Lng, MarkerStyle Style);

// Render 1M marker — nhưng chỉ có vài MarkerStyle object tồn tại
var markers = new List<Marker>();
foreach (var poi in poisFromDb) // 1M row
{
    var style = MarkerStyleFactory.Get(poi.Category.Icon, poi.Category.Color, 32);
    markers.Add(new Marker(poi.Lat, poi.Lng, style));
}
```

**Xuất hiện trong codebase thực:** `string.Intern`, tránh boxing `Enum`, cache font/icon trong UI framework, `StringPool` trong `Microsoft.Extensions.ObjectPool`.

---

## 7. Proxy

**Ý định:** Cung cấp placeholder hoặc surrogate cho object khác để kiểm soát truy cập.

**Biến thể thực tế:**

* **Virtual Proxy** — lazy / deferred loading.
* **Protection Proxy** — kiểm tra authorization.
* **Remote Proxy** — đại diện local cho service remote (gRPC client, Refit).
* **Caching Proxy** — xem cả Decorator.

### Sample thực tế — authorization proxy quanh service nhạy cảm

```csharp
public interface IUserService
{
    Task<User> GetAsync(Guid userId, CancellationToken ct);
    Task DeleteAsync(Guid userId, CancellationToken ct);
}

public sealed class UserService : IUserService { /* implementation EF Core thật */ }

public sealed class AuthorizedUserService : IUserService
{
    private readonly IUserService _inner;
    private readonly ICurrentUser _current;

    public AuthorizedUserService(IUserService inner, ICurrentUser current)
    { _inner = inner; _current = current; }

    public Task<User> GetAsync(Guid userId, CancellationToken ct)
    {
        // User có thể đọc profile của mình; admin có thể đọc của bất kỳ ai.
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

### Built-in proxy trong .NET

* **EF Core lazy-loading proxy** (`UseLazyLoadingProxies()`) — `Order.Customer` là proxy generated lazily load customer.
* **gRPC** và **Refit** generate **remote proxy** tại compile time — object C# local mà method gọi HTTP.
* **Castle DynamicProxy** — engine sau Moq, NSubstitute, AutoFac interceptor.

**Decorator vs. Proxy:** giống nhau về cơ chế (bọc object cùng interface). **Intent** khác:

* Decorator → **thêm** behavior mà client muốn (logging, caching).
* Proxy → **kiểm soát** truy cập (auth, lazy load, remote call) — client có thể không biết.

---

## Bảng tổng kết

| Pattern    | Ý định một dòng                          | Cách dùng phổ biến nhất trong .NET                    |
| ---------- | ---------------------------------------- | ----------------------------------------------------- |
| Adapter    | Dịch một interface sang interface khác    | Bọc SDK third-party sau interface domain              |
| Bridge     | Vary abstraction & implementation        | Notification đa kênh / report engine                  |
| Composite  | Treat cây đồng nhất                       | Syntax tree, UI component, cây folder/permission      |
| Decorator  | Thêm behavior runtime                    | Caching/logging/retry quanh service; ASP.NET middleware |
| Facade     | Đơn giản hoá subsystem phức tạp          | Service "use-case" điều phối nhiều dependency         |
| Flyweight  | Share intrinsic state                    | Pool icon/font/style, `string.Intern`                 |
| Proxy      | Kiểm soát truy cập object                | EF lazy loading, client gRPC/Refit, wrapper authorization |
