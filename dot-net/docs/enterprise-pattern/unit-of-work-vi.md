# Unit of Work Pattern

> Một **Enterprise Pattern** (Fowler PoEAA, 2002): *"Duy trì danh sách các object bị ảnh hưởng bởi một business transaction và điều phối việc ghi thay đổi cũng như xử lý các vấn đề concurrency."*

Tóm gọn: Unit of Work **track những gì đã thay đổi** trong một business operation và **commit tất cả như một transaction** — atomic — ở cuối.

> 🇻🇳 Phiên bản tiếng Việt. English: [`unit-of-work.md`](./unit-of-work.md)

---

## Bài toán nó giải

Không có Unit of Work, application code chịu trách nhiệm track mọi thay đổi và save từng cái rõ ràng:

```csharp
// Không có UoW — dễ gãy
await _orderRepo.SaveAsync(order);
await _customerRepo.SaveAsync(customer);
await _inventoryRepo.SaveAsync(inventory);
// Nếu save thứ 3 fail, hai save đầu đã commit. Inconsistency.
```

Unit of Work thay đổi câu chuyện:

```csharp
// Với UoW
_orderRepo.Add(order);
customer.IncrementOrderCount();   // tracked tự động
inventory.Decrement(quantity);    // tracked tự động
await _uow.SaveChangesAsync(ct);  // một transaction, all-or-nothing
```

---

## DbContext của EF Core **chính là** Unit of Work

Đây là câu quan trọng nhất trong file này:

> **`DbContext` là Unit of Work. `DbSet<T>` là Repository.**

Docs của Microsoft xác nhận điều này rõ ràng. EF Core implement **Identity Map**, **Unit of Work**, và **(generic) Repository** pattern out-of-the-box:

```csharp
public sealed class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
}

// Đây CHÍNH LÀ Unit of Work
public async Task PlaceOrderAsync(...)
{
    var customer = await _db.Customers.FindAsync(customerId);
    var order = customer.PlaceOrder(items);

    _db.Orders.Add(order);
    customer.IncrementOrderCount();

    await _db.SaveChangesAsync(ct); // một transaction commit cả hai thay đổi
}
```

Bạn **không cần** bọc `DbContext` trong `IUnitOfWork` của mình để "tuân theo pattern". Bạn đã làm rồi.

---

## Khi nào bọc DbContext trong `IUnitOfWork`

Bạn sẽ thấy điều này thường xuyên trong codebase nghiêng DDD:

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

public sealed class AppDbContext : DbContext, IUnitOfWork { /* ... */ }
```

### Tại sao người ta làm vậy

* **Application layer** không được reference EF Core trực tiếp (luật Clean Architecture).
* Vì vậy họ expose `IUnitOfWork` từ domain/application layer, ẩn EF Core sau nó trong infrastructure.
* Đây là **lý do hợp lệ duy nhất** để introduce `IUnitOfWork` explicit.

### Khi KHÔNG nên làm

* Nếu architecture của bạn đã cho application gọi `_db.SaveChangesAsync(ct)` trực tiếp (phổ biến trong monolith thực dụng), thêm `IUnitOfWork` interface không thêm gì.
* Nếu bạn combine nhiều `DbContext` dưới một `IUnitOfWork` — cẩn thận. Transaction scope của EF Core là per-`DbContext`. Bạn cần `TransactionScope` hoặc `IDbContextTransaction`.

---

## Một `IUnitOfWork` thân thiện với domain

```csharp
// Domain / Application layer (không reference EF)
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct);
}

// Infrastructure layer
public sealed class AppDbContext : DbContext, IUnitOfWork
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    // SaveChangesAsync đã có trên DbContext — interface thoả mãn
}

// Wire-up
services.AddDbContext<AppDbContext>(opts => opts.UseNpgsql(connectionString));
services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
services.AddScoped<IOrderRepository, EfOrderRepository>();

// Use case ở application layer
public sealed class PlaceOrderHandler(
    IOrderRepository orders,
    ICustomerRepository customers,
    IUnitOfWork uow) : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var customer = await customers.GetByIdAsync(cmd.CustomerId, ct)
                       ?? throw new NotFoundException();
        var order = customer.PlaceOrder(cmd.Items);
        await orders.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct); // commit cả order mới và mutation của customer
        return order.Id;
    }
}
```

Giữ code use-case **không có EF** trong khi vẫn dùng Unit of Work built-in của EF dưới mui.

---

## Unit of Work vs. Database Transaction

Hai cái này **không** giống nhau.

| Khái niệm            | Phạm vi                                          | Cơ chế                                   |
| -------------------- | ------------------------------------------------ | ---------------------------------------- |
| **Database Transaction** | `BEGIN`/`COMMIT` trong DB                    | `BeginTransaction()` / SQL                |
| **Unit of Work**     | Change tracker ở application layer               | `DbContext` (track thay đổi trong memory) |

`SaveChangesAsync` mở **một DB transaction**, apply mọi thay đổi đã track, commit. Trong thực tế chúng trùng nhau — nhưng về khái niệm Unit of Work có thể batch và retry; transaction chỉ là bước persist bền.

Cho kịch bản **multi-DbContext** hoặc **multi-resource**:

```csharp
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await _ordersDb.SaveChangesAsync(ct);
await _inventoryDb.SaveChangesAsync(ct);
scope.Complete();
```

(Sẽ promote thành distributed transaction — thường là dấu hiệu bạn nên dùng [Saga](../architectural-pattern/saga-vi.md) hơn.)

---

## Anti-Pattern

* **Tự build `UnitOfWork`** chỉ bọc `DbContext` với method `Begin`/`Commit`/`Rollback`. EF Core đã cho atomicity qua `SaveChanges`.
* **Method repository gọi `SaveChanges` bên trong.** Buộc mỗi write vào transaction riêng — phá atomicity xuyên nhiều repository. Chỉ commit ở use-case level.
* **UoW sống lâu.** UoW nên match một use case / HTTP request. `DbContext` scoped của EF Core enforce điều này tự nhiên.

---

## TL;DR

* **Unit of Work** là Enterprise Pattern thật sự.
* **`DbContext` của EF Core đã là Unit of Work**.
* Bọc trong `IUnitOfWork` **chỉ** để application layer không phụ thuộc EF.
* Commit **một lần cho mỗi use case**, không bao giờ trong từng repository.

---

## Tham khảo

* Martin Fowler — *PoEAA* (2002). [Mục Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html).
* Microsoft Docs — *Implement the infrastructure persistence layer with Entity Framework Core*. ([learn.microsoft.com](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core)).
