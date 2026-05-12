# Core Principles — DRY, KISS, YAGNI, SoC

> The four most-quoted, most-misunderstood principles in software engineering. Master these first — they underpin every other principle in this guide.

---

## Table of Contents

1. [DRY — Don't Repeat Yourself](#1-dry--dont-repeat-yourself)
2. [KISS — Keep It Simple, Stupid](#2-kiss--keep-it-simple-stupid)
3. [YAGNI — You Aren't Gonna Need It](#3-yagni--you-arent-gonna-need-it)
4. [SoC — Separation of Concerns](#4-soc--separation-of-concerns)
5. [How They Interact](#5-how-they-interact)

---

## 1. DRY — Don't Repeat Yourself

> *"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."*
> — Andy Hunt & Dave Thomas, *The Pragmatic Programmer* (1999)

### 1.1 What It Really Means

DRY is about **knowledge duplication**, not code duplication. Two code blocks that look identical but represent *different* business rules are **not** a DRY violation — they only happen to coincide.

### 1.2 Bad — Knowledge Duplicated

```csharp
// Tax rate hard-coded in three places
public decimal CalcOrderTotal(Order o)   => o.Subtotal * 1.10m;
public decimal CalcInvoiceTotal(Invoice i) => i.Subtotal * 1.10m;
public decimal CalcQuoteTotal(Quote q)   => q.Subtotal * 1.10m;
```

When the tax rate changes, you must hunt and replace.

### 1.3 Good — Single Source of Truth

```csharp
public static class TaxPolicy
{
    public const decimal Rate = 0.10m;
    public static decimal Apply(decimal subtotal) => subtotal * (1 + Rate);
}

public decimal CalcOrderTotal(Order o)     => TaxPolicy.Apply(o.Subtotal);
public decimal CalcInvoiceTotal(Invoice i) => TaxPolicy.Apply(i.Subtotal);
```

### 1.4 When DRY Hurts (False DRY)

Don't unify code just because it *looks* the same.

```csharp
// Both classes compute "discount" but for unrelated reasons
public decimal EmployeeDiscount(Employee e) => e.Salary * 0.10m;
public decimal LoyaltyDiscount(Customer c)  => c.SpendThisYear * 0.10m;
```

Extracting a shared `ApplyTenPercent()` couples Employees to Customers — a bad trade.

### 1.5 Rule of Thumb

* **Same knowledge?** → Refactor.
* **Same code, different reasons to change?** → Leave it.
* **Wait for the third occurrence** before extracting (Rule of Three).

---

## 2. KISS — Keep It Simple, Stupid

> *"Simplicity is the ultimate sophistication."* — Leonardo da Vinci (attributed)

Coined in the **U.S. Navy in 1960** by aircraft engineer **Kelly Johnson**: a jet must be repairable by an average mechanic in a combat zone using common tools.

### 2.1 What It Really Means

Choose the **simplest design that solves today's problem** — not the simplest-looking one, and definitely not the cleverest one.

### 2.2 Bad — Clever, Hard to Read

```csharp
// One-line "clever"
var grouped = items
    .GroupBy(i => i.Date.AddDays(-(int)i.Date.DayOfWeek))
    .ToDictionary(g => g.Key, g => g.Sum(x => x.Amount * (x.IsRefund ? -1 : 1)));
```

### 2.3 Good — Simple, Explicit

```csharp
DateTime StartOfWeek(DateTime d) => d.AddDays(-(int)d.DayOfWeek);

decimal SignedAmount(Transaction t) => t.IsRefund ? -t.Amount : t.Amount;

var grouped = items
    .GroupBy(StartOfWeek)
    .ToDictionary(g => g.Key, g => g.Sum(SignedAmount));
```

Same logic — but each step is named, testable, and obvious.

### 2.4 Signs You're Violating KISS

* Generic type parameters nobody uses.
* "Configurable" classes with no second configuration.
* Five layers of indirection to call one method.
* A framework where a function would do.

### 2.5 Heuristic

> *"Would a tired engineer at 3 a.m. understand this in 30 seconds?"*

If no — simplify.

---

## 3. YAGNI — You Aren't Gonna Need It

> *"Always implement things when you actually need them, never when you just foresee that you need them."*
> — Ron Jeffries, Extreme Programming (XP), late 1990s

### 3.1 What It Really Means

Speculative features are **liabilities** even when they're free to write:

* They must be **maintained** forever.
* They constrain future decisions.
* They confuse readers ("why is this here?").
* They are often wrong — the actual need looks different when it arrives.

### 3.2 Bad — Speculative Generality

```csharp
public interface IUserRepository
{
    User GetById(int id);
    User GetByEmail(string email);
    User GetByPhone(string phone);       // not used anywhere
    User GetByPassportNumber(string p);  // not used anywhere
    Task<User> GetByIdAsync(int id);     // sync version still in use
}
```

### 3.3 Good — Build for Today

```csharp
public interface IUserRepository
{
    User GetById(int id);
    User GetByEmail(string email);
}
```

Add `GetByPhone` the day a real caller needs it.

### 3.4 The Cost of "Just In Case"

| Hidden cost     | Symptom                                                          |
| --------------- | ---------------------------------------------------------------- |
| Maintenance     | Tests, docs, and refactors for unused branches.                  |
| Mental load     | Readers wonder *"is this important?"*                            |
| Coupling        | Unused parameters/interfaces propagate through the codebase.     |
| Wrong shape     | The real future requirement rarely matches the speculation.      |

### 3.5 Heuristic

> *"Delete the line. Does anything break today?"*
> If no — it shouldn't be there.

---

## 4. SoC — Separation of Concerns

> *"Concerns are the different aspects of software functionality. The 'business logic' concern is separated from the 'persistence' concern, and so on."*
> — Edsger W. Dijkstra, 1974

### 4.1 What It Really Means

A **concern** is anything that can change for its own reason: UI, business rules, persistence, logging, validation, authentication. Each should live in its own module, layer, or function so that a change to one doesn't ripple across all.

### 4.2 Bad — All Concerns in One Place

```csharp
public class OrderController
{
    public IActionResult Place(OrderDto dto)
    {
        // 1. Validation
        if (dto.Items.Count == 0) return BadRequest();

        // 2. Business logic
        var total = dto.Items.Sum(i => i.Price * i.Qty) * 1.10m;

        // 3. Persistence
        using var conn = new SqlConnection("...");
        conn.Open();
        new SqlCommand("INSERT INTO Orders ...", conn).ExecuteNonQuery();

        // 4. Email
        new SmtpClient("smtp.acme.com").Send(new MailMessage(...));

        // 5. Logging
        File.AppendAllText("log.txt", $"Order placed: {total}\n");
        return Ok();
    }
}
```

Five reasons to change. Untestable. Unscalable.

### 4.3 Good — Each Concern Isolated

```csharp
public class OrderController(IOrderService orders) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Place(OrderDto dto)
    {
        await orders.PlaceAsync(dto);
        return Ok();
    }
}

public class OrderService(
    IOrderValidator validator,
    IPricingService pricing,
    IOrderRepository repo,
    INotificationService notifier,
    ILogger<OrderService> logger)
{
    public async Task PlaceAsync(OrderDto dto)
    {
        validator.Validate(dto);
        var total = pricing.Compute(dto);
        var id    = await repo.SaveAsync(dto, total);
        await notifier.NotifyAsync(id);
        logger.LogInformation("Order {Id} placed for {Total}", id, total);
    }
}
```

Each collaborator can be **swapped, tested, and scaled** independently.

### 4.4 Where SoC Shows Up

* **Layered architecture** — UI / Application / Domain / Infrastructure.
* **MVC / MVVM** — model, view, controller/viewmodel.
* **Aspect-oriented concerns** — logging, transactions, auth.
* **Microservices** — bounded contexts as deployment units.

---

## 5. How They Interact

These four principles **reinforce and constrain** each other:

```
            DRY ─────────┐
             │           │
             │ pulls     │ pushes
             ▼ toward    ▼ against
        abstraction ◀──── KISS / YAGNI
             │
             ▼
     Separation of Concerns
       (where to draw lines)
```

* **DRY pushes you to abstract.**
* **KISS and YAGNI pull you back** from over-abstraction.
* **SoC tells you *where* the line goes** when you decide to abstract.

The art is in the balance — and that balance is what makes a senior engineer.

---

> *"Make it work. Make it right. Make it fast."* — Kent Beck
> Apply DRY/KISS/YAGNI/SoC in roughly that order over a feature's lifetime.
