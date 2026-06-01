# Service Lifetimes & Scopes — The Complete Guide

> The most common source of subtle bugs in DI-based applications: choosing the **wrong lifetime**. This file explains **Transient**, **Scoped**, and **Singleton** in depth, what a "scope" actually is, the **captive dependency** trap, disposal rules, thread safety, and shows a multi-lifetime real-project example you can copy. Examples in **C# / ASP.NET Core** — Java/Spring and Node/Nest have the same concepts under different names.

---

## Table of Contents

1. [What Is a Service Lifetime?](#1-what-is-a-service-lifetime)
2. [The Three Lifetimes](#2-the-three-lifetimes)
   - [Transient](#21-transient)
   - [Scoped](#22-scoped)
   - [Singleton](#23-singleton)
3. [What Is a "Scope"?](#3-what-is-a-scope)
4. [The Captive Dependency Problem](#4-the-captive-dependency-problem)
5. [Disposal Rules](#5-disposal-rules)
6. [Thread-Safety Implications](#6-thread-safety-implications)
7. [Real Project — Multi-Lifetime E-Commerce Walkthrough](#7-real-project--multi-lifetime-e-commerce-walkthrough)
8. [Key Points Cheat Sheet](#8-key-points-cheat-sheet)
9. [Common Pitfalls & How to Detect Them](#9-common-pitfalls--how-to-detect-them)
10. [Diagnostics — Catching Bugs Early](#10-diagnostics--catching-bugs-early)
11. [Cross-Stack Comparison](#11-cross-stack-comparison)

---

## Quick Reference (What · Why · When · Where)

- **What** — The three lifetimes the DI container can give a service: **Transient** (new per resolution), **Scoped** (new per HTTP request / unit of work), **Singleton** (one for the app's lifetime). Plus the **Captive Dependency** trap and disposal rules.
- **Why** — Wrong lifetime is the single most common DI bug. Holding a per-request `DbContext` in a singleton corrupts state across requests; making a stateful service singleton crashes under concurrency.
- **When** — Every time you register a service. Default to **Scoped** for application logic, **Singleton** for stateless/thread-safe helpers, **Transient** for lightweight stateless helpers.
- **Where** — Composition root (`Program.cs`). Pair with `ioc-di.md` (the broader DI story) and `data-access.md` (why `DbContext` is Scoped).

---

## 1. What Is a Service Lifetime?

A **service lifetime** tells the DI container **how long** an instance lives and **how often** it is created. It's the single answer to two questions:

* When the container is asked for `IFoo`, should it **build a new one** or **return an existing one**?
* When the instance is no longer needed, **when** does the container dispose it?

Choosing the wrong lifetime causes:

* **Memory leaks** (holding a per-request thing forever).
* **Stale data** (caching a per-request thing in a singleton).
* **Thread bugs** (sharing a non-thread-safe service across requests).
* **Disposed-object exceptions** (a singleton holding a disposed scoped dep).

> Lifetime is **not** "how fast can I make this". It's **"who shares this instance with whom"**.

---

## 2. The Three Lifetimes

ASP.NET Core / `Microsoft.Extensions.DependencyInjection` defines three lifetimes:

| Lifetime    | One instance per…                  |
| ----------- | ---------------------------------- |
| **Transient** | Each resolution (every `Resolve<T>`) |
| **Scoped**    | Each scope (HTTP request, by default) |
| **Singleton** | Application lifetime               |

### 2.1 Transient

> A **brand-new** instance every time it's resolved or injected.

```csharp
services.AddTransient<IGuidGenerator, GuidGenerator>();
```

* **Created**: every time the container is asked.
* **Disposed**: when the **scope that resolved it** ends. (Important — see [§5 Disposal Rules](#5-disposal-rules).)
* **State sharing**: never.

#### When to use

* Lightweight, **stateless** helpers (formatters, validators).
* Cheap to construct.
* You want **a fresh instance every call**.

#### When NOT to use

* The class is **expensive** to build.
* The class is **`IDisposable` and holds unmanaged resources** — every transient stays alive until the parent scope ends, potentially exhausting connections/file handles.
* You actually want **shared per-request state** (use Scoped instead).

#### Example

```csharp
public class CorrelationIdGenerator
{
    public Guid New() => Guid.NewGuid();
}

services.AddTransient<CorrelationIdGenerator>();
```

Two consumers in the same request → two instances. Stateless, doesn't matter.

---

### 2.2 Scoped

> **One** instance **per scope** — by default, **per HTTP request**.

```csharp
services.AddScoped<IOrderRepository, EfOrderRepository>();
```

* **Created**: the first time the scope asks for it.
* **Reused**: by every dependency within the same scope.
* **Disposed**: when the scope ends.
* **State sharing**: within one request, yes; across requests, no.

#### When to use

* **`DbContext`** and anything wrapping it.
* **Per-request state** (current user, correlation context, unit of work).
* Services that **track changes** for a single business operation.

#### Why `DbContext` Must Be Scoped

* It is **not thread-safe** — one DbContext per concurrent operation.
* It **caches** loaded entities (identity map) — those caches should be **discarded** between requests.
* It holds an **open DB connection** — leaking it across requests destroys pooling.

`AddDbContext<T>` registers it as Scoped automatically.

#### Example

```csharp
public class CurrentUserContext
{
    public Guid? UserId { get; set; }
}

services.AddScoped<CurrentUserContext>();
```

Set once early in the request (in middleware), read by every service downstream — without passing it through every method.

---

### 2.3 Singleton

> **One** instance for the **entire application lifetime**.

```csharp
services.AddSingleton<IClock, SystemClock>();
```

* **Created**: the first time it's resolved (or eagerly, if you call `GetService<T>()` at startup).
* **Reused**: by every consumer, forever.
* **Disposed**: when the application shuts down.
* **State sharing**: by every request, every thread, everywhere.

#### When to use

* **Truly stateless** services (e.g., a pure calculator).
* **In-memory caches** with their own thread-safety.
* **Configuration** (`IOptions<T>` is singleton by default).
* **Resource-pooling clients** like `HttpClient` (preferred via `IHttpClientFactory`).
* **Logging factories**, **metrics collectors**.
* **Clocks**, **system info**.

#### When NOT to use

* The service depends on a **scoped** service → captive dependency.
* The service holds **per-request** or **per-user** state.
* The service is **not thread-safe**.
* The service holds a connection that should be returned to a pool per request.

#### Example — Safe Singleton

```csharp
public interface IClock { DateTime UtcNow { get; } }
public class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }

services.AddSingleton<IClock, SystemClock>();
```

Stateless, thread-safe, no dependencies — a perfect singleton.

---

## 3. What Is a "Scope"?

A scope is **a unit of work**. In ASP.NET Core, the **framework creates a new scope for every incoming HTTP request**, then disposes it when the request finishes.

```
HTTP request arrives
   │
   ├── New IServiceScope created
   │      ├── Scoped services resolved on demand within this scope
   │      ├── Transient services resolved by this scope are also tracked
   │      └── Singletons looked up from the root provider
   │
   └── Request finishes → scope disposed → scoped + transient services disposed
```

### 3.1 You Can Create Scopes Manually

For **background workers**, **scheduled jobs**, or **CLIs**, there's no incoming request — so you must create your own scope:

```csharp
public class ImportWorker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopeFactory.CreateScope();
            var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
            var uow  = scope.ServiceProvider.GetRequiredService<ShopDbContext>();

            await ProcessBatchAsync(repo, uow, ct);
            // scope disposed here → DbContext disposed, EF connection returned
        }
    }
}
```

> **Without this, a long-running `BackgroundService` would hold one `DbContext` forever.**

### 3.2 The Root Scope

The container itself is the **root scope**. Singletons live here. Scoped services should **never** be resolved from the root — only from a child scope. `ValidateScopes = true` (default in Development) will throw if you do.

---

## 4. The Captive Dependency Problem

> A **long-lived** service capturing a **short-lived** service in a field.

The single most-cited DI bug.

### 4.1 The Bug

```csharp
// WRONG
services.AddSingleton<IOrderProcessor, OrderProcessor>();   // singleton
services.AddScoped<IOrderRepository, EfOrderRepository>(); // scoped

public class OrderProcessor(IOrderRepository repo) : IOrderProcessor
{
    // _repo is captured here, FOREVER — even after the request that produced it ended.
}
```

What happens:

* Request 1: `OrderProcessor` is built. It grabs the request's `EfOrderRepository` (and its `DbContext`).
* Request 1 ends → the framework disposes the `DbContext`.
* Request 2 arrives → reuses the same `OrderProcessor` → still pointing at the **disposed** `DbContext`.
* You get `ObjectDisposedException` or, worse, **stale data and concurrency bugs**.

### 4.2 The Rule

> A service can only depend on services **of the same or longer lifetime**.

| Has lifetime ↓ / depends on → | Singleton | Scoped | Transient   |
| ----------------------------- | --------- | ------ | ----------- |
| **Singleton**                 | ✅         | ❌      | ⚠️ (capture) |
| **Scoped**                    | ✅         | ✅      | ✅           |
| **Transient**                 | ✅         | ✅      | ✅           |

* Singleton + Transient: technically allowed, but the transient gets **captured for the app's life** — usually wrong.
* Singleton + Scoped: **never** allowed. ASP.NET Core throws in dev mode.

### 4.3 The Fix — `IServiceScopeFactory`

If a singleton truly needs a scoped service per operation, inject `IServiceScopeFactory` instead and create a scope on demand:

```csharp
public class OrderProcessor(IServiceScopeFactory scopeFactory) : IOrderProcessor
{
    public async Task ProcessAsync(Guid orderId, CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        await repo.ProcessAsync(orderId, ct);
        // DbContext disposed here, next call gets a fresh one
    }
}
```

---

## 5. Disposal Rules

The container **owns** disposal for the things it builds. Rules:

1. **Singletons** are disposed when the **root container** is disposed (app shutdown).
2. **Scoped services** are disposed when the **scope** is disposed.
3. **Transient services** are disposed when the **scope that resolved them** is disposed.
4. The container only disposes services that **implement `IDisposable`** (and `IAsyncDisposable`).
5. The container does **NOT** dispose instances you provide via `AddSingleton<T>(instance)` — you own that lifecycle.

### 5.1 The Transient Disposal Gotcha

```csharp
services.AddTransient<HeavyConnection>();   // disposable

public class Reporter(IServiceProvider sp)
{
    public void RunForEachItem()
    {
        for (var i = 0; i < 10_000; i++)
        {
            var conn = sp.GetRequiredService<HeavyConnection>();
            // ... use conn ...
            // conn is NOT disposed here! It will be disposed only when the SCOPE ends.
        }
    }
}
```

10,000 connections held until the request ends. If `Reporter` is itself a singleton, that's the **application** lifetime.

**Fix:** create a child scope per iteration, or take an explicit `IDisposable` lifetime in your own code.

---

## 6. Thread-Safety Implications

The lifetime decides who can hit the service in parallel.

| Lifetime    | Concurrent callers per instance              | Must be thread-safe?      |
| ----------- | -------------------------------------------- | ------------------------- |
| **Transient** | One (per resolution)                       | No                        |
| **Scoped**    | One request thread (usually)               | Usually no                |
| **Singleton** | Every thread in the application            | **Yes** — strictly        |

### 6.1 Singletons Are Shared Across Threads

```csharp
public class BadCounter
{
    public int Value;                          // not thread-safe
    public void Increment() => Value++;
}

services.AddSingleton<BadCounter>();
```

Two requests calling `Increment()` simultaneously will lose updates. Use `Interlocked`, locks, or a concurrent collection.

### 6.2 Scoped Services and `async`

A single request can fan out across threads with `Task.WhenAll`. If your scoped service is stateful, **don't share it across parallel tasks** in the same request — the `DbContext` will throw, and any mutable state will race.

```csharp
// BAD
await Task.WhenAll(
    db.Orders.Where(...).ToListAsync(),
    db.Customers.Where(...).ToListAsync()
);
// DbContext is not thread-safe → InvalidOperationException
```

Run them sequentially, or `await db.Orders.ToListAsync()` then `await db.Customers.ToListAsync()`.

---

## 7. Real Project — Multi-Lifetime E-Commerce Walkthrough

A realistic checkout module mixing all three lifetimes correctly.

### 7.1 The Service Registry

```csharp
// Program.cs

var b = WebApplication.CreateBuilder(args);

// ─── Singletons (shared, thread-safe, stateless or carefully synchronized) ──
b.Services.AddSingleton<IClock, SystemClock>();
b.Services.AddSingleton<IExchangeRateCache, InMemoryExchangeRateCache>();
b.Services.AddSingleton<IMetrics, PrometheusMetrics>();
b.Services
   .AddOptions<StripeOptions>()
   .Bind(b.Configuration.GetSection("Stripe"))
   .ValidateDataAnnotations()
   .ValidateOnStart();

// HttpClientFactory manages pooled HTTP connections (singleton internally)
b.Services.AddHttpClient<StripePaymentGateway>(c =>
{
    c.BaseAddress = new Uri("https://api.stripe.com");
    c.Timeout     = TimeSpan.FromSeconds(15);
});

// ─── Scoped (per HTTP request — most application logic lives here) ───────
b.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer(b.Configuration.GetConnectionString("Db")));
b.Services.AddScoped<ICustomerRepository, EfCustomerRepository>();
b.Services.AddScoped<IOrderRepository,    EfOrderRepository>();
b.Services.AddScoped<IProductRepository,  EfProductRepository>();
b.Services.AddScoped<IPaymentGateway,     StripePaymentGateway>();
b.Services.AddScoped<IEmailSender,        SendGridEmailSender>();
b.Services.AddScoped<CurrentUserContext>();   // set in middleware, read by services
b.Services.AddScoped<CheckoutUseCase>();

// ─── Transient (lightweight stateless helpers) ───────────────────────────
b.Services.AddTransient<IOrderValidator,    OrderValidator>();
b.Services.AddTransient<IInvoiceFormatter,  HtmlInvoiceFormatter>();
b.Services.AddTransient<IGuidGenerator,     GuidGenerator>();

// ─── Background worker (creates its own scope per iteration) ─────────────
b.Services.AddHostedService<NightlyReconciliationWorker>();

var app = b.Build();
```

### 7.2 The Application Code

```csharp
// SINGLETON — pure, stateless, thread-safe
public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

public class InMemoryExchangeRateCache : IExchangeRateCache
{
    private readonly ConcurrentDictionary<string, decimal> _rates = new();
    public decimal Get(string ccy) => _rates.GetValueOrDefault(ccy, 1m);
    public void Set(string ccy, decimal rate) => _rates[ccy] = rate;
}

// SCOPED — holds per-request state
public class CurrentUserContext
{
    public Guid? UserId { get; set; }
    public string? Email { get; set; }
}

// SCOPED — orchestrates one request. Notice: depends on scoped + transient + singleton freely.
public class CheckoutUseCase(
    ICustomerRepository       customers,   // scoped
    IOrderRepository          orders,      // scoped
    IPaymentGateway           payment,     // scoped
    IEmailSender              email,       // scoped
    IOrderValidator           validator,   // transient
    IClock                    clock,       // singleton ← OK (longer lifetime)
    IMetrics                  metrics,     // singleton ← OK
    CurrentUserContext        user,        // scoped
    ShopDbContext             uow,         // scoped (Unit of Work)
    ILogger<CheckoutUseCase>  log)
{
    public async Task<PlaceOrderResult> ExecuteAsync(PlaceOrderRequest req, CancellationToken ct)
    {
        validator.Validate(req);
        var customer = await customers.GetAsync(user.UserId!.Value, ct) ?? throw new();
        var order = Order.Start(customer.Id, clock.UtcNow);
        // ... build, charge, save ...
        await orders.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);
        metrics.IncrementOrdersPlaced();
        return new PlaceOrderResult(order.Id);
    }
}

// MIDDLEWARE — fills the scoped CurrentUserContext per request
app.Use(async (ctx, next) =>
{
    var user = ctx.RequestServices.GetRequiredService<CurrentUserContext>();
    user.UserId = ctx.User.GetUserId();
    user.Email  = ctx.User.FindFirstValue(ClaimTypes.Email);
    await next();
});

// BACKGROUND SERVICE — creates its OWN scope per iteration
public class NightlyReconciliationWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<NightlyReconciliationWorker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            // Wait until 2 AM, then run one batch:
            await Task.Delay(TimeUntilNextRun(), ct);

            using var scope = scopeFactory.CreateScope();
            var orders = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
            var uow    = scope.ServiceProvider.GetRequiredService<ShopDbContext>();

            await ReconcileAsync(orders, uow, ct);
            // scope disposed → fresh DbContext next iteration
        }
    }
}
```

### 7.3 What Goes Where — And Why

| Service                       | Lifetime   | Why                                                          |
| ----------------------------- | ---------- | ------------------------------------------------------------ |
| `IClock`                      | Singleton  | Stateless, thread-safe, no deps.                             |
| `IExchangeRateCache`          | Singleton  | App-wide cache; internally thread-safe.                      |
| `StripeOptions`               | Singleton  | Configuration. Doesn't change at runtime.                    |
| `HttpClient` (via factory)    | Pooled     | `IHttpClientFactory` handles connection pooling.             |
| `ShopDbContext`               | Scoped     | Not thread-safe; identity map per request.                   |
| Repositories                  | Scoped     | Wrap `DbContext` → must match its lifetime.                  |
| `CheckoutUseCase`             | Scoped     | Orchestrates per-request work; injects per-request deps.     |
| `CurrentUserContext`          | Scoped     | Mutable per-request state.                                   |
| `IPaymentGateway`             | Scoped     | Holds correlation IDs, log scopes, retry state per request.  |
| `IOrderValidator`             | Transient  | Stateless; cheap; no shared state needed.                    |
| `IInvoiceFormatter`           | Transient  | Stateless utility.                                           |
| `NightlyReconciliationWorker` | Singleton (Hosted) | Lives for the app; creates its own scope per batch.  |

### 7.4 What Breaks If You Get It Wrong

| Mistake                                     | Symptom                                                                |
| ------------------------------------------- | ---------------------------------------------------------------------- |
| `CheckoutUseCase` as Singleton              | Stale `DbContext`, race conditions, `ObjectDisposedException`.         |
| `ShopDbContext` as Singleton                | Concurrent EF errors, leaked DB connections, identity map pollution.   |
| `CurrentUserContext` as Singleton           | Every request sees the **last** logged-in user. Severe security bug.   |
| `IClock` as Scoped                          | Wastes allocations, no functional bug.                                 |
| Background service injecting `DbContext` directly | DbContext lives forever; connection leak; concurrency bugs.       |
| `services.AddSingleton<HttpClient>()`       | Socket exhaustion or DNS staleness. Use `IHttpClientFactory`.          |
| Singleton depending on `IOrderRepository`   | Captive dependency; ASP.NET Core throws in Development.                |

---

## 8. Key Points Cheat Sheet

* **One instance per…**
  Transient → resolution. Scoped → request. Singleton → app.

* **The lifetime rule**
  A service may only depend on services of **equal or longer** lifetime.

* **`DbContext` is Scoped**. Never make it Singleton. Never inject it into a singleton.

* **Singletons must be thread-safe.** Period.

* **Background services are Singletons** but must create child scopes for scoped deps.

* **Configuration (`IOptions<T>`) is Singleton by default.** Use `IOptionsSnapshot<T>` (Scoped) if you need per-request reloading.

* **`HttpClient` is special.** Use `IHttpClientFactory` to avoid both socket exhaustion (one HttpClient per call) and DNS staleness (one shared singleton).

* **Don't resolve from the root provider.** Always resolve from a scope.

* **Transient + IDisposable is dangerous.** Each call accumulates undisposed instances until the parent scope ends.

* **When in doubt, default to Scoped.** Web applications usually want per-request semantics.

---

## 9. Common Pitfalls & How to Detect Them

| Pitfall                                                      | How to detect                                                                |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| Singleton holding Scoped dep                                 | `ValidateScopes = true` (default in Development) — throws at startup or resolution. |
| Singleton accumulating transient `IDisposable` instances     | Memory grows monotonically; counter via `dotnet-counters`.                   |
| Scoped service used across threads in one request            | `InvalidOperationException` from EF: *"A second operation was started…"*.    |
| Resolving from the root provider                             | `InvalidOperationException`: *"Cannot resolve scoped service from root provider"*. |
| Background service injecting `DbContext` directly            | DbContext is never reset; weird stale data; slow connection leaks.           |
| `CurrentUserContext` registered as Singleton                 | All requests see the same user — caught quickly by integration tests.        |
| Mutable Singleton without thread-safety                      | Hard-to-reproduce data corruption under load.                                |

---

## 10. Diagnostics — Catching Bugs Early

### 10.1 Validate Scopes at Startup

```csharp
var b = WebApplication.CreateBuilder(args);

b.Host.UseDefaultServiceProvider((ctx, opts) =>
{
    opts.ValidateScopes  = true;    // default in Development; force in all envs if you trust your tests
    opts.ValidateOnBuild = true;    // build every service at startup; fail fast on bad graph
});
```

* `ValidateScopes` — throws when a singleton tries to consume a scoped service.
* `ValidateOnBuild` — tries to construct every registered service at app start; surfaces graph errors before the first request.

### 10.2 Useful Tools

* **`dotnet-counters`** — watch GC, working set, scope count.
* **`dotnet-trace`** — measure DI resolution time if hot.
* **EF Core logging** — set the connection-string `Log Level=Information` to spot leaked DbContexts.
* **`Microsoft.Extensions.Diagnostics.HealthChecks`** — verify singletons (caches, clocks) are healthy.

### 10.3 Integration Tests for Lifetimes

Spin up the real DI graph in `WebApplicationFactory<T>` and assert at the seams:

```csharp
[Fact]
public void Service_graph_validates_at_build()
{
    using var factory = new WebApplicationFactory<Program>()
        .WithWebHostBuilder(b => b.UseSetting("Environment", "Development"));
    using var scope = factory.Services.CreateScope();
    // Resolve every type you care about; failure means graph is broken.
    _ = scope.ServiceProvider.GetRequiredService<CheckoutUseCase>();
}
```

---

## 11. Cross-Stack Comparison

The names change, the ideas don't.

| Concept            | .NET (MS DI)        | Spring / Java                | NestJS / TS         | Angular DI            |
| ------------------ | ------------------- | ---------------------------- | ------------------- | --------------------- |
| One per resolution | **Transient**       | `prototype` scope            | `TRANSIENT`         | Function provider     |
| One per request    | **Scoped**          | `request` scope              | `REQUEST`           | `Injector` per component |
| App-wide           | **Singleton**       | `singleton` (default)        | `DEFAULT` (singleton)| Root injector        |
| Session / user     | (build your own)    | `session` scope              | (build your own)    | (build your own)      |

> **Default in most stacks: Singleton.** ASP.NET Core deliberately makes you choose, which catches more bugs.

---

## 12. TL;DR

> A service lifetime is **a contract about sharing**, not a performance knob.
> **Singleton = shared everywhere, must be thread-safe.**
> **Scoped = shared within one request, the everyday default for web apps.**
> **Transient = brand new each time, watch out for `IDisposable`.**
> **Never let a long-lived service capture a short-lived one** — create a scope on demand instead.

If you remember just one rule:

> *"A service can only depend on services of the **same or longer** lifetime."*

---

## See Also

* [`ioc-di.md`](./ioc-di.md) — the broader DI / IoC story.
* [`data-access.md`](./data-access.md) — why `DbContext` (Scoped + Unit of Work) is the canonical example.
* [`../../../principles/solid.md`](../../../principles/solid.md) — DIP, the principle behind all of this.
* [`../../../oop/real-project.md`](../../../oop/real-project.md) — same e-commerce model, end-to-end.
