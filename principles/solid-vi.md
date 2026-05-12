# SOLID — Năm Nguyên Lý Hướng Đối Tượng

> Phổ biến bởi **Robert C. Martin** ("Uncle Bob") khoảng 2000, SOLID là acronym của 5 nguyên lý ảnh hưởng nhất trong thiết kế hướng đối tượng. Master chúng và bạn sẽ viết code **dễ thay đổi, dễ test, và dễ hiểu**.

> 🇻🇳 Phiên bản tiếng Việt. English: [`solid.md`](./solid.md)

---

## Mục lục

1. [S — Single Responsibility Principle (SRP)](#1-s--single-responsibility-principle-srp)
2. [O — Open/Closed Principle (OCP)](#2-o--openclosed-principle-ocp)
3. [L — Liskov Substitution Principle (LSP)](#3-l--liskov-substitution-principle-lsp)
4. [I — Interface Segregation Principle (ISP)](#4-i--interface-segregation-principle-isp)
5. [D — Dependency Inversion Principle (DIP)](#5-d--dependency-inversion-principle-dip)
6. [Bảng tóm tắt](#6-bảng-tóm-tắt)

---

## 1. S — Single Responsibility Principle (SRP)

> *"Class chỉ nên có **một lý do để thay đổi**."* — Robert C. Martin

"Lý do để thay đổi" là **stakeholder** hoặc **trục thay đổi**, không phải số method.

### 1.1 Tệ — Nhiều lý do để thay đổi

```csharp
public class Report
{
    public string Title { get; set; }
    public List<Row> Rows { get; set; }

    public decimal CalculateTotal()           { /* business rule */ }
    public string  RenderHtml()                { /* presentation */ }
    public void    SaveToDatabase()            { /* persistence  */ }
    public void    EmailTo(string address)     { /* delivery     */ }
}
```

Thay đổi vì: luật thuế (business), redesign thương hiệu (HTML), đổi DB, đổi SMTP provider — bốn lực trực giao, một class.

### 1.2 Tốt — Mỗi cái một lý do

```csharp
public record Report(string Title, IReadOnlyList<Row> Rows);

public class ReportCalculator   { public decimal Total(Report r) { ... } }
public class ReportRenderer     { public string  Html(Report r)  { ... } }
public class ReportRepository   { public Task SaveAsync(Report r) { ... } }
public class ReportMailer       { public Task SendAsync(Report r, string to) { ... } }
```

### 1.3 Heuristic

> *"Nếu bạn có thể viết một câu mô tả class dùng **và**, có lẽ nó vi phạm SRP."*

---

## 2. O — Open/Closed Principle (OCP)

> *"Software entity nên **mở cho extension** nhưng **đóng cho modification**."*
> — Bertrand Meyer (1988), refine bởi Robert C. Martin

Bạn nên có thể thêm behavior mới **mà không sửa code đã tested**.

### 2.1 Tệ — Switch tăng theo mỗi type mới

```csharp
public class Discount
{
    public decimal For(Customer c)
    {
        return c.Type switch
        {
            "Regular" => 0m,
            "Premium" => 0.10m,
            "Vip"     => 0.20m,
            _         => 0m
        };
    }
}
```

Mỗi loại customer mới phải sửa method này và re-test.

### 2.2 Tốt — Mở qua Polymorphism

```csharp
public interface ICustomer { decimal DiscountRate(); }

public class Regular : ICustomer { public decimal DiscountRate() => 0m;    }
public class Premium : ICustomer { public decimal DiscountRate() => 0.10m; }
public class Vip     : ICustomer { public decimal DiscountRate() => 0.20m; }

public class Discount
{
    public decimal For(ICustomer c) => c.DiscountRate();
}
```

Thêm `Platinum` chỉ cần **thêm một class** — `Discount` không bị sửa.

### 2.3 Công cụ phổ biến để đạt OCP

* **Polymorphism / inheritance**
* **Strategy pattern**
* **Plugin architecture**
* **Decorator pattern**
* **Configuration over conditionals**

### 2.4 Cạm bẫy

Đừng *trước-thời-hạn* làm mọi thứ extensible. Kết hợp với **YAGNI** — chờ variant thứ hai.

---

## 3. L — Liskov Substitution Principle (LSP)

> *"Subtype phải **có thể substitute** cho base type mà không thay đổi tính đúng của chương trình."*
> — Barbara Liskov, 1987

Nếu `S` extend `T`, mọi chỗ dùng `T` phải tiếp tục hoạt động khi đưa `S` — cùng contract behavior, cùng invariant, cùng exception.

### 3.1 Tệ — Rectangle / Square

```csharp
public class Rectangle
{
    public virtual int Width  { get; set; }
    public virtual int Height { get; set; }
}

public class Square : Rectangle
{
    public override int Width  { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}

void Test(Rectangle r)
{
    r.Width  = 5;
    r.Height = 4;
    Debug.Assert(r.Width * r.Height == 20); // FAIL với Square — area là 16
}
```

`Square` về mặt toán học là một Rectangle, nhưng **về behavior** không substitute được.

### 3.2 Tốt — Đừng ép "Is-A"

Model `Square` và `Rectangle` độc lập (hoặc share interface immutable `IShape` trả về `Area()`).

### 3.3 LSP Rule trong thực tế

Một subclass phải:

* **Không tăng precondition** (không yêu cầu nhiều hơn parent).
* **Không yếu postcondition** (không trả ít hơn).
* **Giữ invariant** của parent.
* **Không throw exception type mới** mà parent không có.

### 3.4 Smell thực tế

```csharp
public class ReadOnlyList<T> : List<T>
{
    public new void Add(T item) => throw new NotSupportedException();
}
```

Ẩn `NotSupportedException` đằng sau signature base class — vi phạm LSP textbook. Dùng interface nhỏ hơn (`IReadOnlyList<T>`) thay vào.

---

## 4. I — Interface Segregation Principle (ISP)

> *"Client không nên bị ép phụ thuộc vào interface chúng không dùng."*
> — Robert C. Martin

Interface "béo" thành nam châm coupling. Ưu tiên **nhiều interface nhỏ, theo role**.

### 4.1 Tệ — Interface béo

```csharp
public interface IMultiFunctionDevice
{
    void Print(Document d);
    void Scan(Document d);
    void Fax(Document d);
    void Staple(Document d);
}

public class HomePrinter : IMultiFunctionDevice
{
    public void Print(Document d) { /* ok */ }
    public void Scan(Document d)  => throw new NotSupportedException();
    public void Fax(Document d)   => throw new NotSupportedException();
    public void Staple(Document d)=> throw new NotSupportedException();
}
```

### 4.2 Tốt — Interface tách rời

```csharp
public interface IPrinter { void Print(Document d); }
public interface IScanner { void Scan(Document d);  }
public interface IFax     { void Fax(Document d);   }
public interface IStapler { void Staple(Document d);}

public class HomePrinter   : IPrinter { ... }
public class OfficeMfp     : IPrinter, IScanner, IFax, IStapler { ... }
```

Client chỉ phụ thuộc cái chúng thực sự dùng.

### 4.3 Heuristic

> *"Interface nên **dựa trên role**, không **dựa trên technology**."*

Một `IUserRepository` với 14 method thường ẩn 3 role riêng (`IUserReader`, `IUserWriter`, `IUserAuthenticator`).

---

## 5. D — Dependency Inversion Principle (DIP)

> *"Module high-level không nên phụ thuộc module low-level. **Cả hai nên phụ thuộc abstraction**. Abstraction không nên phụ thuộc chi tiết. Chi tiết phụ thuộc abstraction."*
> — Robert C. Martin

DIP là nguyên lý đằng sau **dependency injection**, **clean architecture**, và **hexagonal/ports-and-adapters**.

### 5.1 Tệ — Phụ thuộc Concrete

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repo = new();   // coupling concrete
    private readonly SmtpEmailSender    _mail = new();   // coupling concrete

    public void Place(Order o) { _repo.Save(o); _mail.Send(o.Email, "Thanks"); }
}
```

Bạn không thể:

* Swap SQL sang Mongo mà không sửa class này.
* Unit-test mà không có SMTP server thật.

### 5.2 Tốt — Phụ thuộc Abstraction

```csharp
public interface IOrderRepository { void Save(Order o); }
public interface IEmailSender     { void Send(string to, string body); }

public class OrderService(IOrderRepository repo, IEmailSender mail)
{
    public void Place(Order o) { repo.Save(o); mail.Send(o.Email, "Thanks"); }
}

// Composition root
var service = new OrderService(new SqlOrderRepository(), new SmtpEmailSender());
```

### 5.3 Hai "đảo ngược"

1. **Đảo ngược source-code dependency** — code high-level không còn import chi tiết low-level.
2. **Đảo ngược control** — framework hoặc composition root *cho* dependency thay vì class *lấy* chúng.

### 5.4 Abstraction sống ở đâu

> *"Abstraction thuộc về layer **dùng** nó, không phải layer **implement** nó."*

`IOrderRepository` sống cạnh `OrderService`, không cạnh `SqlOrderRepository`.

---

## 6. Bảng tóm tắt

| Chữ    | Nguyên lý               | Smell nó ngăn chặn             | Công cụ nó cho phép              |
| ------ | ----------------------- | ------------------------------ | -------------------------------- |
| **S**  | Single Responsibility   | God class, shotgun surgery     | Module cohesive, testable        |
| **O**  | Open/Closed             | Sửa lan qua code               | Strategy, plugin, extension      |
| **L**  | Liskov Substitution     | Behavior bất ngờ trong subclass| Polymorphism an toàn             |
| **I**  | Interface Segregation   | Interface phình, coupling      | Abstraction theo role            |
| **D**  | Dependency Inversion    | Coupling concrete, untestable  | DI, clean architecture, mocking  |

---

> **Mnemonic:** *"**S**ingle, **O**pen, **L**iskov, **I**nterface, **D**ependency."*
> Áp dụng SOLID phục vụ rõ ràng — không bao giờ như bureaucracy. Nếu một nguyên lý làm phức tạp code hôm nay mà không trả công, bạn có thể chưa cần *vẫn*.
