# Hexagonal vs. Onion vs. Clean Architecture

> Ba **architectural style** rất gần nhau, đều enforce cùng một luật cốt lõi: **dependency trỏ vào trong**. Domain không biết gì về database, HTTP, hay framework. Khác biệt nằm ở cách đặt tên, phân lớp, và nhấn mạnh — không phải tinh thần.

> 🇻🇳 Phiên bản tiếng Việt. English: [`hexagonal-onion-clean.md`](./hexagonal-onion-clean.md)

---

## Cốt lõi chung

Cả ba từ chối giả định của N-Layer rằng "database ở dưới đáy". Thay vào đó:

```
   ┌──────────────────────────────────────────┐
   │   Infrastructure / Adapters (ngoài cùng) │   EF Core, HTTP, gRPC, file system, SMTP
   │   ┌──────────────────────────────────┐   │
   │   │    Application / Use Cases       │   │   Orchestrate domain
   │   │    ┌──────────────────────┐      │   │
   │   │    │   Domain (trong cùng) │      │   │   Entity, value object, business rule
   │   │    └──────────────────────┘      │   │
   │   └──────────────────────────────────┘   │
   └──────────────────────────────────────────┘
```

* **Domain** ở trung tâm **không phụ thuộc** vào bất cứ gì bên ngoài.
* Mỗi vòng chỉ phụ thuộc vào vòng **gần trung tâm hơn**.
* Các concern bên ngoài (DB, HTTP) là **adapter** *implement* interface định nghĩa trong vòng trong.

Đây thường gọi là **Dependency Rule** (Uncle Bob): *source code dependency chỉ được trỏ vào trong.*

---

## 1. Hexagonal — "Ports and Adapters" (Alistair Cockburn, 2005)

Bản gốc. Trung tâm là **application**; xung quanh là **adapter** (driving bên trái, driven bên phải). Interface biên gọi là **port**.

```
          Driving (primary) adapters                         Driven (secondary) adapters
          ┌───────────────────────┐                         ┌────────────────────────┐
          │   HTTP Controller     │                         │   EF Core Repository    │
          │   gRPC Service        │   ┌──────────────────┐   │   SMTP Email Adapter    │
          │   CLI Command         │ ─▶│  Application     │ ─▶│   S3 File Storage       │
          │   Background Worker   │   │  Core + Domain   │   │   Twilio SMS Adapter    │
          └───────────────────────┘   └──────────────────┘   └────────────────────────┘
                  (drive app)            (define port)          (implement port)
```

* **Port** = interface trong application/domain (vd. `IPaymentGateway`, `IOrderRepository`).
* **Adapter** = implementation cụ thể trong infrastructure (vd. `StripeGateway`, `EfOrderRepository`).
* Đối xứng — cùng shape ở cả hai phía hexagon.

## 2. Onion (Jeffrey Palermo, 2008)

Các vòng đồng tâm. Trong cùng = **Domain Model**, rồi **Domain Services**, **Application Services**, **Infrastructure / UI**. Cùng Dependency Rule. Nhấn mạnh **domain ở trung tâm tuyệt đối**.

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

Trình bày lại cùng ý tưởng với tên rõ ràng hơn:

* **Entities** (trong cùng) — business rule cấp doanh nghiệp.
* **Use Cases** — business rule cụ thể của application.
* **Interface Adapters** — controller, presenter, gateway.
* **Frameworks & Drivers** (ngoài cùng) — ASP.NET Core, EF Core, SDK third-party.

Diagram của Uncle Bob về cơ bản là Hexagonal với tên layer rõ ràng và **Dependency Inversion Principle** được nêu rõ ở mỗi biên.

---

## Chúng có khác nhau thật không?

Trong thực tế — **không**. Cả ba đều:

* Đặt domain ở trung tâm.
* Đảo dependency qua interface.
* Đẩy framework ra rìa.
* Làm domain testable mà không cần infrastructure.

| Tên gọi          | Hexagonal             | Onion                  | Clean                       |
| ---------------- | --------------------- | ---------------------- | --------------------------- |
| Layer trong cùng | Application           | Domain Model           | Entities                    |
| Interface biên   | Port                  | Repository/Service     | Use Case Interactor / Gateway |
| Layer ngoài      | Adapter               | Infrastructure / UI    | Frameworks & Drivers        |

Chọn vocabulary nào team thích. Không cần tranh cãi.

---

## C# — Layout "Clean / Hexagonal / Onion" solution điển hình

```
MyApp.sln
├── MyApp.Domain                 (entity, value object, domain event, port/interface)
│   └── Orders/
│       ├── Order.cs
│       ├── OrderStatus.cs
│       └── IOrderRepository.cs   ← port
│
├── MyApp.Application             (use case, command/query handler, application service)
│   └── Orders/
│       ├── PlaceOrderCommand.cs
│       └── PlaceOrderHandler.cs
│
├── MyApp.Infrastructure          (adapter: EF Core, SDK third-party, file system)
│   └── Persistence/
│       ├── AppDbContext.cs
│       └── EfOrderRepository.cs  ← adapter implement IOrderRepository
│
├── MyApp.Api                     (controller ASP.NET Core / minimal API)
│
└── MyApp.Tests
    ├── MyApp.Domain.Tests        (không cần infra)
    └── MyApp.Application.Tests   (mock port)
```

### Project reference (Dependency Rule, được compiler enforce)

```
Domain        ← Application      ← Infrastructure
                              ↖   Api
```

* `Application` reference `Domain`.
* `Infrastructure` reference `Application` và `Domain`.
* `Api` reference `Application` (và `Infrastructure` chỉ để đăng ký DI).
* `Domain` reference **không gì cả**.

Có thể verify trong file `.csproj`. Tool như **NetArchTest** hoặc **ArchUnitNET** có thể fail build nếu ai đó vi phạm.

### Wire adapter ở startup (`Api/Program.cs`)

```csharp
// "Cắm" mỗi port vào adapter của nó
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentAdapter>();
builder.Services.AddScoped<IEmailService, SendGridEmailAdapter>();

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<PlaceOrderHandler>());
```

Composition root (Program.cs / Startup.cs) là **nơi duy nhất** biết cả abstraction và implementation cụ thể.

---

## Trade-off

✅ **Lợi:**

* Domain có thể **unit test mà không cần database**.
* Thay infra bất kỳ (EF → Dapper, SQL → Mongo, Stripe → Adyen) mà không đụng domain hay application.
* Bắt buộc **đặt tên** cho boundary — cải thiện thảo luận thiết kế.

⚠️ **Chi phí:**

* **Indirection** — nhiều interface, nhiều project, nhiều file.
* **Mapping** — DTO ở API edge, domain type ở trung tâm, persistence type ở DB edge. Ba shape cho cùng một data.
* **Quá đà cho app CRUD** — với admin tool 5 màn hình, N-Layer nhanh hơn.

---

## Khi nào chọn gì

| Nếu…                                                                  | Thì…                               |
| --------------------------------------------------------------------- | ---------------------------------- |
| App CRUD đơn giản, team nhỏ, đời sống ngắn                            | Dùng **N-Layer**                   |
| Domain phức tạp, business rule sẽ tiến hoá, nhiều kênh input          | Dùng **Hexagonal / Clean / Onion** |
| Đang làm DDD với domain model thật sự                                 | **Clean + DDD** là cặp đôi chuẩn   |

---

## Các Anti-Pattern thường gặp

* **Domain reference EF Core** — entity kế thừa base class có DbContext awareness. Phá Dependency Rule.
* **Anemic domain model** — entity chỉ có getter/setter, mọi behavior trong "service" class. Bạn có layer nhưng không có domain thật.
* **Generic repository ở domain layer** — xem [`../enterprise-pattern/repository-vi.md`](../enterprise-pattern/repository-vi.md).
* **DTO explosion** — ba bộ type gần như giống hệt nhau chỉ vì "phải vậy". Chỉ map khi shape thật sự khác.

---

## Tham khảo

* Alistair Cockburn — *Hexagonal Architecture* (2005). [alistair.cockburn.us/hexagonal-architecture](https://alistair.cockburn.us/hexagonal-architecture/).
* Jeffrey Palermo — *The Onion Architecture* (2008). [jeffreypalermo.com/2008/07/the-onion-architecture-part-1](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/).
* Robert C. Martin — *Clean Architecture* (2017).
* Steve Smith — [`ardalis/CleanArchitecture` template cho .NET](https://github.com/ardalis/CleanArchitecture).
* Jason Taylor — [`jasontaylordev/CleanArchitecture`](https://github.com/jasontaylordev/CleanArchitecture).
