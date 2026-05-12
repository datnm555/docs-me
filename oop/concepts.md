# OOP Core Concepts — Language Constructs

> The language-level building blocks that implement the four pillars. Examples in **C#**; equivalents in Java/TypeScript noted where they differ meaningfully.

---

## Table of Contents

1. [Class & Object](#1-class--object)
2. [Field, Property, Method](#2-field-property-method)
3. [Constructor](#3-constructor)
4. [`this`, `base`, `static`](#4-this-base-static)
5. [Abstract Class vs Interface](#5-abstract-class-vs-interface)
6. [`virtual`, `override`, `sealed`](#6-virtual-override-sealed)
7. [Value vs Reference Types](#7-value-vs-reference-types)
8. [Record / Data Class](#8-record--data-class)
9. [Generics](#9-generics)
10. [Nested & Partial Classes](#10-nested--partial-classes)
11. [Object Lifecycle & Equality](#11-object-lifecycle--equality)

---

## 1. Class & Object

A **class** is a blueprint; an **object** is an instance of that blueprint.

```csharp
public class Customer
{
    public string Name { get; }
    public Customer(string name) => Name = name;
}

var alice = new Customer("Alice");   // alice is an object; Customer is the class
```

| Class                            | Object                                  |
| -------------------------------- | --------------------------------------- |
| Definition / template            | Concrete instance                       |
| Declared once                    | Created many times                      |
| Lives in source code             | Lives in memory at runtime              |
| No state                         | Has state (field values)                |

---

## 2. Field, Property, Method

| Member       | Purpose                              | Example                                   |
| ------------ | ------------------------------------ | ----------------------------------------- |
| **Field**    | Raw state, usually `private`         | `private int _age;`                       |
| **Property** | Get/set wrapper around state         | `public int Age { get; set; }`            |
| **Method**   | Behavior — a function on the object  | `public void Birthday() => _age++;`       |

### 2.1 Auto-Properties

```csharp
public class Person
{
    public string Name { get; init; }       // settable only in constructor / object init
    public int    Age  { get; private set; } // mutable, but only by the class itself
}
```

### 2.2 Computed Properties

```csharp
public class Rectangle(double w, double h)
{
    public double Width  => w;
    public double Height => h;
    public double Area   => w * h;          // computed each time, no backing field
}
```

---

## 3. Constructor

A constructor builds an object and **enforces invariants** at creation time.

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

### 3.1 Primary Constructors (C# 12+)

```csharp
public class Order(Customer customer)
{
    public Guid Id { get; } = Guid.NewGuid();
    public Customer Customer { get; } = customer;
}
```

### 3.2 Constructor Overloading & Chaining

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

| Keyword     | Meaning                                                |
| ----------- | ------------------------------------------------------ |
| `this`      | Reference to the current instance                      |
| `base`      | Reference to the base class (parent)                   |
| `static`    | Belongs to the **type**, not to instances              |

```csharp
public class Counter
{
    public static int InstancesCreated;   // shared across all instances
    public int Count;                     // per-instance

    public Counter() => InstancesCreated++;
}

new Counter();
new Counter();
Console.WriteLine(Counter.InstancesCreated); // 2
```

### Static Members — When to Use

* Pure utilities (`Math.Sqrt`).
* Factory methods (`DateTime.UtcNow`).
* Constants (`Math.PI`).

### Static — When *Not* to Use

* Anything stateful and shared — kills testability and concurrency.
* Anything that depends on configuration — make it injectable.

---

## 5. Abstract Class vs Interface

```csharp
public abstract class Vehicle
{
    public string Make { get; }            // shared state
    protected Vehicle(string make) => Make = make;

    public abstract void Start();          // each child must implement
    public virtual  void Honk() => Console.WriteLine("Beep");  // optional override
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
| Multiple inheritance     | ❌ (one base)             | ✅ (many)                              |
| Constructor              | ✅                        | ❌                                     |
| Stored state (fields)    | ✅                        | ❌ (only properties)                   |
| Default implementations  | ✅                        | ✅ (since C# 8)                        |
| Use for…                 | Shared *partial* behavior| Shared *contract* / *capability*      |

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

public sealed class FixedCircle : Circle    // no further inheritance allowed
{
    public FixedCircle(double r) : base(r) { }
}
```

* `virtual` — *can* be overridden.
* `override` — replaces the base implementation.
* `sealed` — locks the door; no further overrides / inheritance.
* `new` (method) — *hides* the base method (rarely what you want; usually a smell).

> Prefer `sealed` by default on concrete classes. Open only what you intend to allow.

---

## 7. Value vs Reference Types

| Aspect             | Value types (`struct`)     | Reference types (`class`)         |
| ------------------ | -------------------------- | --------------------------------- |
| Lives on…          | Stack (typically)          | Heap                              |
| Copy semantics     | By value                   | By reference                      |
| Equality default   | Member-wise                | Reference identity                |
| Nullable?          | Only via `T?`              | Yes                               |
| Inheritance        | ❌                          | ✅                                 |

```csharp
struct Point { public int X, Y; }       // value
class  Node  { public int Data; }       // reference

Point a = new() { X = 1 };
Point b = a; b.X = 5;
Console.WriteLine(a.X);                 // 1   — a was copied

Node n = new() { Data = 1 };
Node m = n; m.Data = 5;
Console.WriteLine(n.Data);              // 5   — both point to the same object
```

Use `struct` for small, immutable, value-like data (`Point`, `Color`, `Money`). Everything else → `class`.

---

## 8. Record / Data Class

Records are a compact way to declare **immutable, value-equality** types:

```csharp
public record Money(decimal Amount, string Currency);

var a = new Money(10, "USD");
var b = new Money(10, "USD");
Console.WriteLine(a == b);              // True — structural equality

var c = a with { Amount = 20 };         // non-destructive copy
```

Records eliminate boilerplate for **DTOs, value objects, and read models**. They are still reference types unless declared `record struct`.

Java's equivalent: `record` (since Java 14). TypeScript's equivalent: a `readonly` interface or a frozen object.

---

## 9. Generics

Generics enable **type-parameterized** classes and methods — write once, work with any compatible type.

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

### Why Generics Matter

* **Type safety** — no casting, no boxing.
* **Performance** — value types stay unboxed.
* **Code reuse** — one implementation, many types.

---

## 10. Nested & Partial Classes

```csharp
public class Outer
{
    public class Inner { /* tightly bound to Outer's responsibilities */ }
}

// Spread across multiple files (e.g. generated + hand-written code)
public partial class Customer { public string Name { get; set; } }
public partial class Customer { public void Greet() => ...; }
```

Nested classes are useful for **types that only make sense inside another type** (e.g. an `Order.Builder`). Partial classes are mostly for **code generation** — designers, source generators, EF migrations.

---

## 11. Object Lifecycle & Equality

### 11.1 Lifecycle (CLR)

```
new Foo() ─►  constructor runs ─► object on heap ─► reachable
                                                       │
                            references go out of scope │
                                                       ▼
                                              GC eligible ─► finalizer (if any) ─► collected
```

### 11.2 Equality

| Equality kind     | C# tool                                        | Notes                                     |
| ----------------- | ---------------------------------------------- | ----------------------------------------- |
| Reference         | `ReferenceEquals(a, b)`                        | Same object?                              |
| Default value     | `a == b` for `struct`                          | Member-wise                               |
| Structural        | `record` / override `Equals` + `GetHashCode`   | Two distinct objects with same data       |
| Domain identity   | Compare by `Id`                                | Entities — same person across time        |

> **Rule:** if you override `Equals`, override `GetHashCode`. They must agree.

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

> **Next:** put it all together in [`real-project.md`](./real-project.md), then check [`key-points.md`](./key-points.md) for the cheat sheet.
