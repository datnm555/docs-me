# Khái niệm cốt lõi OOP — Construct Ngôn ngữ

> Building block cấp ngôn ngữ thực thi bốn trụ cột. Ví dụ trong **C#**; điểm tương đương ở Java/TypeScript được ghi chú khi khác biệt đáng kể.

> 🇻🇳 Phiên bản tiếng Việt. English: [`concepts.md`](./concepts.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Construct cấp ngôn ngữ implement bốn trụ cột: class & object, field/property/method, constructor, `this`/`base`/`static`, abstract class vs. interface, `virtual`/`override`/`sealed`, value vs. reference type, record, generic, equality.
- **Tại sao** — Trụ cột trừu tượng; construct là cách bạn thực sự viết chúng. Biết keyword nào express intent nào là khác biệt giữa code survive refactor và code không.
- **Khi nào** — Quyết định coding hằng ngày: cái này nên là record hay class? Khi nào `override` vs. `new`? Có nên `sealed` cái này? Tại sao struct này copy?
- **Ở đâu** — Mọi file source C# (và Java/Kotlin/TypeScript). Pair với `pillars-vi.md` (cái *tại sao*) và `principles/` (rule judge bạn dùng chúng đúng không).

---

## Mục lục

1. [Class & Object](#1-class--object)
2. [Field, Property, Method](#2-field-property-method)
3. [Constructor](#3-constructor)
4. [`this`, `base`, `static`](#4-this-base-static)
5. [Abstract Class vs Interface](#5-abstract-class-vs-interface)
6. [`virtual`, `override`, `sealed`](#6-virtual-override-sealed)
7. [Value vs Reference Type](#7-value-vs-reference-type)
8. [Record / Data Class](#8-record--data-class)
9. [Generics](#9-generics)
10. [Nested & Partial Class](#10-nested--partial-class)
11. [Object Lifecycle & Equality](#11-object-lifecycle--equality)

---

## 1. Class & Object

**Class** là blueprint; **object** là instance của blueprint đó.

```csharp
public class Customer
{
    public string Name { get; }
    public Customer(string name) => Name = name;
}

var alice = new Customer("Alice");   // alice là object; Customer là class
```

| Class                            | Object                                  |
| -------------------------------- | --------------------------------------- |
| Definition / template            | Concrete instance                       |
| Khai báo một lần                 | Tạo nhiều lần                           |
| Sống trong source code           | Sống trong memory runtime               |
| Không có state                   | Có state (giá trị field)                |

---

## 2. Field, Property, Method

| Member       | Mục đích                              | Ví dụ                                     |
| ------------ | ------------------------------------ | ----------------------------------------- |
| **Field**    | State thô, thường `private`          | `private int _age;`                       |
| **Property** | Wrapper get/set quanh state          | `public int Age { get; set; }`            |
| **Method**   | Behavior — function trên object      | `public void Birthday() => _age++;`       |

### 2.1 Auto-Properties

```csharp
public class Person
{
    public string Name { get; init; }       // chỉ set được trong constructor / object init
    public int    Age  { get; private set; } // mutable, nhưng chỉ bởi class
}
```

### 2.2 Computed Property

```csharp
public class Rectangle(double w, double h)
{
    public double Width  => w;
    public double Height => h;
    public double Area   => w * h;          // tính lại mỗi lần, không có backing field
}
```

---

## 3. Constructor

Constructor build object và **enforce invariant** lúc tạo.

```csharp
public class Order
{
    public Guid Id { get; }
    public DateTime CreatedAt { get; }
    public Customer Customer { get; }

    public Order(Customer customer)
    {
        ArgumentNullException.ThrowIfNull(customer);
        Id        = Guid.NewGuid();
        CreatedAt = DateTime.UtcNow;
        Customer  = customer;
    }
}
```

### 3.1 Primary Constructor (C# 12+)

```csharp
public class Order(Customer customer)
{
    public Guid Id { get; } = Guid.NewGuid();
    public Customer Customer { get; } = customer;
}
```

### 3.2 Overload & Chain Constructor

```csharp
public class Money
{
    public decimal Amount   { get; }
    public string  Currency { get; }

    public Money(decimal amount) : this(amount, "USD") { }
    public Money(decimal amount, string currency)
    {
        Amount   = amount;
        Currency = currency;
    }
}
```

---

## 4. `this`, `base`, `static`

| Keyword     | Nghĩa                                                  |
| ----------- | ------------------------------------------------------ |
| `this`      | Reference tới instance hiện tại                        |
| `base`      | Reference tới base class (parent)                      |
| `static`    | Thuộc về **type**, không phải instance                 |

```csharp
public class Counter
{
    public static int InstancesCreated;   // share giữa mọi instance
    public int Count;                     // per-instance

    public Counter() => InstancesCreated++;
}

new Counter();
new Counter();
Console.WriteLine(Counter.InstancesCreated); // 2
```

### Static Member — Khi nào dùng

* Utility thuần (`Math.Sqrt`).
* Factory method (`DateTime.UtcNow`).
* Constant (`Math.PI`).

### Static — Khi *KHÔNG* dùng

* Bất cứ gì stateful và shared — giết testability và concurrency.
* Bất cứ gì phụ thuộc configuration — làm cho injectable.

---

## 5. Abstract Class vs Interface

```csharp
public abstract class Vehicle
{
    public string Make { get; }            // state share
    protected Vehicle(string make) => Make = make;

    public abstract void Start();          // mỗi child phải implement
    public virtual  void Honk() => Console.WriteLine("Beep");  // override tuỳ chọn
}

public interface ITradable
{
    decimal Price { get; }
    void Sell(Customer to);
}

public class Car(string make, decimal price) : Vehicle(make), ITradable
{
    public decimal Price => price;
    public override void Start() => Console.WriteLine("Vroom");
    public void Sell(Customer to) { /* ... */ }
}
```

| Feature                  | Abstract Class           | Interface                             |
| ------------------------ | ------------------------ | ------------------------------------- |
| Multiple inheritance     | ❌ (một base)             | ✅ (nhiều)                             |
| Constructor              | ✅                        | ❌                                     |
| Stored state (field)     | ✅                        | ❌ (chỉ property)                      |
| Default implementation   | ✅                        | ✅ (từ C# 8)                           |
| Dùng cho…                | Behavior *partial* share | *Contract* / *capability* share       |

---

## 6. `virtual`, `override`, `sealed`

```csharp
public class Shape
{
    public virtual double Area() => 0;
}

public class Circle : Shape
{
    private readonly double _r;
    public Circle(double r) => _r = r;
    public override double Area() => Math.PI * _r * _r;
}

public sealed class FixedCircle : Circle    // không cho inheritance tiếp
{
    public FixedCircle(double r) : base(r) { }
}
```

* `virtual` — *có thể* được override.
* `override` — thay implementation base.
* `sealed` — khoá cửa; không cho override / inheritance tiếp.
* `new` (method) — *ẩn* method base (hiếm khi đúng ý; thường là smell).

> Mặc định `sealed` cho concrete class. Chỉ mở cái bạn có ý cho phép.

---

## 7. Value vs Reference Type

| Khía cạnh          | Value type (`struct`)      | Reference type (`class`)          |
| ------------------ | -------------------------- | --------------------------------- |
| Sống ở…            | Stack (thường)             | Heap                              |
| Copy semantics     | By value                   | By reference                      |
| Equality default   | Member-wise                | Reference identity                |
| Nullable?          | Chỉ qua `T?`               | Có                                |
| Inheritance        | ❌                          | ✅                                 |

```csharp
struct Point { public int X, Y; }       // value
class  Node  { public int Data; }       // reference

Point a = new() { X = 1 };
Point b = a; b.X = 5;
Console.WriteLine(a.X);                 // 1   — a đã được copy

Node n = new() { Data = 1 };
Node m = n; m.Data = 5;
Console.WriteLine(n.Data);              // 5   — cả hai trỏ tới cùng object
```

Dùng `struct` cho data nhỏ, immutable, dạng value (`Point`, `Color`, `Money`). Mọi cái khác → `class`.

---

## 8. Record / Data Class

Record là cách compact để khai báo type **immutable, value-equality**:

```csharp
public record Money(decimal Amount, string Currency);

var a = new Money(10, "USD");
var b = new Money(10, "USD");
Console.WriteLine(a == b);              // True — structural equality

var c = a with { Amount = 20 };         // copy không phá huỷ
```

Record loại bỏ boilerplate cho **DTO, value object, và read model**. Vẫn là reference type trừ khi khai báo `record struct`.

Tương đương Java: `record` (từ Java 14). Tương đương TypeScript: interface `readonly` hoặc object frozen.

---

## 9. Generics

Generics cho phép class và method **được parameterize bởi type** — viết một lần, work với type tương thích nào cũng được.

```csharp
public class Repository<TEntity> where TEntity : class, IEntity
{
    private readonly List<TEntity> _items = new();
    public void Add(TEntity item) => _items.Add(item);
    public TEntity? FindById(Guid id) => _items.FirstOrDefault(i => i.Id == id);
}

var customers = new Repository<Customer>();
var orders    = new Repository<Order>();
```

### Tại sao Generics quan trọng

* **Type safety** — không cast, không box.
* **Performance** — value type không bị box.
* **Code reuse** — một implementation, nhiều type.

---

## 10. Nested & Partial Class

```csharp
public class Outer
{
    public class Inner { /* gắn chặt với trách nhiệm của Outer */ }
}

// Trải qua nhiều file (vd. code generated + viết tay)
public partial class Customer { public string Name { get; set; } }
public partial class Customer { public void Greet() => ...; }
```

Nested class hữu ích cho **type chỉ có ý nghĩa bên trong type khác** (vd. `Order.Builder`). Partial class chủ yếu cho **code generation** — designer, source generator, EF migration.

---

## 11. Object Lifecycle & Equality

### 11.1 Lifecycle (CLR)

```
new Foo() ─►  constructor chạy ─► object trên heap ─► reachable
                                                       │
                            references out of scope │
                                                       ▼
                                              GC eligible ─► finalizer (nếu có) ─► collected
```

### 11.2 Equality

| Loại equality     | Công cụ C#                                     | Ghi chú                                   |
| ----------------- | ---------------------------------------------- | ----------------------------------------- |
| Reference         | `ReferenceEquals(a, b)`                        | Cùng object?                              |
| Default value     | `a == b` cho `struct`                          | Member-wise                               |
| Structural        | `record` / override `Equals` + `GetHashCode`   | Hai object khác nhau cùng data            |
| Domain identity   | So sánh theo `Id`                              | Entity — cùng người qua thời gian         |

> **Rule:** nếu override `Equals`, phải override `GetHashCode`. Chúng phải đồng nhất.

```csharp
public class Customer
{
    public Guid Id { get; }

    public override bool Equals(object? obj)
        => obj is Customer c && c.Id == Id;

    public override int GetHashCode() => Id.GetHashCode();
}
```

---

> **Bước tiếp:** ghép tất cả lại trong [`real-project-vi.md`](./real-project-vi.md), rồi xem [`key-points-vi.md`](./key-points-vi.md) cho cheat sheet.
