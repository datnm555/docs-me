# Hexagonal vs. Onion vs. Clean Architecture

> Three closely-related **architectural styles** that all enforce the same core rule: **dependencies point inward**. The domain knows nothing about databases, HTTP, or frameworks. Differences are in naming, layering, and emphasis — not in spirit.

---

## The Common Core

All three reject the N-Layer assumption that "the database is at the bottom." Instead:

```
   ┌──────────────────────────────────────────┐
   │   Infrastructure / Adapters (outer)      │   EF Core, HTTP, gRPC, file system, SMTP
   │   ┌──────────────────────────────────┐   │
   │   │    Application / Use Cases       │   │   Orchestrate the domain
   │   │    ┌──────────────────────┐      │   │
   │   │    │   Domain (inner)     │      │   │   Entities, value objects, business rules
   │   │    └──────────────────────┘      │   │
   │   └──────────────────────────────────┘   │
   └──────────────────────────────────────────┘
```

* The **domain** at the center has **no dependencies** on anything outside.
* Each ring depends only on rings **closer to the center**.
* External concerns (DB, HTTP) are **adapters** that *implement* interfaces defined further in.

This is sometimes called the **Dependency Rule** (Uncle Bob): *source code dependencies must point only inwards.*

---

## 1. Hexagonal — "Ports and Adapters" (Alistair Cockburn, 2005)

The original. The center is the **application**; everything around it is **adapters** (driving on the left, driven on the right). Boundary interfaces are called **ports**.

```
              Driving (primary) adapters                            Driven (secondary) adapters
              ┌───────────────────────┐                            ┌────────────────────────┐
              │   HTTP Controller     │                            │   EF Core Repository    │
              │   gRPC Service        │     ┌──────────────────┐    │   SMTP Email Adapter    │
              │   CLI Command         │ ──▶ │  Application     │ ──▶│   S3 File Storage       │
              │   Background Worker   │     │  Core + Domain   │    │   Twilio SMS Adapter    │
              └───────────────────────┘     └──────────────────┘    └────────────────────────┘
                       (drive the app)         (defines ports)         (implement ports)
```

* **Ports** = interfaces in the application/domain (e.g. `IPaymentGateway`, `IOrderRepository`).
* **Adapters** = concrete implementations in infrastructure (e.g. `StripeGateway`, `EfOrderRepository`).
* Symmetric — the same shape on both sides of the hexagon.

## 2. Onion (Jeffrey Palermo, 2008)

Concentric rings. Innermost = **Domain Model**, then **Domain Services**, then **Application Services**, then **Infrastructure / UI**. Same Dependency Rule. Emphasis on **the domain at the very core**.

```
        ┌─────────────────────────────────────────┐
        │   UI / Infrastructure                    │
        │   ┌─────────────────────────────────┐   │
        │   │   Application Services           │   │
        │   │   ┌──────────────────────┐      │   │
        │   │   │   Domain Services    │      │   │
        │   │   │   ┌──────────────┐   │      │   │
        │   │   │   │  Domain Model │   │      │   │
        │   │   │   └──────────────┘   │      │   │
        │   │   └──────────────────────┘      │   │
        │   └─────────────────────────────────┘   │
        └─────────────────────────────────────────┘
```

## 3. Clean Architecture (Robert C. Martin, 2012)

A re-presentation of the same idea with sharper naming:

* **Entities** (innermost) — enterprise-wide business rules.
* **Use Cases** — application-specific business rules.
* **Interface Adapters** — controllers, presenters, gateways.
* **Frameworks & Drivers** (outermost) — ASP.NET Core, EF Core, third-party SDKs.

Uncle Bob's diagram is essentially Hexagonal with explicit layer names and the **Dependency Inversion Principle** spelled out at each boundary.

---

## Are They Really Different?

In practice — **no**. All three:

* Put the domain in the center.
* Invert dependencies via interfaces.
* Push frameworks to the edge.
* Make the domain testable without infrastructure.

| Term used        | Hexagonal             | Onion                  | Clean                       |
| ---------------- | --------------------- | ---------------------- | --------------------------- |
| Innermost layer  | Application           | Domain Model           | Entities                    |
| Boundary interface | Port                | Repository/Service     | Use Case Interactor / Gateway |
| Outer layer      | Adapter              | Infrastructure / UI     | Frameworks & Drivers        |

Pick whichever vocabulary your team prefers. Don't argue about it.

---

## C# — A Typical "Clean / Hexagonal / Onion" Solution Layout

```
MyApp.sln
├── MyApp.Domain                 (entities, value objects, domain events, ports/interfaces)
│   └── Orders/
│       ├── Order.cs
│       ├── OrderStatus.cs
│       └── IOrderRepository.cs   ← port
│
├── MyApp.Application             (use cases, command/query handlers, application services)
│   └── Orders/
│       ├── PlaceOrderCommand.cs
│       └── PlaceOrderHandler.cs
│
├── MyApp.Infrastructure          (adapters: EF Core, third-party SDKs, file system)
│   └── Persistence/
│       ├── AppDbContext.cs
│       └── EfOrderRepository.cs  ← adapter implements IOrderRepository
│
├── MyApp.Api                     (ASP.NET Core controllers / minimal APIs)
│
└── MyApp.Tests
    ├── MyApp.Domain.Tests        (no infra needed)
    └── MyApp.Application.Tests   (mocks the ports)
```

### Project references (the Dependency Rule, enforced by the compiler)

```
Domain        ← Application      ← Infrastructure
                              ↖   Api
```

* `Application` references `Domain`.
* `Infrastructure` references `Application` and `Domain`.
* `Api` references `Application` (and `Infrastructure` only for DI registration).
* `Domain` references **nothing**.

You can verify this in `.csproj` files. Tools like **NetArchTest** or **ArchUnitNET** can fail the build if someone violates it.

### Wiring the adapters at startup (`Api/Program.cs`)

```csharp
// "Plug" each port into its adapter
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentAdapter>();
builder.Services.AddScoped<IEmailService, SendGridEmailAdapter>();

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<PlaceOrderHandler>());
```

The composition root (Program.cs / Startup.cs) is **the only place** that knows both the abstraction and the concrete implementation.

---

## Trade-offs

✅ **Wins:**

* The domain can be **unit-tested without a database**.
* Swap any infrastructure (EF → Dapper, SQL → Mongo, Stripe → Adyen) without touching domain or application.
* Forces you to **name** your boundaries — improves design conversations.

⚠️ **Costs:**

* **Indirection** — more interfaces, more projects, more files.
* **Mapping** — DTOs at the API edge, domain types in the center, persistence types at the DB edge. Three shapes for the same data.
* **Overkill for CRUD apps** — for a 5-screen admin tool, N-Layer is faster.

---

## When to Choose Which

| If…                                                                  | Then…                              |
| -------------------------------------------------------------------- | ---------------------------------- |
| Simple CRUD app, small team, short-lived                             | Use **N-Layer**                    |
| Complex domain, business rules will evolve, multiple input channels  | Use **Hexagonal / Clean / Onion**  |
| You are also doing DDD with a real domain model                      | **Clean + DDD** is the standard duo |

---

## Common Anti-Patterns to Avoid

* **Domain references EF Core** — entities inheriting from `DbContext`-aware base classes. Hard break of the Dependency Rule.
* **Anemic domain model** — entities with only getters/setters, all behavior in "service" classes. You have layers but no real domain.
* **Generic repository at the domain layer** — see [`../enterprise-pattern/repository.md`](../enterprise-pattern/repository.md).
* **DTO explosion** — three sets of nearly-identical types just for the sake of it. Map only when shapes really differ.

---

## References

* Alistair Cockburn — *Hexagonal Architecture* (2005). [alistair.cockburn.us/hexagonal-architecture](https://alistair.cockburn.us/hexagonal-architecture/).
* Jeffrey Palermo — *The Onion Architecture* (2008). [jeffreypalermo.com/2008/07/the-onion-architecture-part-1](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/).
* Robert C. Martin — *Clean Architecture* (2017).
* Steve Smith — [`ardalis/CleanArchitecture` template for .NET](https://github.com/ardalis/CleanArchitecture).
* Jason Taylor — [`jasontaylordev/CleanArchitecture`](https://github.com/jasontaylordev/CleanArchitecture).
