# Clean Architecture — Tóm tắt

> **Robert C. Martin (Uncle Bob)** — *Clean Architecture: A Craftsman's Guide to Software Structure and Design*, Prentice Hall, 2017.

> **Tại sao sách này quan trọng:** sách codify cái phần lớn team .NET, Java, TypeScript hiện đại gọi là "Clean / Onion / Hexagonal". Uncle Bob tổng hợp ~50 năm programming structured, OO, và functional thành một kiến trúc duy nhất **driven bởi dependency rule**, rồi chỉ ra cái gì sai khi rule bị phá. Diagram concentric circle nổi tiếng sống ở đây.

> **Tóm tắt này cover:** lịch sử programming paradigm, refresher SOLID, sáu **component principle** (REP/CCP/CRP và ADP/SDP/SAP), **dependency rule**, diagram 4 ring, Humble Object pattern, cái gì tính là "detail" (database, web, framework), và bits triết lý ở cuối. Code example là của tôi, illustrative.

> 🇻🇳 Phiên bản tiếng Việt. English: [`clean-architecture-robert-martin.md`](./clean-architecture-robert-martin.md)

---

## Mục lục

1. [Về cuốn sách](#1-về-cuốn-sách)
2. [Lập luận Uncle Bob trong một câu](#2-lập-luận-uncle-bob-trong-một-câu)
3. [Phần I — Giới thiệu: Design và Architecture là gì?](#3-phần-i--giới-thiệu-design-và-architecture-là-gì)
4. [Phần II — Bắt đầu với Brick: Programming Paradigm](#4-phần-ii--bắt-đầu-với-brick-programming-paradigm)
5. [Phần III — Design Principle: SOLID](#5-phần-iii--design-principle-solid)
6. [Phần IV — Component Principle](#6-phần-iv--component-principle)
7. [Phần V — Architecture](#7-phần-v--architecture)
8. [Diagram Clean Architecture](#8-diagram-clean-architecture)
9. [Dependency Rule](#9-dependency-rule)
10. [Boundary — Logical, Local, và Full](#10-boundary--logical-local-và-full)
11. [Humble Object Pattern](#11-humble-object-pattern)
12. [Phần VI — Detail: Database, Web, Framework](#12-phần-vi--detail-database-web-framework)
13. [Screaming Architecture](#13-screaming-architecture)
14. [Test Boundary](#14-test-boundary)
15. [Clean Embedded Architecture](#15-clean-embedded-architecture)
16. [Take-Away bức tranh lớn](#16-take-away-bức-tranh-lớn)
17. [Sách này fit với sách khác ra sao](#17-sách-này-fit-với-sách-khác-ra-sao)
18. [Khuyến nghị thứ tự đọc](#18-khuyến-nghị-thứ-tự-đọc)
19. [Hiểu lầm phổ biến](#19-hiểu-lầm-phổ-biến)
20. [Tham khảo](#20-tham-khảo)

---

## 1. Về cuốn sách

* **Tiêu đề:** *Clean Architecture: A Craftsman's Guide to Software Structure and Design*
* **Tác giả:** Robert C. Martin ("Uncle Bob")
* **NXB:** Prentice Hall (Pearson), tháng 9/2017
* **Độ dài:** ~432 trang
* **Audience:** engineer mid-to-senior, tech lead, và architect muốn defense first-principles cho kiến trúc layered, dependency-inverted.
* **Lineage:** sequel tự nhiên của *Clean Code* (2008) và *The Clean Coder* (2011); toàn bộ trilogy "Clean".

Sách là tổng hợp — kéo Hexagonal của Cockburn, Onion của Palermo, Boundary, Use Case, DDD, và 50 năm nghiên cứu programming-paradigm vào một lập luận thống nhất. Output cụ thể nhất là **một diagram** (4 ring đồng tâm) và **một rule** (Dependency Rule).

---

## 2. Lập luận Uncle Bob trong một câu

> Mục tiêu của software architecture là **giảm thiểu nguồn lực con người cần để build và maintain hệ thống yêu cầu** — và cách đạt được đó là giữ **policy** (business rule giá trị cao, thay đổi chậm) **độc lập với detail** (lựa chọn technology biến động, giá trị thấp).

Mọi thứ khác trong sách là hệ quả. Dependency Rule, các ring, các boundary — chúng tồn tại để *bảo vệ policy khỏi detail* để team có thể thay đổi một mà không đụng cái kia.

---

## 3. Phần I — Giới thiệu: Design và Architecture là gì?

* **Architecture = *shape* bạn cho hệ thống.** Không tách khỏi design; "architecture" và "design" mô tả cùng một thứ ở scale khác nhau.
* **Mục tiêu là maximize productivity** qua lifetime của hệ thống. Hệ thống "thắng" nếu cost of change giữ flat theo thời gian.
* **Hai giá trị** một software system cung cấp:
  * **Behavior** — hệ thống làm gì (cái khẩn).
  * **Structure** — khả năng thay đổi của nó (cái quan trọng).
  * Behavior khẩn; structure quan trọng. Phần lớn team prioritize cái khẩn và starve cái quan trọng tới khi structure collapse.
* **Eisenhower's matrix** áp dụng cho software: làm việc quan trọng-không-khẩn *trước* (structure), hoặc bạn sẽ tốn phần còn lại sự nghiệp firefighting việc khẩn-quan-trọng đã trở thành khẩn vì bạn ignore cái quan trọng.

---

## 4. Phần II — Bắt đầu với Brick: Programming Paradigm

Uncle Bob lập luận chỉ có **ba** programming paradigm từng được phát minh, và **mỗi cái là constraint** — không phải capability.

| Paradigm        | Năm   | Cái nó **cấm**                                       | Tại sao quan trọng                      |
| --------------- | ----- | ---------------------------------------------------- | --------------------------------------- |
| **Structured**  | 1968  | Direct transfer of control (`goto`)                  | Enable decomposition và proof            |
| **Object-Oriented** | 1966 | Indirect transfer of control (function pointer as plain data) | Enable polymorphism — và **dependency inversion** |
| **Functional**  | 1957 / 1958 | Assignment (mutable state)                      | Enable concurrency safety               |

Bài học provocative: **paradigm remove power; không add it.** Discipline mỗi cái enforce là cái làm hệ thống maintainable.

> Tại sao điều này quan trọng cho architecture: **OO's gift là polymorphism**, cho phép vẽ software boundary mà hướng source-code dependency **ngược chiều** với hướng runtime control flow. Capability đơn này làm Clean Architecture khả thi.

---

## 5. Phần III — Design Principle: SOLID

Refresher nhanh (sách giả định quen thuộc):

| Chữ    | Nguyên lý                       | Statement ngắn gọn                                                            |
| ------ | ------------------------------- | ---------------------------------------------------------------------------- |
| **S**  | Single Responsibility           | Module nên có một và chỉ một lý do thay đổi — vd., một *actor*.              |
| **O**  | Open/Closed                     | Mở extension, đóng modification — driven bởi polymorphism.                   |
| **L**  | Liskov Substitution             | Subtype phải substitute được về behavior, không chỉ structurally.            |
| **I**  | Interface Segregation           | Đừng phụ thuộc cái bạn không dùng; ưu tiên interface tập trung.              |
| **D**  | Dependency Inversion            | Source code dependency chỉ trỏ tới **abstraction**, không concretion.        |

Re-framing quan trọng trong sách này:

* **SRP là về actor, không phải "thứ"**. Class vi phạm SRP khi hai *con người khác* sẽ yêu cầu thay đổi nó. Ví dụ classic của sách: class `Employee` với `calculatePay`, `reportHours`, `save` — 3 stakeholder khác, 3 lý do thay đổi khác.
* **Dependency Inversion là trụ chịu lực của tất cả Clean Architecture.** Mọi thứ khác (boundary, ring, dependency rule) là hệ quả cấu trúc.

→ Cross-reference: [`../principles/solid-vi.md`](../principles/solid-vi.md) cho SOLID đầy đủ.

---

## 6. Phần IV — Component Principle

SOLID áp dụng cho *class*. **Component principle** áp dụng cho unit deployable (DLL, JAR, npm package). Uncle Bob list 6 cái:

### Component Cohesion (cái gì đi trong component)

| Nguyên lý                                       | Hỏi gì                                                                        |
| ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **REP** — Reuse / Release Equivalence           | Component là unit của reuse *và* unit của release. Version nó.                |
| **CCP** — Common Closure Principle              | Group class thay đổi cùng nhau vì cùng lý do. (SRP ở mức component.)           |
| **CRP** — Common Reuse Principle                | Đừng ép consumer phụ thuộc class chúng không dùng. (ISP ở mức component.)     |

Ba cái pull theo hướng khác nhau — **tension diagram** của Uncle Bob cho thấy thoả mãn cả ba hoàn hảo là không thể. Team prioritize khác nhau khi project mature.

### Component Coupling (component liên hệ ra sao)

| Nguyên lý                                       | Hỏi gì                                                                        |
| ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **ADP** — Acyclic Dependencies Principle        | Graph component dependency phải là **DAG**. Không vòng, mãi mãi.              |
| **SDP** — Stable Dependencies Principle         | Phụ thuộc theo hướng stability. Component có nhiều dependent là stable; cái có ít dependent và nhiều dependency là volatile. |
| **SAP** — Stable Abstractions Principle         | Component stable nên **abstract**. Component volatile có thể concrete.        |

### Main Sequence

Plot mỗi component trên chart 2D: x = instability (0 stable → 1 unstable), y = abstractness (0 concrete → 1 abstract). Component nên ngồi gần đường chéo **y = 1 - x**.

* **Zone of Pain** (stable + concrete, dưới trái): code concrete không thể thay đổi (kiểu `java.util.String`). Khó extend; nếu stop fit, kẹt.
* **Zone of Uselessness** (unstable + abstract, trên phải): interface không ai phụ thuộc. Pure noise.
* **Main Sequence** (đường chéo): sweet spot.

Chart đơn này là một trong những idea được trích dẫn nhiều nhất trong sách — nhiều tool static-analysis (NDepend, SonarQube) report distance-from-main-sequence như metric.

---

## 7. Phần V — Architecture

Section dài nhất. Theme:

### Architecture *để làm gì*

* Để **defer quyết định** — *ngày bạn phải chọn database* nên muộn trong project, không sớm.
* Để **giữ option mở** — job của architect là đảm bảo quyết định đau nhất vẫn reversible càng lâu càng tốt.
* Architect produce **shape**; job của shape là làm policy độc lập với detail.

### Độc lập

Hệ thống well-architected độc lập theo nhiều nghĩa:

* **Use-case độc lập** — thêm use case mới không nên ripple qua hệ thống.
* **Operability** — UI, batch, scheduled, distributed deployment nên configurable.
* **Development độc lập** — nhiều team nên có thể work parallel mà không đụng nhau.
* **Deployability** — phần nên deployable độc lập.

### Boundary (idea trung tâm)

**Boundary** là một đường trong source code mà qua đó dependency *một chiều*. Sách dành nhiều chương để vẽ chúng đúng:

* **Anatomy boundary** — thường là một interface cộng một cặp DTO (một cho input, một cho output).
* **Boundary crossing** — data cross boundary được convert sang form tiện cho module trong. **Module trong không bao giờ biết form của module ngoài.**
* **Plugin architecture** — module ngoài plug vào module trong. Module trong định nghĩa interface; module ngoài implement. **Component "main" (composition root) wire chúng.**

### Policy và Level

* **Policy** — business rule. Thay đổi chậm, giá trị cao.
* **Detail** — framework, database, UI. Thay đổi nhanh, giá trị thấp.
* **Level** = khoảng cách từ input và output. Use case là *high-level*; HTTP controller là *low-level*. Source-code dependency flow từ low level → high level.

---

## 8. Diagram Clean Architecture

Đồ hoạ iconic của sách. Bốn vòng đồng tâm:

```
                ┌──────────────────────────────────────────────┐
                │ Frameworks & Drivers  (UI, DB, Web, Device)  │
                │ ┌──────────────────────────────────────┐     │
                │ │ Interface Adapters  (Controller,     │     │
                │ │   Gateway, Presenter)                │     │
                │ │ ┌──────────────────────────────┐     │     │
                │ │ │ Application Business Rule    │     │     │
                │ │ │   (Use Case / Interactor)    │     │     │
                │ │ │ ┌─────────────────────┐      │     │     │
                │ │ │ │ Enterprise Business │      │     │     │
                │ │ │ │   Rule (Entity)     │      │     │     │
                │ │ │ └─────────────────────┘      │     │     │
                │ │ └──────────────────────────────┘     │     │
                │ └──────────────────────────────────────┘     │
                └──────────────────────────────────────────────┘

           Source-code dependency chỉ trỏ VÀO TRONG ─────►
```

### 4 vòng

| Vòng                          | Cái gì sống ở đây                                                            | Volatility       |
| ----------------------------- | ---------------------------------------------------------------------------- | ---------------- |
| **Entity** (trong cùng)       | Business rule enterprise-wide. Object thuần với method.                       | Chậm nhất        |
| **Use Case**                  | Business rule application-specific. Orchestrate Entity.                       | Chậm             |
| **Interface Adapter**         | Controller, presenter, gateway — dịch giữa use case và bên ngoài.            | Nhanh hơn        |
| **Frameworks & Drivers** (ngoài cùng) | Web framework, ORM, SDK third-party, code browser.                    | Nhanh nhất       |

### Boundary crossing — input boundary, output boundary

Sách vẽ **mũi tên cho runtime control flow** (có thể trỏ cả hai hướng) và **mũi tên cho source-code dependency** (phải trỏ vào trong). Khi control cần rời vòng trong, vòng trong định nghĩa **output port** (interface) mà vòng ngoài implement.

Một interaction use-case điển hình:

```csharp
// Vòng Use Case — định nghĩa contract nó cần (output port)
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Bản thân use case
public sealed class PlaceOrderUseCase
{
    private readonly IOrderRepository _orders;
    public PlaceOrderUseCase(IOrderRepository orders) => _orders = orders;

    public async Task<Guid> Execute(PlaceOrderRequest req, CancellationToken ct)
    {
        var order = Order.Place(req.CustomerId, req.Items);
        await _orders.AddAsync(order, ct);
        return order.Id;
    }
}

// Vòng Interface Adapter — implement port
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => _db.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
    public Task AddAsync(Order order, CancellationToken ct) => _db.AddAsync(order, ct).AsTask();
}

// Composition Root (ngoài cùng) — nơi DUY NHẤT wire concrete sang abstract
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<PlaceOrderUseCase>();
```

`PlaceOrderUseCase` không biết gì về EF Core, ASP.NET, hay HTTP. Có thể unit-test với in-memory repository trong microsecond.

→ Cross-reference: [`../dot-net/docs/architectural-style/clean-architecture-vi.md`](../dot-net/docs/architectural-style/clean-architecture-vi.md) cho deep-dive implementation .NET.

---

## 9. Dependency Rule

> **Source code dependency chỉ trỏ vào trong, hướng về policy cấp cao hơn.**

Stated tiêu cực: **tên khai báo trong vòng ngoài không được mention trong vòng trong.** Không trong code, không trong comment, không trong type signature.

Khi programmer trong vòng trong cần invoke code ở vòng ngoài, họ không. Thay vào:

1. Họ khai báo **interface** trong vòng trong mô tả cái họ cần.
2. Vòng ngoài **implement** interface đó.
3. Composition root wire chúng lúc startup.

Đây là **Dependency Inversion** ở form architectural nhất. Tại moment compile, mũi tên dependency được đảo từ mũi tên control-flow tự nhiên:

```
Control flow:        Use Case  ──►  IRepository  ──►  EfRepository
Source dependency:   Use Case  ◄────IRepository  ◄────EfRepository
                                    (vòng trong)     (vòng ngoài)
```

Một khi bạn internalize bức tranh đó, mọi thứ khác trong sách follow.

---

## 10. Boundary — Logical, Local, và Full

Bạn không luôn cần boundary mức process đầy đủ. Sách list các cấp tách, sắp xếp theo cost:

| Loại Boundary            | Là gì                                                                | Cost           |
| ------------------------ | ------------------------------------------------------------------- | -------------- |
| **Source-level**         | Chỉ interface và DTO — cùng project, cùng process                    | Thấp           |
| **Deployment-level**     | DLL / JAR / package riêng — vẫn cùng process                         | Trung bình     |
| **Thread-level**         | Thread khác — shared memory, không IPC                               | Trung bình     |
| **Local process**        | Process riêng trên cùng máy — cần IPC                                | Cao            |
| **Service-level**        | Máy khác — protocol mạng                                             | Cao nhất       |

**Khởi đầu với source-level boundary**, rồi harden khi pain xuất hiện. Đi thẳng sang microservices ("full service boundary từ ngày một") là lỗi over-engineering phổ biến nhất.

---

## 11. Humble Object Pattern

Trick làm Clean Architecture *testable*: split bất cứ thứ gì khó test thành hai mảnh:

* Mảnh **humble** — chỉ bits tương tác với thế giới (UI rendering, DB connection, network I/O, threading). Logic tối thiểu, không test hoặc test bằng tay.
* Mảnh **testable** — logic, pure. Dễ unit-test.

Ví dụ:

* Controller là humble object; use case handler nó dispatch tới là testable.
* `DbContext` là humble object; domain code mutate entity là testable.
* View WPF/Blazor là humble object; view model / presenter là testable.

```csharp
// Humble: controller. Chỉ HTTP translation sống ở đây.
[ApiController, Route("orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly PlaceOrderUseCase _useCase;
    public OrdersController(PlaceOrderUseCase useCase) => _useCase = useCase;

    [HttpPost]
    public async Task<IActionResult> Post(PlaceOrderRequest req, CancellationToken ct)
    {
        var id = await _useCase.Execute(req, ct);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
}

// Testable: use case. Logic thuần. Unit test không cần web host.
public sealed class PlaceOrderUseCase { /* ... xem earlier ... */ }
```

Humble Object pattern là cách bạn **honor dependency rule** ở framework boundary mà không sacrifice testability.

---

## 12. Phần VI — Detail: Database, Web, Framework

Các chương ngạc nhiên và được trích dẫn nhiều nhất.

### Database là Detail

Database **không phải** foundation của kiến trúc. Là I/O device. Business rule không quan tâm data sống trong SQL Server, Postgres, Cosmos DB, S3, hay file CSV. Việc ngành conflate "application" và "database nó nói chuyện" là tai nạn lịch sử, không phải sự thật design.

Hệ quả thực hành: **định nghĩa repository interface trong vòng Use-Case**, implement trong Infrastructure. Database thành pluggable.

### Web là Detail

Web framework — ASP.NET, Spring MVC, Express, Rails — chỉ là một trong nhiều cơ chế delivery khả thi. Business rule nên deliverable qua HTTP, gRPC, CLI, scheduled job, hoặc test harness, không có thay đổi.

Hệ quả thực hành: **vòng Use-Case nhận input thuần và return output thuần.** Controller dịch; không compute.

### Framework là Detail

Framework là tool bạn **dùng**, không phải base class bạn inherit từ. Khoảnh khắc domain entity inherit từ class DbContext-aware, bạn đã **kết hôn với framework**. Ly hôn sẽ đau đớn và đắt.

> Analogy hôn nhân của Uncle Bob: *"Framework powerful và rất hữu ích. Nhưng chúng không phải architecture. Chúng là tool. Đừng yêu framework. Đừng cưới chúng. Stay sceptical. Adopt framework, nhưng giữ chúng ở khoảng cánh tay."*

Hệ quả thực hành: **đừng reference type framework từ Entity hoặc Use Case của bạn.** Framework sống trong vòng ngoài. Luôn luôn.

---

## 13. Screaming Architecture

> *"Architecture của bạn nên thét intent của hệ thống."*

Nếu người lạ mở source tree của bạn, tên folder top-level nên mô tả **hệ thống làm gì**, không phải **framework nào nó dùng**:

❌ Layout thét framework:
```
src/
  Controllers/
  Services/
  Models/
  Repositories/
```

✅ Layout thét domain:
```
src/
  Ordering/
  Billing/
  Shipping/
  Inventory/
```

Hệ thống health-records nên thét "Healthcare", không phải "ASP.NET Core". Architecture có ý nghĩa communicate **purpose** trước, *technology* sau.

---

## 14. Test Boundary

Test là phần của hệ thống, và theo cùng dependency rule:

* **Unit test** test code trong vòng trong — không cần infrastructure.
* **Integration test** test vòng ngoài tương tác với I/O thật.
* **Functional/E2E test** drive cả stack từ bên ngoài.

Hai cạm bẫy sách cảnh báo:

* **Test fragile.** Nếu test vỡ mỗi lần bạn refactor, test couple với *implementation*, không phải behavior. Fix: test qua boundary interface, không qua class internal.
* **Bẫy "Testing API".** Một số team expose internal chỉ để test. Resist. Test nên drive public API; nếu test khó, API sai.

> Cross-reference: [`unit-testing-vladimir-khorikov-vi.md`](./unit-testing-vladimir-khorikov-vi.md) — sách Khorikov là đọc companion tự nhiên về test design.

---

## 15. Clean Embedded Architecture

Một chương surprise người nghĩ "embedded" nghĩa là "không architecture". Uncle Bob lập luận embedded system *đặc biệt* cần Clean Architecture vì **firmware thường outlive hardware** vài năm hoặc thập kỷ. Cùng discipline dependency-inversion áp dụng:

* HAL (hardware abstraction layer) là boundary.
* OS là detail.
* Driver là detail.
* Application logic là policy.

Bài học generalize: *mọi* hệ thống có detail thay đổi nhanh hơn policy. Bảo vệ policy.

---

## 16. Take-Away bức tranh lớn

1. **Architecture là shape của hệ thống.** Shape đó tồn tại để maximize productivity của team qua lifetime project.
2. **Behavior khẩn; structure quan trọng.** Phần lớn team sacrifice structure cho urgency, rồi chìm trong technical debt.
3. **Policy phải độc lập với detail.** Đây là toàn bộ điểm.
4. **Dependency Rule** — dependency trỏ vào trong — là hệ quả operational của (3).
5. **OO's gift là polymorphism**, là cách ta đảo source-code dependency ngược với control flow.
6. **SOLID mô tả class; 6 component principle mô tả component.** Cùng nhau chúng làm Clean Architecture khả thi cơ học.
7. **Boundary là interface + DTO.** Sống trong vòng trong; vòng ngoài implement.
8. **Database, web, UI, framework đều là detail.** Defer chúng. Giữ chúng ở khoảng cánh tay.
9. **Humble Object pattern** cô lập bits không testable để phần còn lại có thể unit-test.
10. **Architecture nên *thét* business**, không technology.
11. **Bạn sẽ không đúng từ ngày một.** Architecture là iterative. Architect tốt nhất *defer* quyết định tới moment muộn nhất có trách nhiệm.

---

## 17. Sách này fit với sách khác ra sao

| Sách                                                         | Quan hệ                                                                       |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| Eric Evans — *Domain-Driven Design* (Blue Book, 2003)        | Bổ sung. DDD cho bạn **cái gì** (model domain); Clean Architecture cho **ở đâu** (vòng nào nó sống). |
| Alistair Cockburn — *Hexagonal Architecture* (2005)          | Tổ tiên trực tiếp. Hexagonal là Clean Architecture với ít prescription về layer internal. |
| Jeffrey Palermo — *Onion Architecture* (2008)                | Cùng essence như Clean. Vocabulary khác.                                      |
| Vaughn Vernon — *Implementing Domain-Driven Design* (2013)   | Companion thực hành dùng Clean-style layer trong mọi ví dụ.                   |
| Martin Fowler — *PoEAA* (2002)                                | Cũ hơn. Một số pattern của nó (Repository, Unit of Work) sống trong vòng Interface Adapters của Clean Architecture. |
| Robert C. Martin — *Clean Code* (2008)                       | Cleanliness mức class. *Clean Architecture* là counterpart structural.       |
| Khorikov — *Unit Testing* (2020)                              | Cho bạn cách test vòng trong tốt — explicit về cái gì mock và không.         |

---

## 18. Khuyến nghị thứ tự đọc

* **Lần đầu:** đọc thẳng qua. Phần II–IV (paradigm, SOLID, component principle) cảm giác chậm nhưng pay off trong Phần V.
* **Reference mode:** Phần V (Architecture) và chương về Database / Web / Framework như detail là section được trích dẫn nhiều nhất.
* **Pair với:** `architectural-style/clean-architecture-vi.md` trong repo này cho implementation .NET cụ thể. Chúng reinforce nhau — sách cho *tại sao*, doc repo cho *cách*.

---

## 19. Hiểu lầm phổ biến

Người skim sách thường rút ra kết luận sai. Position thực của sách:

| Hiểu lầm                                                                | Thực tế                                                                                                    |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| "Clean Architecture cần 4 project tên Entity/UseCase/Adapter/Framework." | Tên không quan trọng. Dependency Rule quan trọng. Diagram 4 ring là *concept*, không phải template.        |
| "Clean Architecture cấm EF Core / Hibernate / Spring Data."             | Không. Cấm chúng *trong vòng Entity và Use Case*. Chúng sống trong Infrastructure (vòng ngoài cùng).        |
| "Clean Architecture giống DDD."                                          | DDD model *domain*; Clean Architecture đặt *layer*. Pair tốt nhưng giải vấn đề khác.                       |
| "Mọi application cần Clean Architecture."                                | Không. App CRUD nhỏ và prototype sống ngắn over-engineer bởi nó. Sách explicit: trả structural cost khi cost of change cao. |
| "Microservices = Clean Architecture."                                    | Không. Microservices là chiến lược deployment. Microservice có thể Clean-architected hoặc rối rắm.        |

---

## 20. Tham khảo

* Robert C. Martin — *Clean Architecture: A Craftsman's Guide to Software Structure and Design*, Prentice Hall, 2017.
* Robert C. Martin — *Clean Code: A Handbook of Agile Software Craftsmanship*, Prentice Hall, 2008.
* Alistair Cockburn — [*Hexagonal Architecture*](https://alistair.cockburn.us/hexagonal-architecture/) (2005).
* Jeffrey Palermo — [*The Onion Architecture* (blog series)](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/) (2008).
* Jason Taylor — [Clean Architecture Solution Template](https://github.com/jasontaylordev/CleanArchitecture).
* Microsoft — [eShopOnWeb reference application](https://github.com/dotnet-architecture/eShopOnWeb).
* Cross-reference trong repo này: [`../dot-net/docs/architectural-style/clean-architecture-vi.md`](../dot-net/docs/architectural-style/clean-architecture-vi.md).
