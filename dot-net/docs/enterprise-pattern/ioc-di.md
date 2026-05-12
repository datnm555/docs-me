# Inversion of Control & Dependency Injection

> The single most important pattern family in modern application development. Almost every framework you use — ASP.NET Core, Spring, NestJS, Angular — is built around it. Master IoC and DI and you have the foundation for everything else.

---

## Table of Contents

1. [Inversion of Control (IoC)](#1-inversion-of-control-ioc)
2. [Dependency Injection (DI)](#2-dependency-injection-di)
3. [The Three Flavors of DI](#3-the-three-flavors-of-di)
4. [DI Containers](#4-di-containers)
5. [Service Lifetimes](#5-service-lifetimes)
6. [The Composition Root](#6-the-composition-root)
7. [Service Locator — The Anti-Pattern](#7-service-locator--the-anti-pattern)
8. [Common Pitfalls](#8-common-pitfalls)
9. [When Not to Use DI](#9-when-not-to-use-di)

---

## 1. Inversion of Control (IoC)

> *"Don't call us — we'll call you."* (The **Hollywood Principle**.)

**IoC is a principle**, not a pattern. It describes a shift in *who controls the flow* of a program:

| Traditional flow             | Inverted flow                                  |
| ---------------------------- | ---------------------------------------------- |
| Your code drives the program | The framework drives the program               |
| Your code calls libraries    | The framework calls your code                  |
| You build the wiring         | The framework / container builds the wiring    |

### 1.1 Everyday Examples of IoC

* **Web framework** — you write a controller method; the framework decides when to call it based on incoming HTTP requests.
* **Event handler** — you register a callback; the runtime invokes it when the event fires.
* **GUI toolkit** — your code does not run a main loop; the toolkit does, and calls your handlers.
* **Template Method (GoF)** — the base class controls the algorithm; your subclass plugs in the variable steps.
* **DI container** — you describe dependencies declaratively; the container constructs the graph.

### 1.2 IoC ≠ DI

IoC is the broader idea. DI is **one specific way** to achieve it (specifically, the inversion of *dependency creation*). Other ways:

* **Events / observers** — invert who decides when to react.
* **Callbacks / continuations** — invert who decides what comes next.
* **Service Locator** — invert *where* dependencies come from (though pull-based, so usually worse).

---

## 2. Dependency Injection (DI)

> *"Don't construct your collaborators; have them handed to you."*

A class that *creates* its dependencies is tightly coupled to specific implementations. A class that *receives* its dependencies is decoupled, testable, and swappable.

### 2.1 Bad — Hard-coded Dependencies

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

Problems:

* Can't swap SQL for Mongo without editing this class.
* Can't unit test without a live database and SMTP server.
* Can't add logging / retry / caching around either dependency.

### 2.2 Good — Injected Dependencies

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

Now `OrderService` depends only on **abstractions**. Concrete implementations are decided **elsewhere** (the composition root).

### 2.3 Benefits

* **Testability** — pass in fakes / mocks.
* **Flexibility** — swap providers via configuration.
* **Decoupling** — depend on contracts, not implementations.
* **Cross-cutting concerns** — wrap with decorators (logging, retry, caching).
* **Single Responsibility** — class focuses on its job, not on building its world.

> DI is the practical mechanism behind the **Dependency Inversion Principle** (the *D* in SOLID).

---

## 3. The Three Flavors of DI

### 3.1 Constructor Injection — **Preferred**

```csharp
public class OrderService(IOrderRepository repo, IEmailSender mail)
{
    // deps are required, immutable, visible in the signature
}
```

* **Required** — object can't exist without its deps.
* **Immutable** — `readonly` once assigned.
* **Discoverable** — the constructor IS the contract.
* **Fails fast** — missing deps surface at construction time.

> Use constructor injection unless you have a very specific reason not to.

### 3.2 Property / Setter Injection

```csharp
public class OrderService
{
    public IOrderRepository Repo { get; set; } = default!;
    public IEmailSender     Mail { get; set; } = default!;
}
```

Use only when:

* The dependency is genuinely **optional** (and there's a no-op default).
* You're stuck in a framework that constructs the object before deps are known (rare in modern stacks).

Downsides: deps can be `null`, can be reassigned, hidden from the constructor signature.

### 3.3 Method Injection

```csharp
public class ReportRenderer
{
    public string Render(Report report, IFormatter formatter)
        => formatter.Format(report);
}
```

Use when the dependency is **per-call**, not per-object — e.g., the formatter changes by request.

### 3.4 Comparison

| Flavor        | Required? | Visibility            | Use when…                                  |
| ------------- | --------- | --------------------- | ------------------------------------------ |
| Constructor   | Yes       | Constructor signature | Almost always                              |
| Property      | No        | Class definition      | Truly optional dependency                  |
| Method        | Per call  | Method signature      | Dep varies per invocation                  |

---

## 4. DI Containers

A **DI container** (a.k.a. **IoC container**) is a runtime registry that:

1. **Registers** — you tell it which concrete type implements each interface, and at what lifetime.
2. **Resolves** — when something needs `IOrderRepository`, the container constructs the right thing and recursively builds its dependency graph.

### 4.1 ASP.NET Core (Microsoft.Extensions.DependencyInjection)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IEmailSender,     SendGridEmailSender>();
builder.Services.AddScoped<OrderService>();

var app = builder.Build();

// Resolve happens automatically when a controller asks for OrderService:
// public class OrdersController(OrderService svc) { ... }
```

### 4.2 Popular Containers

| Stack            | Built-in / common containers                          |
| ---------------- | ----------------------------------------------------- |
| .NET             | `Microsoft.Extensions.DependencyInjection`, Autofac, Lamar |
| Java             | Spring, Guice, CDI                                    |
| TypeScript / Node| NestJS DI, tsyringe, InversifyJS                      |
| Python           | dependency-injector, FastAPI's DI                     |
| Angular          | Built-in injector                                     |

The mechanics differ; the ideas are identical.

### 4.3 What a Container Buys You

* **Object-graph wiring** — no manual `new` chains.
* **Lifetime management** — scope per request, per app, per instance.
* **Cross-cutting decoration** — register an `IEmailSender` and a `LoggingEmailSender(IEmailSender)` decorator, transparently.
* **Configuration binding** — `IOptions<T>` patterns, tied to `appsettings.json`.

---

## 5. Service Lifetimes

In ASP.NET Core there are three, and the names are universal:

| Lifetime    | One instance per…                  | Typical use                                 |
| ----------- | ---------------------------------- | ------------------------------------------- |
| **Transient** | Each request to the container    | Lightweight, stateless utilities             |
| **Scoped**    | HTTP request / unit of work      | Repositories, `DbContext`, per-request state |
| **Singleton** | The whole application lifetime   | Configuration, in-memory caches, clocks      |

### 5.1 Captive Dependency — A Classic Trap

> A **longer-lived** service holding a **shorter-lived** dependency.

```csharp
services.AddSingleton<IOrderService, OrderService>();
services.AddScoped<IOrderRepository, EfOrderRepository>(); // EF DbContext is per-request

// At runtime: OrderService is built once, holding a reference to a per-request
//             repository forever. The repository's DbContext is now leaked
//             across requests, causing concurrency bugs and stale data.
```

**Rule:** a singleton may only depend on singletons. A scoped service may depend on scoped or singletons. A transient may depend on anything.

### 5.2 Which Lifetime?

* Default to **Scoped** for application services in a web context.
* Use **Singleton** sparingly — only for truly stateless or carefully synchronized state.
* Use **Transient** for tiny, cheap, stateless helpers.

---

## 6. The Composition Root

> *"There should be a single place in the application where the object graph is wired."* — Mark Seemann

Concretely, this is your `Program.cs` (or `Startup.cs`, or `main()` in console apps). **Only this file knows about concrete classes.** Every other class depends on interfaces.

### Why It Matters

* **Single point of change** for swapping implementations.
* **No service locator** scattered through the code.
* **Clear architectural boundary** between policy and mechanism.

```csharp
// Program.cs — the ONLY file that knows EfOrderRepository exists
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IEmailSender,     SendGridEmailSender>();
builder.Services.AddSingleton<IClock,        SystemClock>();
```

---

## 7. Service Locator — The Anti-Pattern

> Pulling a dependency from a static registry instead of receiving it via the constructor.

### 7.1 What It Looks Like

```csharp
public class OrderService
{
    public void Place(Order o)
    {
        var repo = ServiceLocator.Get<IOrderRepository>();   // HIDDEN dep
        var mail = ServiceLocator.Get<IEmailSender>();       // HIDDEN dep
        repo.Save(o);
        mail.Send(o.Email, "Thanks!");
    }
}
```

### 7.2 Why It's Bad

* **Hidden dependencies** — you can't tell what the class needs without reading the body.
* **Untestable** — tests must configure a global registry, which can leak between tests.
* **Couples to the locator** — every class depends on the locator's API.
* **Defeats the point of DI** — control is *not* inverted; the class still decides where its deps come from.

### 7.3 When It's Acceptable

* **Inside the composition root** — the *root* itself needs to resolve top-level services.
* **In legacy code** with no other migration path (and only as a stepping stone).
* **In rare factory scenarios** where dependencies are chosen at runtime — and then prefer a typed factory (`IDbContextFactory<T>`) over a generic locator.

> If you can convert a Service Locator usage to constructor injection without pain, do it.

---

## 8. Common Pitfalls

### 8.1 Too Many Dependencies

A constructor with 8+ parameters is a smell — *not of DI*, but of the class. Probably violates **SRP**.

**Fix:** split the class, or group related deps into a **facade / parameter object**.

### 8.2 New-ing Things Inside

```csharp
public class OrderService(IOrderRepository repo)
{
    public void Place(Order o)
    {
        var auditor = new AuditLogger();   // ← hidden dependency, bypasses DI
        ...
    }
}
```

If `AuditLogger` has side effects (writes to disk / DB), it must be injected.

### 8.3 Resolving in Methods

```csharp
public void Handle(IServiceProvider sp)
{
    var repo = sp.GetRequiredService<IOrderRepository>();  // service locator in disguise
}
```

Pass `IOrderRepository` directly, or use a typed factory.

### 8.4 Circular Dependencies

`A` depends on `B`, `B` depends on `A`. The container will throw at construction. This indicates a missing third class to break the cycle (often a domain event / mediator).

### 8.5 Overusing Generics in Registrations

```csharp
services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>)); // sometimes fine
```

Open-generic registration is great for thin abstractions; misused, it hides specifics that should be explicit.

---

## 9. When Not to Use DI

* **Tiny scripts / one-shot CLIs** — `new` is fine.
* **Value objects and entities** — domain types should not depend on services. (Pass services to methods instead.)
* **Plain functions** — sometimes a static method is just clearer.
* **Performance-critical hot paths** — virtual calls and indirections cost in inner loops; rarely the bottleneck but worth measuring.

---

## Quick Reference

| Question                              | Answer                                              |
| ------------------------------------- | --------------------------------------------------- |
| IoC vs DI?                            | IoC is the *principle*; DI is one *pattern* for it. |
| Default DI flavor?                    | Constructor injection.                              |
| Singleton holding Scoped dep — ok?    | No (captive dependency).                            |
| Service Locator?                      | Anti-pattern outside the composition root.          |
| 10-param constructor?                 | The class is doing too much.                        |
| Where is the wiring?                  | The composition root (`Program.cs`).                |

---

> **Next:** [`data-access.md`](./data-access.md) — Repository, Unit of Work, and the lazy/eager loading story.
