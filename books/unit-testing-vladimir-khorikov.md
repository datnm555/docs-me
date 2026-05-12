# Unit Testing — Summary

> **Vladimir Khorikov** — *Unit Testing: Principles, Practices, and Patterns*, Manning, 2020.

> **Why this book matters:** the most rigorous treatment of "what makes a unit test *good*" published in the last decade. Khorikov's central contribution is the **four pillars** model — a small, testable framework that lets you score any test against four objectively measurable properties and explains *why* every famous testing controversy (mocks vs. classical, integration vs. unit, what to test, what to mock) is a tradeoff between those properties.

> **This summary covers:** the four pillars, the two schools (classical / London), the precise definition of a "unit", observable behavior vs. implementation details, when to use mocks, mocking only what you own, integration testing, anti-patterns to retire, and the relationship between architectural style (functional / hexagonal) and test quality. Code examples are mine, illustrative — C# matches the book; the lessons translate cleanly to Java, TypeScript, and Kotlin.

> 🇻🇳 Vietnamese version: [`unit-testing-vladimir-khorikov-vi.md`](./unit-testing-vladimir-khorikov-vi.md)

---

## Table of Contents

1. [About the Book](#1-about-the-book)
2. [The Big Idea](#2-the-big-idea)
3. [The Four Pillars of a Good Unit Test](#3-the-four-pillars-of-a-good-unit-test)
4. [The Two Schools — Classical (Detroit) vs. London (Mockist)](#4-the-two-schools--classical-detroit-vs-london-mockist)
5. [What Is a "Unit"?](#5-what-is-a-unit)
6. [Observable Behavior vs. Implementation Details](#6-observable-behavior-vs-implementation-details)
7. [Mocks vs. Stubs — A Precise Distinction](#7-mocks-vs-stubs--a-precise-distinction)
8. [The Three Test Types — Output-Based, State-Based, Communication-Based](#8-the-three-test-types--output-based-state-based-communication-based)
9. [The Anatomy of a Good Test (AAA)](#9-the-anatomy-of-a-good-test-aaa)
10. [Mocking — When, What, and What Not](#10-mocking--when-what-and-what-not)
11. [Mock Only What You Own (And Only at the Boundary)](#11-mock-only-what-you-own-and-only-at-the-boundary)
12. [Functional / Hexagonal Architecture and Testability](#12-functional--hexagonal-architecture-and-testability)
13. [Integration Testing](#13-integration-testing)
14. [Working With Time, Randomness, and Other Hard Dependencies](#14-working-with-time-randomness-and-other-hard-dependencies)
15. [Anti-Patterns to Retire](#15-anti-patterns-to-retire)
16. [Big-Picture Take-Aways](#16-big-picture-take-aways)
17. [How This Book Fits the Repo](#17-how-this-book-fits-the-repo)
18. [Reading Order Recommendations](#18-reading-order-recommendations)
19. [Common Misreadings](#19-common-misreadings)
20. [References](#20-references)

---

## 1. About the Book

* **Title:** *Unit Testing: Principles, Practices, and Patterns*
* **Author:** Vladimir Khorikov (Microsoft MVP, Pluralsight instructor)
* **Publisher:** Manning, 2020
* **Length:** ~304 pages
* **Audience:** mid-level and senior engineers who already write tests but suspect they're writing the *wrong* tests. The book is opinionated — it will challenge habits that come from "common best practices" articles.

The book uses C# in its examples, but every principle is language-agnostic. The lessons apply identically to JUnit/Jest/PyTest/RSpec.

---

## 2. The Big Idea

Most "best practice" advice about unit tests is **vague** or **contradictory**. ("Use mocks." "Don't use mocks." "Test private methods." "Never test private methods." "Aim for 100% coverage." "Coverage isn't a quality metric.")

Khorikov's central claim: **all of these arguments resolve once you adopt a precise definition of what tests are *for*** — and his definition is the **four pillars**. With that framework in hand, you can score any test design and explain *why* one approach is better than another for a given context.

The book is, fundamentally, a tool for making test-design decisions defensible.

---

## 3. The Four Pillars of a Good Unit Test

Every test, no matter how it's written, has four measurable properties:

| Pillar                          | What it asks                                                                                | High value means…                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **1. Protection against regressions** | Does the test catch real bugs when they're introduced?                                  | The test covers business-critical code paths and assertions.    |
| **2. Resistance to refactoring** | Does the test stay green when you refactor *without* changing observable behavior?         | The test doesn't break when internal structure changes.         |
| **3. Fast feedback**            | How quickly does the test run and tell you about failure?                                   | Sub-second tests; failures arrive in seconds, not minutes.       |
| **4. Maintainability**           | How hard is the test to understand and update?                                              | Short, readable, no scaffolding needed to grasp it.              |

### The Cruel Theorem

You cannot have all four pillars at maximum. The first three are *mutually exclusive at the extremes*:

* A test that maximizes **regression protection** *and* **resistance to refactoring** must exercise as much real code as possible (slow). → loses **fast feedback**.
* A test that maximizes **regression protection** *and* **fast feedback** must mock everything outside a small unit (heavily coupled to implementation). → loses **resistance to refactoring**.
* A test that maximizes **resistance to refactoring** *and* **fast feedback** can't exercise much real code. → loses **regression protection**.

Every test design is a point in this triangle. Khorikov's reframing: **the goal of good testing is to maximize the area of the triangle**, not to maximize one corner.

> **The fourth pillar (maintainability) is non-negotiable.** A test you can't maintain is dead code that produces false confidence. The first three trade off; maintainability has to be paid for.

### A test's *value* is the product of the four pillars

Khorikov suggests treating the value of a test as roughly **`P × R × F × M`** — multiplicative, not additive. A test that scores zero on any pillar has zero value. Useless tests are not neutral — they cost time to maintain and create false confidence.

---

## 4. The Two Schools — Classical (Detroit) vs. London (Mockist)

The book defuses one of the longest-running debates in testing.

| Aspect                  | **Classical (Detroit / Chicago)**                            | **London (Mockist)**                                          |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------- |
| **Definition of "unit"** | A unit of *behavior*; may span many classes                  | One class                                                     |
| **Isolation**           | Tests are isolated from *each other* (no shared state)       | Each class is isolated from its *collaborators* with mocks    |
| **Mock usage**          | Use sparingly — only at the boundary of *unmanaged* dependencies | Use heavily — replace every collaborator with a mock        |
| **Test focus**          | Verify *outcomes* (state, return values)                     | Verify *interactions* (which method was called with what args)|
| **Resistance to refactoring** | High                                                   | Low                                                           |
| **Regression protection** | Higher per test (more real code exercised)                | Sometimes lower (mocks let bugs slip through)                 |

Khorikov is openly biased toward the **classical school**. His core argument: the London school produces tests that **couple tightly to implementation details**. Every internal refactor breaks tests. The fourth pillar — maintainability — collapses, and over time the test suite becomes a brake on the project rather than a safety net.

That said, he is fair about when mocks **are** appropriate (see §10).

---

## 5. What Is a "Unit"?

The single most-important corrective in the book.

The London school says: a unit = a class. So every test mocks every collaborator.

The classical school — and Khorikov — say: **a unit = a unit of *behavior***. It might be one class, or it might be three classes that always work together to implement one piece of business logic. **You test behavior, not classes.**

```csharp
// Behavior: "transferring money between accounts moves the amount and emits an event"
//
// Implementation: this might span Account, TransferService, EventBus, and Money — four classes.
// The unit test treats them as ONE unit, mocking nothing internal, asserting on observable outcome.

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

The test exercises four classes and zero mocks — and that's correct, because the *unit of behavior* spans those four classes. A bug in any of them fails this test. A pure refactor of any of them (rename a method, change a private field, extract a class) doesn't.

---

## 6. Observable Behavior vs. Implementation Details

Khorikov's most-quoted concept after the four pillars.

**Observable behavior** = anything a client of the code can detect. Return values, exceptions, state changes the client can query, calls to dependencies the client explicitly registered.

**Implementation details** = how the behavior is achieved internally. Private methods, private fields, the specific data structures, the call sequences between internal classes.

**The rule:** *tests should assert against observable behavior and ignore implementation details.*

| Observable                              | Implementation detail                      |
| --------------------------------------- | ------------------------------------------ |
| Return value of a public method         | A private helper method                    |
| State change visible via a public getter | An internal field (`_total`, `_count`)      |
| Exception thrown                        | The order of internal method calls         |
| A message published to an event bus     | The internal data structure used to queue it |
| A row inserted into a DB                | Whether a `using` block was used            |

A test that asserts an implementation detail breaks every time you refactor — even when the behavior is identical. That fails the **resistance to refactoring** pillar.

A useful gut check: **if a refactor preserves the observable behavior, the test must still pass.** If it doesn't, your test is testing implementation detail.

---

## 7. Mocks vs. Stubs — A Precise Distinction

Khorikov insists on Meszaros's vocabulary (*xUnit Test Patterns*, 2007). Most developers conflate these terms; the precision matters because it changes when each is appropriate.

### Test doubles (overview)

| Type            | Purpose                                                              | Example                                                  |
| --------------- | -------------------------------------------------------------------- | -------------------------------------------------------- |
| **Dummy**       | Fills a parameter slot; never called                                 | `null` or default-constructed object passed in           |
| **Fake**        | A working implementation, simplified for tests                       | In-memory repository                                     |
| **Stub**        | Returns canned data when queried; supplies *input* to the SUT       | `repo.GetById(id)` returns a fixed `Order`              |
| **Spy**         | Records calls; assertions check what happened                        | Records all emails "sent"; test asserts count            |
| **Mock**        | A pre-configured object that *verifies* interactions in advance     | "Expect `email.Send` called with X" — fails if not       |

### The book's headline rule

> **Asserting on a stub is an anti-pattern. Asserting on a mock is sometimes appropriate.**

Reason: a **stub** supplies *input* to the system under test. Asserting that the SUT *called* the stub is asserting on an implementation detail — *how* the SUT got the data, not *what it did with it*.

A **mock** verifies that the SUT triggered the *right side-effect* — and side-effects on *unmanaged* dependencies (third-party APIs, message buses, email senders) are themselves observable behavior of the system.

```csharp
// BAD — asserting on a stub (implementation-detail test, breaks on refactor)
[Fact]
public void Bad_test()
{
    var repo = new Mock<IOrderRepository>();
    repo.Setup(r => r.GetById(42)).Returns(new Order());

    var sut = new OrderService(repo.Object);
    sut.GetOrder(42);

    repo.Verify(r => r.GetById(42), Times.Once);   // ← brittle, doesn't add value
}

// GOOD — asserting on observable behavior (the return value)
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

## 8. The Three Test Types — Output-Based, State-Based, Communication-Based

Khorikov classifies tests by *what they assert*:

### 1. Output-based testing (functional)

The SUT is a pure function: input → output. The test verifies the output.

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

**Verdict:** the gold standard. Maximizes all four pillars simultaneously. **Strive to write code in a style that makes output-based tests possible** — pure functions, no hidden state, no side effects.

### 2. State-based testing

The SUT changes some state; the test verifies the resulting state via a query.

```csharp
[Fact]
public void Adding_an_item_increases_cart_total()
{
    var cart = new ShoppingCart();
    cart.Add(new LineItem(ProductId: 1, Quantity: 2, Price: 50m));
    Assert.Equal(100m, cart.Total);
}
```

**Verdict:** second-best. Good when the SUT genuinely owns state. Asserts on observable state via public API.

### 3. Communication-based testing (interaction)

The SUT collaborates with something else; the test verifies the *interaction* (often with mocks).

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

**Verdict:** the most fragile. Tests that *the SUT calls* something are testing implementation. *Sometimes* unavoidable — when the SUT's contract really *is* "to trigger a side-effect on an unmanaged dependency". But use sparingly.

> **Khorikov's guidance:** prefer output-based, fall back to state-based, use communication-based only at the application boundary.

---

## 9. The Anatomy of a Good Test (AAA)

Every test, regardless of type, follows the **Arrange / Act / Assert** structure:

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

### Khorikov's specific advice on AAA

* **Separate the three sections clearly.** Blank lines or comments. Don't mix them.
* **One "Act" per test.** Multiple acts = multiple behaviors = should be multiple tests. (Exception: integration tests sometimes legitimately need multiple steps.)
* **Naming matters.** The test name should describe the *behavior under test*, not the method name. `OrderService_Pay_Success` → `Paying_an_order_marks_it_as_paid_and_records_the_transaction`.
* **Avoid `if`/`switch` inside tests.** Each test exercises one path. Branching in tests means you're testing two things in one test.
* **The Given-When-Then style** (`BDD`) is the same idea with different keywords; both are fine.

### Test name patterns

* `[StateUnderTest]_[ExpectedBehavior]` — `EmptyCart_Total_IsZero`
* Plain English sentence — `Empty cart's total is zero`
* Khorikov prefers plain English. Readers don't need to decode camelCase.

---

## 10. Mocking — When, What, and What Not

The chapter that sparks the most arguments in code reviews.

### The two categories of dependencies

Khorikov draws a precise distinction:

| Category               | Definition                                                                | Examples                                              |
| ---------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Managed dependency**  | A dependency *only* this application uses; we control its lifecycle entirely | The application's own database (write-side)          |
| **Unmanaged dependency** | A dependency shared with other applications, or observable externally   | Third-party APIs, message buses, SMTP servers, payment gateways |

### The rule

> **Mock unmanaged dependencies. Don't mock managed dependencies — use the real thing.**

* **Unmanaged dependency** → side-effects on it are observable to *other systems*. Treat those side-effects as part of the SUT's observable behavior. Mock + verify.
* **Managed dependency** → side-effects on it are observable only via the SUT itself. Use a real DB in integration tests; assert on the *state* after, not on the *call*.

### Examples

| Dependency                         | Type       | How to test                                                        |
| ---------------------------------- | ---------- | ------------------------------------------------------------------ |
| Your application's SQL Server DB   | Managed    | Real DB (Testcontainers); assert on state via query                |
| Payment gateway (Stripe)           | Unmanaged  | Mock; verify `Charge` was called with the right amount             |
| SendGrid / SMTP                    | Unmanaged  | Mock; verify `Send` was called                                     |
| Internal message bus (consumed only by you) | Managed   | Use in-memory or real Kafka; assert state of consumers             |
| External webhook to a partner      | Unmanaged  | Mock; verify the partner contract                                  |
| Time / `DateTime.Now`              | Unmanaged-ish | Inject `IClock`; pass deterministic values                       |

The principle: **side-effects visible *outside* the application boundary are part of the SUT's observable behavior; side-effects visible *only inside* are implementation detail.**

---

## 11. Mock Only What You Own (And Only at the Boundary)

Khorikov endorses the **"Don't mock what you don't own"** rule (from *Growing Object-Oriented Software, Guided by Tests*):

* **You own:** interfaces you defined to abstract third-party libraries. Always wrap third-party concretes in your own port.
* **You don't own:** the third-party type itself (`HttpClient`, `Stripe.PaymentIntentService`, `SendGridClient`).

```csharp
// Good — wrap the third party in your own port, mock the port
public interface IPaymentGateway
{
    Task<PaymentResult> Charge(Money amount, string customerId, CancellationToken ct);
}

public sealed class StripePaymentGateway : IPaymentGateway
{
    private readonly Stripe.PaymentIntentService _stripe;
    public Task<PaymentResult> Charge(Money amount, string customerId, CancellationToken ct) { /* call Stripe */ }
}

// Test mocks IPaymentGateway, never Stripe directly
public void Order_payment_failure_marks_order_as_payment_failed()
{
    var gateway = new Mock<IPaymentGateway>();
    gateway.Setup(g => g.Charge(It.IsAny<Money>(), It.IsAny<string>(), default))
           .ReturnsAsync(PaymentResult.Declined("insufficient funds"));

    // ...
}
```

Mocking the third-party type directly couples your tests to the third party's API. When they change it, your tests break — but your behavior didn't.

---

## 12. Functional / Hexagonal Architecture and Testability

A whole chapter on how **architectural style** influences how much you can test with output-based tests.

The thesis: **the more you push business logic into pure functions / pure domain code, the better your tests become.**

### The functional core / imperative shell pattern

* **Functional core** — pure domain logic. No I/O, no side effects. Returns *what* should happen.
* **Imperative shell** — thin glue that performs the side effects the core decided on.

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

// Imperative shell: just orchestrates I/O around the core
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

* Tests on `OrderRules.DecideOnOrder` are **output-based**, run in microseconds, no mocks, no infrastructure.
* Tests on `OrderUseCase` are *integration tests* — one per high-level happy path or error path, real DB, mocked unmanaged dependencies.

This separation **maximizes the area of the four-pillars triangle** for the largest portion of your code.

### Hexagonal architecture as a special case

Hexagonal Architecture (ports & adapters) is *one* way to enforce this functional core / imperative shell split. The book endorses it. **Domain → 100% output-based; Application → integration; Adapters → almost no testing (humble objects).**

> Cross-reference: [`../dot-net/docs/architectural-style/hexagonal-architecture.md`](../dot-net/docs/architectural-style/hexagonal-architecture.md).

---

## 13. Integration Testing

The book's testing pyramid for an application following the functional-core pattern:

```
       /\
      /  \   ← E2E (1–5 most-critical scenarios)
     /----\
    /      \  ← Integration (per use case; real DB; mocked unmanaged deps)
   /--------\
  /          \ ← Unit (the functional core; pure; output-based; thousands of these)
 /____________\
```

### Khorikov's integration-test rules

1. **One integration test per high-level path through a use case.** Happy path, key failure paths. Not every variation — those are output-based unit tests on the core.
2. **Real database.** Use Testcontainers / a real local DB. In-memory replacements (EF Core's InMemory provider, SQLite-as-Postgres) hide bugs and provide false confidence.
3. **Mock unmanaged dependencies.** Stripe, email, message bus going to other systems.
4. **Don't mock the database.** It's a managed dependency.
5. **One Act per test, still.** Integration tests just exercise more code per act.
6. **Reset state between tests.** A clean database is part of the arrange step. (Transaction rollback, table truncation, or a fresh container.)
7. **Avoid shared mutable test fixtures.** They cause order-dependent test flakiness.

```csharp
// Integration test against a real Postgres via Testcontainers
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

## 14. Working With Time, Randomness, and Other Hard Dependencies

The chapter that finally answers: how do I test code that depends on `DateTime.Now`?

### The pattern: lift hard dependencies into ports

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

// Test passes a fixed clock — fully deterministic
[Fact]
public void Token_expires_one_hour_after_issuance()
{
    var fixedClock = new FixedClock(new DateTime(2026, 1, 1, 12, 0, 0));
    var sut = new TokenService(fixedClock);

    var token = sut.IssueToken(new User(1), TimeSpan.FromHours(1));

    Assert.Equal(new DateTime(2026, 1, 1, 13, 0, 0), token.ExpiresAt);
}
```

The same approach works for:
* `Random` → inject `IRandom`
* `Guid.NewGuid()` → inject `IGuidGenerator`
* Filesystem → inject `IFileSystem`
* Environment variables → inject `IConfig`

This is the **functional core** pattern again: push impurity to the edges, keep the core deterministic.

---

## 15. Anti-Patterns to Retire

Khorikov enumerates testing habits he considers **harmful**:

| Anti-pattern                                  | Why it's bad                                                                 |
| --------------------------------------------- | ---------------------------------------------------------------------------- |
| **Testing private methods**                   | Private methods are implementation details. If you must test one, the method is *yours* but hidden — make it `internal` and write a test through the public API that exercises it. |
| **Asserting on stubs (`Verify(stub)`)**       | Pure waste. The stub is input; asserting it was called doesn't validate behavior. |
| **Mocks for managed dependencies**            | Mocking the DB leads to tests that pass but the DB schema is wrong.          |
| **One test per method ("structural coverage")** | Counts but doesn't matter. Test *behaviors*, not methods.                  |
| **Time-coupled tests** (`Thread.Sleep`, real `DateTime.Now`) | Flaky and slow. Inject a clock.                              |
| **Test inheritance hierarchies for shared setup** | Hidden state, hard to read. Use simple helper methods or factories.       |
| **High mock count per test**                  | Symptom of *implementation testing*. Either refactor toward purity, or merge into one larger unit-of-behavior test. |
| **Tests that change after every refactor**    | Failing the resistance-to-refactoring pillar. The tests are coupled to implementation. |
| **100% coverage as a goal**                   | Coverage measures lines, not behaviors. Better to have 70% coverage of meaningful behaviors than 100% coverage of getters and trivial code. |
| **Test code without code review**             | Tests are production code. Review them.                                      |

---

## 16. Big-Picture Take-Aways

1. **Tests are valuable in proportion to all four pillars.** Optimize the *product*, not any one corner.
2. **Define a "unit" as a unit of *behavior*, not a class.** This single shift collapses most of the mocks-vs-classical debate.
3. **Test observable behavior; ignore implementation details.** The smell of "implementation-detail testing" is tests that break on every refactor.
4. **Mock only at the boundary** with unmanaged dependencies. The database is *not* a boundary.
5. **Wrap third-party libraries behind your own ports** and mock the ports, not the libraries.
6. **Functional core / imperative shell** is the architectural style that maximizes testability — pure logic in the core, I/O at the edge.
7. **Output-based > state-based > communication-based.** Lean toward writing code that admits the first.
8. **Use a real database in integration tests.** In-memory substitutes lie.
9. **Integration tests cover the use case; unit tests cover the domain logic.** Don't try to do both with one test type.
10. **Coverage is a side effect, not a goal.** Aim for the right behaviors covered, not the most lines.

---

## 17. How This Book Fits the Repo

| Concept in the book                              | Repo doc                                                                                  |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Functional core / imperative shell               | [`../dot-net/docs/architectural-style/hexagonal-architecture.md`](../dot-net/docs/architectural-style/hexagonal-architecture.md), [`../dot-net/docs/architectural-style/clean-architecture.md`](../dot-net/docs/architectural-style/clean-architecture.md) |
| Wrap third-party libs (ports)                    | [`../dot-net/docs/design-pattern/structural.md`](../dot-net/docs/design-pattern/structural.md) (Adapter section) |
| DI / testability                                  | [`../dot-net/docs/enterprise-pattern/ioc-di.md`](../dot-net/docs/enterprise-pattern/ioc-di.md), [`../dot-net/docs/enterprise-pattern/lifetimes.md`](../dot-net/docs/enterprise-pattern/lifetimes.md) |
| DDD aggregates (testing rich domain)             | [`../dot-net/docs/ddd/tactical-patterns.md`](../dot-net/docs/ddd/tactical-patterns.md)    |
| Test pyramid in microservices                    | [`building-microservices-sam-newman.md`](./building-microservices-sam-newman.md) (testing chapter) |
| SOLID principles, especially DIP                 | [`../principles/solid.md`](../principles/solid.md)                                       |

---

## 18. Reading Order Recommendations

* **First time:** read straight through. The four-pillars framework in Chapter 4 is the key chapter; everything before is setup, everything after is application.
* **If you only have an hour:** read Chapters 4 (four pillars), 5 (mocks vs. stubs), 7 (observable behavior), and 11 (functional architecture).
* **If your team is fighting about mocks:** Chapters 5, 8, and 11 together end the debate.
* **If your tests are slow and flaky:** Chapter 8 (integration tests) + the anti-patterns chapter.

---

## 19. Common Misreadings

| Misreading                                                          | Reality                                                                                                |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| "Khorikov says don't use mocks."                                    | He says use them sparingly — at the boundary, for unmanaged dependencies. He's against *over-mocking*.  |
| "A unit is one class."                                              | A unit is one *behavior*. It may span many classes.                                                     |
| "Coverage doesn't matter."                                          | Coverage matters; it's a *necessary* but not sufficient signal. 100% as a *goal* leads to bad tests.    |
| "Integration tests are unit tests with a DB."                       | They're a different test type with different goals. They protect more, refactor less, and run slower.   |
| "The book is C#-specific."                                          | No. Every principle is language-agnostic. C# is the *vehicle*, not the *subject*.                       |
| "Testing private methods is fine if I make them `internal`."        | Don't. Test the behavior through the public API. If a behavior requires a private method but isn't testable through the public API, the public API is the wrong shape. |

---

## 20. References

* Vladimir Khorikov — *Unit Testing: Principles, Practices, and Patterns*, Manning, 2020.
* Gerard Meszaros — *xUnit Test Patterns: Refactoring Test Code*, Addison-Wesley, 2007 (the canonical taxonomy of test doubles).
* Steve Freeman & Nat Pryce — *Growing Object-Oriented Software, Guided by Tests*, Addison-Wesley, 2009 (the canonical London-school book; Khorikov disagrees with it on key points, fairly).
* Kent Beck — *Test-Driven Development by Example*, Addison-Wesley, 2002.
* Martin Fowler — [*Mocks Aren't Stubs*](https://martinfowler.com/articles/mocksArentStubs.html) (2007). The article Khorikov builds on.
* Vladimir Khorikov's Pluralsight courses on testing — the book's content is a more polished version of those courses.
