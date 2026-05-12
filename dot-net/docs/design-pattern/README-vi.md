# Design Pattern trong .NET — Hướng dẫn đầy đủ

> Tham chiếu thực hành cho 23 design pattern kinh điển của **Gang of Four (GoF)**, kèm sample C# idiomatic cho .NET hiện đại (.NET 6 / 7 / 8 / 9).

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Mục lục

1. [Design Pattern là gì?](#1-design-pattern-là-gì)
2. [Có bao nhiêu Design Pattern?](#2-có-bao-nhiêu-design-pattern)
3. [Ba category](#3-ba-category)
4. [Catalog đầy đủ (23 GoF Pattern)](#4-catalog-đầy-đủ-23-gof-pattern)
5. [Cách chọn Pattern](#5-cách-chọn-pattern)
6. [Pattern được dùng nhiều nhất trong dự án thực tế](#6-pattern-được-dùng-nhiều-nhất-trong-dự-án-thực-tế)
7. [Pattern vs. Nguyên lý vs. Architecture](#7-pattern-vs-nguyên-lý-vs-architecture)
8. [.NET hiện đại đã implement sẵn nhiều cái](#8-net-hiện-đại-đã-implement-sẵn-nhiều-cái)
9. [Anti-Pattern cần tránh](#9-anti-pattern-cần-tránh)
10. [Đọc thêm](#10-đọc-thêm)

---

## 1. Design Pattern là gì?

**Design pattern** là *giải pháp có tên, tái sử dụng được* cho một *bài toán thiết kế lặp lại* trong phần mềm hướng đối tượng. Pattern không phải code hoàn chỉnh — nó là **template** mô tả cách cấu trúc class và object để giải bài toán theo cách linh hoạt và dễ bảo trì.

Các tính chất của một pattern thực sự:

* **Bài toán lặp lại** — xuất hiện trong nhiều dự án.
* **Giải pháp đã được chứng minh** — đã được dùng và refine qua thời gian.
* **Trade-off** — mọi pattern thêm indirection; chỉ dùng khi nó đáng.
* **Vocabulary** — cho developer một ngôn ngữ chung ("dùng Strategy ở đây", "đây là Decorator").

---

## 2. Có bao nhiêu Design Pattern?

Câu trả lời chuẩn là **23 pattern**, định nghĩa bởi *Gang of Four* (Gamma, Helm, Johnson, Vlissides) trong cuốn **"Design Patterns: Elements of Reusable Object-Oriented Software"** (1994).

| Category       | Số lượng | Mục đích                                       |
| -------------- | ------- | --------------------------------------------- |
| **Creational** | 5       | Cách object được **tạo**                       |
| **Structural** | 7       | Cách object được **kết hợp**                   |
| **Behavioral** | 11      | Cách object **giao tiếp** và phân chia công việc |
| **Tổng**       | **23**  |                                               |

Ngoài 23 của GoF, ngành đã thêm nhiều cái khác (vd. **Repository**, **Unit of Work**, **Specification**, **CQRS**, **Mediator (kiểu MediatR)**, **Null Object**, **Dependency Injection**, **Service Locator**, **Lazy Load**) — chúng thường được nhóm dưới *Enterprise Pattern* (Martin Fowler, *PoEAA*) thay vì GoF.

Hướng dẫn này tập trung vào **23 GoF pattern** như nền tảng.

---

## 3. Ba category

### 3.1 Creational Pattern (5)

Liên quan đến **tạo object** — abstract và decouple *cách* instantiate. Hữu ích khi `new SomeClass()` trở nên quá cứng nhắc.

→ Xem [`creational-vi.md`](./creational-vi.md)

### 3.2 Structural Pattern (7)

Liên quan đến **kết hợp object** — cách class và object được lắp ráp thành cấu trúc lớn hơn mà vẫn giữ linh hoạt và hiệu quả.

→ Xem [`structural-vi.md`](./structural-vi.md)

### 3.3 Behavioral Pattern (11)

Liên quan đến **giao tiếp và trách nhiệm** giữa các object — cách chúng tương tác, phân chia công việc, và quản lý thuật toán.

→ Xem [`behavioral-vi.md`](./behavioral-vi.md)

---

## 4. Catalog đầy đủ (23 GoF Pattern)

### Creational (5)

| #  | Pattern              | Ý định một dòng                                                                  |
| -- | -------------------- | -------------------------------------------------------------------------------- |
| 1  | **Singleton**        | Đảm bảo một class có **một instance duy nhất** và cho global access point.       |
| 2  | **Factory Method**   | Cho subclass quyết định **class cụ thể nào** sẽ được instantiate.                |
| 3  | **Abstract Factory** | Tạo **gia đình object liên quan** mà không chỉ định class cụ thể.                |
| 4  | **Builder**          | Construct object phức tạp **từng bước**; cùng process, kết quả khác.             |
| 5  | **Prototype**        | Tạo object mới bằng **clone** một object có sẵn.                                  |

### Structural (7)

| #  | Pattern         | Ý định một dòng                                                                         |
| -- | --------------- | --------------------------------------------------------------------------------------- |
| 6  | **Adapter**     | Chuyển một interface thành interface client muốn (translator).                          |
| 7  | **Bridge**      | Decouple **abstraction** khỏi **implementation** để cả hai có thể vary độc lập.         |
| 8  | **Composite**   | Treat object đơn lẻ và composition **đồng nhất** (cấu trúc cây).                         |
| 9  | **Decorator**   | Gắn trách nhiệm mới vào object **động**.                                                |
| 10 | **Facade**      | Cung cấp **interface đơn giản hơn** trên subsystem phức tạp.                            |
| 11 | **Flyweight**   | Share object nhỏ hiệu quả để giảm memory.                                               |
| 12 | **Proxy**       | Cung cấp **placeholder/surrogate** để kiểm soát truy cập tới object khác.               |

### Behavioral (11)

| #  | Pattern                    | Ý định một dòng                                                                |
| -- | -------------------------- | ----------------------------------------------------------------------------- |
| 13 | **Chain of Responsibility**| Truyền request qua một chuỗi cho tới khi ai đó xử lý.                         |
| 14 | **Command**                | Đóng gói request thành object (queue, log, undo).                              |
| 15 | **Interpreter**            | Định nghĩa grammar và interpreter cho câu trong grammar đó.                    |
| 16 | **Iterator**               | Duyệt collection tuần tự mà không expose internal.                             |
| 17 | **Mediator**               | Tập trung giao tiếp phức tạp giữa các object.                                  |
| 18 | **Memento**                | Capture và restore state nội bộ của object (undo).                             |
| 19 | **Observer**               | Notification one-to-many khi state thay đổi (pub/sub).                         |
| 20 | **State**                  | Cho object thay đổi behavior khi state nội bộ thay đổi.                        |
| 21 | **Strategy**               | Đóng gói thuật toán có thể đổi đằng sau một interface chung.                   |
| 22 | **Template Method**        | Định nghĩa skeleton của thuật toán; cho subclass fill các step.                |
| 23 | **Visitor**                | Thêm operation mới lên object mà không modify chúng.                           |

---

## 5. Cách chọn Pattern

Đặt câu hỏi trước; pattern theo sau.

| Nếu cần…                                                   | Cân nhắc                                              |
| ---------------------------------------------------------- | ----------------------------------------------------- |
| Ẩn logic construction phức tạp                             | **Builder**, **Factory Method**, **Abstract Factory** |
| Đảm bảo đúng một instance                                  | **Singleton** (hoặc DI với Singleton lifetime)        |
| Thêm behavior vào object mà không inheritance              | **Decorator**                                         |
| Làm cho interface không tương thích làm việc với nhau       | **Adapter**                                           |
| Đơn giản hoá API phức tạp cho client                       | **Facade**                                            |
| Đổi thuật toán runtime                                     | **Strategy**                                          |
| Phản ứng với thay đổi state ở chỗ khác                     | **Observer**                                          |
| Xử lý request qua nhiều handler tuỳ chọn                   | **Chain of Responsibility**                           |
| Hỗ trợ undo / redo                                         | **Command** + **Memento**                             |
| Đi qua cây nodes hỗn hợp leaf/branch                       | **Composite** + **Visitor**                           |
| Hoãn hoặc bọc truy cập object (cache, lazy, security)      | **Proxy**                                             |
| Giảm memory của nhiều object giống nhau                    | **Flyweight**                                         |
| Đổi behavior dựa trên lifecycle/state machine              | **State**                                             |

---

## 6. Pattern được dùng nhiều nhất trong dự án thực tế

Không phải 23 pattern được dùng bằng nhau. Trong codebase .NET production (web API, microservices, line-of-business app), phân bố lệch mạnh. Đây là xếp hạng thực dụng — dựa trên độ thường gặp trong stack ASP.NET Core / EF Core / MediatR-style thực tế.

### Tier 1 — Dùng trong **hầu hết mọi** dự án .NET

Phổ biến đến mức bạn thường dùng mà không gọi tên.

| Pattern              | Xuất hiện ở đâu                                                       |
| -------------------- | --------------------------------------------------------------------- |
| **Strategy**         | Bất cứ lúc nào bạn register nhiều implementation của một interface trong DI (auth provider, pricing rule, exporter). |
| **Singleton**        | `AddSingleton<T>()` — config cache, HTTP factory, in-memory store.    |
| **Factory (Method)** | `IHttpClientFactory`, `ILoggerFactory`, keyed service, MediatR resolve handler. |
| **Decorator**        | Cross-cutting concern: caching/logging/retry quanh service; ASP.NET middleware; MediatR pipeline behavior. |
| **Iterator**         | Mỗi `foreach`; `IAsyncEnumerable<T>` để stream query.                  |
| **Observer**         | `event`, domain event, MediatR `INotification`, `IHostedService`, Reactive Extensions. |
| **Builder**          | `WebApplicationBuilder`, `HostBuilder`, `StringBuilder`, EF model builder, fluent test builder. |
| **Facade**           | Class "application service" / "use case" điều phối nhiều dependency sau một method. |

### Tier 2 — **Phổ biến** trong dự án vừa-lớn

Gặp vài tuần một lần.

| Pattern                  | Xuất hiện ở đâu                                                   |
| ------------------------ | ----------------------------------------------------------------- |
| **Command**              | CQRS, MediatR `IRequest`, message bus payload, audit log.          |
| **Mediator**             | MediatR — chuẩn de-facto trong .NET architecture hiện đại.         |
| **Adapter**              | Bọc SDK third-party (Stripe, Twilio, S3) sau interface của bạn.    |
| **Template Method**      | `BackgroundService.ExecuteAsync`, base controller, base job.       |
| **Chain of Responsibility** | ASP.NET middleware, `HttpClient` `DelegatingHandler`.           |
| **Proxy**                | EF Core lazy loading, gRPC/Refit client, authorization wrapper.    |
| **State**                | Workflow order/booking/loan với `Stateless` hoặc Workflow Core.    |

### Tier 3 — **Hữu ích trong niche cụ thể**

Lấy ra khi bài toán khớp.

| Pattern             | Khi nào nó đáng                                                  |
| ------------------- | ----------------------------------------------------------------- |
| **Abstract Factory**| Theming, multi-tenant provider, đổi cả gia đình cùng lúc.         |
| **Composite**       | Data dạng cây: AST, file system, permission/menu tree.             |
| **Bridge**          | Hai trục biến đổi độc lập (vd. message-type × channel).             |
| **Visitor**         | Roslyn analyzer, expression-tree rewriter, AST walker.              |
| **Interpreter**     | DSL parser, rule engine, search-query language.                     |
| **Memento**         | Undo/redo trong editor; transactional rollback của in-memory state. |
| **Prototype**       | `record` + `with`; clone template đắt build.                       |

### Tier 4 — **Hiếm** trong code app .NET hiện đại

| Pattern       | Tại sao hiếm                                                                   |
| ------------- | ------------------------------------------------------------------------------ |
| **Flyweight** | Vấn đề áp lực memory hiếm trong app web/business điển hình. Dành cho rendering, gaming, large-document tooling. |

### Tóm tắt thẳng thắn

Nếu chỉ học **8 pattern**, học những cái này — bao 80% việc bạn làm trong codebase .NET thực tế:

> **Strategy, Decorator, Factory, Singleton, Builder, Observer, Mediator/Command, Facade.**

Thêm **Adapter** và **Template Method** đạt ~90%. Còn lại tuỳ tình huống.

---

## 7. Pattern vs. Nguyên lý vs. Architecture

Ba khái niệm này thường bị nhầm. Chúng vận hành ở các cấp khác nhau.

| Khái niệm         | Phạm vi            | Ví dụ                                                  |
| ----------------- | ------------------ | ------------------------------------------------------ |
| **Nguyên lý**     | Hướng dẫn          | SOLID, DRY, KISS, YAGNI, Law of Demeter                |
| **Pattern**       | Class/object level | Strategy, Observer, Builder (guide này)                |
| **Architecture**  | System level       | N-Layer, Clean, Onion, Hexagonal, CQRS, Microservices  |

Pattern **implement** nguyên lý; architecture **tổ chức** pattern.

---

## 8. .NET hiện đại đã implement sẵn nhiều cái

Đừng reinvent. Nhận ra pattern đã sẵn trong framework:

| Pattern             | Built-in trong .NET                                                    |
| ------------------- | ---------------------------------------------------------------------- |
| **Singleton**       | `services.AddSingleton<T>()` trong DI container.                       |
| **Factory**         | `IHttpClientFactory`, `ILoggerFactory`, `IServiceProvider`.            |
| **Builder**         | `StringBuilder`, `HostBuilder`, `WebApplicationBuilder`, EF model builder. |
| **Iterator**        | `IEnumerable<T>` / `IEnumerator<T>` — mỗi `foreach` dùng nó.           |
| **Observer**        | `IObservable<T>`/`IObserver<T>`, `events`, `IHostedService`, Reactive Extensions. |
| **Decorator**       | ASP.NET Core middleware pipeline; `Scrutor`'s `Decorate<T>()`.          |
| **Strategy**        | DI một interface với nhiều implementation + `IEnumerable<T>`.           |
| **Chain of Resp.**  | ASP.NET middleware; delegating handler trong `HttpClient`.              |
| **Adapter**         | `IAsyncEnumerable<T>` wrapper; SDK adapter.                            |
| **Proxy**           | EF Core lazy loading proxy; Castle DynamicProxy; `dispatch_proxy`.     |
| **Command**         | MediatR's `IRequest`; CQRS command.                                    |
| **Mediator**        | MediatR.                                                               |
| **Template Method** | Base controller, `BackgroundService.ExecuteAsync`.                     |
| **State**           | Thư viện `Stateless`, workflow engine.                                 |

Bài học: phần lớn thời gian bạn **consume** một pattern qua framework, không phải implement từ đầu.

---

## 9. Anti-Pattern cần tránh

| Anti-Pattern              | Mô tả                                                                          |
| ------------------------- | ------------------------------------------------------------------------------ |
| **Patternitis**           | Dùng pattern vì pattern. Thêm indirection mà không có lợi.                      |
| **Lạm dụng Singleton**    | Global mutable state. Ẩn dependency và giết testability.                       |
| **God Factory**           | Factory biết về mọi type cụ thể — thành switch dài.                              |
| **Anemic Domain Model**   | Entity domain chỉ có getter/setter, mọi logic trong class "service".            |
| **Service Locator**       | Ẩn dependency sau container lookup; nên dùng constructor injection.             |
| **Premature Abstraction** | Tạo strategy/factory trước khi có hai implementation thật.                      |

**Quy tắc:** chờ đến khi có **ít nhất hai biến thể thật** trước khi extract một pattern. Lần đầu chỉ là code; lần thứ hai mới là lúc pattern đáng được dùng.

---

## 10. Đọc thêm

* Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (1994). Sách GoF gốc.
* Freeman & Robson — *Head First Design Patterns* (2nd ed., 2020). Giới thiệu thân thiện với ví dụ Java.
* Martin Fowler — *Patterns of Enterprise Application Architecture* (2002). Pattern enterprise/persistence.
* Mark Seemann — *Dependency Injection Principles, Practices, and Patterns* (2019). Pattern DI-aware trong .NET.
* [refactoring.guru/design-patterns](https://refactoring.guru/design-patterns) — giải thích trực quan, hiện đại với ví dụ C#.

---

## Danh mục file

* [`README.md`](./README.md) — bản tiếng Anh
* [`README-vi.md`](./README-vi.md) — file này
* [`creational.md`](./creational.md) / [`creational-vi.md`](./creational-vi.md) — 5 creational pattern với sample C#
* [`structural.md`](./structural.md) / [`structural-vi.md`](./structural-vi.md) — 7 structural pattern với sample C#
* [`behavioral.md`](./behavioral.md) / [`behavioral-vi.md`](./behavioral-vi.md) — 11 behavioral pattern với sample C#
