# Enterprise Patterns (Fowler PoEAA)

> Các pattern được Martin Fowler tổng hợp trong **Patterns of Enterprise Application Architecture** (2002). Chúng nằm *giữa* GoF design pattern và architectural style — định hình cách một application **tương tác với data và các application khác** ở mức layer-and-component.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## "Enterprise Pattern" là gì?

* **Phạm vi:** layer data-access, layer application service, các biên presentation/integration của một enterprise application điển hình.
* **Mức chi tiết:** to hơn class collaboration (GoF), nhỏ hơn một service hay system style.
* **Nguồn chuẩn:** Martin Fowler — *Patterns of Enterprise Application Architecture* (PoEAA).

Đây **KHÔNG PHẢI** GoF design pattern, dù nhìn tương tự. Đây là catalog riêng với tên riêng. Thành viên nổi tiếng nhất — **Repository** — cũng được DDD tái sử dụng như một *tactical pattern*.

---

## Index

| Pattern                  | Bài toán nó giải                                                              | File                                                |
| ------------------------ | ----------------------------------------------------------------------------- | --------------------------------------------------- |
| **Repository**           | Decouple domain khỏi data access; query như in-memory collection.             | [`repository-vi.md`](./repository-vi.md)            |
| **Unit of Work**         | Track thay đổi qua nhiều object và commit cùng nhau trong một transaction.    | [`unit-of-work-vi.md`](./unit-of-work-vi.md)        |
| **IoC / DI**             | Đảo construction object; phụ thuộc abstraction, không concretion.              | [`ioc-di.md`](./ioc-di.md) (chưa dịch)              |
| **Service Lifetimes**    | Transient / Scoped / Singleton — khi nào mỗi instance được tạo.                | [`lifetimes.md`](./lifetimes.md) (chưa dịch)        |
| **Data Access (Lazy/Eager/Identity Map/…)** | Loading strategy và pattern liên quan persistence.          | [`data-access.md`](./data-access.md) (chưa dịch)    |
| **Other Enterprise Patterns** | Specification, Mediator, Options, Result, Decorator pipeline, …          | [`other-enterprise.md`](./other-enterprise.md) (chưa dịch) |

## Các PoEAA Pattern khác (chưa mở rộng, tóm tắt)

| Pattern              | Một dòng                                                                  |
| -------------------- | ------------------------------------------------------------------------- |
| **Active Record**    | Một object cho mỗi table; object tự biết save/load. (Style của RoR.)       |
| **Data Mapper**      | Domain object không biết persistence; mapper chuyển data vào/ra.           |
| **Service Layer**    | Layer mỏng expose use case của application trên domain.                    |
| **Specification**    | Đóng gói predicate query thành object hạng nhất (có thể compose).           |
| **Identity Map**     | Một object cho mỗi identity trong session — EF Core change tracker.        |
| **Lazy Load**        | Hoãn load data liên quan tới khi cần — EF Core lazy proxy.                  |
| **DTO**              | Chuyển data giữa các layer / qua wire mà không leak domain type.           |
| **Front Controller** | Một entry point route mọi request — ASP.NET MVC/routing.                    |
| **MVC / MVP / MVVM** | Các pattern phân tách UI.                                                  |

---

## Quan hệ với .NET hiện đại

Nhiều pattern này **đã được implement trong framework**. Nhận ra trước khi tự build lại.

| Pattern         | Đã có sẵn trong .NET như…                                    |
| --------------- | ------------------------------------------------------------ |
| Repository      | `DbContext.Set<T>()` đã cho `IQueryable<T>`                  |
| Unit of Work    | Bản thân `DbContext` — `SaveChanges` commit toàn bộ thay đổi |
| Identity Map    | EF Core change tracker                                       |
| Lazy Load       | `UseLazyLoadingProxies()` trong EF Core                      |
| Data Mapper     | EF Core / Dapper                                             |
| Service Layer   | Application service / MediatR handler                        |
| Specification   | LINQ `Expression<Func<T,bool>>` predicate                    |
| DTO             | Record / API model                                           |
| Front Controller| Routing pipeline ASP.NET Core                                |

Vì lý do này, **áp dụng các pattern này một cách máy móc lên EF Core thường không có lợi và còn gây hại** — xem thảo luận về "Generic Repository" trong [`repository-vi.md`](./repository-vi.md).

---

## Tham khảo

* Martin Fowler — *Patterns of Enterprise Application Architecture* (2002). Tham chiếu chuẩn.
* Catalog online: [martinfowler.com/eaaCatalog](https://martinfowler.com/eaaCatalog/).
