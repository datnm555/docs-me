# Nguyên Lý Bổ Sung — Ngoài DRY, SOLID, và GRASP

> Catalog các nguyên lý khác mọi kỹ sư đang làm việc cuối cùng cũng internalize. Không cái nào là phần của SOLID hay GRASP, nhưng cùng nhau chúng cover các khía cạnh *behavioral*, *organizational*, và *defensive* của code tốt.

> 🇻🇳 Phiên bản tiếng Việt. English: [`additional.md`](./additional.md)

---

## Mục lục

1. [Law of Demeter (LoD)](#1-law-of-demeter-lod)
2. [Composition over Inheritance](#2-composition-over-inheritance)
3. [Tell, Don't Ask](#3-tell-dont-ask)
4. [Principle of Least Astonishment (POLA)](#4-principle-of-least-astonishment-pola)
5. [Fail Fast](#5-fail-fast)
6. [Convention over Configuration (CoC)](#6-convention-over-configuration-coc)
7. [Boy Scout Rule](#7-boy-scout-rule)
8. [Encapsulation](#8-encapsulation)
9. [Principle of Least Privilege](#9-principle-of-least-privilege)
10. [Single Source of Truth (SoT)](#10-single-source-of-truth-sot)
11. [Curly's Law (Do One Thing)](#11-curlys-law-do-one-thing)
12. [Hollywood Principle](#12-hollywood-principle)
13. [Worse Is Better](#13-worse-is-better)

---

## 1. Law of Demeter (LoD)

> *"Chỉ nói chuyện với bạn ngay sát."* — Northeastern University, 1987

Method `m` của class `C` chỉ nên call method của:

1. Bản thân `C`
2. Field của nó
3. Parameter của nó
4. Object nó tạo

Không phải trên object được trả về bởi chúng — đó là reach qua một object để chộp một object khác (gọi là **train wreck** code).

### Tệ — Train Wreck

```csharp
decimal total = order.Customer.Address.PostalCode.Zone.TaxRate * order.Subtotal;
```

Sáu class coupling. Rename ở bất kỳ đâu cũng vỡ cái này.

### Tốt — Tell, Don't Reach

```csharp
decimal total = order.CalculateTotal();   // Order biết customer của nó, customer biết address...
```

### Lưu ý

LoD **không** cấm method chaining khi cùng object được trả về (fluent API, LINQ). Nó cấm chain qua object không liên quan.

---

## 2. Composition over Inheritance

> *"Ưu tiên **composition** object hơn **inheritance** class."* — Gang of Four

Inheritance tạo liên kết vĩnh viễn, compile-time. Composition là quan hệ runtime — linh hoạt, có thể thay thế, mockable.

### Tệ — Lạm dụng Inheritance

```csharp
public class Bird { public virtual void Fly() { ... } }
public class Penguin : Bird { public override void Fly() => throw new NotSupportedException(); }
```

Penguin là chim **sinh học** nhưng không **về behavior** (vi phạm LSP nữa).

### Tốt — Compose Capability

```csharp
public interface IFlyBehavior { void Fly(); }
public class CanFly  : IFlyBehavior { public void Fly() { /* lift off */ } }
public class CantFly : IFlyBehavior { public void Fly() { /* no-op */ } }

public class Bird(IFlyBehavior fly)
{
    public void Fly() => fly.Fly();
}

var sparrow = new Bird(new CanFly());
var penguin = new Bird(new CantFly());
```

Đây chính xác là **Strategy pattern** — và gần như luôn linh hoạt hơn inheritance.

### Khi inheritance đúng

* Quan hệ *is-a* thật (một `SqlConnection` IS-A `DbConnection`).
* Bạn muốn polymorphism mà không cần per-instance configuration.
* Base class ổn định và child chỉ specialize.

---

## 3. Tell, Don't Ask

> *"Code procedural lấy thông tin rồi quyết định. Code object-oriented bảo object làm."* — Alec Sharp

### Tệ — Ask

```csharp
if (order.Status == OrderStatus.Pending && order.TotalAmount > 1000)
{
    order.Status = OrderStatus.RequiresApproval;
    notifier.NotifyManager(order);
}
```

Caller đang quyết định những thứ thuộc về `Order`.

### Tốt — Tell

```csharp
order.RequestApprovalIfNeeded(notifier);
```

```csharp
public void RequestApprovalIfNeeded(IManagerNotifier notifier)
{
    if (Status == OrderStatus.Pending && TotalAmount > 1000)
    {
        Status = OrderStatus.RequiresApproval;
        notifier.NotifyManager(this);
    }
}
```

Liên quan: **Information Expert**, **Encapsulation**, và **Law of Demeter**.

---

## 4. Principle of Least Astonishment (POLA)

> *"Component nên hành xử theo cách phần lớn user mong đợi."*

Nếu method tên `GetUser` *cũng* ghi log, gửi email, và update cache — astonish. Code astonishing trở thành **code hút bug**.

### Quy tắc thực hành

* Method `Get*` không mutate state.
* Method `Save*` không silent call HTTP tới third-party.
* Operator (`==`, `+`) hành xử như primitive analog.
* Cùng input cho cùng output (purity nếu có thể).

### Tệ

```csharp
public bool IsAuthorized(User u)
{
    if (u == null) Environment.Exit(1);   // astonishing — kill process
    ...
}
```

### Tốt

```csharp
public bool IsAuthorized(User u)
{
    ArgumentNullException.ThrowIfNull(u);
    ...
}
```

---

## 5. Fail Fast

> *"Error nên được phát hiện **càng sớm càng tốt** — ưu tiên compile time, kế là startup, kế là lần invalid call đầu tiên."*

Bug xuất hiện gần nguyên nhân thì rẻ. Bug xuất hiện trong production lúc 3 giờ sáng tốn tiền.

### Tệ — Silent Drift

```csharp
public class Config
{
    public string ConnectionString { get; set; }   // null cho tới khi có gì đó dùng
}
```

Typo trong `appsettings.json` không phát hiện được tới khi query chạy.

### Tốt — Validate ở Startup

```csharp
public class Config
{
    [Required, MinLength(1)] public required string ConnectionString { get; init; }
}

builder.Services.AddOptions<Config>()
    .Bind(builder.Configuration.GetSection("Db"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Fail ngay lúc boot — visible, fixable, không trong production.

### Áp dụng ở đâu

* Constructor argument check (`ArgumentNullException`, `ArgumentException`).
* Strong type thay primitive (vd. value object `EmailAddress`).
* Schema validation cho incoming request.
* Field `readonly` / `init` để ngăn mutation sau.

---

## 6. Convention over Configuration (CoC)

> *"Developer chỉ cần chỉ định các khía cạnh **không quy ước** của application."* — David Heinemeier Hansson, Rails

Nếu framework dùng default hợp lý, developer viết ít code hơn. Chỉ deviation cần configuration.

### Ví dụ

* **ASP.NET Core**: `Controllers/HomeController.cs` tự map sang `/Home/*`.
* **Entity Framework**: `Id` là primary key trừ khi bạn nói khác.
* **Maven / npm**: layout project chuẩn nghĩa là không setup.

### Trade-off

Default kỳ diệu rất tốt cho tới khi bất ngờ bạn. POLA giới hạn CoC: quy ước nên **dễ phát hiện**, **được document**, và **có thể override**.

---

## 7. Boy Scout Rule

> *"Luôn để khu cắm trại sạch hơn lúc bạn tìm thấy."* — Robert C. Martin

Khi bạn touch một file, làm cải thiện nhỏ: một rename, thêm null check, xoá comment, extract method. Cải thiện nhỏ liên tục cộng dồn — đó là cách codebase legacy phục hồi mà không cần "refactor sprint" riêng.

### Quy tắc tham gia

* Đừng gói drive-by refactor vào feature PR — mở PR riêng.
* Đừng fix cả file; fix gần thay đổi của bạn.
* Đừng reformat code bạn không thay đổi (giết `git blame`).

---

## 8. Encapsulation

> *"Gom data và behavior; chỉ expose cái caller cần."*

Encapsulation là **tường chịu lực** của thiết kế OO — gần như mọi nguyên lý khác giả định nó.

### Tệ — Anemic Object

```csharp
public class BankAccount
{
    public decimal Balance;
}

// Ở bất kỳ đâu trong hệ thống:
account.Balance -= 100;  // không validate, không audit, không concurrency
```

### Tốt — Behavior sở hữu Data

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Must be positive");
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}
```

Bây giờ invariant được enforce ở **một chỗ**, và caller không thể vỡ chúng.

---

## 9. Principle of Least Privilege

> *"Mỗi module chỉ được phép truy cập thông tin và resource cần thiết cho mục đích hợp pháp của nó."* — Saltzer & Schroeder, 1975

Ban đầu là nguyên lý security, nhưng áp dụng tương đương cho thiết kế code:

* `private` hơn `public`.
* `readonly` hơn mutable.
* `IReadOnlyList<T>` hơn `List<T>` trong giá trị trả về.
* Database user với grant *tối thiểu* cần.
* Service account scope cho API *tối thiểu* chúng touch.

Mỗi cửa đóng là một bề mặt tấn công ít đi và một footgun ít đi.

---

## 10. Single Source of Truth (SoT)

> *"Mỗi mẩu data nên có chính xác **một** nhà có thẩm quyền."*

Họ hàng gần của DRY, nhưng cụ thể về *state* (DRY về *knowledge*).

### Tệ

```csharp
public class Order
{
    public List<LineItem> Items { get; }
    public int TotalQuantity { get; set; }   // trùng với Items.Sum(i => i.Qty)
}
```

`TotalQuantity` có thể drift khỏi thực tế. Hai source of truth = một cái sẽ sai.

### Tốt

```csharp
public class Order
{
    public IReadOnlyList<LineItem> Items { get; }
    public int TotalQuantity => Items.Sum(i => i.Qty);   // derived, không bao giờ store
}
```

---

## 11. Curly's Law (Do One Thing)

> *"Một biến, function, hoặc class nên có nghĩa **một thứ** và chỉ một thứ."* — Tim Ottinger / Jeff Atwood

Họ hàng gần của SRP, nhưng ở mọi mức granularity:

* Một **biến** không nên reuse cho hai mục đích (`var x = ...; x = ...somethingElse...`).
* Một **function** không nên "save và return formatted" — tách ra.
* Một **class** nên có mục đích đặt tên được, đơn lẻ.

Nếu bạn không thể đặt tên cái gì đó bằng noun hoặc verb-noun, nó đang làm quá nhiều thứ.

---

## 12. Hollywood Principle

> *"Đừng call chúng tôi — chúng tôi sẽ call bạn."*

Framework đảo control: thay vì code của bạn call vào library, library call code của bạn. Đây là nền của:

* **Dependency Injection container**
* **Hệ thống event-driven**
* **Template Method pattern**
* **Lifecycle hook** (`OnInit`, `OnDestroy`, `BeforeSave`)
* **Callback / observer**

Cùng idea với **DIP** nhìn từ góc nhìn của tác giả framework.

---

## 13. Worse Is Better

> *"Một chương trình 90% đúng, đơn giản, và nhanh thường thắng cái 100% đúng, phức tạp, và chậm."* — Richard Gabriel, 1989

Đối trọng với chủ nghĩa hoàn hảo. *Thực dụng* thắng *thuần khiết*. Ship giải pháp tẻ nhạt giải 95% case; xem lại khi 5% còn lại thành vấn đề thật.

### Bản dịch

* Worse Is Better → **YAGNI** + **KISS** + **Lean**
* Framework hoàn hảo ship trong 18 tháng thua cái có lỗi ship trong 3.
* Thiết kế library nên tối ưu cho **adoption**, không tinh tế lý thuyết.

---

## Tham chiếu nhanh

| Nguyên lý                    | Bài học một dòng                                                    |
| ---------------------------- | ------------------------------------------------------------------- |
| Law of Demeter               | Chỉ nói chuyện với bạn ngay sát.                                    |
| Composition over Inheritance | Ưu tiên has-a hơn is-a.                                             |
| Tell, Don't Ask              | Gửi command; đừng probe state.                                      |
| POLA                         | Không bất ngờ trong public behavior.                                |
| Fail Fast                    | Bắt bug ở thời điểm sớm nhất có thể.                                |
| Convention over Configuration| Default hợp lý; chỉ configure deviation.                            |
| Boy Scout Rule               | Để file sạch hơn lúc tìm thấy.                                      |
| Encapsulation                | Behavior sở hữu data nó phụ thuộc.                                  |
| Least Privilege              | Cấp truy cập tối thiểu cần thiết.                                   |
| Single Source of Truth       | Mỗi fact sống ở chính xác một nơi.                                  |
| Curly's Law                  | Một tên = một mục đích.                                             |
| Hollywood Principle          | Đừng call chúng tôi — chúng tôi sẽ call bạn.                        |
| Worse Is Better              | Ship cái đơn giản trên cái hoàn hảo.                                |

---

> **Suy nghĩ cuối:** nguyên lý cộng dồn. Encapsulation cho phép Tell-Don't-Ask. Tell-Don't-Ask cho phép Law of Demeter. Law of Demeter giảm coupling. Coupling thấp làm SRP dễ hơn. SRP làm OCP không đau. Bắt đầu ở đâu cũng được; cả công trình mạnh hơn cùng nhau.
