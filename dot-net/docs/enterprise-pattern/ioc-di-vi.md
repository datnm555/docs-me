# Inversion of Control & Dependency Injection

> Họ pattern quan trọng nhất trong phát triển application hiện đại. Gần như mọi framework bạn dùng — ASP.NET Core, Spring, NestJS, Angular — được xây quanh nó. Master IoC và DI và bạn có nền tảng cho mọi cái khác.

> 🇻🇳 Phiên bản tiếng Việt. English: [`ioc-di.md`](./ioc-di.md)

---

## Mục lục

1. [Inversion of Control (IoC)](#1-inversion-of-control-ioc)
2. [Dependency Injection (DI)](#2-dependency-injection-di)
3. [Ba flavor của DI](#3-ba-flavor-của-di)
4. [DI Container](#4-di-container)
5. [Service Lifetime](#5-service-lifetime)
6. [Composition Root](#6-composition-root)
7. [Service Locator — Anti-Pattern](#7-service-locator--anti-pattern)
8. [Cạm bẫy phổ biến](#8-cạm-bẫy-phổ-biến)
9. [Khi nào không dùng DI](#9-khi-nào-không-dùng-di)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — **IoC** là nguyên lý (framework call code của bạn, không ngược lại). **DI** là pattern realize IoC bằng cách pass dependency vào class thay vì construct bên trong.
- **Tại sao** — DI enable testability (substitute fake), flexibility (swap implementation qua config), single responsibility (class không build thế giới của nó), và decorator (wrap behavior không sửa core).
- **Khi nào** — Gần như luôn luôn trong .NET/Spring/NestJS hiện đại. Default **constructor injection**; reach setter/property chỉ cho dep optional; skip DI chỉ cho script tí xíu hoặc prototype sống ngắn.
- **Ở đâu** — Wire ở **composition root** (`Program.cs`). Container quản lý lifetime (xem `lifetimes-vi.md`). Foundation cho gần như mọi pattern trong folder này (Repository, Mediator, Decorator pipeline, Options).

---

## 1. Inversion of Control (IoC)

> *"Đừng call chúng tôi — chúng tôi sẽ call bạn."* (**Hollywood Principle**.)

**IoC là nguyên lý**, không phải pattern. Mô tả shift trong *ai kiểm soát flow* của chương trình:

| Flow truyền thống            | Flow inverted                                   |
| ---------------------------- | ----------------------------------------------- |
| Code của bạn drive chương trình | Framework drive chương trình                  |
| Code của bạn call library    | Framework call code của bạn                     |
| Bạn build wiring             | Framework / container build wiring              |

### 1.1 Ví dụ hằng ngày của IoC

* **Web framework** — bạn viết method controller; framework quyết định khi nào call dựa trên HTTP request đến.
* **Event handler** — bạn register callback; runtime invoke khi event fire.
* **GUI toolkit** — code bạn không chạy main loop; toolkit chạy, và call handler của bạn.
* **Template Method (GoF)** — base class kiểm soát thuật toán; subclass của bạn plug vào step biến.
* **DI container** — bạn mô tả dependency declarative; container construct graph.

### 1.2 IoC ≠ DI

IoC là idea rộng hơn. DI là **một cách cụ thể** để đạt nó (cụ thể, đảo *creation của dependency*). Cách khác:

* **Event / observer** — đảo ai quyết định khi react.
* **Callback / continuation** — đảo ai quyết định kế tiếp.
* **Service Locator** — đảo *nơi* dependency đến từ (nhưng pull-based, nên thường tệ hơn).

---

## 2. Dependency Injection (DI)

> *"Đừng construct collaborator; có chúng được trao cho bạn."*

Class *tạo* dependency của nó couple chặt với implementation cụ thể. Class *nhận* dependency được decouple, testable, swappable.

### 2.1 Tệ — Dependency hard-code

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repo = new();
    private readonly SmtpEmailSender    _mail = new();

    public void Place(Order o)
    {
        _repo.Save(o);
        _mail.Send(o.Email, "Thanks!");
    }
}
```

Vấn đề:

* Không thể swap SQL sang Mongo mà không sửa class này.
* Không thể unit test mà không có DB và SMTP server thật.
* Không thể thêm logging / retry / caching quanh dependency nào.

### 2.2 Tốt — Dependency được inject

```csharp
public class OrderService(IOrderRepository repo, IEmailSender mail)
{
    public void Place(Order o)
    {
        repo.Save(o);
        mail.Send(o.Email, "Thanks!");
    }
}
```

Bây giờ `OrderService` chỉ phụ thuộc **abstraction**. Implementation cụ thể được quyết định **chỗ khác** (composition root).

### 2.3 Lợi ích

* **Testability** — pass fake / mock.
* **Flexibility** — swap provider qua configuration.
* **Decoupling** — phụ thuộc contract, không phải implementation.
* **Cross-cutting concern** — bọc với decorator (logging, retry, caching).
* **Single Responsibility** — class tập trung công việc, không build thế giới của nó.

> DI là cơ chế thực hành đằng sau **Dependency Inversion Principle** (*D* trong SOLID).

---

## 3. Ba flavor của DI

### 3.1 Constructor Injection — **Ưu tiên**

```csharp
public class OrderService(IOrderRepository repo, IEmailSender mail)
{
    // dep là required, immutable, visible trong signature
}
```

* **Required** — object không thể tồn tại không có dep.
* **Immutable** — `readonly` khi gán.
* **Discoverable** — constructor LÀ contract.
* **Fail fast** — dep thiếu xuất hiện ở construction time.

> Dùng constructor injection trừ khi có lý do rất cụ thể không.

### 3.2 Property / Setter Injection

```csharp
public class OrderService
{
    public IOrderRepository Repo { get; set; } = default!;
    public IEmailSender     Mail { get; set; } = default!;
}
```

Chỉ dùng khi:

* Dependency thực sự **optional** (và có no-op default).
* Bạn kẹt trong framework construct object trước khi dep được biết (hiếm trong stack hiện đại).

Nhược: dep có thể `null`, có thể reassign, ẩn khỏi signature constructor.

### 3.3 Method Injection

```csharp
public class ReportRenderer
{
    public string Render(Report report, IFormatter formatter)
        => formatter.Format(report);
}
```

Dùng khi dep là **per-call**, không per-object — vd., formatter đổi theo request.

### 3.4 So sánh

| Flavor        | Required? | Visibility            | Dùng khi…                                  |
| ------------- | --------- | --------------------- | ------------------------------------------ |
| Constructor   | Có        | Constructor signature | Gần như luôn luôn                           |
| Property      | Không     | Class definition      | Dependency thực sự optional                 |
| Method        | Per call  | Method signature      | Dep vary theo invocation                    |

---

## 4. DI Container

**DI container** (aka **IoC container**) là runtime registry mà:

1. **Register** — bạn bảo nó concrete type nào implement mỗi interface, ở lifetime nào.
2. **Resolve** — khi có gì đó cần `IOrderRepository`, container construct cái đúng và đệ quy build dependency graph.

### 4.1 ASP.NET Core (Microsoft.Extensions.DependencyInjection)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IEmailSender,     SendGridEmailSender>();
builder.Services.AddScoped<OrderService>();

var app = builder.Build();

// Resolve tự động khi controller xin OrderService:
// public class OrdersController(OrderService svc) { ... }
```

### 4.2 Container phổ biến

| Stack            | Container built-in / phổ biến                          |
| ---------------- | ----------------------------------------------------- |
| .NET             | `Microsoft.Extensions.DependencyInjection`, Autofac, Lamar |
| Java             | Spring, Guice, CDI                                    |
| TypeScript / Node| NestJS DI, tsyringe, InversifyJS                      |
| Python           | dependency-injector, FastAPI's DI                     |
| Angular          | Injector built-in                                     |

Cơ chế khác nhau; idea giống hệt.

### 4.3 Container cho bạn gì

* **Wire object-graph** — không có manual `new` chain.
* **Lifetime management** — scope per request, per app, per instance.
* **Cross-cutting decoration** — register `IEmailSender` và một `LoggingEmailSender(IEmailSender)` decorator, transparent.
* **Configuration binding** — pattern `IOptions<T>`, gắn với `appsettings.json`.

---

## 5. Service Lifetime

Trong ASP.NET Core có ba, và tên là phổ quát:

| Lifetime    | Một instance per…                  | Dùng điển hình                              |
| ----------- | ---------------------------------- | ------------------------------------------- |
| **Transient** | Mỗi request tới container        | Utility nhẹ, stateless                       |
| **Scoped**    | HTTP request / unit of work      | Repository, `DbContext`, state per-request   |
| **Singleton** | Toàn bộ lifetime application     | Configuration, in-memory cache, clock        |

### 5.1 Captive Dependency — Bẫy kinh điển

> Service **sống dài hơn** giữ dependency **sống ngắn hơn**.

```csharp
services.AddSingleton<IOrderService, OrderService>();
services.AddScoped<IOrderRepository, EfOrderRepository>(); // EF DbContext là per-request

// Runtime: OrderService build một lần, giữ reference tới
//          repository per-request mãi. DbContext của repository giờ
//          leak xuyên qua request, gây bug concurrency và data stale.
```

**Rule:** singleton chỉ có thể phụ thuộc singleton. Scoped có thể phụ thuộc scoped hoặc singleton. Transient có thể phụ thuộc bất kỳ.

### 5.2 Lifetime nào?

* Mặc định **Scoped** cho application service trong web context.
* Dùng **Singleton** sparingly — chỉ cho thật stateless hoặc state được sync cẩn thận.
* Dùng **Transient** cho helper nhỏ, rẻ, stateless.

---

## 6. Composition Root

> *"Nên có một chỗ duy nhất trong application nơi object graph được wire."* — Mark Seemann

Cụ thể, đây là `Program.cs` (hoặc `Startup.cs`, hoặc `main()` trong console app). **Chỉ file này biết về concrete class.** Mọi class khác phụ thuộc interface.

### Tại sao quan trọng

* **Một điểm thay đổi** để swap implementation.
* **Không có service locator** rải rác trong code.
* **Boundary architectural rõ ràng** giữa policy và mechanism.

```csharp
// Program.cs — file DUY NHẤT biết EfOrderRepository tồn tại
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IEmailSender,     SendGridEmailSender>();
builder.Services.AddSingleton<IClock,        SystemClock>();
```

---

## 7. Service Locator — Anti-Pattern

> Pull dependency từ static registry thay vì nhận qua constructor.

### 7.1 Trông như thế nào

```csharp
public class OrderService
{
    public void Place(Order o)
    {
        var repo = ServiceLocator.Get<IOrderRepository>();   // dep ẨN
        var mail = ServiceLocator.Get<IEmailSender>();       // dep ẨN
        repo.Save(o);
        mail.Send(o.Email, "Thanks!");
    }
}
```

### 7.2 Tại sao tệ

* **Dependency ẩn** — không thể biết class cần gì không đọc thân.
* **Untestable** — test phải configure global registry, có thể leak giữa test.
* **Couple với locator** — mọi class phụ thuộc API của locator.
* **Đánh bại điểm của DI** — control *không* inverted; class vẫn quyết định dep đến từ đâu.

### 7.3 Khi nào chấp nhận được

* **Bên trong composition root** — bản thân *root* cần resolve service cấp top.
* **Trong legacy code** không có path migration khác (và chỉ như bước đệm).
* **Trong kịch bản factory hiếm** nơi dep chọn runtime — và rồi ưu tiên typed factory (`IDbContextFactory<T>`) hơn generic locator.

> Nếu có thể convert dùng Service Locator sang constructor injection mà không đau, hãy làm.

---

## 8. Cạm bẫy phổ biến

### 8.1 Quá nhiều dependency

Constructor có 8+ parameter là smell — *không phải của DI*, mà của class. Có lẽ vi phạm **SRP**.

**Fix:** tách class, hoặc nhóm dep liên quan vào **facade / parameter object**.

### 8.2 New thứ bên trong

```csharp
public class OrderService(IOrderRepository repo)
{
    public void Place(Order o)
    {
        var auditor = new AuditLogger();   // ← dependency ẩn, bypass DI
        ...
    }
}
```

Nếu `AuditLogger` có side effect (ghi disk / DB), phải được inject.

### 8.3 Resolve trong method

```csharp
public void Handle(IServiceProvider sp)
{
    var repo = sp.GetRequiredService<IOrderRepository>();  // service locator đội lốt
}
```

Pass `IOrderRepository` trực tiếp, hoặc dùng typed factory.

### 8.4 Circular Dependency

`A` phụ thuộc `B`, `B` phụ thuộc `A`. Container sẽ throw lúc construction. Báo hiệu thiếu class thứ ba để phá vòng (thường là domain event / mediator).

### 8.5 Lạm dụng generic trong registration

```csharp
services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>)); // đôi khi OK
```

Open-generic registration tốt cho abstraction mỏng; lạm dụng ẩn specifics nên explicit.

---

## 9. Khi nào không dùng DI

* **Script nhỏ / CLI một-shot** — `new` OK.
* **Value object và entity** — type domain không nên phụ thuộc service. (Pass service tới method thay vào.)
* **Function thuần** — đôi khi static method rõ hơn.
* **Hot path nhạy performance** — virtual call và indirection tốn trong inner loop; hiếm khi bottleneck nhưng đáng đo.

---

## Tham chiếu nhanh

| Câu hỏi                               | Trả lời                                             |
| ------------------------------------- | --------------------------------------------------- |
| IoC vs DI?                            | IoC là *nguyên lý*; DI là một *pattern* cho nó.     |
| DI flavor mặc định?                   | Constructor injection.                              |
| Singleton giữ Scoped dep — OK?        | Không (captive dependency).                         |
| Service Locator?                      | Anti-pattern ngoài composition root.                |
| Constructor 10 param?                 | Class đang làm quá nhiều.                            |
| Wiring ở đâu?                         | Composition root (`Program.cs`).                    |

---

> **Tiếp:** [`data-access-vi.md`](./data-access-vi.md) — Repository, Unit of Work, và câu chuyện lazy/eager loading.
