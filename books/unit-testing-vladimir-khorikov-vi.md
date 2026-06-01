# Unit Testing — Tóm tắt

> **Vladimir Khorikov** — *Unit Testing: Principles, Practices, and Patterns*, Manning, 2020.

> **Tại sao sách này quan trọng:** treatment rigorous nhất về "cái gì làm unit test *tốt*" publish trong thập kỷ qua. Đóng góp trung tâm của Khorikov là model **bốn trụ cột** — framework nhỏ, testable cho phép score bất kỳ test nào theo bốn property objectively measurable và giải thích *tại sao* mọi controversy testing nổi tiếng (mock vs. classical, integration vs. unit, test gì, mock gì) là tradeoff giữa các property đó.

> **Tóm tắt này cover:** bốn trụ cột, hai trường phái (classical / London), định nghĩa chính xác của "unit", observable behavior vs. implementation detail, khi nào dùng mock, mock chỉ cái bạn own, integration testing, anti-pattern cần retire, và quan hệ giữa architectural style (functional / hexagonal) và chất lượng test. Code example là của tôi, illustrative — C# match sách; bài học translate sạch sang Java, TypeScript, Kotlin.

> 🇻🇳 Phiên bản tiếng Việt. English: [`unit-testing-vladimir-khorikov.md`](./unit-testing-vladimir-khorikov.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Tóm tắt sách Khorikov 2020 — bốn trụ cột của good unit test (regression protection × resistance to refactoring × fast feedback × maintainability), trường phái classical vs. London, vocabulary chính xác mock-vs-stub, và boundary giữa unit và integration test.
- **Tại sao** — End debate kinh điển "có nên dùng mock?" với framework measurable objectively và cho biết tại sao phần lớn suite test over-mocked thành phanh trên project.
- **Khi nào** — Test của bạn chậm, fragile, hoặc break mỗi refactor; introduce mock vào project mới; quyết định cái gì đặt trong unit test vs. integration test vs. E2E.
- **Ở đâu** — Pair với `dot-net/docs/architectural-style/hexagonal-architecture-vi.md` (pattern functional core / imperative shell maximize testability) và `dot-net/docs/enterprise-pattern/ioc-di-vi.md` (shape DI làm test khả thi).

---

## Mục lục

1. [Về cuốn sách](#1-về-cuốn-sách)
2. [Idea lớn](#2-idea-lớn)
3. [Bốn trụ cột của Good Unit Test](#3-bốn-trụ-cột-của-good-unit-test)
4. [Hai trường phái — Classical (Detroit) vs. London (Mockist)](#4-hai-trường-phái--classical-detroit-vs-london-mockist)
5. [Cái gì là "Unit"?](#5-cái-gì-là-unit)
6. [Observable Behavior vs. Implementation Detail](#6-observable-behavior-vs-implementation-detail)
7. [Mock vs. Stub — Phân biệt chính xác](#7-mock-vs-stub--phân-biệt-chính-xác)
8. [Ba loại test — Output-Based, State-Based, Communication-Based](#8-ba-loại-test--output-based-state-based-communication-based)
9. [Anatomy của Good Test (AAA)](#9-anatomy-của-good-test-aaa)
10. [Mocking — Khi nào, cái gì, và cái gì không](#10-mocking--khi-nào-cái-gì-và-cái-gì-không)
11. [Mock chỉ cái bạn Own (Và chỉ ở Boundary)](#11-mock-chỉ-cái-bạn-own-và-chỉ-ở-boundary)
12. [Functional / Hexagonal Architecture và Testability](#12-functional--hexagonal-architecture-và-testability)
13. [Integration Testing](#13-integration-testing)
14. [Work với Time, Random, và Hard Dependency khác](#14-work-với-time-random-và-hard-dependency-khác)
15. [Anti-Pattern cần Retire](#15-anti-pattern-cần-retire)
16. [Take-Away bức tranh lớn](#16-take-away-bức-tranh-lớn)
17. [Sách này fit Repo ra sao](#17-sách-này-fit-repo-ra-sao)
18. [Khuyến nghị thứ tự đọc](#18-khuyến-nghị-thứ-tự-đọc)
19. [Hiểu lầm phổ biến](#19-hiểu-lầm-phổ-biến)
20. [Tham khảo](#20-tham-khảo)

---

## 1. Về cuốn sách

* **Tiêu đề:** *Unit Testing: Principles, Practices, and Patterns*
* **Tác giả:** Vladimir Khorikov (Microsoft MVP, instructor Pluralsight)
* **NXB:** Manning, 2020
* **Độ dài:** ~304 trang
* **Audience:** engineer mid-level và senior đã viết test nhưng nghi ngờ họ viết test *sai*. Sách có opinion — sẽ challenge habit từ article "common best practices".

Sách dùng C# trong example, nhưng mọi nguyên lý language-agnostic. Bài học áp dụng identical cho JUnit/Jest/PyTest/RSpec.

---

## 2. Idea lớn

Phần lớn advice "best practice" về unit test là **vague** hoặc **mâu thuẫn**. ("Dùng mock." "Đừng dùng mock." "Test private method." "Không bao giờ test private method." "Aim 100% coverage." "Coverage không phải quality metric.")

Claim trung tâm của Khorikov: **mọi argument resolve một khi bạn adopt định nghĩa chính xác của test *để làm gì*** — và định nghĩa của anh ấy là **bốn trụ cột**. Với framework đó trong tay, bạn có thể score bất kỳ design test nào và giải thích *tại sao* một approach tốt hơn cái khác cho context nhất định.

Sách, về cơ bản, là tool cho làm test-design decision defensible.

---

## 3. Bốn trụ cột của Good Unit Test

Mọi test, dù viết ra sao, có bốn property measurable:

| Trụ cột                          | Hỏi gì                                                                                       | Value cao nghĩa là…                                              |
| ------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **1. Protection against regression** | Test có catch bug thật khi chúng được introduce?                                       | Test cover code path critical với business và assertion.        |
| **2. Resistance to refactoring** | Test có stay green khi bạn refactor *mà không* thay đổi observable behavior?                | Test không break khi internal structure thay đổi.               |
| **3. Fast feedback**            | Test chạy và bảo bạn về failure bao nhanh?                                                  | Test sub-second; failure đến trong giây, không phải phút.        |
| **4. Maintainability**          | Test khó hiểu và update tới đâu?                                                            | Ngắn, readable, không cần scaffolding để grasp.                  |

### Cruel Theorem

Bạn không thể có cả bốn trụ cột ở max. Ba cái đầu *mutually exclusive ở cực*:

* Test maximize **regression protection** *và* **resistance to refactoring** phải exercise càng nhiều code thật càng tốt (chậm). → mất **fast feedback**.
* Test maximize **regression protection** *và* **fast feedback** phải mock mọi thứ ngoài unit nhỏ (couple heavy với implementation). → mất **resistance to refactoring**.
* Test maximize **resistance to refactoring** *và* **fast feedback** không thể exercise nhiều code thật. → mất **regression protection**.

Mỗi test design là một point trong triangle này. Reframing của Khorikov: **mục tiêu của testing tốt là maximize area của triangle**, không phải maximize một corner.

> **Trụ cột thứ tư (maintainability) không thương lượng.** Test bạn không thể maintain là code chết produce false confidence. Ba cái đầu tradeoff; maintainability phải trả.

### Value của test là product của bốn trụ cột

Khorikov suggest treat value của test khoảng **`P × R × F × M`** — multiplicative, không additive. Test score zero trên trụ cột nào có value zero. Test vô dụng không neutral — chúng cost time để maintain và tạo false confidence.

---

## 4. Hai trường phái — Classical (Detroit) vs. London (Mockist)

Sách defuse một trong những debate dài nhất trong testing.

| Khía cạnh                | **Classical (Detroit / Chicago)**                            | **London (Mockist)**                                          |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------- |
| **Định nghĩa "unit"**     | Unit của *behavior*; có thể span nhiều class                  | Một class                                                     |
| **Isolation**            | Test isolated khỏi *nhau* (không shared state)               | Mỗi class isolated khỏi *collaborator* với mock              |
| **Mock usage**           | Dùng sparingly — chỉ ở boundary của *unmanaged* dependency   | Dùng heavy — replace mọi collaborator với mock                |
| **Test focus**           | Verify *outcome* (state, return value)                       | Verify *interaction* (method nào được call với arg nào)       |
| **Resistance to refactoring** | Cao                                                    | Thấp                                                          |
| **Regression protection** | Cao hơn per test (nhiều code thật exercise hơn)             | Đôi khi thấp hơn (mock cho bug slip through)                  |

Khorikov openly bias về **classical school**. Argument cốt lõi: London school produce test **couple chặt với implementation detail**. Mỗi internal refactor break test. Trụ cột thứ tư — maintainability — collapse, và qua thời gian suite test thành phanh trên project thay vì safety net.

Nói vậy, anh ấy fair về khi mock **là** appropriate (xem §10).

---

## 5. Cái gì là "Unit"?

Corrective quan trọng nhất trong sách.

London school nói: unit = một class. Nên mọi test mock mọi collaborator.

Classical school — và Khorikov — nói: **unit = một unit của *behavior***. Có thể là một class, hoặc ba class luôn work cùng nhau để implement một mảnh business logic. **Bạn test behavior, không test class.**

```csharp
// Behavior: "transfer money giữa account move amount và emit event"
//
// Implementation: cái này có thể span Account, TransferService, EventBus, và Money — bốn class.
// Unit test treat chúng như MỘT unit, mock nothing internal, assert trên observable outcome.

[Fact]
public void Transfer_between_accounts_moves_the_amount_and_emits_event()
{
    var a = new Account("A", initial: new Money(100m, "USD"));
    var b = new Account("B", initial: new Money(0m, "USD"));
    var bus = new InMemoryEventBus();
    var service = new TransferService(bus);

    service.Transfer(a, b, new Money(30m, "USD"));

    Assert.Equal(new Money(70m, "USD"), a.Balance);
    Assert.Equal(new Money(30m, "USD"), b.Balance);
    Assert.Single(bus.Published.OfType<MoneyTransferred>());
}
```

Test exercise bốn class và zero mock — và đó đúng, vì *unit của behavior* span những class đó. Bug trong bất kỳ cái nào fail test này. Pure refactor của bất kỳ cái nào (rename method, đổi private field, extract class) không.

---

## 6. Observable Behavior vs. Implementation Detail

Concept của Khorikov được quote nhiều nhất sau bốn trụ cột.

**Observable behavior** = bất cứ gì client của code có thể detect. Return value, exception, state change client có thể query, call tới dependency mà client explicit register.

**Implementation detail** = cách behavior được achieve internal. Private method, private field, data structure cụ thể, sequence call giữa class internal.

**Rule:** *test nên assert chống observable behavior và ignore implementation detail.*

| Observable                              | Implementation detail                      |
| --------------------------------------- | ------------------------------------------ |
| Return value của public method          | Private helper method                      |
| State change visible qua public getter  | Internal field (`_total`, `_count`)         |
| Exception throw                          | Order của internal method call             |
| Message publish tới event bus           | Internal data structure dùng để queue nó    |
| Row insert vào DB                        | Whether `using` block được dùng            |

Test assert implementation detail break mỗi lần bạn refactor — kể cả khi behavior identical. Đó fail trụ cột **resistance to refactoring**.

Check gut hữu ích: **nếu refactor preserve observable behavior, test phải vẫn pass.** Nếu không, test bạn test implementation detail.

---

## 7. Mock vs. Stub — Phân biệt chính xác

Khorikov insist vocabulary của Meszaros (*xUnit Test Patterns*, 2007). Phần lớn dev conflate terms này; precision quan trọng vì nó thay đổi khi mỗi cái appropriate.

### Test double (overview)

| Loại            | Mục đích                                                              | Ví dụ                                                    |
| --------------- | --------------------------------------------------------------------- | -------------------------------------------------------- |
| **Dummy**       | Fill parameter slot; không bao giờ call                              | `null` hoặc default-construct object pass vào            |
| **Fake**        | Implementation work, simplified cho test                              | In-memory repository                                     |
| **Stub**        | Trả canned data khi query; supply *input* cho SUT                    | `repo.GetById(id)` trả fixed `Order`                    |
| **Spy**         | Record call; assertion check chuyện gì xảy ra                         | Record mọi email "sent"; test assert count               |
| **Mock**        | Object pre-configure *verify* interaction trước                       | "Expect `email.Send` call với X" — fail nếu không        |

### Headline rule của sách

> **Assert trên stub là anti-pattern. Assert trên mock đôi khi appropriate.**

Lý do: **stub** supply *input* cho system under test. Assert SUT *call* stub là assert trên implementation detail — *cách* SUT lấy data, không phải *cái nó làm với data*.

**Mock** verify SUT trigger *side-effect đúng* — và side-effect trên *unmanaged* dependency (third-party API, message bus, email sender) là observable behavior của hệ thống.

```csharp
// TỆ — assert trên stub (test implementation-detail, break trên refactor)
[Fact]
public void Bad_test()
{
    var repo = new Mock<IOrderRepository>();
    repo.Setup(r => r.GetById(42)).Returns(new Order());

    var sut = new OrderService(repo.Object);
    sut.GetOrder(42);

    repo.Verify(r => r.GetById(42), Times.Once);   // ← brittle, không add value
}

// TỐT — assert trên observable behavior (return value)
[Fact]
public void Good_test()
{
    var repo = new InMemoryOrderRepository();
    repo.Add(new Order(id: 42, total: 100m));

    var sut = new OrderService(repo);
    var order = sut.GetOrder(42);

    Assert.Equal(100m, order.Total);
}
```

---

## 8. Ba loại test — Output-Based, State-Based, Communication-Based

Khorikov classify test theo *cái họ assert*:

### 1. Output-based testing (functional)

SUT là pure function: input → output. Test verify output.

```csharp
public static decimal CalculateDiscount(decimal subtotal, CustomerTier tier) =>
    tier switch
    {
        CustomerTier.Gold     => subtotal * 0.10m,
        CustomerTier.Platinum => subtotal * 0.15m,
        _                     => 0m,
    };

[Fact]
public void Gold_customer_gets_10_percent_discount()
{
    var discount = PricingService.CalculateDiscount(100m, CustomerTier.Gold);
    Assert.Equal(10m, discount);
}
```

**Verdict:** gold standard. Maximize cả bốn trụ cột đồng thời. **Strive viết code theo style làm output-based test khả thi** — pure function, không hidden state, không side effect.

### 2. State-based testing

SUT thay đổi state nào đó; test verify state kết quả qua query.

```csharp
[Fact]
public void Adding_an_item_increases_cart_total()
{
    var cart = new ShoppingCart();
    cart.Add(new LineItem(ProductId: 1, Quantity: 2, Price: 50m));
    Assert.Equal(100m, cart.Total);
}
```

**Verdict:** thứ hai tốt nhất. Tốt khi SUT genuinely own state. Assert trên observable state qua public API.

### 3. Communication-based testing (interaction)

SUT collaborate với cái khác; test verify *interaction* (thường với mock).

```csharp
[Fact]
public void Confirming_order_sends_email()
{
    var email = new Mock<IEmailSender>();
    var service = new OrderService(email.Object);

    service.Confirm(new Order(id: 1, customerEmail: "x@y.com"));

    email.Verify(e => e.Send("x@y.com", It.IsAny<string>(), It.IsAny<string>()), Times.Once);
}
```

**Verdict:** fragile nhất. Test rằng *SUT call* cái gì đó là test implementation. *Đôi khi* unavoidable — khi contract của SUT thực sự *là* "trigger side-effect trên unmanaged dependency". Nhưng dùng sparingly.

> **Guidance Khorikov:** ưu tiên output-based, fall back state-based, dùng communication-based chỉ ở application boundary.

---

## 9. Anatomy của Good Test (AAA)

Mọi test, regardless loại, theo cấu trúc **Arrange / Act / Assert**:

```csharp
[Fact]
public void Withdrawing_more_than_balance_throws()
{
    // Arrange
    var account = new BankAccount(initial: 100m);

    // Act
    Action action = () => account.Withdraw(150m);

    // Assert
    Assert.Throws<InvalidOperationException>(action);
}
```

### Advice cụ thể của Khorikov về AAA

* **Separate ba section rõ ràng.** Blank line hoặc comment. Đừng trộn.
* **Một "Act" per test.** Nhiều act = nhiều behavior = nên là nhiều test. (Exception: integration test đôi khi legitimately cần nhiều step.)
* **Naming quan trọng.** Tên test nên mô tả *behavior under test*, không method name. `OrderService_Pay_Success` → `Paying_an_order_marks_it_as_paid_and_records_the_transaction`.
* **Tránh `if`/`switch` trong test.** Mỗi test exercise một path. Branch trong test nghĩa là test hai thứ trong một test.
* **Style Given-When-Then** (`BDD`) là cùng idea với keyword khác; cả hai fine.

### Pattern tên test

* `[StateUnderTest]_[ExpectedBehavior]` — `EmptyCart_Total_IsZero`
* Câu plain English — `Empty cart's total is zero`
* Khorikov ưu tiên plain English. Reader không cần decode camelCase.

---

## 10. Mocking — Khi nào, cái gì, và cái gì không

Chương spark nhiều argument nhất trong code review.

### Hai category dependency

Khorikov vẽ distinction chính xác:

| Category               | Định nghĩa                                                                | Ví dụ                                                |
| ---------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Managed dependency**  | Dependency *chỉ* application này dùng; ta control lifecycle entirely     | Database riêng của application (write-side)          |
| **Unmanaged dependency** | Dependency share với application khác, hoặc observable external          | Third-party API, message bus, SMTP server, payment gateway |

### Rule

> **Mock unmanaged dependency. Đừng mock managed dependency — dùng cái thật.**

* **Unmanaged dependency** → side-effect trên nó observable cho *hệ thống khác*. Treat side-effect đó như phần của observable behavior của SUT. Mock + verify.
* **Managed dependency** → side-effect trên nó observable chỉ qua SUT chính nó. Dùng DB thật trong integration test; assert trên *state* sau, không trên *call*.

### Ví dụ

| Dependency                         | Loại        | Cách test                                                          |
| ---------------------------------- | ----------- | ------------------------------------------------------------------ |
| DB SQL Server của application bạn  | Managed     | DB thật (Testcontainers); assert trên state qua query              |
| Payment gateway (Stripe)           | Unmanaged   | Mock; verify `Charge` được call với amount đúng                    |
| SendGrid / SMTP                    | Unmanaged   | Mock; verify `Send` được call                                      |
| Message bus internal (consume chỉ bởi bạn) | Managed | Dùng in-memory hoặc Kafka thật; assert state của consumer       |
| Webhook external tới partner       | Unmanaged   | Mock; verify partner contract                                      |
| Time / `DateTime.Now`              | Unmanaged-ish | Inject `IClock`; pass value deterministic                        |

Nguyên lý: **side-effect visible *bên ngoài* application boundary là phần observable behavior của SUT; side-effect visible *chỉ bên trong* là implementation detail.**

---

## 11. Mock chỉ cái bạn Own (Và chỉ ở Boundary)

Khorikov endorse rule **"Don't mock what you don't own"** (từ *Growing Object-Oriented Software, Guided by Tests*):

* **Bạn own:** interface bạn defined để abstract third-party library. Luôn wrap third-party concrete trong port riêng.
* **Bạn không own:** bản thân type third-party (`HttpClient`, `Stripe.PaymentIntentService`, `SendGridClient`).

```csharp
// Tốt — wrap third party trong port riêng, mock port
public interface IPaymentGateway
{
    Task<PaymentResult> Charge(Money amount, string customerId, CancellationToken ct);
}

public sealed class StripePaymentGateway : IPaymentGateway
{
    private readonly Stripe.PaymentIntentService _stripe;
    public Task<PaymentResult> Charge(Money amount, string customerId, CancellationToken ct) { /* call Stripe */ }
}

// Test mock IPaymentGateway, không bao giờ Stripe trực tiếp
public void Order_payment_failure_marks_order_as_payment_failed()
{
    var gateway = new Mock<IPaymentGateway>();
    gateway.Setup(g => g.Charge(It.IsAny<Money>(), It.IsAny<string>(), default))
           .ReturnsAsync(PaymentResult.Declined("insufficient funds"));

    // ...
}
```

Mock type third-party trực tiếp couple test với API third-party. Khi họ thay đổi nó, test break — nhưng behavior không.

---

## 12. Functional / Hexagonal Architecture và Testability

Cả chương về cách **architectural style** influence bao nhiêu bạn có thể test với output-based test.

Thesis: **càng push business logic vào pure function / pure domain code, test của bạn càng tốt.**

### Pattern functional core / imperative shell

* **Functional core** — pure domain logic. Không I/O, không side effect. Return *cái gì* nên xảy ra.
* **Imperative shell** — glue mỏng perform side effect mà core decided.

```csharp
// Functional core: pure decision logic
public static class OrderRules
{
    public static OrderDecision DecideOnOrder(Order order, Customer customer, Promotions promotions)
    {
        if (customer.IsBlocked) return OrderDecision.Reject("Customer blocked");
        if (order.Total > customer.CreditLimit) return OrderDecision.Reject("Over credit limit");

        var discount = promotions.BestFor(order, customer);
        return OrderDecision.Accept(order, discount);
    }
}

// Imperative shell: chỉ orchestrate I/O quanh core
public sealed class OrderUseCase
{
    private readonly IOrderRepository _orders;
    private readonly ICustomerRepository _customers;
    private readonly IPromotionRepository _promotions;
    private readonly IEmailSender _email;

    public async Task<Result> PlaceOrder(Order o, CancellationToken ct)
    {
        var customer = await _customers.GetAsync(o.CustomerId, ct);
        var promotions = await _promotions.GetActiveAsync(ct);

        var decision = OrderRules.DecideOnOrder(o, customer, promotions);    // ← pure call

        if (decision is OrderDecision.Reject r)
            return Result.Fail(r.Reason);

        await _orders.AddAsync(((OrderDecision.Accept)decision).WithDiscount, ct);
        await _email.SendConfirmationAsync(o, ct);
        return Result.Ok();
    }
}
```

* Test trên `OrderRules.DecideOnOrder` là **output-based**, chạy trong microsecond, không mock, không infrastructure.
* Test trên `OrderUseCase` là *integration test* — một per happy path high-level hoặc error path, DB thật, unmanaged dependency mock.

Separation này **maximize area của triangle bốn trụ cột** cho phần lớn nhất của code bạn.

### Hexagonal architecture như case đặc biệt

Hexagonal Architecture (port & adapter) là *một* cách enforce split functional core / imperative shell này. Sách endorse nó. **Domain → 100% output-based; Application → integration; Adapter → gần như không test (humble object).**

> Cross-reference: [`../dot-net/docs/architectural-style/hexagonal-architecture-vi.md`](../dot-net/docs/architectural-style/hexagonal-architecture-vi.md).

---

## 13. Integration Testing

Testing pyramid của sách cho application theo pattern functional-core:

```
       /\
      /  \   ← E2E (1–5 scenario critical nhất)
     /----\
    /      \  ← Integration (per use case; DB thật; unmanaged dep mock)
   /--------\
  /          \ ← Unit (functional core; pure; output-based; hàng ngàn cái)
 /____________\
```

### Rule integration-test của Khorikov

1. **Một integration test per path high-level qua use case.** Happy path, key failure path. Không phải mọi variation — đó là output-based unit test trên core.
2. **Database thật.** Dùng Testcontainers / DB local thật. In-memory replacement (provider InMemory của EF Core, SQLite-as-Postgres) ẩn bug và cung cấp false confidence.
3. **Mock unmanaged dependency.** Stripe, email, message bus đi sang hệ thống khác.
4. **Đừng mock database.** Là managed dependency.
5. **Một Act per test, vẫn vậy.** Integration test chỉ exercise nhiều code hơn per act.
6. **Reset state giữa test.** DB sạch là phần của step arrange. (Transaction rollback, table truncation, hoặc container fresh.)
7. **Tránh shared mutable test fixture.** Chúng gây flakiness test order-dependent.

```csharp
// Integration test chống Postgres thật qua Testcontainers
public class PlaceOrderUseCaseIntegrationTests : IAsyncLifetime
{
    private PostgreSqlContainer _db;

    public async Task InitializeAsync()
    {
        _db = new PostgreSqlBuilder().Build();
        await _db.StartAsync();
    }

    [Fact]
    public async Task Valid_order_is_persisted_and_confirmation_email_sent()
    {
        // Arrange
        var dbContext = BuildDbContext(_db.GetConnectionString());
        var email = new Mock<IEmailSender>();
        var useCase = new PlaceOrderUseCase(dbContext, email.Object);

        // Act
        var result = await useCase.PlaceOrder(new Order(customerId: 1, total: 100m), default);

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Single(await dbContext.Orders.ToListAsync());
        email.Verify(e => e.SendConfirmationAsync(It.IsAny<Order>(), default), Times.Once);
    }

    public Task DisposeAsync() => _db.DisposeAsync().AsTask();
}
```

---

## 14. Work với Time, Random, và Hard Dependency khác

Chương cuối cùng trả lời: làm sao test code phụ thuộc `DateTime.Now`?

### Pattern: lift hard dependency vào port

```csharp
public interface IClock { DateTime UtcNow { get; } }
public sealed class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }

public sealed class TokenService
{
    private readonly IClock _clock;
    public TokenService(IClock clock) => _clock = clock;

    public Token IssueToken(User user, TimeSpan lifetime)
        => new Token(user.Id, _clock.UtcNow + lifetime);
}

// Test pass clock fixed — fully deterministic
[Fact]
public void Token_expires_one_hour_after_issuance()
{
    var fixedClock = new FixedClock(new DateTime(2026, 1, 1, 12, 0, 0));
    var sut = new TokenService(fixedClock);

    var token = sut.IssueToken(new User(1), TimeSpan.FromHours(1));

    Assert.Equal(new DateTime(2026, 1, 1, 13, 0, 0), token.ExpiresAt);
}
```

Cùng approach work cho:
* `Random` → inject `IRandom`
* `Guid.NewGuid()` → inject `IGuidGenerator`
* Filesystem → inject `IFileSystem`
* Environment variable → inject `IConfig`

Đây là pattern **functional core** lần nữa: push impurity ra rìa, giữ core deterministic.

---

## 15. Anti-Pattern cần Retire

Khorikov enumerate habit testing anh ấy coi **harmful**:

| Anti-pattern                                  | Tại sao tệ                                                                   |
| --------------------------------------------- | ---------------------------------------------------------------------------- |
| **Test private method**                       | Private method là implementation detail. Nếu phải test một, method là *của bạn* nhưng hidden — make nó `internal` và viết test qua public API exercise nó. |
| **Assert trên stub (`Verify(stub)`)**         | Lãng phí thuần. Stub là input; assert nó được call không validate behavior.   |
| **Mock cho managed dependency**               | Mock DB dẫn tới test pass nhưng DB schema sai.                               |
| **Một test per method ("structural coverage")** | Count nhưng không quan trọng. Test *behavior*, không method.               |
| **Test time-coupled** (`Thread.Sleep`, `DateTime.Now` thật) | Flaky và chậm. Inject clock.                                    |
| **Test inheritance hierarchy cho setup share** | Hidden state, khó đọc. Dùng helper method hoặc factory đơn giản.            |
| **High mock count per test**                  | Triệu chứng *implementation testing*. Hoặc refactor về purity, hoặc merge thành một test unit-of-behavior lớn hơn. |
| **Test change sau mọi refactor**              | Fail trụ cột resistance-to-refactoring. Test couple với implementation.       |
| **100% coverage như goal**                    | Coverage measure dòng, không behavior. Tốt hơn có 70% coverage behavior có ý nghĩa hơn 100% coverage getter và code trivial. |
| **Test code không code review**                | Test là production code. Review chúng.                                       |

---

## 16. Take-Away bức tranh lớn

1. **Test valuable theo tỉ lệ với cả bốn trụ cột.** Optimize *product*, không phải corner nào.
2. **Define "unit" như unit của *behavior*, không class.** Single shift này collapse phần lớn debate mocks-vs-classical.
3. **Test observable behavior; ignore implementation detail.** Smell của "implementation-detail testing" là test break trên mọi refactor.
4. **Mock chỉ ở boundary** với unmanaged dependency. Database *không* phải boundary.
5. **Wrap library third-party đằng sau port riêng** và mock port, không library.
6. **Functional core / imperative shell** là architectural style maximize testability — pure logic trong core, I/O ở rìa.
7. **Output-based > state-based > communication-based.** Lean về viết code admit cái đầu.
8. **Dùng database thật trong integration test.** In-memory substitute lie.
9. **Integration test cover use case; unit test cover domain logic.** Đừng try làm cả hai với một loại test.
10. **Coverage là side effect, không goal.** Aim cho behavior đúng được cover, không phải dòng nhiều nhất.

---

## 17. Sách này fit Repo ra sao

| Concept trong sách                               | Doc repo                                                                                  |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Functional core / imperative shell               | [`../dot-net/docs/architectural-style/hexagonal-architecture-vi.md`](../dot-net/docs/architectural-style/hexagonal-architecture-vi.md), [`../dot-net/docs/architectural-style/clean-architecture-vi.md`](../dot-net/docs/architectural-style/clean-architecture-vi.md) |
| Wrap third-party lib (port)                      | [`../dot-net/docs/design-pattern/structural-vi.md`](../dot-net/docs/design-pattern/structural-vi.md) (section Adapter) |
| DI / testability                                  | [`../dot-net/docs/enterprise-pattern/ioc-di-vi.md`](../dot-net/docs/enterprise-pattern/ioc-di-vi.md), [`../dot-net/docs/enterprise-pattern/lifetimes-vi.md`](../dot-net/docs/enterprise-pattern/lifetimes-vi.md) |
| DDD aggregate (test rich domain)                 | [`../dot-net/docs/ddd/tactical-patterns-vi.md`](../dot-net/docs/ddd/tactical-patterns-vi.md) |
| Test pyramid trong microservices                 | [`building-microservices-sam-newman-vi.md`](./building-microservices-sam-newman-vi.md) (chương testing) |
| SOLID principle, đặc biệt DIP                    | [`../principles/solid-vi.md`](../principles/solid-vi.md)                                  |

---

## 18. Khuyến nghị thứ tự đọc

* **Lần đầu:** đọc thẳng qua. Framework bốn trụ cột trong Chương 4 là chương key; mọi thứ trước là setup, mọi thứ sau là application.
* **Nếu chỉ có một giờ:** đọc Chương 4 (bốn trụ cột), 5 (mocks vs. stubs), 7 (observable behavior), và 11 (functional architecture).
* **Nếu team đang fight về mock:** Chương 5, 8, và 11 cùng nhau end debate.
* **Nếu test chậm và flaky:** Chương 8 (integration test) + chương anti-pattern.

---

## 19. Hiểu lầm phổ biến

| Hiểu lầm                                                            | Thực tế                                                                                                |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| "Khorikov nói đừng dùng mock."                                      | Anh ấy nói dùng chúng sparingly — ở boundary, cho unmanaged dependency. Chống *over-mocking*.          |
| "Unit là một class."                                                | Unit là một *behavior*. Có thể span nhiều class.                                                       |
| "Coverage không quan trọng."                                        | Coverage quan trọng; là signal *cần thiết* nhưng không đủ. 100% như *goal* dẫn tới test tệ.            |
| "Integration test là unit test với DB."                             | Là loại test khác với goal khác. Chúng protect more, refactor less, và chạy chậm hơn.                   |
| "Sách C#-specific."                                                 | Không. Mọi nguyên lý language-agnostic. C# là *phương tiện*, không phải *chủ đề*.                       |
| "Test private method fine nếu tôi make chúng `internal`."           | Đừng. Test behavior qua public API. Nếu behavior cần private method nhưng không testable qua public API, public API sai shape. |

---

## 20. Tham khảo

* Vladimir Khorikov — *Unit Testing: Principles, Practices, and Patterns*, Manning, 2020.
* Gerard Meszaros — *xUnit Test Patterns: Refactoring Test Code*, Addison-Wesley, 2007 (taxonomy canonical của test double).
* Steve Freeman & Nat Pryce — *Growing Object-Oriented Software, Guided by Tests*, Addison-Wesley, 2009 (sách London-school canonical; Khorikov disagree với nó ở điểm key, fairly).
* Kent Beck — *Test-Driven Development by Example*, Addison-Wesley, 2002.
* Martin Fowler — [*Mocks Aren't Stubs*](https://martinfowler.com/articles/mocksArentStubs.html) (2007). Article Khorikov build trên.
* Course Pluralsight của Vladimir Khorikov về testing — content sách là version polish hơn của course đó.
