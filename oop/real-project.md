# Real-Project Walkthrough — An E-Commerce Checkout

> A complete, realistic OOP design exercise. We will design an **e-commerce checkout module** end to end — domain modeling, layering, interfaces, polymorphism, persistence — and show how each pillar and principle is applied. Code is C# / .NET, but the design holds in Java / Kotlin / TypeScript.

---

## Table of Contents

1. [The Requirements](#1-the-requirements)
2. [Identify the Domain Concepts](#2-identify-the-domain-concepts)
3. [Layering & Project Layout](#3-layering--project-layout)
4. [Domain Layer — Entities & Value Objects](#4-domain-layer--entities--value-objects)
5. [Domain Layer — Polymorphism in Action](#5-domain-layer--polymorphism-in-action)
6. [Application Layer — The Use Case](#6-application-layer--the-use-case)
7. [Infrastructure Layer — Repositories & Adapters](#7-infrastructure-layer--repositories--adapters)
8. [Presentation Layer — Web API](#8-presentation-layer--web-api)
9. [Dependency Wiring (Composition Root)](#9-dependency-wiring-composition-root)
10. [Testing Strategy](#10-testing-strategy)
11. [Where Each OOP Pillar Showed Up](#11-where-each-oop-pillar-showed-up)
12. [Where Each Principle Showed Up](#12-where-each-principle-showed-up)

---

## 1. The Requirements

An online store needs a **checkout** feature:

1. A customer places an order with one or more line items.
2. The system calculates totals, including **tax** (varies by country) and **discounts** (loyalty tier, coupon code).
3. Payment is taken via one of: **credit card**, **PayPal**, or **Apple Pay**.
4. On success: persist the order, decrement stock, send a confirmation email.
5. Behavior must be **testable** — no real Stripe or SMTP calls in unit tests.
6. Adding a new payment provider or discount rule must not require editing existing classes.

---

## 2. Identify the Domain Concepts

A useful starting exercise — **noun-verb mining** from the requirements:

| Noun / concept    | Becomes                                | Why                                       |
| ----------------- | -------------------------------------- | ----------------------------------------- |
| Customer          | **Entity** (`Customer`)                | Has identity, evolves over time           |
| Order, LineItem   | **Aggregate root + entity**            | Identity + lifecycle                      |
| Product           | **Entity**                             | Has identity                              |
| Money             | **Value Object**                       | No identity; equal by value               |
| Address           | **Value Object**                       | No identity; equal by value               |
| TaxRule           | **Polymorphic strategy**               | Per-country variation                     |
| DiscountRule      | **Polymorphic strategy**               | Many variants, will grow                  |
| PaymentMethod     | **Polymorphic strategy**               | Many variants, will grow                  |
| OrderRepository   | **Pure Fabrication / Indirection**     | Isolates persistence                      |
| EmailSender       | **Pure Fabrication / Indirection**     | Isolates SMTP                             |
| CheckoutService   | **Application service / Controller**   | Orchestrates a use case                   |

Already three places call for **polymorphism** — a strong sign OOP fits this problem.

---

## 3. Layering & Project Layout

```
src/
├── Shop.Domain/                  (no external dependencies)
│   ├── Customers/
│   ├── Orders/
│   ├── Products/
│   ├── Payments/
│   ├── Pricing/
│   └── SharedKernel/             (Money, Address, Result<T>)
│
├── Shop.Application/             (depends only on Domain)
│   ├── Checkout/
│   │   ├── CheckoutUseCase.cs
│   │   ├── PlaceOrderRequest.cs
│   │   └── PlaceOrderResult.cs
│   └── Abstractions/             (interfaces implemented by Infrastructure)
│       ├── IOrderRepository.cs
│       ├── IProductRepository.cs
│       ├── IPaymentGateway.cs
│       └── IEmailSender.cs
│
├── Shop.Infrastructure/          (depends on Application + Domain)
│   ├── Persistence/              (EF Core implementations)
│   ├── Payments/                 (Stripe, PayPal, ApplePay adapters)
│   └── Email/                    (SMTP / SendGrid adapter)
│
└── Shop.Api/                     (depends on Application + Infrastructure)
    └── Controllers/
        └── CheckoutController.cs
```

This is **Clean Architecture** / **Hexagonal** / **Ports & Adapters** — pick your label, the dependency rule is the same: **inner layers never reference outer layers.**

---

## 4. Domain Layer — Entities & Value Objects

### 4.1 `Money` — A Value Object

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

**Why a value object:** `Money` has no identity — two `Money(10, "USD")` are interchangeable. Records give us free structural equality and immutability.

### 4.2 `LineItem` — An Entity Inside an Aggregate

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

* `internal` constructor — only the `Order` aggregate can create line items.
* `private set` on `Quantity` — invariants live in `IncreaseQuantity`.

### 4.3 `Order` — The Aggregate Root

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

* **Encapsulation** — `_items` is fully owned.
* **Static factory** `Order.Start(...)` — clearer than `new Order(...)`, can grow validation.
* **Invariants** centralized in `MarkPaid` (state machine guarded).

---

## 5. Domain Layer — Polymorphism in Action

Three growth points — payment, tax, discount — are perfect candidates for the **Strategy pattern** (polymorphic interfaces).

### 5.1 Payment

```csharp
namespace Shop.Domain.Payments;

public abstract record PaymentResult
{
    public sealed record Success(string TransactionId) : PaymentResult;
    public sealed record Declined(string Reason)        : PaymentResult;
}

// Application-level abstraction — lives in Application layer, not Infrastructure
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

Adding **Google Pay** → add a new `PaymentInstrument` case + a new adapter that implements `IPaymentGateway`. **No existing code changes.** (OCP, GRASP Protected Variations.)

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

Discounts compound: a customer can be Gold tier *and* use a coupon. Use the **Composite** pattern.

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

## 6. Application Layer — The Use Case

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
        // 1. Load aggregates
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

        // 4. Persist + side effects
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

Look at this class — it reads like **prose**. There is no SQL, no HTTP, no Stripe, no SMTP, no JSON. Each collaborator is an interface; each interface has one reason to change.

---

## 7. Infrastructure Layer — Repositories & Adapters

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

> Each adapter knows exactly **one** external system. Swap Stripe → Adyen → replace this file, nothing else.

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

Thin controller — exactly as **GRASP Controller** prescribes. All real work is delegated.

---

## 9. Dependency Wiring (Composition Root)

```csharp
// Program.cs
var b = WebApplication.CreateBuilder(args);

b.Services.AddDbContext<ShopDbContext>(o =>
    o.UseSqlServer(b.Configuration.GetConnectionString("Db")));

// Repositories
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

// Use cases
b.Services.AddScoped<CheckoutUseCase>();

b.Services.AddControllers();
var app = b.Build();
app.MapControllers();
app.Run();
```

**Composition root** — the *only* place that knows about concrete classes. Everything else asks for interfaces.

---

## 10. Testing Strategy

Because every external boundary is behind an interface, the use case is unit-testable without a database, HTTP, or SMTP.

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

Fast, deterministic, no external systems.

---

## 11. Where Each OOP Pillar Showed Up

| Pillar              | Where in this project                                                              |
| ------------------- | ---------------------------------------------------------------------------------- |
| **Encapsulation**   | `Order._items` private; `LineItem.Quantity` private set; invariants in methods.    |
| **Abstraction**     | `IPaymentGateway`, `IOrderRepository`, `IEmailSender` — *what*, not *how*.         |
| **Inheritance**     | `record PaymentInstrument` hierarchy; `PaymentResult.Success / Declined`.          |
| **Polymorphism**    | `ITaxPolicy`, `IDiscountRule`, `IPaymentGateway` swapped at runtime.               |

---

## 12. Where Each Principle Showed Up

| Principle                       | Manifestation                                                                |
| ------------------------------- | ---------------------------------------------------------------------------- |
| **SRP**                         | Each class has one reason to change (repository ≠ gateway ≠ use case).       |
| **OCP**                         | Add new payment / tax / discount → no edits to existing classes.             |
| **LSP**                         | Any `IPaymentGateway` substitutable for any other.                           |
| **ISP**                         | Focused interfaces (`IEmailSender`, not `IInfrastructureFacade`).            |
| **DIP**                         | Use case depends on interfaces; concretes wired in composition root.         |
| **DRY**                         | `Money.Multiply` used everywhere money math is needed.                       |
| **YAGNI**                       | No `IShipper` yet — added when shipping is in scope.                         |
| **KISS**                        | No EventBus / Mediator until a real cross-cutting need arises.               |
| **SoC**                         | Domain ↔ Application ↔ Infrastructure ↔ API — four crisp layers.             |
| **GRASP Information Expert**    | `Order.Subtotal()` lives on `Order`, where the items live.                   |
| **GRASP Controller**            | `CheckoutController` is thin; `CheckoutUseCase` orchestrates.                |
| **GRASP Pure Fabrication**      | `OrderRepository` — no real-world concept, purely technical.                 |
| **GRASP Indirection**           | `IPaymentGateway` between use case and Stripe.                               |
| **GRASP Protected Variations**  | All volatile externals behind stable interfaces.                             |
| **Tell, Don't Ask**             | `order.MarkPaid()` — caller doesn't manipulate status fields.                |
| **Composition over Inheritance**| `CompositeDiscount` composes rules; no deep inheritance.                     |
| **Encapsulation**               | All fields private; behavior owns the data.                                  |
| **Fail Fast**                   | Constructors guard, value objects guard, status transitions guarded.         |

---

> **Key insight:** the four pillars and the principles aren't theory — they emerge naturally when you take a real requirement and ask, repeatedly, *"what changes, what stays the same, and who owns this?"*. The architecture above isn't elegant because we *applied* SOLID — it's elegant because we **modeled the domain** and let SOLID emerge.
