# Creational Design Pattern trong .NET

> 5 GoF pattern liên quan đến **tạo object** — abstract *cách* instantiate để code client không phụ thuộc vào type cụ thể. Mỗi pattern dưới đây có **scenario thực tế** bạn sẽ gặp trong production .NET.

> 🇻🇳 Phiên bản tiếng Việt. English: [`creational.md`](./creational.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — 5 GoF creational pattern: **Singleton**, **Factory Method**, **Abstract Factory**, **Builder**, **Prototype** — liên quan đến cách object được construct.
- **Tại sao** — `new SomeClass()` ngây thơ couple client với type cụ thể. Creational pattern abstract instantiation để type đúng có thể chọn runtime, construction phức tạp có thể ẩn, và single instance có thể enforce.
- **Khi nào** — Construction involve runtime decision (subtype nào?), nhiều parameter tuỳ chọn, setup đắt, hoặc gia đình type liên quan phải stay consistent.
- **Ở đâu** — Mức class. Trong .NET hiện đại, DI container đã implement Singleton + Factory; BCL cung cấp type `*Builder` (`StringBuilder`, `WebApplicationBuilder`, EF model builder); `record` + `with` cho Prototype miễn phí.

---

## Mục lục

1. [Singleton](#1-singleton)
2. [Factory Method](#2-factory-method)
3. [Abstract Factory](#3-abstract-factory)
4. [Builder](#4-builder)
5. [Prototype](#5-prototype)

---

## 1. Singleton

**Ý định:** Đảm bảo một class chỉ có một instance và cung cấp single global access point.

**Khi nào dùng:**

* Resource truly global, đắt khi tạo (configuration, logging sink, in-memory cache).
* Khi có nhiều hơn một instance sẽ gây behavior sai.

**Khi nào KHÔNG dùng:**

* Như biến global tiện lợi. Nên dùng **DI với `AddSingleton`** — cùng lifetime, không global state, fully testable.

### Sample — thread-safe lazy singleton

```csharp
public sealed class AppConfig
{
    private static readonly Lazy<AppConfig> _instance =
        new(() => new AppConfig(), LazyThreadSafetyMode.ExecutionAndPublication);

    public static AppConfig Instance => _instance.Value;

    public string Environment { get; }

    private AppConfig()
    {
        Environment = System.Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
                      ?? "Production";
    }
}

// Usage
Console.WriteLine(AppConfig.Instance.Environment);
```

### Idiomatic .NET (ưu tiên)

```csharp
// Program.cs
builder.Services.AddSingleton<IAppConfig, AppConfig>();

// Inject ở đâu cũng được — cùng lifetime, testable, không static state.
public class HomeController(IAppConfig config) { /* ... */ }
```

---

## 2. Factory Method

**Ý định:** Định nghĩa interface để tạo object, nhưng cho subclass quyết định class cụ thể nào sẽ instantiate.

**Khi nào dùng:**

* Cần tạo object, nhưng type chính xác phụ thuộc data runtime hoặc subclass.
* Bạn muốn tập trung và đặt tên cho logic tạo thay vì rải `new` khắp nơi.

### Sample — payment processor

```csharp
public interface IPaymentProcessor
{
    Task ChargeAsync(decimal amount);
}

public sealed class StripeProcessor : IPaymentProcessor
{
    public Task ChargeAsync(decimal amount) { /* ... */ return Task.CompletedTask; }
}

public sealed class PaypalProcessor : IPaymentProcessor
{
    public Task ChargeAsync(decimal amount) { /* ... */ return Task.CompletedTask; }
}

public abstract class PaymentService
{
    protected abstract IPaymentProcessor CreateProcessor(); // factory method

    public Task PayAsync(decimal amount) => CreateProcessor().ChargeAsync(amount);
}

public sealed class StripePaymentService : PaymentService
{
    protected override IPaymentProcessor CreateProcessor() => new StripeProcessor();
}

public sealed class PaypalPaymentService : PaymentService
{
    protected override IPaymentProcessor CreateProcessor() => new PaypalProcessor();
}
```

### .NET hiện đại — DI keyed service (.NET 8+)

```csharp
builder.Services.AddKeyedScoped<IPaymentProcessor, StripeProcessor>("stripe");
builder.Services.AddKeyedScoped<IPaymentProcessor, PaypalProcessor>("paypal");

public class CheckoutHandler(IServiceProvider sp)
{
    public Task Pay(string gateway, decimal amount) =>
        sp.GetRequiredKeyedService<IPaymentProcessor>(gateway).ChargeAsync(amount);
}
```

---

## 3. Abstract Factory

**Ý định:** Tạo **gia đình** object liên quan hoặc phụ thuộc nhau mà không chỉ định class cụ thể.

**Khi nào dùng:**

* Bạn có nhiều product phải dùng cùng nhau (vd. UI control cho theme: Button đen + TextBox đen + Menu đen).
* Bạn muốn đổi cả gia đình cùng lúc.

### Sample — UI theme factory

```csharp
public interface IButton { void Render(); }
public interface ICheckbox { void Render(); }

public sealed class DarkButton : IButton    { public void Render() => Console.WriteLine("Dark button"); }
public sealed class DarkCheckbox : ICheckbox { public void Render() => Console.WriteLine("Dark checkbox"); }

public sealed class LightButton : IButton    { public void Render() => Console.WriteLine("Light button"); }
public sealed class LightCheckbox : ICheckbox { public void Render() => Console.WriteLine("Light checkbox"); }

public interface IUiFactory
{
    IButton CreateButton();
    ICheckbox CreateCheckbox();
}

public sealed class DarkUiFactory : IUiFactory
{
    public IButton CreateButton()     => new DarkButton();
    public ICheckbox CreateCheckbox() => new DarkCheckbox();
}

public sealed class LightUiFactory : IUiFactory
{
    public IButton CreateButton()     => new LightButton();
    public ICheckbox CreateCheckbox() => new LightCheckbox();
}

// Code client không biết theme
public void RenderForm(IUiFactory ui)
{
    ui.CreateButton().Render();
    ui.CreateCheckbox().Render();
}
```

**Factory Method vs. Abstract Factory:**

* Factory Method tạo **một** type product.
* Abstract Factory tạo **gia đình** product liên quan cùng nhau.

---

## 4. Builder

**Ý định:** Tách construction của object phức tạp khỏi representation của nó để cùng process có thể tạo nhiều representation.

**Khi nào dùng:**

* Object có nhiều parameter tuỳ chọn (tránh telescoping constructor).
* Construction từng bước mà thứ tự quan trọng.
* Build object immutable theo kiểu fluent.

### Sample — HTTP request builder fluent

```csharp
public sealed class HttpRequestBuilder
{
    private HttpMethod _method = HttpMethod.Get;
    private string? _url;
    private readonly Dictionary<string, string> _headers = new();
    private HttpContent? _content;

    public HttpRequestBuilder Method(HttpMethod m)         { _method = m; return this; }
    public HttpRequestBuilder Url(string url)              { _url = url; return this; }
    public HttpRequestBuilder Header(string k, string v)   { _headers[k] = v; return this; }
    public HttpRequestBuilder JsonBody(object body)
    {
        _content = JsonContent.Create(body);
        return this;
    }

    public HttpRequestMessage Build()
    {
        var req = new HttpRequestMessage(_method, _url) { Content = _content };
        foreach (var (k, v) in _headers) req.Headers.Add(k, v);
        return req;
    }
}

// Usage
var request = new HttpRequestBuilder()
    .Method(HttpMethod.Post)
    .Url("https://api.example.com/users")
    .Header("Authorization", "Bearer abc")
    .JsonBody(new { name = "Quan" })
    .Build();
```

### Ví dụ built-in trong .NET

* `StringBuilder`
* `HostBuilder`, `WebApplicationBuilder`
* `EntityTypeBuilder<T>` trong EF Core
* `JsonSerializerOptions` (de-facto builder qua initializer)

---

## 5. Prototype

**Ý định:** Tạo object mới bằng **clone** một object có sẵn thay vì instantiate từ đầu.

**Khi nào dùng:**

* Tạo object đắt (query phức tạp, parsing, network call).
* Cần nhiều object tương tự với khác biệt nhỏ.
* Tránh couple với constructor cụ thể.

### Sample — deep clone object configuration

```csharp
public sealed class ReportTemplate : ICloneable
{
    public string Title { get; set; } = "";
    public List<string> Columns { get; set; } = new();
    public Dictionary<string, string> Filters { get; set; } = new();

    public ReportTemplate Clone()
    {
        return new ReportTemplate
        {
            Title   = Title,
            Columns = new List<string>(Columns),
            Filters = new Dictionary<string, string>(Filters),
        };
    }

    object ICloneable.Clone() => Clone();
}

// Usage
var baseTpl = new ReportTemplate
{
    Title = "Sales",
    Columns = { "Date", "Amount", "Region" },
    Filters = { ["Region"] = "APAC" }
};

var q2Tpl = baseTpl.Clone();
q2Tpl.Title = "Sales Q2";
q2Tpl.Filters["Quarter"] = "Q2"; // KHÔNG mutate baseTpl
```

### Phương án C# hiện đại — `record` với `with` expression

```csharp
public record ReportTemplate(string Title, IReadOnlyList<string> Columns);

var baseTpl = new ReportTemplate("Sales", new[] { "Date", "Amount" });
var q2Tpl   = baseTpl with { Title = "Sales Q2" }; // shallow clone, built-in
```

`with` là implementation Prototype shallow do language C# cung cấp.

---

## Bảng tổng kết

| Pattern          | Dùng khi…                                            | Thực tế trong .NET                          |
| ---------------- | ---------------------------------------------------- | ------------------------------------------- |
| Singleton        | Cần đúng một instance                                | `services.AddSingleton<T>()`                |
| Factory Method   | Subclass/runtime data chọn type cụ thể                | Keyed service, `IHttpClientFactory`         |
| Abstract Factory | Đổi cả *gia đình* product liên quan                   | Abstraction theme/provider                  |
| Builder          | Construction nhiều bước phức tạp                      | Type `*Builder` khắp BCL/ASP.NET            |
| Prototype        | Clone rẻ hơn rebuild                                  | `record` + `with`; `ICloneable`             |
