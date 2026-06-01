# GRASP — 9 Nguyên Lý Gán Trách Nhiệm

> **GRASP** = **G**eneral **R**esponsibility **A**ssignment **S**oftware **P**atterns.
> Giới thiệu bởi **Craig Larman** trong *Applying UML and Patterns* (1997). Trong khi SOLID hỏi *"class này có được thiết kế tốt không?"*, GRASP hỏi câu cơ bản hơn: ***"trách nhiệm này nên thuộc class nào?"***

> 🇻🇳 Phiên bản tiếng Việt. English: [`grasp.md`](./grasp.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — 9 nguyên lý GRASP (Craig Larman): **Information Expert**, **Creator**, **Controller**, **Low Coupling**, **High Cohesion**, **Polymorphism**, **Pure Fabrication**, **Indirection**, **Protected Variations**.
- **Tại sao** — Khi SOLID hỏi *"class này có được thiết kế tốt?"*, GRASP hỏi câu cơ bản hơn: ***"trách nhiệm này nên thuộc class nào?"***. Cùng nhau chúng cover cả design và allocation của trách nhiệm.
- **Khi nào** — Quyết định đặt method mới ở đâu; review smell "Feature Envy"; giải thích tại sao Repository (Pure Fabrication) không represent concept domain thật; thiết kế Mediator (Indirection).
- **Ở đâu** — Allocation trách nhiệm mức class. Foundation cho hiểu *tại sao* pattern GoF được shape theo cách chúng được.

---

## Mục lục

1. [Information Expert](#1-information-expert)
2. [Creator](#2-creator)
3. [Controller](#3-controller)
4. [Low Coupling](#4-low-coupling)
5. [High Cohesion](#5-high-cohesion)
6. [Polymorphism](#6-polymorphism)
7. [Pure Fabrication](#7-pure-fabrication)
8. [Indirection](#8-indirection)
9. [Protected Variations](#9-protected-variations)
10. [Tóm tắt](#10-tóm-tắt)

---

## 1. Information Expert

> *"Gán trách nhiệm cho class **có thông tin** cần để thực hiện."*

Nếu một class đã sở hữu data, đặt behavior ở chỗ khác tạo **Feature Envy**.

### Tệ

```csharp
public class Order { public IList<LineItem> Items { get; } = new List<LineItem>(); }

public class OrderTotalService
{
    public decimal Total(Order o) => o.Items.Sum(i => i.Price * i.Qty);
}
```

`OrderTotalService` chỉ tồn tại để đọc data của `Order` — order nên tự tính.

### Tốt

```csharp
public class Order
{
    public IList<LineItem> Items { get; } = new List<LineItem>();
    public decimal Total() => Items.Sum(i => i.Price * i.Qty);
}
```

---

## 2. Creator

> *"Class B nên tạo object của class A khi ít nhất một trong các điều này đúng:*
> *— B aggregate A*
> *— B chứa A*
> *— B record A*
> *— B dùng A chặt chẽ*
> *— B có data khởi tạo cho A."*

### Ví dụ

`Order` chứa `LineItem` — nên `Order` nên tạo chúng.

```csharp
public class Order
{
    private readonly List<LineItem> _items = new();

    public void AddItem(Product p, int qty)
    {
        _items.Add(new LineItem(p, qty, p.Price));   // Order là Creator
    }
}
```

Tránh leak logic construction vào client và giữ invariant tập trung.

---

## 3. Controller

> *"Gán trách nhiệm xử lý **system event** cho class đại diện **toàn hệ thống, use-case scenario, hoặc façade**."*

Controller nằm giữa UI và domain — nó không *làm* việc, nó **orchestrate**.

### Ví dụ

```csharp
// ASP.NET Core — controller mỏng, delegate cho use-case service
public class CheckoutController(ICheckoutUseCase useCase) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Place([FromBody] PlaceOrderRequest req)
    {
        var result = await useCase.PlaceOrderAsync(req);
        return result.IsSuccess ? Ok(result.Value) : BadRequest(result.Error);
    }
}
```

### Form tệ (Fat Controller)

Controller mà validate, tính total, charge payment, gửi email, và write DB vi phạm SRP **và** Controller.

---

## 4. Low Coupling

> *"Gán trách nhiệm sao cho **coupling vẫn thấp**."*

Coupling = mức độ một class phụ thuộc class khác. Coupling thấp nghĩa là:

* Dễ thay đổi một class mà không đụng class khác.
* Dễ test isolate.
* Dễ reuse.

### Triệu chứng Coupling cao

* Class biết field *internal* của nhau.
* Thay đổi module A vỡ test module B.
* Không compile được một module mà không có 10 cái khác.

### Công cụ giảm Coupling

* Phụ thuộc **interface**, không phải concrete (DIP).
* Dùng **event / message** thay vì call trực tiếp.
* Áp dụng **Indirection** (xem dưới).
* Tôn trọng **Law of Demeter**.

---

## 5. High Cohesion

> *"Gán trách nhiệm sao cho **cohesion vẫn cao**."*

Cohesion = mức độ các trách nhiệm bên trong một class thuộc về nhau.

### Tệ — Cohesion thấp (Utility Class)

```csharp
public static class Helpers
{
    public static string  Capitalize(string s)        { ... }
    public static decimal CalculateTax(decimal d)      { ... }
    public static void    SendEmail(string to)         { ... }
    public static byte[]  CompressZip(byte[] data)     { ... }
}
```

Bốn trách nhiệm không liên quan — một class. Mọi thay đổi đụng nó.

### Tốt — Cohesion cao

```csharp
public static class StringFormatting { public static string Capitalize(string s) { ... } }
public static class TaxPolicy        { public static decimal Apply(decimal d) { ... } }
public class MailService             { public void Send(string to) { ... } }
public static class ZipCompression   { public static byte[] Compress(byte[] data) { ... } }
```

> **Low coupling + high cohesion** là cặp metric quan trọng nhất trong thiết kế OO. Gần như mọi nguyên lý khác tồn tại để phục vụ chúng.

---

## 6. Polymorphism

> *"Khi alternative hoặc behavior vary theo type, **dùng operation polymorphic**, không phải type check."*

Đây là **OCP enabler** ở mức gán trách nhiệm.

### Tệ

```csharp
public decimal Area(Shape s)
{
    if (s is Circle c)    return Math.PI * c.R * c.R;
    if (s is Square sq)   return sq.Side * sq.Side;
    if (s is Triangle t)  return 0.5m * t.B * t.H;
    throw new InvalidOperationException();
}
```

### Tốt

```csharp
public abstract class Shape { public abstract decimal Area(); }
public class Circle(decimal r)               : Shape { public override decimal Area() => (decimal)Math.PI * r * r; }
public class Square(decimal side)            : Shape { public override decimal Area() => side * side; }
public class Triangle(decimal b, decimal h)  : Shape { public override decimal Area() => 0.5m * b * h; }
```

Thêm `Pentagon` không đụng `Area()` — đóng cho modification, mở cho extension.

---

## 7. Pure Fabrication

> *"Khi không class domain nào fit, **phát minh class** không có domain counterpart để giữ cohesion và coupling sạch."*

Pure Fabrication = class *được phát minh* không model khái niệm domain nhưng có giá trị về mặt kỹ thuật.

### Ví dụ

* `OrderRepository` — không có "repository" trong business domain, nhưng cô lập persistence.
* `EmailSender` — service object tạo thuần vì nhu cầu kỹ thuật.
* `TaxCalculator` — extract ra để `Order` không phình.

### Tại sao quan trọng

Không có Pure Fabrication bạn sẽ overload domain class (Order tự persist, tự mail, tự print) — vi phạm rõ SRP và Cohesion.

---

## 8. Indirection

> *"Gán một trách nhiệm cho **object trung gian** để mediate giữa các component khác."*

Đây là meta-principle đằng sau:

* **Adapter pattern**
* **Mediator pattern**
* **Facade pattern**
* **Proxy / Decorator**
* **Service bus / message broker**

### Ví dụ

```csharp
// Trực tiếp: ApplicationLayer nói chuyện với MailKit -> coupling chặt với library
// Indirection: ApplicationLayer nói chuyện với IEmailSender ; MailKitEmailSender implement nó

public interface IEmailSender { Task SendAsync(string to, string subject, string body); }

public class MailKitEmailSender : IEmailSender { /* biết MailKit */ }
public class SendGridEmailSender : IEmailSender { /* biết SendGrid */ }
```

Swap provider thành thay đổi configuration, không phải refactor.

### Cẩn thận

Indirection có chi phí: nhiều file, nhiều vocabulary. Chỉ áp dụng khi **decoupling đáng giá**.

---

## 9. Protected Variations

> *"Xác định điểm **variation hoặc bất ổn dự đoán**; gán trách nhiệm để tạo **interface ổn định** quanh chúng."*

Bọc biến động đằng sau contract. Bên trong tường, mọi thứ có thể thay đổi; bên ngoài, caller được bảo vệ.

### Ví dụ

* `IPaymentGateway` che codebase khỏi thay đổi "Stripe", "Adyen", "PayPal".
* `IClock` che test khỏi `DateTime.Now`.
* `IFileStorage` che code khỏi "local filesystem vs S3".

### Quan hệ với nguyên lý khác

Protected Variations là **mục tiêu**. Indirection, Polymorphism, OCP, và DIP là **phương tiện** để đạt nó.

---

## 10. Tóm tắt

| #   | Nguyên lý                  | Câu hỏi nó trả lời                                               |
| --- | -------------------------- | ---------------------------------------------------------------- |
| 1   | **Information Expert**     | *Ai có data để làm việc này?*                                    |
| 2   | **Creator**                | *Ai nên instantiate cái này?*                                    |
| 3   | **Controller**             | *Ai xử lý system event này?*                                     |
| 4   | **Low Coupling**           | *Có thể giảm dependency giữa class không?*                       |
| 5   | **High Cohesion**          | *Mọi thứ trong class này có thuộc về nhau không?*                |
| 6   | **Polymorphism**           | *Xử lý variation theo type không có `switch` ra sao?*             |
| 7   | **Pure Fabrication**       | *Không class domain nào fit — có nên phát minh một cái?*         |
| 8   | **Indirection**            | *Có nên đặt người trung gian giữa hai bên này không?*             |
| 9   | **Protected Variations**   | *Cái gì khả năng thay đổi, và che caller khỏi nó ra sao?*         |

---

> **Mental model:**
> *SOLID mô tả **class** tốt. GRASP quyết định **class nào** mỗi trách nhiệm thuộc về.* Dùng chúng cùng nhau — chúng trả lời câu hỏi khác nhau.
