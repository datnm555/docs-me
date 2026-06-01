# Service Lifetime & Scope — Hướng dẫn đầy đủ

> Nguồn bug tinh tế phổ biến nhất trong application dùng DI: chọn **sai lifetime**. File này giải thích **Transient**, **Scoped**, và **Singleton** sâu, "scope" thực sự là gì, bẫy **captive dependency**, quy tắc dispose, thread safety, và walkthrough dự án thật multi-lifetime. Ví dụ trong **C# / ASP.NET Core** — Java/Spring và Node/Nest có cùng khái niệm với tên khác.

> 🇻🇳 Phiên bản tiếng Việt. English: [`lifetimes.md`](./lifetimes.md)

---

## Mục lục

1. [Service Lifetime là gì?](#1-service-lifetime-là-gì)
2. [Ba Lifetime](#2-ba-lifetime)
3. [Scope là gì?](#3-scope-là-gì)
4. [Captive Dependency Problem](#4-captive-dependency-problem)
5. [Quy tắc Dispose](#5-quy-tắc-dispose)
6. [Implication Thread-Safety](#6-implication-thread-safety)
7. [Real Project — Walkthrough E-Commerce Multi-Lifetime](#7-real-project--walkthrough-e-commerce-multi-lifetime)
8. [Cheat Sheet Key Point](#8-cheat-sheet-key-point)
9. [Cạm bẫy phổ biến & cách phát hiện](#9-cạm-bẫy-phổ-biến--cách-phát-hiện)
10. [Diagnostic — Bắt bug sớm](#10-diagnostic--bắt-bug-sớm)
11. [So sánh Cross-Stack](#11-so-sánh-cross-stack)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Ba lifetime DI container có thể cho service: **Transient** (mới per resolution), **Scoped** (mới per HTTP request / unit of work), **Singleton** (một cho lifetime app). Cộng bẫy **Captive Dependency** và rule disposal.
- **Tại sao** — Lifetime sai là bug DI phổ biến nhất. Giữ `DbContext` per-request trong singleton corrupt state xuyên request; làm service stateful thành singleton crash dưới concurrency.
- **Khi nào** — Mỗi lần bạn register service. Default **Scoped** cho application logic, **Singleton** cho helper stateless/thread-safe, **Transient** cho helper nhẹ stateless.
- **Ở đâu** — Composition root (`Program.cs`). Pair với `ioc-di-vi.md` (câu chuyện DI rộng hơn) và `data-access-vi.md` (tại sao `DbContext` là Scoped).

---

## 1. Service Lifetime là gì?

**Service lifetime** bảo DI container instance sống **bao lâu** và được tạo **bao thường xuyên**. Trả lời hai câu:

* Khi container được hỏi cho `IFoo`, **build mới** hay **trả về cái có sẵn**?
* Khi instance không còn cần, **khi nào** container dispose nó?

Chọn sai lifetime gây:

* **Memory leak** (giữ thứ per-request mãi).
* **Data stale** (cache thứ per-request trong singleton).
* **Thread bug** (share service không thread-safe xuyên request).
* **Disposed-object exception** (singleton giữ scoped dep đã dispose).

> Lifetime **không phải** "nhanh ra sao". Là **"ai share instance này với ai"**.

---

## 2. Ba Lifetime

ASP.NET Core / `Microsoft.Extensions.DependencyInjection` định nghĩa ba:

| Lifetime    | Một instance per…                  |
| ----------- | ---------------------------------- |
| **Transient** | Mỗi resolution (mọi `Resolve<T>`) |
| **Scoped**    | Mỗi scope (HTTP request, mặc định) |
| **Singleton** | Lifetime application               |

### 2.1 Transient

> Instance **mới toanh** mỗi lần resolve hoặc inject.

```csharp
services.AddTransient<IGuidGenerator, GuidGenerator>();
```

* **Tạo**: mỗi lần container được hỏi.
* **Dispose**: khi **scope resolve nó** kết thúc. (Quan trọng — xem [§5](#5-quy-tắc-dispose).)
* **Share state**: không bao giờ.

**Dùng khi:** helper nhẹ, stateless (formatter, validator); rẻ construct; muốn instance mới mỗi call.

**KHÔNG dùng khi:** class đắt build; class `IDisposable` giữ unmanaged resource (mọi transient sống cho tới khi parent scope kết thúc); thực sự muốn shared per-request state (dùng Scoped).

### 2.2 Scoped

> **Một** instance **per scope** — mặc định, **per HTTP request**.

```csharp
services.AddScoped<IOrderRepository, EfOrderRepository>();
```

* **Tạo**: lần đầu scope hỏi.
* **Reuse**: bởi mọi dep trong cùng scope.
* **Dispose**: khi scope kết thúc.
* **Share state**: trong một request, có; xuyên request, không.

**Dùng khi:** `DbContext` và bất cứ thứ gì bọc nó; state per-request (current user, correlation, unit of work); service track changes cho một business operation.

**Tại sao `DbContext` phải Scoped:** không thread-safe; cache loaded entity (identity map) phải discard giữa request; giữ DB connection.

`AddDbContext<T>` register Scoped tự động.

### 2.3 Singleton

> **Một** instance cho **toàn bộ lifetime application**.

```csharp
services.AddSingleton<IClock, SystemClock>();
```

* **Tạo**: lần đầu resolve (hoặc eagerly nếu call `GetService<T>()` lúc startup).
* **Reuse**: bởi mọi consumer, mãi.
* **Dispose**: khi application shutdown.
* **Share state**: mọi request, mọi thread, khắp nơi.

**Dùng khi:** service thực sự stateless; in-memory cache với thread-safety riêng; configuration (`IOptions<T>` mặc định singleton); resource-pooling client như `HttpClient` (ưu tiên qua `IHttpClientFactory`); logging factory, metrics collector; clock, system info.

**KHÔNG dùng khi:** service phụ thuộc scoped (captive dependency); giữ state per-request hoặc per-user; không thread-safe; giữ connection nên trả về pool per request.

**Ví dụ — Singleton an toàn:**

```csharp
public interface IClock { DateTime UtcNow { get; } }
public class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }

services.AddSingleton<IClock, SystemClock>();
```

Stateless, thread-safe, không dependency — singleton hoàn hảo.

---

## 3. Scope là gì?

Scope là **unit of work**. Trong ASP.NET Core, **framework tạo scope mới cho mỗi HTTP request đến**, rồi dispose khi request kết thúc.

```
HTTP request đến
   │
   ├── IServiceScope mới được tạo
   │      ├── Scoped service resolve theo nhu cầu trong scope này
   │      ├── Transient service resolve bởi scope này cũng được track
   │      └── Singleton lookup từ root provider
   │
   └── Request kết thúc → scope dispose → scoped + transient service dispose
```

### 3.1 Có thể tạo scope thủ công

Cho **background worker**, **scheduled job**, hoặc **CLI**, không có request đến — phải tạo scope riêng:

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
            // scope dispose ở đây → DbContext dispose, connection EF trả về
        }
    }
}
```

> **Không có cái này, `BackgroundService` chạy lâu sẽ giữ một `DbContext` mãi.**

### 3.2 Root Scope

Bản thân container là **root scope**. Singleton sống ở đây. Scoped service **không bao giờ** nên resolve từ root — chỉ từ child scope. `ValidateScopes = true` (mặc định trong Development) sẽ throw nếu làm.

---

## 4. Captive Dependency Problem

> Service **sống lâu** capture service **sống ngắn** vào field.

Bug DI được nhắc nhiều nhất.

### 4.1 Bug

```csharp
// SAI
services.AddSingleton<IOrderProcessor, OrderProcessor>();   // singleton
services.AddScoped<IOrderRepository, EfOrderRepository>(); // scoped

public class OrderProcessor(IOrderRepository repo) : IOrderProcessor
{
    // _repo capture ở đây, MÃI MÃI — kể cả khi request tạo ra nó kết thúc.
}
```

Chuyện xảy ra:

* Request 1: `OrderProcessor` build. Chộp `EfOrderRepository` của request (và `DbContext` của nó).
* Request 1 kết thúc → framework dispose `DbContext`.
* Request 2 đến → reuse cùng `OrderProcessor` → vẫn trỏ vào `DbContext` **đã dispose**.
* Bạn nhận `ObjectDisposedException` hoặc tệ hơn, **data stale và concurrency bug**.

### 4.2 Quy tắc

> Service chỉ có thể phụ thuộc service **cùng hoặc lifetime dài hơn**.

| Có lifetime ↓ / phụ thuộc → | Singleton | Scoped | Transient   |
| ----------------------------- | --------- | ------ | ----------- |
| **Singleton**                 | ✅         | ❌      | ⚠️ (capture) |
| **Scoped**                    | ✅         | ✅      | ✅           |
| **Transient**                 | ✅         | ✅      | ✅           |

* Singleton + Transient: kỹ thuật cho phép, nhưng transient **bị capture cho life của app** — thường sai.
* Singleton + Scoped: **không bao giờ** cho phép. ASP.NET Core throw trong dev mode.

### 4.3 Fix — `IServiceScopeFactory`

Nếu singleton thực sự cần scoped service per operation, inject `IServiceScopeFactory` thay và tạo scope theo nhu cầu:

```csharp
public class OrderProcessor(IServiceScopeFactory scopeFactory) : IOrderProcessor
{
    public async Task ProcessAsync(Guid orderId, CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        await repo.ProcessAsync(orderId, ct);
        // DbContext dispose ở đây, call sau lấy cái mới
    }
}
```

---

## 5. Quy tắc Dispose

Container **sở hữu** dispose cho thứ nó build. Quy tắc:

1. **Singleton** dispose khi **root container** dispose (app shutdown).
2. **Scoped service** dispose khi **scope** dispose.
3. **Transient service** dispose khi **scope resolve nó** dispose.
4. Container chỉ dispose service **implement `IDisposable`** (và `IAsyncDisposable`).
5. Container **KHÔNG** dispose instance bạn cung cấp qua `AddSingleton<T>(instance)` — bạn sở hữu lifecycle đó.

### 5.1 Gotcha về Transient Disposal

```csharp
services.AddTransient<HeavyConnection>();   // disposable

public class Reporter(IServiceProvider sp)
{
    public void RunForEachItem()
    {
        for (var i = 0; i < 10_000; i++)
        {
            var conn = sp.GetRequiredService<HeavyConnection>();
            // ... dùng conn ...
            // conn KHÔNG dispose ở đây! Chỉ dispose khi SCOPE kết thúc.
        }
    }
}
```

10,000 connection giữ tới khi request kết thúc. Nếu `Reporter` bản thân là singleton, đó là lifetime **application**.

**Fix:** tạo child scope per iteration, hoặc lấy lifetime `IDisposable` explicit trong code của bạn.

---

## 6. Implication Thread-Safety

Lifetime quyết định ai có thể hit service song song.

| Lifetime    | Caller đồng thời per instance                | Phải thread-safe?         |
| ----------- | -------------------------------------------- | ------------------------- |
| **Transient** | Một (per resolution)                       | Không                     |
| **Scoped**    | Một thread request (thường)                | Thường không              |
| **Singleton** | Mọi thread trong application               | **Có** — nghiêm ngặt      |

### 6.1 Singleton được share giữa các thread

```csharp
public class BadCounter
{
    public int Value;                          // không thread-safe
    public void Increment() => Value++;
}

services.AddSingleton<BadCounter>();
```

Hai request call `Increment()` đồng thời sẽ mất update. Dùng `Interlocked`, lock, hoặc concurrent collection.

### 6.2 Scoped Service và `async`

Một request có thể fan out qua thread với `Task.WhenAll`. Nếu scoped service của bạn stateful, **đừng share xuyên parallel task** trong cùng request — `DbContext` sẽ throw, và mutable state nào sẽ race.

```csharp
// TỆ
await Task.WhenAll(
    db.Orders.Where(...).ToListAsync(),
    db.Customers.Where(...).ToListAsync()
);
// DbContext không thread-safe → InvalidOperationException
```

Chạy sequentially, hoặc `await db.Orders.ToListAsync()` rồi `await db.Customers.ToListAsync()`.

---

## 7. Real Project — Walkthrough E-Commerce Multi-Lifetime

Module checkout thực tế kết hợp đúng cả ba lifetime.

### 7.1 Service Registry

```csharp
// Program.cs

var b = WebApplication.CreateBuilder(args);

// ─── Singleton (share, thread-safe, stateless hoặc sync cẩn thận) ──
b.Services.AddSingleton<IClock, SystemClock>();
b.Services.AddSingleton<IExchangeRateCache, InMemoryExchangeRateCache>();
b.Services.AddSingleton<IMetrics, PrometheusMetrics>();
b.Services
   .AddOptions<StripeOptions>()
   .Bind(b.Configuration.GetSection("Stripe"))
   .ValidateDataAnnotations()
   .ValidateOnStart();

// HttpClientFactory quản lý pool HTTP connection (singleton bên trong)
b.Services.AddHttpClient<StripePaymentGateway>(c =>
{
    c.BaseAddress = new Uri("https://api.stripe.com");
    c.Timeout     = TimeSpan.FromSeconds(15);
});

// ─── Scoped (per HTTP request — phần lớn application logic ở đây) ───
b.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer(b.Configuration.GetConnectionString("Db")));
b.Services.AddScoped<ICustomerRepository, EfCustomerRepository>();
b.Services.AddScoped<IOrderRepository,    EfOrderRepository>();
b.Services.AddScoped<IProductRepository,  EfProductRepository>();
b.Services.AddScoped<IPaymentGateway,     StripePaymentGateway>();
b.Services.AddScoped<IEmailSender,        SendGridEmailSender>();
b.Services.AddScoped<CurrentUserContext>();   // set trong middleware, đọc bởi service
b.Services.AddScoped<CheckoutUseCase>();

// ─── Transient (helper nhẹ stateless) ───────────────────────────
b.Services.AddTransient<IOrderValidator,    OrderValidator>();
b.Services.AddTransient<IInvoiceFormatter,  HtmlInvoiceFormatter>();
b.Services.AddTransient<IGuidGenerator,     GuidGenerator>();
```

### 7.2 Middleware set state per-request

```csharp
app.Use(async (ctx, next) =>
{
    var current = ctx.RequestServices.GetRequiredService<CurrentUserContext>();
    if (Guid.TryParse(ctx.User.FindFirstValue("sub"), out var uid))
        current.UserId = uid;
    await next();
});
```

### 7.3 Service Map theo Lifetime

| Lifetime    | Service                                              | Tại sao                                            |
| ----------- | ---------------------------------------------------- | -------------------------------------------------- |
| Singleton   | `IClock`, `IMetrics`, `IExchangeRateCache`            | Stateless / thread-safe / app-wide               |
| Singleton   | `IHttpClientFactory` (qua AddHttpClient)             | Pool HTTP connection                              |
| Scoped      | `ShopDbContext`, repository, payment gateway          | Per-request DB / API state                        |
| Scoped      | `CurrentUserContext`, `CheckoutUseCase`               | Per-request context                                |
| Transient   | `IOrderValidator`, `IInvoiceFormatter`                | Stateless, nhẹ                                    |

---

## 8. Cheat Sheet Key Point

* **Default Scoped** cho application logic.
* **Singleton** chỉ cho thật stateless / thread-safe.
* **Transient** cho helper nhỏ, không state.
* **Singleton không bao giờ giữ Scoped trực tiếp** — dùng `IServiceScopeFactory`.
* **`DbContext` luôn Scoped** — không bao giờ singleton.
* **Background worker tạo scope riêng** với `IServiceScopeFactory`.
* **`HttpClient` qua `IHttpClientFactory`** — không `new HttpClient()`.
* **Đo lifetime bằng "ai share?"**, không phải "nhanh ra sao".

---

## 9. Cạm bẫy phổ biến & cách phát hiện

| Cạm bẫy                                  | Triệu chứng                                            | Cách phát hiện                          |
| ---------------------------------------- | ------------------------------------------------------ | --------------------------------------- |
| Singleton giữ Scoped                     | `ObjectDisposedException`, data stale                  | `ValidateScopes = true` trong dev       |
| Singleton stateful không thread-safe     | Data corruption ngẫu nhiên dưới tải                    | Code review + stress test               |
| Transient `IDisposable` tích tụ trong scope dài | Memory grow, file handle exhaust                   | Profiling                               |
| Scope không được tạo trong background worker | DbContext leak                                     | Log + observability                     |
| `Task.WhenAll` trên cùng DbContext       | `InvalidOperationException`                            | Test integration                        |

---

## 10. Diagnostic — Bắt bug sớm

* Bật `builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });` ở dev.
* Test: viết integration test resolve service và assert lifetime.
* Profiler: nhìn allocation / GC pressure cho transient `IDisposable` lẩn quẩn.
* Distributed tracing: track DbContext lifetime với OpenTelemetry.

---

## 11. So sánh Cross-Stack

| Stack         | Transient                | Scoped              | Singleton             |
| ------------- | ------------------------ | ------------------- | --------------------- |
| .NET          | `AddTransient<T>`         | `AddScoped<T>`       | `AddSingleton<T>`      |
| Spring (Java) | `@Scope("prototype")`     | `@RequestScope`     | `@Singleton` (default) |
| NestJS        | `@Injectable({scope: Scope.TRANSIENT})` | `Scope.REQUEST` | Default               |
| Angular       | `@Injectable({providedIn: 'any'})` | `providedIn: route` | `providedIn: 'root'` |

---

## See Also

* [`ioc-di-vi.md`](./ioc-di-vi.md) — câu chuyện DI / IoC rộng hơn.
* [`data-access-vi.md`](./data-access-vi.md) — tại sao `DbContext` (Scoped + Unit of Work) là ví dụ canonical.
* [`../../../principles/solid-vi.md`](../../../principles/solid-vi.md) — DIP, nguyên lý đằng sau tất cả.
* [`../../../oop/real-project-vi.md`](../../../oop/real-project-vi.md) — cùng model e-commerce, end-to-end.
