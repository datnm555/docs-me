# Pattern Data-Access — Repository, UoW, Lazy & Eager Loading

> Các pattern quyết định cách domain nói chuyện với database. Phần lớn từ **PoEAA của Martin Fowler (2002)** và vẫn là vocabulary mọi dev đang làm việc dùng với ORM như **EF Core**, **Hibernate**, **Dapper**, **Mongoose**.

> 🇻🇳 Phiên bản tiếng Việt. English: [`data-access.md`](./data-access.md)

---

## Mục lục

1. [Repository](#1-repository)
2. [Unit of Work](#2-unit-of-work)
3. [Lazy Loading](#3-lazy-loading)
4. [Eager Loading](#4-eager-loading)
5. [Explicit Loading](#5-explicit-loading)
6. [Identity Map](#6-identity-map)
7. [Active Record vs Data Mapper](#7-active-record-vs-data-mapper)
8. [Query Object & Specification](#8-query-object--specification)
9. [Chọn chiến lược](#9-chọn-chiến-lược)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Các pattern data-access từ PoEAA: **Repository**, **Unit of Work**, **Lazy / Eager / Explicit Loading**, **Identity Map**, **Active Record vs. Data Mapper**, **Query Object** và **Specification**.
- **Tại sao** — Cho bạn vocabulary chính xác để nói về cách domain code reach database. Phần lớn bug production trong code data đến từ trộn strategy (lazy-load khi cần eager, generic repo khi cần specific, Active Record khi cần Data Mapper).
- **Khi nào** — Bất cứ lúc nào bạn chọn cách load data liên quan, structure query, hoặc wrap ORM. Lựa chọn thường được làm implicit — làm explicit là bước đầu tới fix N+1 query và Cartesian explosion.
- **Ở đâu** — **Infrastructure** layer (implementation) và boundary **Application/Domain** (interface). EF Core đã implement nhiều cái (Identity Map, Lazy Load qua proxy, Unit of Work qua `DbContext`).

---

## 1. Repository

> *"Repository làm trung gian giữa domain và data-mapping layer, **hành xử như in-memory collection** của domain object."* — Eric Evans, *DDD*

### 1.1 Contract

```csharp
public interface IOrderRepository
{
    Task<Order?> GetAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task<IReadOnlyList<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct);
}
```

* Domain nói chuyện **chỉ** với interface này — không bao giờ với SQL, không bao giờ với `DbContext`.
* Method có **intent** (`FindByCustomerAsync`), không generic (`Where(predicate)`).
* Trả về **domain object**, không phải DTO hoặc `IQueryable`.

### 1.2 Implementation EF Core

```csharp
public class EfOrderRepository(ShopDbContext db) : IOrderRepository
{
    public Task<Order?> GetAsync(Guid id, CancellationToken ct)
        => db.Orders
             .Include(o => o.Items)
             .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct)
    {
        await db.Orders.AddAsync(order, ct);
        // SaveChanges là việc của Unit of Work
    }

    public Task<IReadOnlyList<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct)
        => db.Orders
             .Where(o => o.CustomerId == customerId)
             .ToListAsync(ct)
             .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);
}
```

### 1.3 Tranh luận "Generic Repository"

```csharp
public interface IRepository<T> where T : IEntity
{
    Task<T?> GetAsync(Guid id);
    IQueryable<T> Query();        // ← oh no
    Task AddAsync(T entity);
    Task DeleteAsync(T entity);
}
```

`IRepository<T>` generic trên EF Core thường **vô nghĩa** vì:

* `DbSet<T>` đã là repository.
* Expose `IQueryable<T>` leak persistence concern vào caller.
* Mọi method "find" kết thúc trên cùng interface, vi phạm ISP.

> **Quy tắc:** viết repository **cụ thể** per aggregate khi muốn contract domain-side ổn định. Bỏ qua abstraction và dùng `DbSet<T>` trực tiếp nếu service sẽ nói chuyện với EF dù sao.

(Xem [`repository-vi.md`](./repository-vi.md) cho deep dive — bao gồm tại sao Generic Repository là anti-pattern trong DDD.)

### 1.4 Khi Repository đáng giá

* Muốn **test domain logic** mà không spin up database.
* Muốn **swap data store** (EF → Dapper → in-memory) mà không rewrite caller.
* Đang theo **DDD aggregate** và muốn boundary nghiêm ngặt.

---

## 2. Unit of Work

> *"Duy trì danh sách object bị ảnh hưởng bởi business transaction và điều phối ghi thay đổi cũng như giải concurrency."* — Fowler

Tóm gọn: **track changes; commit như một operation atomic.**

### 2.1 Pattern

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}
```

### 2.2 Sống ở đâu trong EF Core

`DbContext` **CHÍNH LÀ** Unit of Work. Bạn không cần viết của riêng:

```csharp
public class CheckoutUseCase(IOrderRepository orders, ShopDbContext uow)
{
    public async Task PlaceAsync(Order o, CancellationToken ct)
    {
        await orders.AddAsync(o, ct);
        // ... nhiều thay đổi ...
        await uow.SaveChangesAsync(ct);    // một transaction
    }
}
```

### 2.3 Khi nào bọc

Bọc `SaveChangesAsync` đằng sau `IUnitOfWork` chỉ khi:

* Không muốn code domain / application biết về `DbContext`.
* Muốn swap data store (hiếm trong thực tế).
* Enforce **transaction boundary per-request** trong middleware.

Nếu không, phụ thuộc `DbContext` trực tiếp là OK — và EF Core ship hỗ trợ first-class.

Xem [`unit-of-work-vi.md`](./unit-of-work-vi.md) cho deep dive.

### 2.4 UoW + Repository cùng nhau

```
Use case ──► Repository  (add / remove / find)
   │
   └────────► Unit of Work (SaveChangesAsync ở cuối)
```

Repository thêm **cái gì** thay đổi. UoW quyết định **khi nào** flush.

---

## 3. Lazy Loading

> Hoãn load data liên quan tới khi nó thực sự được truy cập.

### 3.1 Idea

```csharp
// Order entity với lazy navigation property
public class Order
{
    public Guid Id { get; set; }
    public virtual ICollection<LineItem> Items { get; set; } = new List<LineItem>();
    //     ^^^^^^^ — EF proxy override để load lúc truy cập đầu
}

var order = db.Orders.Find(id);             // SQL: SELECT * FROM Orders WHERE Id = ...
var first = order.Items.First();            // SQL: SELECT * FROM LineItems WHERE OrderId = ...
//          └─ truy cập này trigger query thứ hai
```

### 3.2 Cách EF Core enable

```csharp
services.AddDbContext<ShopDbContext>(o => o
    .UseLazyLoadingProxies()
    .UseSqlServer(connectionString));
```

Navigation property phải `virtual` để EF có thể override với proxy.

### 3.3 Lợi & Hại

| Lợi                                    | Hại                                                  |
| --------------------------------------- | --------------------------------------------------- |
| Query khởi đầu nhỏ                      | Query ẩn — bất ngờ runtime                          |
| Tiện — chỉ truy cập property            | **N+1 problem** (xem dưới)                          |
| Hoạt động với code không biết sẽ cần data liên quan | Cần DbContext mở khi truy cập           |
|                                         | Behavior untestable không có DB thật                |

### 3.4 N+1 Problem

Footgun performance nổi tiếng nhất trong lịch sử ORM:

```csharp
var orders = db.Orders.ToList();             // 1 query
foreach (var o in orders)
    Console.WriteLine(o.Customer.Name);      // N query — một per order
```

`1 + N` query thay vì một — chết bởi nghìn query nhỏ. Lazy loading làm điều này **vô hình** cho tới production.

> **Khuyến nghị mạnh:** tắt lazy loading **off** trong service xử lý request bên ngoài. Dùng **eager** hoặc **explicit** loading.

---

## 4. Eager Loading

> Load data liên quan **upfront**, trong cùng query, trước khi ai cần.

### 4.1 EF Core — `Include` / `ThenInclude`

```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .Where(o => o.PlacedAt > since)
    .ToListAsync(ct);
```

* **Predictable** — một SQL statement (hoặc số nhỏ, cố định).
* **Visible** — developer chọn cái gì load.
* **Tunable** — performance có thể lý luận.

### 4.2 Dùng khi

* Caller **luôn** cần data liên quan.
* Đang render screen / API response với shape biết trước.
* Cardinality quan hệ nhỏ (tránh eagerly load 10k child row không cần).

### 4.3 Lợi & Hại

| Lợi                               | Hại                                              |
| --------------------------------- | ----------------------------------------------- |
| Không N+1                         | Payload khởi đầu to                              |
| Performance predictable           | Lãng phí data nếu caller không cần              |
| Explicit, dễ audit                | Verbose cho graph sâu                            |

### 4.4 Cảnh báo Cartesian Explosion

`Include` translate thành SQL `JOIN`. Hai `Include` collection không liên quan có thể nhân hàng:

```csharp
db.Orders
  .Include(o => o.Items)     // 1 order × 10 item
  .Include(o => o.Comments)  // 1 order × 5 comment
  // SQL row count: 1 × 10 × 5 = 50 hàng cho một order
```

EF Core 5+ mặc định **split query** trong một số case, và bạn có thể ép:

```csharp
db.Orders.AsSplitQuery().Include(...).Include(...);
```

---

## 5. Explicit Loading

> Caller quyết định chính xác khi nào load data liên quan — không lazy không eager.

### 5.1 API EF Core

```csharp
var order = await db.Orders.FindAsync(id);

// sau, nếu cần:
await db.Entry(order).Collection(o => o.Items).LoadAsync();
await db.Entry(order).Reference(o => o.Customer).LoadAsync();
```

### 5.2 Dùng khi

* Loading có điều kiện: *"nếu X thì cũng fetch Y"*.
* Streaming / pagination access bạn kiểm soát batching.
* Graph lớn muốn materialize **từng phần**.

Đây là middle ground — không có magic của lazy, nhưng linh hoạt hơn always-eager.

---

## 6. Identity Map

> *"Đảm bảo mỗi object chỉ load một lần bằng cách giữ mọi object đã load trong map. Lookup object dùng map khi tham chiếu."* — Fowler

Một `DbContext` trong EF Core (và `Session` trong Hibernate/NHibernate) là Identity Map.

```csharp
var a = db.Orders.Find(id);
var b = db.Orders.Find(id);     // trả về cùng instance — không query thứ hai
Console.WriteLine(ReferenceEquals(a, b));    // True
```

### Tại sao quan trọng

* **Consistency** — trong một unit of work, mọi reference tới "customer id X" trỏ cùng object.
* **Change tracking** — modification tới một reference được thấy bởi mọi nơi.
* **Performance** — duplicate lookup miễn phí.

Identity Map **gắn với unit of work** (per request, per session). Ngoài boundary đó, hai query có thể trả về hai object in-memory khác nhau cùng đại diện cho hàng giống nhau.

---

## 7. Active Record vs Data Mapper

Hai cách tiếp cận đối lập kết nối domain object với database.

### 7.1 Active Record

> Entity *biết tự persist*.

```csharp
public class Order : ActiveRecord
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }

    public void Save()    { /* INSERT/UPDATE hàng này */ }
    public void Delete()  { /* DELETE hàng này */ }

    public static Order Find(Guid id) { /* SELECT ... */ }
}
```

**Ví dụ:** Rails ActiveRecord, Laravel Eloquent, Django ORM.

| Lợi                             | Hại                                          |
| ------------------------------- | --------------------------------------------- |
| Boilerplate nhỏ                 | Couple domain với persistence                 |
| Nhanh khởi đầu                  | Khó unit-test logic domain thuần              |
| Tuyệt cho app CRUD              | Khó swap data store                           |
|                                 | Bloat entity khi rule tăng                    |

### 7.2 Data Mapper

> Entity **không biết persistence**; một **mapper** riêng chuyển data giữa object và database.

```csharp
public class Order   // domain thuần, không biết SQL
{
    public Guid Id { get; }
    public Money Total { get; private set; }
    public void MarkPaid() { ... }
}

public class OrderMapper   // làm mọi persistence
{
    public Order Load(Guid id) { ... }
    public void  Save(Order o) { ... }
}
```

**Ví dụ:** EF Core (với thiết kế aggregate đúng), Hibernate, Doctrine.

| Lợi                                    | Hại                                 |
| -------------------------------------- | ----------------------------------- |
| Tách concern sạch                       | Plumbing nhiều hơn                  |
| Domain dễ unit-test                    | Config ORM mapping có thể phức tạp  |
| Data store có thể swap                 | Learning curve cao                  |
| Hợp với DDD aggregate                  |                                     |

### 7.3 Chọn cái nào?

* **App nhỏ / CRUD-heavy, iterate nhanh**? Active Record trung thực với nhu cầu.
* **Domain phong phú, hệ thống sống lâu, IO đa dạng, core testable**? Data Mapper.
* **Lai?** Có — EF Core có thể là cái nào tuỳ bạn giữ persistence concern khỏi entity nghiêm ngặt đến đâu.

---

## 8. Query Object & Specification

Khi query lớn lên có điều kiện, dynamic, hoặc reuse, một LINQ expression ad-hoc thành gánh nặng maintain.

### 8.1 Query Object

> Đóng gói query thành object với API explicit.

```csharp
public class FindHighValueOrdersQuery
{
    public DateTime Since { get; init; }
    public decimal  MinTotal { get; init; }
    public string?  Country { get; init; }

    public IQueryable<Order> Apply(IQueryable<Order> source)
    {
        var q = source.Where(o => o.PlacedAt >= Since && o.Total >= MinTotal);
        if (Country is not null) q = q.Where(o => o.Customer.Country == Country);
        return q;
    }
}
```

### 8.2 Specification

> Đóng gói **predicate** thành object có thể combine.

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T candidate);
    Expression<Func<T, bool>> ToExpression();
}

public class HighValueOrderSpec(decimal min) : ISpecification<Order>
{
    public Expression<Func<Order, bool>> ToExpression() => o => o.Total >= min;
    public bool IsSatisfiedBy(Order o) => o.Total >= min;
}

// Compose:
var spec = new HighValueOrderSpec(1000m).And(new InCountrySpec("US"));
var orders = db.Orders.Where(spec.ToExpression()).ToList();
```

Specification tuyệt khi:

* Cùng predicate dùng cho query **và** in-memory validation.
* Business rule là object explicit, có tên, testable.

(Xem [`other-enterprise-vi.md`](./other-enterprise-vi.md) cho more về Specification.)

---

## 9. Chọn chiến lược

Cây quyết định thực hành cho loading data liên quan:

```
                ┌─────────────────────────────────────────┐
                │ Caller có luôn cần data này không?      │
                └──────────────┬──────────────────────────┘
                               │
                  ┌────────────┴────────────┐
                 Có                        Không
                  ▼                         ▼
            EAGER (Include)          ┌─────────────────────────┐
                                     │ Sẽ cần đôi khi /        │
                                     │ có điều kiện?           │
                                     └──────────┬──────────────┘
                                                │
                                  ┌─────────────┴────────────┐
                                 Có                         Không
                                  ▼                          ▼
                        EXPLICIT (LoadAsync)            Đừng load.
                                                       (Đơn giản đừng.)
```

> **Mặc định:** tắt lazy loading toàn cục. Làm mỗi fetch data là **quyết định có ý thức**.

---

## Tham chiếu nhanh

| Khái niệm           | Sở hữu…                      | Trả về…                       |
| ------------------- | ---------------------------- | ----------------------------- |
| **Repository**      | Một domain aggregate          | Domain object                 |
| **Unit of Work**    | Change tracking + transaction | Số hàng saved                 |
| **Lazy Loading**    | Fetch hoãn lúc truy cập       | Query thêm ẩn                 |
| **Eager Loading**   | Join upfront                  | Một query predictable         |
| **Explicit Load**   | Fetch caller kiểm soát        | Chính xác cái xin             |
| **Identity Map**    | Cache per-context theo id     | Cùng instance cho cùng hàng   |
| **Active Record**   | Entity = persistence object   | CRUD trên `this`              |
| **Data Mapper**     | Class mapper                  | Domain object thuần           |
| **Query Object**    | Query có tên                  | Kết quả filter                |
| **Specification**   | Predicate có tên              | Rule composable               |

---

> **Tiếp:** [`other-enterprise-vi.md`](./other-enterprise-vi.md) — Specification (sâu hơn), CQRS, Mediator, Result, Options, Decorator pipeline.
