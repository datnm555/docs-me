# Nguyên Lý Cốt Lõi — DRY, KISS, YAGNI, SoC

> Bốn nguyên lý được trích dẫn nhiều nhất, hiểu lầm nhiều nhất trong software engineering. Master chúng trước — chúng là nền cho mọi nguyên lý khác trong guide này.

> 🇻🇳 Phiên bản tiếng Việt. English: [`core.md`](./core.md)

---

## Mục lục

1. [DRY — Don't Repeat Yourself](#1-dry--dont-repeat-yourself)
2. [KISS — Keep It Simple, Stupid](#2-kiss--keep-it-simple-stupid)
3. [YAGNI — You Aren't Gonna Need It](#3-yagni--you-arent-gonna-need-it)
4. [SoC — Separation of Concerns](#4-soc--separation-of-concerns)
5. [Chúng tương tác ra sao](#5-chúng-tương-tác-ra-sao)

---

## 1. DRY — Don't Repeat Yourself

> *"Mỗi mẩu kiến thức phải có representation duy nhất, không mơ hồ, có thẩm quyền trong một hệ thống."*
> — Andy Hunt & Dave Thomas, *The Pragmatic Programmer* (1999)

### 1.1 Thực sự nghĩa là gì

DRY là về **trùng lặp kiến thức**, không phải trùng lặp code. Hai block code nhìn giống hệt nhưng biểu diễn business rule *khác* **không** là vi phạm DRY — chúng chỉ tình cờ trùng.

### 1.2 Tệ — Kiến thức trùng lặp

```csharp
// Tax rate hard-code ở 3 nơi
public decimal CalcOrderTotal(Order o)   => o.Subtotal * 1.10m;
public decimal CalcInvoiceTotal(Invoice i) => i.Subtotal * 1.10m;
public decimal CalcQuoteTotal(Quote q)   => q.Subtotal * 1.10m;
```

Khi tax rate thay đổi, bạn phải đi săn và replace.

### 1.3 Tốt — Single Source of Truth

```csharp
public static class TaxPolicy
{
    public const decimal Rate = 0.10m;
    public static decimal Apply(decimal subtotal) => subtotal * (1 + Rate);
}

public decimal CalcOrderTotal(Order o)     => TaxPolicy.Apply(o.Subtotal);
public decimal CalcInvoiceTotal(Invoice i) => TaxPolicy.Apply(i.Subtotal);
```

### 1.4 Khi DRY gây hại (False DRY)

Đừng gộp code chỉ vì nó *trông* giống.

```csharp
// Cả hai class tính "discount" nhưng vì lý do không liên quan
public decimal EmployeeDiscount(Employee e) => e.Salary * 0.10m;
public decimal LoyaltyDiscount(Customer c)  => c.SpendThisYear * 0.10m;
```

Extract một `ApplyTenPercent()` chung sẽ couple Employee với Customer — trade-off tệ.

### 1.5 Quy tắc

* **Cùng kiến thức?** → Refactor.
* **Cùng code, khác lý do thay đổi?** → Để yên.
* **Chờ lần xuất hiện thứ ba** trước khi extract (Rule of Three).

---

## 2. KISS — Keep It Simple, Stupid

> *"Đơn giản là đỉnh cao của tinh tế."* — Leonardo da Vinci (gán)

Đặt tên trong **U.S. Navy 1960** bởi kỹ sư máy bay **Kelly Johnson**: một jet phải sửa được bởi thợ máy trung bình trong khu chiến đấu dùng dụng cụ phổ thông.

### 2.1 Thực sự nghĩa là gì

Chọn **thiết kế đơn giản nhất giải bài toán hôm nay** — không phải cái trông đơn giản nhất, và chắc chắn không phải cái thông minh nhất.

### 2.2 Tệ — Thông minh, khó đọc

```csharp
// "Thông minh" một dòng
var grouped = items
    .GroupBy(i => i.Date.AddDays(-(int)i.Date.DayOfWeek))
    .ToDictionary(g => g.Key, g => g.Sum(x => x.Amount * (x.IsRefund ? -1 : 1)));
```

### 2.3 Tốt — Đơn giản, rõ ràng

```csharp
DateTime StartOfWeek(DateTime d) => d.AddDays(-(int)d.DayOfWeek);

decimal SignedAmount(Transaction t) => t.IsRefund ? -t.Amount : t.Amount;

var grouped = items
    .GroupBy(StartOfWeek)
    .ToDictionary(g => g.Key, g => g.Sum(SignedAmount));
```

Cùng logic — nhưng mỗi step được đặt tên, testable, và rõ ràng.

### 2.4 Dấu hiệu bạn vi phạm KISS

* Generic type parameter không ai dùng.
* Class "configurable" mà không có configuration thứ hai.
* 5 layer indirection để call một method.
* Một framework chỗ một function là đủ.

### 2.5 Heuristic

> *"Một kỹ sư mệt mỏi lúc 3 giờ sáng có hiểu cái này trong 30 giây không?"*

Không — đơn giản hoá.

---

## 3. YAGNI — You Aren't Gonna Need It

> *"Luôn implement khi bạn thực sự cần, không bao giờ khi bạn chỉ tiên đoán cần."*
> — Ron Jeffries, Extreme Programming (XP), cuối thập niên 1990

### 3.1 Thực sự nghĩa là gì

Feature đầu cơ là **gánh nặng** kể cả khi viết miễn phí:

* Phải **maintain** mãi.
* Constrain quyết định tương lai.
* Confuse người đọc ("tại sao cái này ở đây?").
* Thường là sai — nhu cầu thật trông khác khi đến.

### 3.2 Tệ — Speculative Generality

```csharp
public interface IUserRepository
{
    User GetById(int id);
    User GetByEmail(string email);
    User GetByPhone(string phone);       // không dùng ở đâu
    User GetByPassportNumber(string p);  // không dùng ở đâu
    Task<User> GetByIdAsync(int id);     // version sync vẫn đang dùng
}
```

### 3.3 Tốt — Build cho hôm nay

```csharp
public interface IUserRepository
{
    User GetById(int id);
    User GetByEmail(string email);
}
```

Thêm `GetByPhone` cái ngày một caller thực sự cần.

### 3.4 Chi phí của "Just In Case"

| Chi phí ẩn      | Triệu chứng                                                      |
| --------------- | ---------------------------------------------------------------- |
| Maintenance     | Test, doc, và refactor cho branch không dùng.                    |
| Tải tâm trí     | Người đọc tự hỏi *"có quan trọng không?"*                        |
| Coupling        | Parameter/interface không dùng lan trong codebase.               |
| Shape sai       | Requirement tương lai thật hiếm khi match speculation.            |

### 3.5 Heuristic

> *"Xoá dòng đó. Có gì hỏng hôm nay không?"*
> Không — nó không nên ở đó.

---

## 4. SoC — Separation of Concerns

> *"Concern là các khía cạnh khác nhau của software functionality. Concern 'business logic' tách khỏi concern 'persistence', v.v."*
> — Edsger W. Dijkstra, 1974

### 4.1 Thực sự nghĩa là gì

**Concern** là bất cứ gì có thể thay đổi vì lý do riêng: UI, business rule, persistence, logging, validation, authentication. Mỗi cái nên sống trong module, layer, hoặc function riêng để thay đổi một cái không lan qua tất cả.

### 4.2 Tệ — Mọi concern ở một chỗ

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

5 lý do để thay đổi. Untestable. Unscalable.

### 4.3 Tốt — Mỗi concern được cô lập

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

Mỗi collaborator có thể **swap, test, scale** độc lập.

### 4.4 SoC xuất hiện ở đâu

* **Layered architecture** — UI / Application / Domain / Infrastructure.
* **MVC / MVVM** — model, view, controller/viewmodel.
* **Aspect-oriented concern** — logging, transaction, auth.
* **Microservices** — bounded context như deployment unit.

---

## 5. Chúng tương tác ra sao

Bốn nguyên lý này **củng cố và ràng buộc** lẫn nhau:

```
            DRY ─────────┐
             │           │
             │ kéo       │ đẩy
             ▼ về        ▼ chống lại
        abstraction ◀──── KISS / YAGNI
             │
             ▼
     Separation of Concerns
       (vạch ranh giới ở đâu)
```

* **DRY đẩy bạn về abstract.**
* **KISS và YAGNI kéo bạn lại** khỏi over-abstraction.
* **SoC bảo bạn *vạch ranh giới ở đâu*** khi bạn quyết định abstract.

Nghệ thuật ở chỗ cân bằng — và cân bằng đó là cái làm nên senior engineer.

---

> *"Make it work. Make it right. Make it fast."* — Kent Beck
> Áp dụng DRY/KISS/YAGNI/SoC theo thứ tự đó qua đời sống feature.
