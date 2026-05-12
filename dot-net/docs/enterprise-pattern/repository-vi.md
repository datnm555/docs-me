# Repository Pattern (và Anti-Pattern "Generic Repository")

> **Repository** là một **Enterprise Pattern** (Fowler PoEAA, 2002) và một **Tactical DDD pattern** (Evans, 2003). Nó làm trung gian giữa **domain** và **data mapping layer**, hành xử như một **in-memory collection** của domain object.

Nó **KHÔNG PHẢI** GoF design pattern.

> 🇻🇳 Phiên bản tiếng Việt. English: [`repository.md`](./repository.md)

---

## Repository là gì

Một Repository:

1. Expose domain bằng **các thao tác giống collection** (`Add`, `Get`, `Remove`).
2. **Ẩn** data store (SQL, NoSQL, file, web API).
3. Được định nghĩa bằng **interface trong domain** và **implement trong infrastructure**.
4. Trong DDD: **một repository cho mỗi Aggregate Root** — không phải mỗi table.

```csharp
// Trong domain — abstraction
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<Order>> GetUnpaidByCustomerAsync(Guid customerId, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Trong infrastructure — implementation
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct) =>
        _db.Orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);

    public Task<IReadOnlyList<Order>> GetUnpaidByCustomerAsync(Guid customerId, CancellationToken ct) =>
        _db.Orders.Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Placed)
                  .ToListAsync(ct)
                  .ContinueWith(t => (IReadOnlyList<Order>)t.Result, ct);

    public Task AddAsync(Order order, CancellationToken ct)
    {
        _db.Orders.Add(order);
        return Task.CompletedTask; // commit qua Unit of Work (SaveChangesAsync)
    }
}
```

Chú ý:

* Các method có **ý nghĩa business** — `GetUnpaidByCustomerAsync`, không phải `GetByPredicate(x => ...)`.
* Interface không biết gì về EF Core; implementation biết hết.
* Không có method `Update` — EF Core track thay đổi; bạn mutate entity rồi `SaveChangesAsync`.

Đây là **Repository pattern đúng nghĩa** như Fowler và Evans định nghĩa.

---

## Generic Repository — và tại sao nhiều tác giả DDD coi nó là Anti-Pattern

**Generic Repository** là cám dỗ:

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct);
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task AddAsync(T entity, CancellationToken ct);
    void Update(T entity);
    void Remove(T entity);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly AppDbContext _db;
    public Repository(AppDbContext db) => _db = db;
    public Task<T?> GetByIdAsync(object id, CancellationToken ct) =>
        _db.Set<T>().FindAsync([id], ct).AsTask();
    public Task<IEnumerable<T>> GetAllAsync(CancellationToken ct) =>
        _db.Set<T>().ToListAsync(ct).ContinueWith<IEnumerable<T>>(t => t.Result, ct);
    public Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct) =>
        _db.Set<T>().Where(predicate).ToListAsync(ct).ContinueWith<IEnumerable<T>>(t => t.Result, ct);
    public Task AddAsync(T entity, CancellationToken ct) { _db.Set<T>().Add(entity); return Task.CompletedTask; }
    public void Update(T entity) => _db.Set<T>().Update(entity);
    public void Remove(T entity) => _db.Set<T>().Remove(entity);
}

// Đăng ký: services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```

### Tại sao cái này bị coi là **anti-pattern** trong DDD / Clean architecture

1. **Đã tồn tại rồi.** `DbContext` + `DbSet<T>` của EF Core *là* một generic repository. Bọc thêm một generic repository bên ngoài **chỉ thêm layer mà không có behavior** — thuần thủ tục.

2. **Leak persistence concern vào domain.** Signature `Find(Expression<Func<T, bool>>)` có nghĩa là **caller viết query**. Đây ngược lại với "ẩn data store". Mỗi controller gọi `repo.Find(o => o.Status == OrderStatus.Placed)` đang làm data access.

3. **Khuyến khích interface anemic, không có intent.** Repository trở thành cửa CRUD không mặt — không có `GetUnpaidByCustomer`, không có `GetOverdueInvoices`. Bạn mất *domain vocabulary* — cái chính của repository.

4. **Giả vờ testable nhưng không thật.** Người ta mock `IRepository<T>` trong test, nhưng các mock này chấp nhận mọi `Expression<Func<T,bool>>` — chúng không enforce rằng expression trả về đúng thứ. Test pass nhưng SQL sai.

5. **Phức tạp hoá real query.** Join, projection, paging, include — không cái nào fit `IRepository<T>` gọn ghẽ. Cuối cùng bạn expose `IQueryable<T>` từ repository, mà đó cũng chỉ là `_db.Set<T>()`.

6. **Vi phạm quy tắc "một repository cho mỗi aggregate root".** Bạn có `IRepository<OrderItem>`, `IRepository<Address>`, v.v. — nhưng `OrderItem` **không phải aggregate root**; nó chỉ được modify qua `Order`. Generic repo cho client bypass quy tắc đó.

### Những câu nói nổi tiếng

> *"Đừng dùng Generic Repository. Nó là abstraction sai."* — **Tim McCarthy**, *.NET Domain-Driven Design with C#*.
>
> *"Một repository không nên là wrapper mỏng quanh data access technology. Nếu vậy, nó không có lý do tồn tại."* — **Vaughn Vernon**, *Implementing Domain-Driven Design*.
>
> *"Generic repository … thực sự không abstract gì cả. Nó chỉ thêm một layer indirection."* — **Jimmy Bogard** (tác giả MediatR / AutoMapper).

---

## Generic Repository thực sự là gì

Nó **không phải** GoF, PoEAA, hay DDD pattern có tên. Nó là một **idiom / kỹ thuật implementation** trong C# dùng generic. Một số tên gọi khác:

* "Generic Repository (over an ORM)"
* "Repository over IQueryable"
* "Type-parameterized DAO"
* Trong Java: tương tự **Spring Data's `JpaRepository<T, ID>`** — nhưng Spring Data *sinh implementation* từ tên method, khác hẳn về bản chất.

**Tóm lại:** *Generic Repository* là tên người ta dùng, nhưng không có trong bất kỳ catalog pattern chính thống nào. Gọi nó là "design pattern" là quá hào phóng.

---

## Khi nào Generic Repository chấp nhận được?

Bạn có thể bảo vệ nó trong tình huống **hẹp**:

* **App CRUD admin đơn giản** — bạn thật sự chỉ cần `Add/Update/Remove/GetById` cho mỗi table và không gì hơn.
* **Library nền tảng tái sử dụng** — bạn ship library dùng cho nhiều service nhỏ, đều CRUD. Generic repo tiết kiệm boilerplate.
* **Typed wrapper trên store không-LINQ** (vd. raw MongoDB hoặc DynamoDB) — bạn muốn shape thống nhất.

Nhưng trong codebase DDD nghiêm túc: **đừng**. Viết repository cụ thể nói ngôn ngữ domain.

---

## C# — Một Repository DDD-style tốt

```csharp
public interface ICustomerRepository
{
    Task<Customer?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<Customer?> GetByEmailAsync(Email email, CancellationToken ct);
    Task<IReadOnlyList<Customer>> FindHighValueAsync(Money threshold, CancellationToken ct);
    Task AddAsync(Customer customer, CancellationToken ct);
}
```

Mỗi tên method là một **danh từ-động từ từ business**, không phải "find with predicate". Interface định nghĩa **các câu hỏi mà domain muốn được trả lời**, không phải cách trả lời.

---

## Repository vs. Service vs. Query Object

| Nếu cần…                                                | Dùng…                                       |
| ------------------------------------------------------- | ------------------------------------------ |
| Load / persist một **aggregate root**                   | **Repository** (một cho mỗi aggregate)      |
| Chạy read phức tạp (join, projection, paging)           | **Query object** (Dapper, raw SQL) — *không phải* repo |
| Orchestrate domain + infrastructure cho một use case    | **Application Service** / MediatR handler   |
| Đóng gói **predicate có thể compose**                   | **Specification pattern**                    |

CQRS làm điều này rõ ràng: **command dùng repository**, **query bỏ qua domain hoàn toàn** (Dapper thẳng tới DTO).

---

## Biến thể phổ biến

* **Read-only repository** — chỉ `GetById` / query. Cho aggregate không thay đổi.
* **Specification + Repository** — method `GetBySpec(ISpecification<T>)` compose expression. Sạch hơn raw predicate nếu dùng cẩn thận.
* **In-memory repository cho test** — implement cùng interface backed bởi `List<T>`.

---

## TL;DR

* **Repository** — đúng là pattern thật (Fowler PoEAA + DDD tactical). Dùng làm interface ngôn ngữ domain, một cho mỗi aggregate root.
* **Generic Repository** — không phải pattern có tên; trong giới DDD được coi rộng rãi là anti-pattern trên EF Core vì EF đã làm cùng việc và abstraction leak data-access concern vào domain.
* **Gọi nó là gì** — nếu cần tên, "Generic Repository idiom" hoặc "Repository over `IQueryable<T>`" là trung thực.

---

## Tham khảo

* Eric Evans — *Domain-Driven Design* (2003). Chương về Repository.
* Martin Fowler — *PoEAA* (2002). [Mục Repository](https://martinfowler.com/eaaCatalog/repository.html).
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Chương 12.
* Jimmy Bogard — [*Repository is the new Singleton*](https://lostechies.com/jimmybogard/2009/05/15/repositories-and-the-laws-of-thermodynamics/).
* Mark Seemann — [*IQueryable is Tight Coupling*](https://blog.ploeh.dk/2012/03/26/IQueryableTightCouplingPart1/).
