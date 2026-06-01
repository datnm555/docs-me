# Walkthrough Dự Án Thật — E-Commerce Checkout

> Bài tập thiết kế OOP đầy đủ, thực tế. Chúng ta sẽ thiết kế **module checkout e-commerce** từ đầu đến cuối — modeling domain, layer, interface, polymorphism, persistence — và chỉ ra mỗi trụ cột và nguyên lý được áp dụng ra sao. Code là C# / .NET, nhưng thiết kế đúng cho Java / Kotlin / TypeScript.

> 🇻🇳 Phiên bản tiếng Việt. English: [`real-project.md`](./real-project.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Bài tập thiết kế full end-to-end: module checkout e-commerce model với cả bốn trụ cột OO, layered thành Domain / Application / Infrastructure / API, với cross-reference explicit tới nguyên lý nào mỗi mảnh phục vụ.
- **Tại sao** — Pattern và nguyên lý chỉ click khi bạn thấy chúng compose trong codebase thật. Walkthrough này kết nối "Strategy" / "SRP" / "DIP" tới dòng code nơi chúng sống.
- **Khi nào** — Sau khi đọc `pillars-vi.md` và `concepts-vi.md`, trước khi tackle content `dot-net/docs/` sâu. Hoặc như reference example khi thiết kế module tương tự.
- **Ở đâu** — Layout solution giả định — nhưng cấu trúc (Domain ← Application ← Infrastructure ← Web) là shape Clean/Hexagonal canonical bạn sẽ thấy trong project .NET thật.

---

## Mục lục

1. [Yêu cầu](#1-yêu-cầu)
2. [Xác định khái niệm domain](#2-xác-định-khái-niệm-domain)
3. [Layer & Project Layout](#3-layer--project-layout)
4. [Domain Layer — Entity & Value Object](#4-domain-layer--entity--value-object)
5. [Domain Layer — Polymorphism trong action](#5-domain-layer--polymorphism-trong-action)
6. [Application Layer — Use Case](#6-application-layer--use-case)
7. [Infrastructure Layer — Repository & Adapter](#7-infrastructure-layer--repository--adapter)
8. [Presentation Layer — Web API](#8-presentation-layer--web-api)
9. [Wire Dependency (Composition Root)](#9-wire-dependency-composition-root)
10. [Chiến lược test](#10-chiến-lược-test)
11. [Mỗi trụ cột OOP xuất hiện ở đâu](#11-mỗi-trụ-cột-oop-xuất-hiện-ở-đâu)
12. [Mỗi nguyên lý xuất hiện ở đâu](#12-mỗi-nguyên-lý-xuất-hiện-ở-đâu)

---

## 1. Yêu cầu

Một cửa hàng online cần feature **checkout**:

1. Customer place order với một hoặc nhiều line item.
2. Hệ thống tính total, bao gồm **tax** (vary theo quốc gia) và **discount** (loyalty tier, coupon code).
3. Payment lấy qua một trong: **credit card**, **PayPal**, hoặc **Apple Pay**.
4. Khi thành công: persist order, decrement stock, gửi email confirmation.
5. Behavior phải **testable** — không có call Stripe hay SMTP thật trong unit test.
6. Thêm payment provider mới hoặc discount rule không được sửa class hiện có.

---

## 2. Xác định khái niệm domain

Bài tập khởi đầu hữu ích — **đào noun-verb** từ yêu cầu:

| Noun / khái niệm  | Trở thành                              | Tại sao                                   |
| ----------------- | -------------------------------------- | ----------------------------------------- |
| Customer          | **Entity** (`Customer`)                | Có identity, tiến hoá theo thời gian      |
| Order, LineItem   | **Aggregate root + entity**            | Identity + lifecycle                      |
| Product           | **Entity**                             | Có identity                               |
| Money             | **Value Object**                       | Không identity; bằng theo value           |
| Address           | **Value Object**                       | Không identity; bằng theo value           |
| TaxRule           | **Polymorphic strategy**               | Variation theo quốc gia                   |
| DiscountRule      | **Polymorphic strategy**               | Nhiều variant, sẽ tăng                    |
| PaymentMethod     | **Polymorphic strategy**               | Nhiều variant, sẽ tăng                    |
| OrderRepository   | **Pure Fabrication / Indirection**     | Cô lập persistence                        |
| EmailSender       | **Pure Fabrication / Indirection**     | Cô lập SMTP                               |
| CheckoutService   | **Application service / Controller**   | Orchestrate một use case                  |

Đã có ba chỗ cần **polymorphism** — dấu hiệu mạnh là OOP fit bài toán này.

---

## 3. Layer & Project Layout

```
src/
├── Shop.Domain/                  (không có external dependency)
│   ├── Customers/
│   ├── Orders/
│   ├── Products/
│   ├── Payments/
│   ├── Pricing/
│   └── SharedKernel/             (Money, Address, Result<T>)
│
├── Shop.Application/             (chỉ phụ thuộc Domain)
│   ├── Checkout/
│   │   ├── CheckoutUseCase.cs
│   │   ├── PlaceOrderRequest.cs
│   │   └── PlaceOrderResult.cs
│   └── Abstractions/             (interface implement bởi Infrastructure)
│       ├── IOrderRepository.cs
│       ├── IProductRepository.cs
│       ├── IPaymentGateway.cs
│       └── IEmailSender.cs
│
├── Shop.Infrastructure/          (phụ thuộc Application + Domain)
│   ├── Persistence/              (implementation EF Core)
│   ├── Payments/                 (adapter Stripe, PayPal, ApplePay)
│   └── Email/                    (adapter SMTP / SendGrid)
│
└── Shop.Api/                     (phụ thuộc Application + Infrastructure)
    └── Controllers/
        └── CheckoutController.cs
```

Đây là **Clean Architecture** / **Hexagonal** / **Ports & Adapters** — chọn nhãn nào tuỳ, dependency rule giống nhau: **layer trong không bao giờ reference layer ngoài.**

---

## 4. Domain Layer — Entity & Value Object

### 4.1 `Money` — Value Object

```csharp
namespace Shop.Domain.SharedKernel;

public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Zero(string ccy) => new(0m, ccy);

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return this with { Amount = Amount + other.Amount };
    }

    public Money Multiply(decimal factor)
        => this with { Amount = Math.Round(Amount * factor, 2) };
}
```

**Tại sao value object:** `Money` không có identity — hai `Money(10, "USD")` tương đương. Record cho ta structural equality và immutability miễn phí.

### 4.2 `LineItem` — Entity bên trong Aggregate

```csharp
namespace Shop.Domain.Orders;

public class LineItem
{
    public Guid    ProductId { get; }
    public string  Name      { get; }
    public Money   UnitPrice { get; }
    public int     Quantity  { get; private set; }

    public Money Subtotal => UnitPrice.Multiply(Quantity);

    internal LineItem(Guid productId, string name, Money unitPrice, int qty)
    {
        if (qty <= 0) throw new ArgumentException("Quantity must be positive");
        ProductId = productId;
        Name      = name;
        UnitPrice = unitPrice;
        Quantity  = qty;
    }

    internal void IncreaseQuantity(int by)
    {
        if (by <= 0) throw new ArgumentException();
        Quantity += by;
    }
}
```

* `internal` constructor — chỉ aggregate `Order` mới có thể tạo line item.
* `private set` trên `Quantity` — invariant sống trong `IncreaseQuantity`.

### 4.3 `Order` — Aggregate Root

```csharp
namespace Shop.Domain.Orders;

public class Order
{
    private readonly List<LineItem> _items = new();

    public Guid     Id         { get; }
    public Guid     CustomerId { get; }
    public DateTime PlacedAt   { get; }
    public OrderStatus Status  { get; private set; }
    public IReadOnlyList<LineItem> Items => _items;

    private Order(Guid customerId)
    {
        Id         = Guid.NewGuid();
        CustomerId = customerId;
        PlacedAt   = DateTime.UtcNow;
        Status     = OrderStatus.Pending;
    }

    public static Order Start(Guid customerId) => new(customerId);

    public void AddItem(Product product, int qty)
    {
        var existing = _items.FirstOrDefault(i => i.ProductId == product.Id);
        if (existing is null)
            _items.Add(new LineItem(product.Id, product.Name, product.Price, qty));
        else
            existing.IncreaseQuantity(qty);
    }

    public Money Subtotal(string currency)
        => _items.Aggregate(Money.Zero(currency), (acc, i) => acc.Add(i.Subtotal));

    public void MarkPaid()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException($"Cannot pay an order in status {Status}");
        Status = OrderStatus.Paid;
    }
}

public enum OrderStatus { Pending, Paid, Cancelled, Refunded }
```

* **Encapsulation** — `_items` được sở hữu hoàn toàn.
* **Static factory** `Order.Start(...)` — rõ hơn `new Order(...)`, có thể thêm validation.
* **Invariant** tập trung trong `MarkPaid` (state machine bảo vệ).

---

## 5. Domain Layer — Polymorphism trong action

Ba điểm tăng trưởng — payment, tax, discount — là ứng viên hoàn hảo cho **Strategy pattern** (interface polymorphic).

### 5.1 Payment

```csharp
namespace Shop.Domain.Payments;

public abstract record PaymentResult
{
    public sealed record Success(string TransactionId) : PaymentResult;
    public sealed record Declined(string Reason)        : PaymentResult;
}

// Abstraction cấp application — sống ở Application layer, không phải Infrastructure
public interface IPaymentGateway
{
    string Name { get; }
    Task<PaymentResult> ChargeAsync(Money amount, PaymentInstrument instrument, CancellationToken ct);
}

public abstract record PaymentInstrument
{
    public sealed record CreditCard(string Token)              : PaymentInstrument;
    public sealed record PayPal(string AccountToken)           : PaymentInstrument;
    public sealed record ApplePay(string DeviceTransactionId)  : PaymentInstrument;
}
```

Thêm **Google Pay** → thêm case `PaymentInstrument` mới + adapter mới implement `IPaymentGateway`. **Không sửa code cũ.** (OCP, GRASP Protected Variations.)

### 5.2 Tax

```csharp
namespace Shop.Domain.Pricing;

public interface ITaxPolicy
{
    Money Apply(Money subtotal, Address shippingTo);
}

public class VatTaxPolicy(decimal rate) : ITaxPolicy
{
    public Money Apply(Money subtotal, Address _) => subtotal.Multiply(rate);
}

public class UsSalesTaxPolicy(IReadOnlyDictionary<string, decimal> ratesByState) : ITaxPolicy
{
    public Money Apply(Money subtotal, Address address)
    {
        var rate = ratesByState.GetValueOrDefault(address.State, 0m);
        return subtotal.Multiply(rate);
    }
}
```

### 5.3 Discount — Composite Strategy

Discount cộng dồn: customer có thể là Gold tier *và* dùng coupon. Dùng pattern **Composite**.

```csharp
namespace Shop.Domain.Pricing;

public interface IDiscountRule
{
    Money ComputeDiscount(Money subtotal, Customer customer);
}

public class LoyaltyTierDiscount : IDiscountRule
{
    public Money ComputeDiscount(Money subtotal, Customer c) => c.Tier switch
    {
        Tier.Gold     => subtotal.Multiply(0.10m),
        Tier.Platinum => subtotal.Multiply(0.15m),
        _             => Money.Zero(subtotal.Currency)
    };
}

public class CouponDiscount(string code, decimal percent) : IDiscountRule
{
    public Money ComputeDiscount(Money subtotal, Customer c)
        => c.RedeemableCoupons.Contains(code)
             ? subtotal.Multiply(percent)
             : Money.Zero(subtotal.Currency);
}

public class CompositeDiscount(IEnumerable<IDiscountRule> rules) : IDiscountRule
{
    public Money ComputeDiscount(Money subtotal, Customer c)
        => rules.Aggregate(Money.Zero(subtotal.Currency),
                           (acc, r) => acc.Add(r.ComputeDiscount(subtotal, c)));
}
```

---

## 6. Application Layer — Use Case

```csharp
namespace Shop.Application.Checkout;

public record PlaceOrderRequest(
    Guid CustomerId,
    IReadOnlyList<(Guid ProductId, int Qty)> Items,
    PaymentInstrument Payment,
    string Currency);

public record PlaceOrderResult(Guid OrderId, string TransactionId);

public class CheckoutUseCase(
    ICustomerRepository  customers,
    IProductRepository   products,
    IOrderRepository     orders,
    ITaxPolicy           tax,
    IDiscountRule        discounts,
    IPaymentGateway      payment,
    IEmailSender         email,
    ILogger<CheckoutUseCase> log)
{
    public async Task<PlaceOrderResult> ExecuteAsync(
        PlaceOrderRequest req, CancellationToken ct)
    {
        // 1. Load aggregate
        var customer = await customers.GetAsync(req.CustomerId, ct)
            ?? throw new InvalidOperationException("Customer not found");

        var order = Order.Start(customer.Id);
        foreach (var (pid, qty) in req.Items)
        {
            var product = await products.GetAsync(pid, ct)
                ?? throw new InvalidOperationException($"Product {pid} not found");
            order.AddItem(product, qty);
        }

        // 2. Pricing
        var subtotal = order.Subtotal(req.Currency);
        var discount = discounts.ComputeDiscount(subtotal, customer);
        var taxed    = tax.Apply(subtotal.Add(discount.Multiply(-1)), customer.ShippingAddress);
        var total    = subtotal.Add(discount.Multiply(-1)).Add(taxed);

        // 3. Payment
        var result = await payment.ChargeAsync(total, req.Payment, ct);
        if (result is PaymentResult.Declined d)
            throw new PaymentDeclinedException(d.Reason);

        var success = (PaymentResult.Success)result;

        // 4. Persist + side effect
        order.MarkPaid();
        await orders.SaveAsync(order, ct);
        await email.SendAsync(customer.Email,
                              $"Order {order.Id} confirmed",
                              $"Thanks! Total: {total.Amount} {total.Currency}");

        log.LogInformation("Order {OrderId} placed for {Total}", order.Id, total);
        return new PlaceOrderResult(order.Id, success.TransactionId);
    }
}
```

Nhìn class này — đọc như **văn xuôi**. Không có SQL, không HTTP, không Stripe, không SMTP, không JSON. Mỗi collaborator là interface; mỗi interface có một lý do để thay đổi.

---

## 7. Infrastructure Layer — Repository & Adapter

### 7.1 Persistence

```csharp
namespace Shop.Infrastructure.Persistence;

public class EfOrderRepository(ShopDbContext db) : IOrderRepository
{
    public Task SaveAsync(Order order, CancellationToken ct)
    {
        db.Orders.Add(order);
        return db.SaveChangesAsync(ct);
    }

    public Task<Order?> GetAsync(Guid id, CancellationToken ct)
        => db.Orders.Include(o => o.Items).SingleOrDefaultAsync(o => o.Id == id, ct);
}
```

### 7.2 Stripe Payment Adapter

```csharp
namespace Shop.Infrastructure.Payments;

public class StripePaymentGateway(StripeOptions opts, HttpClient http) : IPaymentGateway
{
    public string Name => "Stripe";

    public async Task<PaymentResult> ChargeAsync(
        Money amount, PaymentInstrument instr, CancellationToken ct)
    {
        if (instr is not PaymentInstrument.CreditCard cc)
            return new PaymentResult.Declined("Unsupported instrument");

        var resp = await http.PostAsJsonAsync("/v1/charges",
            new { amount = (int)(amount.Amount * 100), currency = amount.Currency, source = cc.Token },
            ct);

        if (!resp.IsSuccessStatusCode) return new PaymentResult.Declined("Stripe error");
        var body = await resp.Content.ReadFromJsonAsync<StripeChargeResponse>(cancellationToken: ct);
        return new PaymentResult.Success(body!.Id);
    }
}
```

> Mỗi adapter biết chính xác **một** hệ thống bên ngoài. Đổi Stripe → Adyen → thay file này, không gì khác.

---

## 8. Presentation Layer — Web API

```csharp
namespace Shop.Api.Controllers;

[ApiController, Route("api/checkout")]
public class CheckoutController(CheckoutUseCase useCase) : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<PlaceOrderResult>> Place(
        [FromBody] PlaceOrderRequest req, CancellationToken ct)
    {
        try
        {
            return Ok(await useCase.ExecuteAsync(req, ct));
        }
        catch (PaymentDeclinedException ex)
        {
            return BadRequest(new { error = "payment_declined", reason = ex.Message });
        }
    }
}
```

Controller mỏng — đúng như **GRASP Controller** quy định. Mọi việc thật được delegate.

---

## 9. Wire Dependency (Composition Root)

```csharp
// Program.cs
var b = WebApplication.CreateBuilder(args);

b.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer(b.Configuration.GetConnectionString("Db")));

// Repository
b.Services.AddScoped<ICustomerRepository, EfCustomerRepository>();
b.Services.AddScoped<IProductRepository,  EfProductRepository>();
b.Services.AddScoped<IOrderRepository,    EfOrderRepository>();

// Pricing
b.Services.AddSingleton<ITaxPolicy>(_ => new VatTaxPolicy(0.20m));
b.Services.AddSingleton<IDiscountRule>(sp => new CompositeDiscount(new IDiscountRule[]
{
    new LoyaltyTierDiscount(),
    new CouponDiscount("WELCOME10", 0.10m)
}));

// Payment
b.Services.AddHttpClient<StripePaymentGateway>();
b.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();

// Email
b.Services.AddScoped<IEmailSender, SendGridEmailSender>();

// Use case
b.Services.AddScoped<CheckoutUseCase>();

b.Services.AddControllers();
var app = b.Build();
app.MapControllers();
app.Run();
```

**Composition root** — *nơi duy nhất* biết về concrete class. Mọi thứ khác xin interface.

---

## 10. Chiến lược test

Vì mọi external boundary đằng sau interface, use case có thể unit-test mà không cần database, HTTP, hay SMTP.

```csharp
public class CheckoutUseCaseTests
{
    [Fact]
    public async Task Charges_customer_total_with_tax_and_discount()
    {
        var customer = new Customer(Guid.NewGuid(), "Alice", "alice@x.com",
                                    new Address("US", "CA", ...),
                                    Tier.Gold, redeemable: new());

        var products = new InMemoryProductRepo(new Product(_pid, "Book", new Money(100, "USD")));
        var orders   = new InMemoryOrderRepo();
        var customers= new InMemoryCustomerRepo(customer);
        var payment  = new FakePaymentGateway(success: "tx_123");
        var email    = new FakeEmailSender();

        var sut = new CheckoutUseCase(customers, products, orders,
            tax:       new VatTaxPolicy(0.20m),
            discounts: new LoyaltyTierDiscount(),
            payment:   payment,
            email:     email,
            log:       NullLogger<CheckoutUseCase>.Instance);

        var result = await sut.ExecuteAsync(new PlaceOrderRequest(
            customer.Id, new[]{(_pid, 2)}, new PaymentInstrument.CreditCard("tok"), "USD"),
            default);

        // Subtotal 200 → -10% loyalty = 180 → +20% VAT = 216
        Assert.Equal(216m, payment.LastCharged.Amount);
        Assert.Single(email.Sent);
        Assert.Equal(OrderStatus.Paid, orders.Saved.Single().Status);
    }
}
```

Nhanh, deterministic, không có hệ thống bên ngoài.

---

## 11. Mỗi trụ cột OOP xuất hiện ở đâu

| Trụ cột             | Ở đâu trong project này                                                            |
| ------------------- | ---------------------------------------------------------------------------------- |
| **Encapsulation**   | `Order._items` private; `LineItem.Quantity` private set; invariant trong method.   |
| **Abstraction**     | `IPaymentGateway`, `IOrderRepository`, `IEmailSender` — *cái gì*, không *cách*.    |
| **Inheritance**     | Hierarchy `record PaymentInstrument`; `PaymentResult.Success / Declined`.          |
| **Polymorphism**    | `ITaxPolicy`, `IDiscountRule`, `IPaymentGateway` swap runtime.                     |

---

## 12. Mỗi nguyên lý xuất hiện ở đâu

| Nguyên lý                       | Biểu hiện                                                                    |
| ------------------------------- | ---------------------------------------------------------------------------- |
| **SRP**                         | Mỗi class một lý do để thay đổi (repository ≠ gateway ≠ use case).           |
| **OCP**                         | Thêm payment / tax / discount mới → không sửa class cũ.                       |
| **LSP**                         | Bất kỳ `IPaymentGateway` có thể substitute cho cái khác.                     |
| **ISP**                         | Interface tập trung (`IEmailSender`, không phải `IInfrastructureFacade`).    |
| **DIP**                         | Use case phụ thuộc interface; concrete wire ở composition root.              |
| **DRY**                         | `Money.Multiply` dùng khắp nơi cần math money.                                |
| **YAGNI**                       | Chưa có `IShipper` — thêm khi shipping vào scope.                            |
| **KISS**                        | Không EventBus / Mediator cho tới khi có nhu cầu cross-cutting thật.         |
| **SoC**                         | Domain ↔ Application ↔ Infrastructure ↔ API — 4 layer rõ.                    |
| **GRASP Information Expert**    | `Order.Subtotal()` sống trên `Order`, nơi item sống.                          |
| **GRASP Controller**            | `CheckoutController` mỏng; `CheckoutUseCase` orchestrate.                    |
| **GRASP Pure Fabrication**      | `OrderRepository` — không có khái niệm thế giới thực, thuần technical.       |
| **GRASP Indirection**           | `IPaymentGateway` giữa use case và Stripe.                                   |
| **GRASP Protected Variations**  | Mọi external biến động đằng sau interface ổn định.                            |
| **Tell, Don't Ask**             | `order.MarkPaid()` — caller không manipulate field status.                    |
| **Composition over Inheritance**| `CompositeDiscount` compose rule; không inheritance sâu.                      |
| **Encapsulation**               | Mọi field private; behavior sở hữu data.                                     |
| **Fail Fast**                   | Constructor guard, value object guard, status transition guard.              |

---

> **Insight chính:** bốn trụ cột và các nguyên lý không phải lý thuyết — chúng xuất hiện tự nhiên khi bạn nhận yêu cầu thật và hỏi đi hỏi lại, *"cái gì thay đổi, cái gì giữ nguyên, và ai sở hữu cái này?"*. Architecture trên không elegant vì ta *áp dụng* SOLID — nó elegant vì ta **model domain** và để SOLID xuất hiện.
