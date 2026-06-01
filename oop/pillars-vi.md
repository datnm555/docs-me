# Bốn Trụ Cột của OOP

> **Encapsulation, Abstraction, Inheritance, Polymorphism.** Nắm vững chúng và bạn hiểu *tại sao* đằng sau mọi quyết định thiết kế OO.

> 🇻🇳 Phiên bản tiếng Việt. English: [`pillars.md`](./pillars.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Bốn trụ cột: **Encapsulation** (ẩn state đằng sau behavior), **Abstraction** (expose cái gì, ẩn cách), **Inheritance** (specialize type có sẵn), **Polymorphism** (một interface, nhiều behavior runtime).
- **Tại sao** — Bốn idea này cùng nhau giải thích *mọi* OO design decision. Master chúng và pattern như Strategy, Template Method, Decorator thành tự nhiên thay vì memorize.
- **Khi nào** — Trước khi học design pattern, SOLID, hoặc DDD; giải thích tại sao `if (x is Y)` là smell; quyết định giữa subclass và composition.
- **Ở đâu** — Fundamental ngôn ngữ (keyword `class`, `interface`, `virtual` của ngôn ngữ là cách mỗi trụ cột show up). Foundation cho mọi thứ trong `principles/`, `design-pattern/`, và `ddd/`.

---

## Mục lục

1. [Encapsulation](#1-encapsulation)
2. [Abstraction](#2-abstraction)
3. [Inheritance](#3-inheritance)
4. [Polymorphism](#4-polymorphism)
5. [Các trụ cột liên hệ ra sao](#5-các-trụ-cột-liên-hệ-ra-sao)

---

## 1. Encapsulation

> *"Gom data và operation trên data đó vào một đơn vị duy nhất. Ẩn internal đằng sau một public interface."*

Encapsulation là **trụ cột chịu lực**. Ba cái còn lại đều dựa vào nó.

### 1.1 Trong code nghĩa là

* **Field là `private`**, không bao giờ chạm từ bên ngoài.
* **Method enforce invariant** (rule luôn phải đúng).
* **Public surface = behavior**, không phải data.

### 1.2 Tệ — Không có Encapsulation (Anemic Object)

```csharp
public class BankAccount
{
    public decimal Balance;          // public field thô
}

// Ở bất kỳ đâu trong codebase:
account.Balance -= 100;              // không validate, không audit, không concurrency
```

Balance âm có thể xuất hiện. Rule "không overdraft" không sống ở đâu — nên nó không sống ở đâu một cách đáng tin cậy.

### 1.3 Tốt — Behavior sở hữu data

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        if (amount > Balance)
            throw new InvalidOperationException("Insufficient funds");

        Balance -= amount;
    }
}
```

Bây giờ rule được enforce ở **một chỗ** — method `Withdraw`.

### 1.4 Các cấp Access Control

| Modifier (C#)           | Visible với                                                 |
| ----------------------- | ----------------------------------------------------------- |
| `private`               | Chỉ cùng class                                              |
| `protected`             | Cùng class + subclass                                       |
| `internal`              | Cùng assembly (≈ package)                                   |
| `protected internal`    | Cùng assembly **hoặc** subclass                             |
| `private protected`     | Cùng assembly **và** subclass                               |
| `public`                | Mọi người                                                   |

> **Mặc định dùng modifier *chặt nhất***. Chỉ mở khi có lý do.

### 1.5 Encapsulation trong C# hiện đại

```csharp
public class Order
{
    private readonly List<LineItem> _items = new();

    public IReadOnlyList<LineItem> Items => _items;   // expose view read-only
    public decimal Total => _items.Sum(i => i.Price * i.Qty);

    public void Add(LineItem item)
    {
        ArgumentNullException.ThrowIfNull(item);
        _items.Add(item);
    }
}
```

List `_items` thuộc sở hữu hoàn toàn. Caller có thể đọc nhưng không thể append, clear, hay sort.

---

## 2. Abstraction

> *"Expose **cái gì** object làm. Ẩn **cách** nó làm."*

Abstraction hoạt động một cấp trên encapsulation: nó hỏi *operation nào* đáng expose.

### 2.1 Encapsulation vs Abstraction

| Khái niệm         | Câu hỏi nó trả lời                                 |
| ----------------- | -------------------------------------------------- |
| **Encapsulation** | *Làm sao ẩn và bảo vệ data này?*                   |
| **Abstraction**   | *Operation nào thậm chí đáng expose?*              |

Encapsulation là cơ chế; abstraction là **quyết định thiết kế**.

### 2.2 Ví dụ — Hai cấp Abstraction

```csharp
// Low-level (cụ thể)
public class SmtpClient
{
    public void Connect(string host, int port);
    public void Authenticate(string user, string pass);
    public void Send(MailMessage msg);
    public void Disconnect();
}

// Abstraction high-level mà application thật sự muốn
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body);
}
```

Application không quan tâm port, TLS, hay auth — chỉ muốn *gửi email*.

### 2.3 Tìm Abstraction đúng

* Đặt tên theo **vai trò**, không phải technology — `IEmailSender`, không phải `ISmtpClient`.
* Mô tả **ý định**, không phải cơ chế — `IClock`, không phải `IDateTimeWrapper`.
* Kháng cự abstraction *đầu cơ*; chờ đến khi có **hai implementation thật** (hoặc một thật + một fake test).

### 2.4 Abstract Class vs Interface

| Cần                                        | Dùng             |
| ------------------------------------------ | ---------------- |
| Pure contract, không có shared state       | `interface`      |
| Shared state HOẶC shared default behavior  | `abstract class` |
| "Inheritance" nhiều khả năng               | Nhiều interface  |
| Marker cho một category                    | `interface`      |

> C# hiện đại cho phép **default interface method** — nhưng nếu cần chúng, bạn có lẽ muốn một abstract base class.

---

## 3. Inheritance

> *"Định nghĩa class mới như một **specialization** của class có sẵn — thừa kế member và refine hoặc thêm behavior."*

Inheritance là trụ cột **bị lạm dụng nhiều nhất**. Dùng tốt, nó thể hiện **is-a**. Dùng tệ, nó thành mớ hỗn độn.

### 3.1 Cấu trúc của Inheritance

```csharp
public class Animal
{
    public string Name { get; set; }
    public virtual void Speak() => Console.WriteLine("...");
}

public class Dog : Animal
{
    public override void Speak() => Console.WriteLine("Woof!");
}

public class Cat : Animal
{
    public override void Speak() => Console.WriteLine("Meow!");
}
```

* `virtual` — method base có thể được override.
* `override` — method derived thay thế base.
* `sealed` — ngăn inheritance/override tiếp theo.

### 3.2 Test "is-a"

Nếu bạn không thể nói thành thật *"`Dog` **là một** `Animal`"*, đừng inherit. Test fail phổ biến:

```csharp
public class Stack<T> : List<T> { ... }   // TỆ: Stack KHÔNG phải List
```

Stack không cho phép mutation random-access; inherit từ `List<T>` expose operation vi phạm invariant của stack. Dùng **composition** thay vào.

### 3.3 Cạm bẫy Inheritance

* **Fragile base class** — thay đổi nhỏ ở base có thể vỡ mọi child.
* **Yo-yo problem** — để hiểu một class bạn phải nhảy lên xuống hierarchy.
* **Forced generalization** — feature thêm "phòng khi" được share.
* **Diamond problem** — nhập nhằng đa kế thừa (tránh trong C# / Java bằng interface).

### 3.4 Quy tắc

> *"Ưu tiên **composition over inheritance**. Chỉ dùng inheritance cho quan hệ **is-a ổn định, thực sự**."*
> — Gang of Four

---

## 4. Polymorphism

> *"Cùng operation hành xử khác trên các type khác nhau."*

Polymorphism là thứ làm inheritance và abstraction **hữu ích tại runtime**.

### 4.1 Bốn loại Polymorphism

| Loại            | Cơ chế trong C#                          | Ví dụ                                              |
| --------------- | ---------------------------------------- | -------------------------------------------------- |
| **Subtype**     | `virtual` / `override`                   | `Animal.Speak()` dispatch tới `Dog` hoặc `Cat`     |
| **Parametric**  | Generics `<T>`                           | `List<T>`, `Dictionary<TKey, TValue>`              |
| **Ad-hoc**      | Method overloading, operator overloading | `Print(int)` vs `Print(string)`                    |
| **Coercion**    | Type conversion implicit                 | `int → long`, `string ↔ ReadOnlySpan<char>`        |

Khi dân OOP nói "polymorphism", họ gần như luôn muốn nói **subtype polymorphism**.

### 4.2 Subtype Polymorphism trong action

```csharp
public abstract class Shape { public abstract double Area(); }

public class Circle(double r)    : Shape { public override double Area() => Math.PI * r * r; }
public class Square(double side) : Shape { public override double Area() => side * side; }

double TotalArea(IEnumerable<Shape> shapes) => shapes.Sum(s => s.Area());
```

`TotalArea` không biết — hay quan tâm — *shape nào* nó có. Thêm `Triangle` chỉ cần viết một class. Không `switch`, không `if`.

### 4.3 Tại sao Polymorphism quan trọng

* **Open/Closed Principle** — variant mới thêm vào mà không sửa code cũ.
* **Testability** — thay fake cho dependency thật.
* **Rõ ràng** — thay thang `switch` dài bằng behavior driven by type.

### 4.4 Khi KHÔNG dùng Polymorphism

* **Logic 2 trường hợp không có variant tương lai** — một `if` đơn giản rõ hơn hai subclass.
* **Loop nhạy cảm performance** — virtual call tốn một indirection. (Hiếm khi là vấn đề thật, nhưng có thật trong inner loop chật.)
* **Cross-type behavior** — pattern matching có thể đọc tốt hơn polymorphism cho code path một lần.

---

## 5. Các trụ cột liên hệ ra sao

```
                ┌──────────────┐
                │ ENCAPSULATION│ (ẩn data)
                └──────┬───────┘
                       │ làm khả thi
                       ▼
                ┌──────────────┐
                │ ABSTRACTION  │ (chọn expose gì)
                └──────┬───────┘
                       │ thực thi qua
              ┌────────┴─────────┐
              ▼                  ▼
       ┌────────────┐     ┌────────────────┐
       │INHERITANCE │     │  COMPOSITION   │
       │ (is-a)     │     │  (has-a)       │
       └─────┬──────┘     └────────┬───────┘
             │                     │
             └──────── cho phép ───┘
                        ▼
                 ┌──────────────┐
                 │POLYMORPHISM  │ (behavior khác qua cùng contract)
                 └──────────────┘
```

* **Encapsulation** là đất.
* **Abstraction** là quyết định thiết kế mọc trên đất đó.
* **Inheritance và composition** là cấu trúc xây từ abstraction.
* **Polymorphism** là behavior những cấu trúc đó biểu hiện.

---

> **Bước tiếp:** đọc [`concepts-vi.md`](./concepts-vi.md) cho các construct ngôn ngữ implement những trụ cột này trong C# / Java.
