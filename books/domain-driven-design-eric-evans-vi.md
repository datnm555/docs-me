# Domain-Driven Design — Tóm tắt

> **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software*, Addison-Wesley, 2003. Thường gọi là **"Blue Book"**.

> **Tại sao sách này quan trọng:** text foundational cho modeling domain phức tạp trong software OO. Evans introduce vocabulary mà ngành vẫn dùng hằng ngày — *Ubiquitous Language, Bounded Context, Aggregate, Domain Event, Anti-Corruption Layer, Core Domain*. Blue Book dense, đôi khi meandering, và 500+ trang; tóm tắt này distill argument của nó thành cấu trúc phần lớn engineer cần biết, với example bằng từ ngữ của tôi và code gốc.

> **Tóm tắt này cover:** triết lý model-driven design, Ubiquitous Language, các tactical pattern (Entity, Value Object, Aggregate, Factory, Repository, Domain Service, Module, Domain Event), các strategic pattern (Bounded Context, Context Map, subdomain Core/Supporting/Generic, Anti-Corruption Layer, và phần còn lại), nguyên lý supple design, và cách sách được nhận và mở rộng (Vernon, EventStorming) từ 2003.

> 🇻🇳 Phiên bản tiếng Việt. English: [`domain-driven-design-eric-evans.md`](./domain-driven-design-eric-evans.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Tóm tắt Blue Book của Eric Evans 2003 — Ubiquitous Language, Model-Driven Design, tactical pattern (Entity, Value Object, Aggregate, Domain Service, Repository, Factory, Domain Event), và strategic pattern (Bounded Context, Context Map, Anti-Corruption Layer, subdomain Core/Supporting/Generic).
- **Tại sao** — Cho bạn vocabulary và discipline để model domain phức tạp sao cho code match cách domain expert nói — loại bỏ friction translation produce 90% bug "nhưng code của tôi làm sai thứ".
- **Khi nào** — Domain phức tạp và trung tâm với business (banking, insurance, logistics, healthcare); hai team dùng cùng từ cho thứ khác; integrating với mớ legacy.
- **Ở đâu** — Pair với `dot-net/docs/ddd/` (treatment .NET của tactical + strategic pattern), `books/clean-architecture-robert-martin-vi.md` (shell architectural mà domain layer DDD sống bên trong), và `books/building-microservices-sam-newman-vi.md` (một bounded context per service).

---

## Mục lục

1. [Về cuốn sách](#1-về-cuốn-sách)
2. [Claim trung tâm của Evans](#2-claim-trung-tâm-của-evans)
3. [Phần I — Putting the Domain Model to Work](#3-phần-i--putting-the-domain-model-to-work)
4. [Ubiquitous Language](#4-ubiquitous-language)
5. [Model-Driven Design](#5-model-driven-design)
6. [Phần II — Các Tactical Pattern](#6-phần-ii--các-tactical-pattern)
7. [Layered Architecture & Anti-Pattern Smart UI](#7-layered-architecture--anti-pattern-smart-ui)
8. [Entity](#8-entity)
9. [Value Object](#9-value-object)
10. [Domain Service](#10-domain-service)
11. [Module](#11-module)
12. [Aggregate](#12-aggregate)
13. [Repository](#13-repository)
14. [Factory](#14-factory)
15. [Domain Event (Thêm sau)](#15-domain-event-thêm-sau)
16. [Phần III — Refactoring Toward Deeper Insight](#16-phần-iii--refactoring-toward-deeper-insight)
17. [Supple Design](#17-supple-design)
18. [Phần IV — Strategic Design](#18-phần-iv--strategic-design)
19. [Bounded Context](#19-bounded-context)
20. [Context Mapping — Pattern quan hệ](#20-context-mapping--pattern-quan-hệ)
21. [Distillation — Core, Supporting, Generic Subdomain](#21-distillation--core-supporting-generic-subdomain)
22. [Large-Scale Structure](#22-large-scale-structure)
23. [Take-Away bức tranh lớn](#23-take-away-bức-tranh-lớn)
24. [Cái sách không cover (Và Companion của Vernon)](#24-cái-sách-không-cover-và-companion-của-vernon)
25. [Sách này fit Repo ra sao](#25-sách-này-fit-repo-ra-sao)
26. [Khuyến nghị thứ tự đọc](#26-khuyến-nghị-thứ-tự-đọc)
27. [Hiểu lầm phổ biến](#27-hiểu-lầm-phổ-biến)
28. [Tham khảo](#28-tham-khảo)

---

## 1. Về cuốn sách

* **Tiêu đề:** *Domain-Driven Design: Tackling Complexity in the Heart of Software*
* **Tác giả:** Eric Evans (consultant, founder Domain Language)
* **NXB:** Addison-Wesley, 2003
* **Độ dài:** ~560 trang
* **Audience:** designer, architect, senior developer làm trên hệ thống mà *bản thân domain* phức tạp — banking, insurance, healthcare, logistics, supply chain, ERP.

Sách famously khó đọc end-to-end. Evans interleave nguyên lý, narrative, code, và phản ánh theo cách reward đọc lại. Content được trích dẫn nhiều nhất split giữa **tactical pattern** (Phần II) và **strategic design** (Phần IV). Nhiều engineer đọc hai phần này và ignore phần còn lại.

---

## 2. Claim trung tâm của Evans

> **Trái tim của software là khả năng giải vấn đề liên quan tới domain. Mọi feature khác, dù vital, support mục đích thiết yếu này.**

Lập luận của Evans unfold:

1. Phức tạp software có nguồn: bản thân **domain**.
2. Cách manage phức tạp đó là develop **deep model** của domain.
3. Model được share giữa domain expert và developer qua **Ubiquitous Language**.
4. Model phải được express *trực tiếp* trong code — không dịch sang "technical model" riêng drift khỏi domain language.
5. Để giữ model coherent ở scale, vẽ **Bounded Context** explicit, document quan hệ chúng trong **Context Map**, và bảo vệ phần valuable nhất (**Core Domain**) với design có discipline.

Mọi thứ trong sách serve 5 idea này.

---

## 3. Phần I — Putting the Domain Model to Work

### Crunch knowledge

Evans khởi đầu với story: dev làm trên domain họ không hiểu produce code technically đúng nhưng vô dụng với business. Cure là **knowledge crunching** — relentlessly work với domain expert để expose rule implicit đằng sau rule explicit.

Takeaway: **modeling là collaboration**, không phải hoạt động solo. Model là medium qua đó developer và domain expert negotiate understanding.

### Câu hỏi "what is a model"

Trong DDD, **domain model** không phải UML diagram, không phải database schema, không phải class hierarchy. Là **cấu trúc conceptual chia sẻ** mà cả code và conversation tham chiếu tới. Diagram capture một snapshot của nó; language là form canonical.

---

## 4. Ubiquitous Language

Idea ảnh hưởng nhất trong sách.

> **Ubiquitous Language** là *ngôn ngữ rigorous, đơn lẻ* share bởi developer, domain expert, tester, và stakeholder — dùng trong conversation, document, ticket, code, và test.

Nơi phần lớn team có *jargon gap* giữa business và engineering, UL loại bỏ nó:

* Domain expert nói "underwrite a policy".
* Ticket nói "underwrite a policy".
* Test scenario chấp nhận nói "underwrite a policy".
* Class là `Policy` với method `Underwrite(...)`.
* Git commit nói "fix bug in policy underwriting".

Khi language nhất quán qua mọi artifact, **friction translation biến mất**, và model có thể evolve an toàn vì mọi người share cùng vocabulary.

### Cơ chế thực hành

* Duy trì **glossary** per Bounded Context.
* **Refactor tên không nương tay** khi vocabulary domain expert shift. Rename class rẻ; nói qua nhau hàng tháng không rẻ.
* **Dùng language trong code review.** "Tại sao gọi cái này là `Container`? Domain expert gọi là Shipment."
* **Một language per Bounded Context, không một language cho toàn company.** "Customer" trong Sales và "Customer" trong Support là concept khác; ép chúng vào một và bạn destroy model.

---

## 5. Model-Driven Design

Evans lập luận cho binding chặt giữa **conceptual model** và **code**. Hai cách team fail cái này:

* **Model là diagram không ai update.** Drift tích luỹ; code thành single source of truth, và diagram lie.
* **Model và code dùng vocabulary khác.** Model nói "underwrite"; code có `update_status_3()`.

Remedy DDD:

* Code được *express trong language của model*.
* Tên class, tên method, tên biến — tất cả match Ubiquitous Language.
* Model evolve qua refactoring; code follow.

### Hands-On Modeler

Chương controversial: Evans lập luận **modeler phải viết code**, và **coder phải tham gia modeling**. Separation truyền thống — analyst produce "design document", coder implement nó — produce model chết. Team own model own code; vice versa.

---

## 6. Phần II — Các Tactical Pattern

Section được trích dẫn nhiều nhất. Đây là building block của model-driven design bên trong một Bounded Context.

Catalog pattern:

* **Layered Architecture** — separation of concern (UI / Application / Domain / Infrastructure).
* **Entity** — có identity tồn tại qua thay đổi.
* **Value Object** — định nghĩa bởi attribute; immutable; không identity.
* **Domain Service** — behavior domain không tự nhiên thuộc Entity hoặc Value Object.
* **Module** — group concept domain liên quan.
* **Aggregate** — cluster Entity và Value Object treat như một unit cho consistency.
* **Repository** — abstraction collection-like cho retrieving Aggregate.
* **Factory** — encapsulate creation phức tạp của Aggregate.
* **Domain Event** — cái gì đã xảy ra trong domain (Evans thêm trong paper 2014 sau; giờ routine include trong DDD).

> Cross-reference: [`../dot-net/docs/ddd/tactical-patterns-vi.md`](../dot-net/docs/ddd/tactical-patterns-vi.md) cho full treatment .NET với code C#.

---

## 7. Layered Architecture & Anti-Pattern Smart UI

Sách define **layered architecture** với 4 layer kinh điển:

| Layer              | Trách nhiệm                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| **User Interface** | Display, accept user command                                                 |
| **Application**    | Orchestrate use case, manage transaction và security; không business rule  |
| **Domain**         | Domain model — Entity, Value Object, Service, Event                         |
| **Infrastructure** | Persistence, messaging, file I/O, external API                              |

Rule strict: mỗi layer phụ thuộc chỉ layer dưới.

### Anti-Pattern Smart UI

Evans candid: **cho app CRUD đơn giản, throwaway, full DDD là overkill.** "Smart UI" — đặt mọi thứ (UI, business logic, persistence) vào một chỗ — là lựa chọn *legitimate* khi project nhỏ và domain shallow. Nhưng bạn sacrifice khả năng evolve. Sách explicit: chọn Smart UI knowingly, không phải accident.

> Câu được overlooked nhất của Blue Book: *"Đừng áp dụng Domain-Driven Design cho vấn đề không cần nó."*

---

## 8. Entity

> **Entity** là object có **identity distinct** chạy qua thời gian và representation khác nhau.

Hai object `Order` *là cùng entity* nếu chúng có cùng `Id`, kể cả nếu attribute khác (status khác, line item khác, total khác). Hai object `Money` với cùng amount và currency *bằng nhau* — chúng là Value Object, không Entity.

### Implement identity

```csharp
public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;

    public override bool Equals(object? obj) =>
        obj is Entity<TId> other && EqualityComparer<TId>.Default.Equals(Id, other.Id);

    public override int GetHashCode() => Id?.GetHashCode() ?? 0;
}

public sealed class Customer : Entity<Guid>
{
    public string Name { get; private set; }
    public EmailAddress Email { get; private set; }

    private Customer() { /* cho EF */ }

    public Customer(Guid id, string name, EmailAddress email)
    {
        Id = id;
        Name = name;
        Email = email;
    }

    public void ChangeEmail(EmailAddress newEmail)
    {
        if (newEmail == Email) return;
        Email = newEmail;
        // Có thể raise CustomerEmailChanged domain event ở đây
    }
}
```

### Rule của Evans

* **Identity được assign lúc tạo** và **không bao giờ thay đổi**.
* **Dùng identifier natural khi có thể** (ISBN, SSN). Nếu không, generate (`Guid`, autoincrement).
* **Identity là attribute duy nhất quan trọng cho equality**. Hai entity với state giống hệt nhưng id khác là *khác*; hai entity với cùng id là *cùng*.
* **Behavior thuộc entity**, không service. `customer.ChangeEmail(...)`, không `customerService.ChangeCustomerEmail(customer, ...)`.

---

## 9. Value Object

> **Value Object** là object mô tả characteristic — định nghĩa chỉ bởi attribute, **không có identity conceptual**.

Money, date, address, color, phone number, range, coordinate. Hai instance `Money(100, "USD")` interchangeable; không có notion *cái* hundred dollar này so với *cái* hundred dollar kia.

### Ba rule

1. **Immutable.** Tạo xong, không bao giờ thay đổi. Operation trả về instance mới.
2. **Equality theo value.** Mọi attribute contribute equality và hash code.
3. **Encapsulate business rule.** Validation, formatting, arithmetic — chúng sống ở đây, không scattered khắp codebase.

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money
    {
        if (Amount < 0) throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(Currency)) throw new ArgumentException("Currency required");
        Currency = Currency.ToUpperInvariant();
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(decimal factor) => this with { Amount = Math.Round(Amount * factor, 2) };

    public static Money operator +(Money a, Money b) => a.Add(b);
}

public sealed record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email", nameof(value));
        Value = value.ToLowerInvariant();
    }

    public override string ToString() => Value;
}
```

### Tại sao Value Object quan trọng đến vậy

Không có chúng, mọi method nhận `string email, decimal price, string currency` và re-validate. Logic spread khắp nơi. Với Value Object, **bản thân type system enforce correctness**:

* Method nhận `EmailAddress` không thể được call với email invalid.
* Method nhận `Money` không thể multiply USD amount với EUR exchange rate nhầm.

Đây là cure cho **primitive obsession** — một trong smell trong *Refactoring* mà DDD eliminate bằng construction.

### Heuristic Evans

> **Ưu tiên Value Object.** Dùng Entity chỉ khi bạn thực sự cần identity. Phần lớn team over-use Entity và kết thúc với object bloated, identity-laden đại diện cho data nên là Value Object.

---

## 10. Domain Service

Khi một mảnh behavior domain **không tự nhiên thuộc về single Entity hoặc Value Object**, model nó như **Domain Service**.

Heuristic: nếu operation involve *nhiều Aggregate từ concept khác* (Customer + Order + Promotion), và đặt nó lên bất kỳ cái nào cảm giác forced — đó là Domain Service.

```csharp
public sealed class OvercreditCheck
{
    public bool IsApproved(Customer customer, Order proposed, IReadOnlyList<Order> outstanding)
    {
        var outstandingTotal = outstanding.Sum(o => o.Total.Amount);
        return outstandingTotal + proposed.Total.Amount <= customer.CreditLimit.Amount;
    }
}
```

### Domain Service vs. Application Service

Phân biệt quan trọng Evans labor:

| Layer                 | Service                                                                   |
| --------------------- | ------------------------------------------------------------------------- |
| **Domain Service**    | Pure domain logic. Không I/O, không transaction, không kiến thức framework. |
| **Application Service** | Orchestration use-case. Load aggregate từ repository, call domain logic, commit qua Unit of Work, publish event. |

`PlaceOrderHandler` (Application) call `PricingPolicy` (Domain Service) để compute price, rồi save Order qua Repository của nó. Separation sạch.

---

## 11. Module

**Module** (gọi *package* hoặc *namespace* trong code) group concept domain liên quan. Rule của Evans: **module nên reflect model**, không technical concern.

Tệ: `Controllers/`, `Services/`, `Models/`, `Repositories/`.

Tốt: `Orders/`, `Pricing/`, `Shipping/`, `Customers/`.

Đây là cùng idea với **Screaming Architecture** (Robert Martin), nhưng Evans pre-date nó.

---

## 12. Aggregate

Pattern được hiểu lầm nhiều nhất trong sách.

> **Aggregate** là cluster object associated (Entity và Value Object) treat như **unit cho mục đích data change**. Cluster có single **root Entity** mà qua đó thế giới ngoài tương tác với nó.

### Tại sao aggregate tồn tại

Trong domain phức tạp, business rule thường span nhiều object. Không có boundary explicit, mọi transaction risk corrupt invariant qua model. Aggregate giải cái này bằng:

* Bound **transactional consistency** — trong một aggregate, invariant holds *immediately* sau mỗi transaction.
* Cho phép **eventual consistency** *giữa* aggregate — aggregate khác nhau có thể inconsistent trong period ngắn.

### Rule

1. **Một aggregate root per aggregate.** Root là Entity duy nhất được tham chiếu externally trong aggregate.
2. **Object ngoài chỉ tham chiếu root** — không bao giờ Entity internal. Entity internal không có identity dùng được externally.
3. **Object ngoài tham chiếu aggregate khác chỉ bằng ID**, không direct object reference.
4. **Một transaction modify một aggregate.** Nếu use case phải modify hai aggregate, làm chúng trong transaction riêng và coordinate qua Domain Event hoặc Saga.
5. **Aggregate root enforce invariant** của aggregate.

```csharp
public sealed class Order : Entity<Guid>     // ← Aggregate root
{
    private readonly List<OrderLine> _lines = new();

    public Guid CustomerId { get; }            // Reference aggregate khác bằng ID (không direct)
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Money Total => _lines.Aggregate(
        new Money(0m, "USD"),
        (sum, line) => sum.Add(line.LineTotal));

    private Order() { }

    public static Order Place(Guid customerId)
    {
        return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
    }

    public void AddLine(Guid productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify a submitted order");
        if (quantity <= 0)
            throw new ArgumentException("Quantity must be positive");

        _lines.Add(new OrderLine(productId, quantity, unitPrice));   // Entity internal
    }

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Cannot submit empty order");
        Status = OrderStatus.Submitted;
        // Raise domain event: OrderSubmitted
    }
}

public sealed class OrderLine : Entity<Guid>   // Entity internal — chỉ Order tạo cái này
{
    public Guid ProductId { get; }
    public int Quantity { get; }
    public Money UnitPrice { get; }
    public Money LineTotal => UnitPrice.Multiply(Quantity);

    internal OrderLine(Guid productId, int quantity, Money unitPrice)
    {
        Id = Guid.NewGuid();
        ProductId = productId;
        Quantity = quantity;
        UnitPrice = unitPrice;
    }
}
```

### Quy tắc thiết kế Aggregate (thường attributed Vaughn Vernon)

* **Aggregate nhỏ.** Aggregate to gây contention và performance chậm.
* **Reference aggregate khác bằng id.** Không bằng navigation property.
* **Một aggregate per transaction.** Nếu cần thay đổi hai, dùng Domain Event.
* **Eventual consistency giữa aggregate.** Hai aggregate briefly inconsistent là bình thường và acceptable.

---

## 13. Repository

> **Repository** cung cấp **abstraction collection-like** cho retrieve và store Aggregate Root. Client tương tác với nó như với collection in-memory.

### Repository nên trông như thế nào

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> FindUnpaidByCustomerAsync(Guid customerId, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}
```

### Rule Evans đặt

1. **Một repository per aggregate root.** Không per table. `IOrderRepository`, không `IOrderLineRepository`.
2. **Method trả về aggregate**, không DTO và không `IQueryable`.
3. **Tên method nói domain language**: `FindUnpaidByCustomer`, không `FindByPredicate`.
4. **Interface sống ở domain layer**; implementation ở infrastructure.
5. **Repository ẩn persistence hoàn toàn.** Domain code dùng repository nên portable qua SQL Server, Postgres, MongoDB, file storage, in-memory — không thay đổi.

### Tại sao Generic Repository (`IRepository<T>`) được coi rộng rãi là anti-pattern trong DDD

* Expose interface CRUD-shaped không có business meaning.
* Method như `Find(Expression<Func<T, bool>>)` push construction query trở lại application code, vi phạm "ẩn data store".
* Thường được áp dụng cho *mọi* entity, bao gồm cái không phải aggregate root — destroy boundary aggregate.

> Cross-reference: [`../dot-net/docs/enterprise-pattern/repository-vi.md`](../dot-net/docs/enterprise-pattern/repository-vi.md) có full discussion Generic Repository vs. proper DDD Repository.

---

## 14. Factory

Khi construct Aggregate phức tạp (nhiều invariant, object liên quan tạo cùng nhau, value derived), encapsulate complexity đó trong **Factory** — hoặc static factory method trên bản thân Aggregate Root, hoặc class Factory dedicated.

```csharp
// Static factory method trên aggregate
public static Order Place(Guid customerId)
{
    return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Draft };
}

// Hoặc Factory dedicated cho creation elaborate hơn
public sealed class OrderFactory
{
    private readonly IClock _clock;
    private readonly IIdGenerator _ids;

    public OrderFactory(IClock clock, IIdGenerator ids)
    {
        _clock = clock;
        _ids = ids;
    }

    public Order CreateFromCart(Customer customer, Cart cart, IReadOnlyList<Promotion> activePromotions)
    {
        if (cart.IsEmpty)
            throw new InvalidOperationException("Cannot place an order from an empty cart");

        var order = Order.Place(customer.Id);
        foreach (var item in cart.Items)
            order.AddLine(item.ProductId, item.Quantity, item.UnitPrice);

        foreach (var promotion in activePromotions.Where(p => p.AppliesTo(order)))
            promotion.Apply(order);

        return order;
    }
}
```

Điểm: **logic construction thuộc một chỗ**, không scattered qua application service.

---

## 15. Domain Event (Thêm sau)

Evans introduce Domain Event trong paper 2014, không trong Blue Book gốc — nhưng pattern giờ được coi canonical DDD.

> **Domain Event** là cái gì đã xảy ra trong domain mà domain expert quan tâm.

Past tense, immutable, raise bởi aggregate như side effect của business operation.

```csharp
public sealed record OrderSubmitted(Guid OrderId, Guid CustomerId, Money Total, DateTime OccurredAt);
public sealed record OrderPaid(Guid OrderId, string TransactionId, DateTime OccurredAt);
public sealed record OrderShipped(Guid OrderId, string TrackingNumber, DateTime OccurredAt);
```

### Cách aggregate raise chúng

```csharp
public abstract class Entity<TId>
{
    private readonly List<INotification> _events = new();
    public IReadOnlyList<INotification> DomainEvents => _events;
    protected void Raise(INotification @event) => _events.Add(@event);
    public void ClearDomainEvents() => _events.Clear();

    public TId Id { get; protected set; } = default!;
}

public sealed class Order : Entity<Guid>
{
    // ... như trên ...

    public void Submit()
    {
        if (_lines.Count == 0)
            throw new InvalidOperationException("Cannot submit empty order");
        Status = OrderStatus.Submitted;
        Raise(new OrderSubmitted(Id, CustomerId, Total, DateTime.UtcNow));
    }
}
```

### Hai loại — domain event vs. integration event

| Loại                 | Scope                          | Subscriber                            |
| -------------------- | ------------------------------ | -------------------------------------- |
| **Domain Event**     | In-process, trong một bounded context | Aggregate khác / read model trong context này |
| **Integration Event** | Cross-service, qua bounded context | Service khác qua message bus           |

Domain event được dispatch sau khi transaction commit. Integration event được publish qua Outbox pattern tới message broker.

> Cross-reference: [`../dot-net/docs/architectural-pattern/outbox-vi.md`](../dot-net/docs/architectural-pattern/outbox-vi.md), [`../dot-net/docs/architectural-style/event-driven-vi.md`](../dot-net/docs/architectural-style/event-driven-vi.md).

---

## 16. Phần III — Refactoring Toward Deeper Insight

Thesis của Evans: **model đầu tiên bạn viết là sai**. Bạn discover model đúng chỉ bằng work với domain hàng tháng. Refactoring không phải tuỳ chọn — là *cơ chế chính* mà model improve.

### Breakthrough

Đôi khi thay đổi nhỏ reveal insight sâu hơn nhiều. Concept mà team đã work quanh (luôn pass cùng parameter, luôn làm cùng special case) đột nhiên thành concept first-class riêng, và mảng code simplify. Evans gọi những moment này **breakthrough**.

### Làm explicit concept implicit

Nguồn lớn nhất của breakthrough: **concept mà domain expert mention nhưng code không name**. Ví dụ: mỗi order có "shipping mode" ảnh hưởng pricing, nhưng code thread nó qua như string ở 12 chỗ. Make `ShippingMode` Value Object explicit, và design trở nên manifest.

### Watch smell

Evans specific về signal để refactor:

* **Tên awkward** — method gọi `process(...)` làm 4 thứ.
* **List parameter** với cùng trio argument mọi nơi — chúng là Value Object thiếu.
* **Conditional dựa trên type** — có lẽ là polymorphism thiếu.
* **"Comment giải thích cái code làm"** — language sai; refactor tên để comment không cần.

---

## 17. Supple Design

Chương 10 — chương dense nhất trong sách, và statement aesthetic của Evans về *code domain tốt trông như thế nào*.

### Nguyên lý

| Nguyên lý                                | Định nghĩa                                                                                                  |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Intention-Revealing Interface**        | Tên communicate *cái gì* và *tại sao*, không *cách*. `customer.MarkAsPreferred()` không `customer.UpdateStatus(3)`. |
| **Side-Effect-Free Function**            | Query không mutate; command mutate nhưng có semantics rõ. Phần lớn operation nên là một hoặc cả hai — không bao giờ sneaky. |
| **Assertion**                            | Invariant của operation explicit. Pre- và post-condition stated trong code (`Guard.Against.Null(...)`) hoặc trong test. |
| **Conceptual Contour**                   | Boundary giữa class match boundary giữa concept trong domain. Resist split/merge concept để fit constraint technical. |
| **Standalone Class**                     | Giảm dependency. Class với ít collaborator dễ lý luận hơn.                                                    |
| **Closure of Operation**                 | Khi operation trên type trả về cùng type (`Money + Money = Money`), composition thành dễ.                    |
| **Declarative Design**                   | Khi cấu trúc domain admit, viết code *mô tả* domain rule — specification, policy object — thay vì thuật toán imperative. |

Supple Design là cái code DDD *expert* feel như — nhỏ, đặt tên, composable, expressive. Các pattern earlier làm Supple Design *khả thi*; chương này là cái cần *aim*.

---

## 18. Phần IV — Strategic Design

Section bị overlooked nhất nhưng arguably valuable nhất của sách. Pay off kể cả nếu bạn adopt zero tactical pattern.

### Tại sao strategic design quan trọng

Tactical pattern work bên trong *một* model. Moment bạn có **hai team**, **nhiều service**, hoặc **legacy system phải integrate**, bạn có nhiều model — và chúng sẽ conflict.

Strategic design cung cấp:

* **Bounded Context** — boundary explicit cho một model.
* **Context Map** — diagram của cách mọi context relate.
* **Distillation** — focus effort modeling của team vào cái quan trọng nhất.
* **Large-Scale Structure** — pattern giữ toàn hệ thống coherent.

> Cross-reference: [`../dot-net/docs/ddd/strategic-patterns-vi.md`](../dot-net/docs/ddd/strategic-patterns-vi.md) cho full treatment .NET.

---

## 19. Bounded Context

> **Bounded Context** là boundary explicit trong đó model cụ thể valid và unambiguous.

Bên trong boundary, term có một nghĩa. Bên ngoài, cùng từ có thể nghĩa khác.

### Tại sao đây là pattern strategic quan trọng nhất

Trong company lớn, từ "Customer" xuất hiện ở vô số nơi:

* Trong **Sales**, Customer có `LeadScore` và `LastQuotedAt`.
* Trong **Billing**, Customer có `PaymentTerms` và `OutstandingBalance`.
* Trong **Support**, Customer có `OpenTickets` và `SatisfactionScore`.
* Trong **Shipping**, Customer là delivery address.

Một "company-wide Customer" model try phục vụ tất cả kết thúc phục vụ không cái nào tốt. **Mỗi context own Customer model riêng.** Đó đúng, không phải duplication.

### Mỗi Bounded Context là

* Đơn vị **model integrity** — model consistent bên trong.
* Đơn vị **team ownership** — thường một team per context.
* Đơn vị **deployment** — thường một microservice per context.
* Đơn vị **Ubiquitous Language** — term unambiguous bên trong.

### Identify Bounded Context

Evans cho heuristic; tác giả sau (Vernon, cộng đồng DDD) thêm cái khác:

* **Boundary linguistic** — chỗ nào từ thay đổi nghĩa?
* **Boundary team** — nhóm nào own cái này?
* **Boundary business capability** — Sales, Billing, Shipping…
* **Boundary subdomain** — Core, Supporting, Generic (section sau).

**EventStorming** (Alberto Brandolini, ~2013) là kỹ thuật post-Evans phổ biến nhất để discover Bounded Context collaboratively. Bản thân Blue Book không cover nó, nhưng mọi DDD practitioner hiện đại dùng nó.

---

## 20. Context Mapping — Pattern quan hệ

**Context Map** là diagram của cách Bounded Context relate. Evans define vài loại quan hệ, mỗi cái với implication cho coordination team và design integration.

### 9 quan hệ canonical

| Pattern                       | Nghĩa                                                                                                    |
| ----------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Partnership**               | Hai team thành công hoặc thất bại cùng nhau; cooperation chặt; phần model share.                          |
| **Shared Kernel**             | Một phần nhỏ model được share explicit giữa hai context. Cost coordination cao.                          |
| **Customer / Supplier**       | Upstream provide; downstream consume; downstream có tiếng nói trong priority upstream.                    |
| **Conformist**                | Downstream conform model upstream nguyên xi. Không dịch. Rẻ, nhưng couple downstream với upstream.        |
| **Anticorruption Layer (ACL)** | Downstream dịch model upstream vào của riêng. Bảo vệ model downstream khỏi mớ hỗn độn upstream.          |
| **Open Host Service**         | Upstream cung cấp protocol/API well-documented cho nhiều consumer downstream.                            |
| **Published Language**         | Ngôn ngữ share well-defined (thường event schema) dùng như lingua franca giữa context.                  |
| **Separate Ways**             | Hai context không integrate gì hết. Đôi khi là call đúng.                                                |
| **Big Ball of Mud**           | Context không có model rõ. Wrap với ACL khi phải integrate.                                              |

### Anti-Corruption Layer trong code

Pattern bạn reach khi phải integrate với legacy system mà model *khác* với của bạn.

```csharp
// Domain model sạch của bạn
public sealed record Address(string Street, string City, string PostalCode, string Country);

// API legacy SOAP expose:
public class LegacyAddressDto
{
    public string? AddressLine1 { get; set; }
    public string? AddressLine2 { get; set; }   // đôi khi chứa city+postal trộn
    public string? CityField { get; set; }
    public string? ZipPostCode { get; set; }
    public int CountryCode { get; set; }        // numeric ISO code
}

// ACL — dịch giữa hai thế giới
public interface IAddressProvider
{
    Task<Address> GetAsync(Guid customerId, CancellationToken ct);
}

public sealed class LegacyAddressAcl : IAddressProvider
{
    private readonly LegacyCustomerClient _legacy;
    private readonly ICountryCodeMapper _countries;

    public LegacyAddressAcl(LegacyCustomerClient legacy, ICountryCodeMapper countries)
    {
        _legacy = legacy;
        _countries = countries;
    }

    public async Task<Address> GetAsync(Guid customerId, CancellationToken ct)
    {
        var dto = await _legacy.GetCustomerAddressAsync(customerId.ToString(), ct);

        var street = string.Join(", ",
            new[] { dto.AddressLine1, dto.AddressLine2 }
                .Where(s => !string.IsNullOrWhiteSpace(s)));

        return new Address(
            Street: street,
            City: dto.CityField ?? "",
            PostalCode: dto.ZipPostCode ?? "",
            Country: _countries.IsoCodeFromLegacy(dto.CountryCode));
    }
}
```

Phần còn lại của codebase dùng `Address` sạch. Mớ hỗn độn sống **chỉ** trong `LegacyAddressAcl`. Khi legacy system thay đổi, chỉ class này thay đổi.

---

## 21. Distillation — Core, Supporting, Generic Subdomain

Một khi bạn có nhiều Bounded Context, bạn spend effort modeling expensive của team ở đâu? Evans trả lời với **classification subdomain**:

| Subdomain                | Là gì                                                                  | Effort modeling |
| ------------------------ | --------------------------------------------------------------------- | --------------- |
| **Core Domain**          | Phần business làm bạn khác với competitor                              | **Tối đa** — đầu tư nặng |
| **Supporting Subdomain** | Quan trọng cho business nhưng không differentiating                     | Vừa phải — build appropriately |
| **Generic Subdomain**    | Vấn đề đã solved; off-the-shelf fine                                     | Tối thiểu — buy, integrate qua ACL |

Ví dụ cho công ty e-commerce:

* **Core**: pricing, recommendation, fraud detection. Bits làm họ tốt hơn competitor.
* **Supporting**: order management, shipping coordination. Custom nhưng không differentiating.
* **Generic**: identity, billing, payment. Dùng off-the-shelf (Auth0, Stripe).

### Tại sao distillation quan trọng

Team finite có capacity modeling finite. Đặt modeler tốt nhất của bạn trên **Generic Subdomain** là lãng phí tragic. Reserve chúng cho **Core Domain** — phần mà modeling sâu produce business advantage.

### Domain Vision Statement

Evans recommend viết **statement ngắn** (paragraph hoặc hai) articulate mục đích của Core Domain. Đọc lại khi nghi ngờ. Giữ attention team vào cái quan trọng.

---

## 22. Large-Scale Structure

Chương strategic cuối. Khi hệ thống huge, individual Bounded Context không đủ — bạn cần pattern để **tổ chức nhiều context coherently**:

* **System Metaphor** — analogy đơn giản giúp mọi người hiểu hệ thống. (Idea XP; Evans endorse.)
* **Responsibility Layer** — partition theo trách nhiệm horizontal trong hoặc qua context.
* **Knowledge Level** — separate behavior instance khỏi behavior type.
* **Pluggable Component Framework** — khi nhiều context plug vào platform chung.

Đây là pattern aspirational nhất và áp dụng ít nhất trong sách. Phần lớn hệ thống hiện đại dùng nguyên lý tổ chức đơn giản hơn (microservices + bounded context + context map).

---

## 23. Take-Away bức tranh lớn

1. **DDD là methodology, không pattern.** Là cách work đặt **domain model** ở trung tâm của design.
2. **Ubiquitous Language là idea quan trọng nhất trong sách.** Adopt nó kể cả nếu bạn adopt nothing else.
3. **Aggregate bound transactional consistency.** Modify một aggregate per transaction; dùng Domain Event giữa chúng.
4. **Value Object là cure cho primitive obsession.** Ưu tiên chúng; dùng Entity chỉ khi identity cần.
5. **Bounded Context cho mỗi model integrity.** Một model per company là fantasy phá clarity.
6. **Context Map là diagram đơn valuable nhất trong hệ thống lớn.** Vẽ nó; label mọi quan hệ; revisit nó.
7. **Distill Core Domain.** Đầu tư effort modeling nơi business differentiate.
8. **Đừng áp dụng DDD cho app CRUD.** Sách explicit: chọn Smart UI knowingly khi domain shallow.
9. **Refactoring là engine.** Model đầu sai; model đúng emerge qua cycle deepening insight.
10. **Pair DDD với Clean / Hexagonal Architecture.** Chúng made for each other; Evans là influence sớm trên cả hai.

---

## 24. Cái sách không cover (Và Companion của Vernon)

Blue Book là 2003. Vài idea DDD giờ canonical được thêm sau:

* **Domain Event** — Evans, paper 2014.
* **EventStorming** — Alberto Brandolini, ~2013.
* **Bounded Context = microservice** — Sam Newman, 2015 (*Building Microservices*).
* **Pairing Event Sourcing / CQRS với DDD** — Greg Young, ~2010.
* **Pattern implementation thực hành** — Vaughn Vernon, *Implementing Domain-Driven Design* ("Red Book"), 2013.
* **DDD-Lite vs. Full DDD** — evolution cộng đồng.

> **Khuyến nghị chuẩn:** đọc Blue Book cho *strategy* và *philosophy*; đọc *Implementing DDD* của Vernon cho *tactic* và *implementation*. Cùng nhau chúng cho bạn picture đầy đủ.

---

## 25. Sách này fit Repo ra sao

| Idea trong sách                             | Doc repo                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Ubiquitous Language & Bounded Context       | [`../dot-net/docs/ddd/strategic-patterns-vi.md`](../dot-net/docs/ddd/strategic-patterns-vi.md)        |
| Entity / Value Object / Aggregate / Service / Factory / Repository | [`../dot-net/docs/ddd/tactical-patterns-vi.md`](../dot-net/docs/ddd/tactical-patterns-vi.md) |
| Repository (và tại sao Generic Repo là anti-DDD) | [`../dot-net/docs/enterprise-pattern/repository-vi.md`](../dot-net/docs/enterprise-pattern/repository-vi.md) |
| Layered Architecture & Clean Architecture   | [`clean-architecture-robert-martin-vi.md`](./clean-architecture-robert-martin-vi.md), [`../dot-net/docs/architectural-style/clean-architecture-vi.md`](../dot-net/docs/architectural-style/clean-architecture-vi.md) |
| Domain Event & Integration Event            | [`../dot-net/docs/architectural-style/event-driven-vi.md`](../dot-net/docs/architectural-style/event-driven-vi.md), [`../dot-net/docs/architectural-pattern/outbox-vi.md`](../dot-net/docs/architectural-pattern/outbox-vi.md) |
| Bounded Context = microservice               | [`building-microservices-sam-newman-vi.md`](./building-microservices-sam-newman-vi.md)                |
| Saga (workflow cross-aggregate)             | [`../dot-net/docs/architectural-pattern/saga-vi.md`](../dot-net/docs/architectural-pattern/saga-vi.md) |

---

## 26. Khuyến nghị thứ tự đọc

Blue Book famously khó đọc sequentially. Path hiện đại phổ biến:

1. **Phần I (Putting the Domain Model to Work)** — ngắn, set philosophy.
2. **Chương 4 (Layered Architecture)** — nhanh.
3. **Phần IV trước, trước Phần II** (controversial; Vernon recommend nó). Context strategic làm tactical pattern hợp lý hơn.
4. **Phần II (Tactical Pattern)** — Entity, Value Object, Aggregate, Repository, Factory, Service.
5. **Chương 10 (Supple Design)** — đọc sau khi practice tactical pattern; hợp lý hơn nhiều khi đó.
6. **Phần III (Refactoring Toward Deeper Insight)** — ngắn và reward đọc lại một khi bạn dùng DDD.

Sách companion đọc parallel:
* Vaughn Vernon — *Implementing Domain-Driven Design* (implementation thực hành).
* Vlad Khononov — *Learning Domain-Driven Design* (2021, hiện đại, accessible).

---

## 27. Hiểu lầm phổ biến

| Hiểu lầm                                                            | Thực tế                                                                                                  |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| "DDD nghĩa là Entity / Value Object / Aggregate / Repository."      | Đó là *tactical pattern* — nửa dễ. Nửa strategic (Bounded Context, Context Map) quan trọng hơn.          |
| "Mọi project nên dùng DDD."                                          | Không. Evans explicit: app CRUD nhỏ không cần. Áp dụng DDD cho domain **phức tạp**.                       |
| "DDD cần microservices."                                            | Không. DDD pre-date microservices một thập kỷ. Modular monolith với Bounded Context là DDD xuất sắc.      |
| "Aggregate có thể tham chiếu nhau trực tiếp."                       | Không. Tham chiếu aggregate khác chỉ bằng **ID**. Dùng Domain Event giữa aggregate.                       |
| "Generic Repository fine trong DDD."                                | Được coi rộng rãi là anti-pattern. Dùng repository specific per aggregate root, với tên method trong Ubiquitous Language. |
| "Bounded Context là về size."                                       | Chúng là về *model integrity*, không size. Một số context lớn, một số nhỏ.                               |
| "DDD = Clean Architecture."                                         | Chúng pair tốt, nhưng giải vấn đề khác. DDD là về modeling domain; Clean là về layer hệ thống.            |
| "Anti-Corruption Layer chỉ là Adapter."                             | Cấu trúc tương tự, conceptually khác. ACL specifically bảo vệ model integrity qua context boundary.       |

---

## 28. Tham khảo

* Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software*, Addison-Wesley, 2003.
* Eric Evans — *Domain-Driven Design Reference* (PDF tóm tắt miễn phí bởi Evans tự mình, 2015). [domainlanguage.com](https://www.domainlanguage.com/ddd/reference/).
* Vaughn Vernon — *Implementing Domain-Driven Design*, Addison-Wesley, 2013 ("Red Book").
* Vaughn Vernon — *Domain-Driven Design Distilled*, Addison-Wesley, 2016 (version ngắn).
* Vlad Khononov — *Learning Domain-Driven Design*, O'Reilly, 2021 (hiện đại, accessible).
* Alberto Brandolini — [*Introducing EventStorming*](https://www.eventstorming.com/) (format workshop để discover Bounded Context).
* Martin Fowler — [*Bounded Context*](https://martinfowler.com/bliki/BoundedContext.html), [*Ubiquitous Language*](https://martinfowler.com/bliki/UbiquitousLanguage.html).
* DDD Crew — [github.com/ddd-crew](https://github.com/ddd-crew) (template cộng đồng cho Context Map, Bounded Context Canvas, EventStorming).
* Cross-reference trong repo này: [`../dot-net/docs/ddd/`](../dot-net/docs/ddd/), [`../dot-net/docs/architectural-pattern/`](../dot-net/docs/architectural-pattern/), [`../dot-net/docs/enterprise-pattern/repository-vi.md`](../dot-net/docs/enterprise-pattern/repository-vi.md).
