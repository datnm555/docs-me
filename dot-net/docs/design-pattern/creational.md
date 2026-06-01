# Creational Design Patterns in .NET

> The 5 GoF patterns concerned with **object creation** — abstracting the *how* of instantiation so client code stays decoupled from concrete types. Each pattern below is shown with a **real-world scenario** you would actually meet in production .NET code.

---

## Quick Reference (What · Why · When · Where)

- **What** — The 5 GoF creational patterns: **Singleton**, **Factory Method**, **Abstract Factory**, **Builder**, **Prototype** — concerned with how objects are constructed.
- **Why** — Naïve `new SomeClass()` couples clients to concrete types. Creational patterns abstract instantiation so the right type can be chosen at runtime, complex construction can be hidden, and a single instance can be enforced.
- **When** — Construction involves runtime decisions (which subtype?), many optional parameters, expensive setup, or families of related types that must stay consistent.
- **Where** — Class level. In modern .NET, the DI container already implements Singleton + Factory; the BCL provides `*Builder` types (`StringBuilder`, `WebApplicationBuilder`, EF model builder); `record` + `with` gives Prototype for free.

---

## Table of Contents

1. [Singleton](#1-singleton)
2. [Factory Method](#2-factory-method)
3. [Abstract Factory](#3-abstract-factory)
4. [Builder](#4-builder)
5. [Prototype](#5-prototype)

---

## 1. Singleton

**Intent:** Ensure a class has only one instance and provide a single global access point to it.

**When to use:**

* Truly global, expensive-to-create resources (configuration, logging sinks, in-memory caches).
* When more than one instance would cause incorrect behavior.

**When NOT to use:**

* As a convenient global variable. Prefer **DI with `AddSingleton`** — same lifetime, no global state, fully testable.

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

### Idiomatic .NET equivalent (preferred)

```csharp
// Program.cs
builder.Services.AddSingleton<IAppConfig, AppConfig>();

// Inject anywhere — same lifetime, testable, no static state.
public class HomeController(IAppConfig config) { /* ... */ }
```

---

## 2. Factory Method

**Intent:** Define an interface for creating an object, but let subclasses decide which concrete class to instantiate.

**When to use:**

* You need to create objects, but the exact type depends on runtime data or subclass.
* You want to centralize and name the creation logic instead of scattering `new`.

### Sample — payment processors

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

### Modern .NET variant — DI keyed services (.NET 8+)

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

**Intent:** Create **families** of related or dependent objects without specifying their concrete classes.

**When to use:**

* You have several products that must be used together (e.g. UI controls for a theme: dark Button + dark TextBox + dark Menu).
* You want to swap an entire family at once.

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

// Client code is theme-agnostic
public void RenderForm(IUiFactory ui)
{
    ui.CreateButton().Render();
    ui.CreateCheckbox().Render();
}
```

**Factory Method vs. Abstract Factory:**

* Factory Method produces **one** product type.
* Abstract Factory produces a **family** of related products together.

---

## 4. Builder

**Intent:** Separate the construction of a complex object from its representation so that the same construction process can create different representations.

**When to use:**

* Objects with many optional parameters (avoids telescoping constructors).
* Step-by-step construction where order matters.
* Building immutable objects fluently.

### Sample — fluent HTTP request builder

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

### Built-in .NET examples

* `StringBuilder`
* `HostBuilder`, `WebApplicationBuilder`
* `EntityTypeBuilder<T>` in EF Core
* `JsonSerializerOptions` (de-facto builder via initializers)

---

## 5. Prototype

**Intent:** Create new objects by **cloning** an existing one rather than instantiating from scratch.

**When to use:**

* Object creation is expensive (deep queries, parsing, network calls).
* You need many similar objects with small variations.
* You want to avoid coupling to concrete constructors.

### Sample — deep clone of a configuration object

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
q2Tpl.Filters["Quarter"] = "Q2"; // does NOT mutate baseTpl
```

### Modern C# alternative — `record` with `with` expression

```csharp
public record ReportTemplate(string Title, IReadOnlyList<string> Columns);

var baseTpl = new ReportTemplate("Sales", new[] { "Date", "Amount" });
var q2Tpl   = baseTpl with { Title = "Sales Q2" }; // shallow clone, built-in
```

`with` is a shallow Prototype implementation provided by the C# language.

---

## Summary Table

| Pattern          | Use when…                                           | .NET reality                                |
| ---------------- | --------------------------------------------------- | ------------------------------------------- |
| Singleton        | Exactly one instance is required                    | `services.AddSingleton<T>()`                |
| Factory Method   | Subclass/runtime data picks the concrete type        | Keyed services, `IHttpClientFactory`        |
| Abstract Factory | Swap a whole *family* of related products            | Theme/provider abstractions                 |
| Builder          | Complex multi-step construction                      | `*Builder` types everywhere in BCL/ASP.NET  |
| Prototype        | Cloning is cheaper than rebuilding                   | `record` + `with`; `ICloneable`             |
