# Tài Liệu .NET Architecture & Patterns

> Folder này tổ chức các software pattern theo **đúng category** của chúng. Từ "pattern" bị overload — Saga, DDD, EDD, Repository, và GoF 23 đều nằm ở các tầng abstraction khác nhau. Dùng map này để tìm đúng tên cho đúng khái niệm.

> 🇻🇳 Phiên bản tiếng Việt. English version: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Taxonomy của "pattern" trong software, sort theo scope: methodology (DDD), architectural style (Layered, Hexagonal, Clean, EDA), architectural pattern (Saga, CQRS, Outbox), enterprise pattern (Repository, Unit of Work), DDD pattern (tactical + strategic), GoF design pattern, và idiom ngôn ngữ.
- **Tại sao** — Từ "pattern" được dùng ở 5 tầng khác. Map này ngăn so sánh Saga với Decorator như thể chúng là alternative — chúng sống trên trục khác.
- **Khi nào** — Bất cứ khi nào ai đó dùng term overloaded ("chúng tôi dùng DDD", "chúng tôi có microservices", "chúng tôi follow Repository"); navigate phần còn lại docs .NET; dạy teammate tìm gì ở đâu.
- **Ở đâu** — Entry point top-level cho `dot-net/docs/`. Mọi doc khác trong folder này reach được từ bảng taxonomy.

---

## Phân loại Pattern (Taxonomy)

| Tầng (Level)                   | Folder / File                                      | Ví dụ                                                                                                  |
| ------------------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Methodology / Practice**     | [`ddd/`](./ddd/)                                   | **DDD**, TDD, BDD, Agile                                                                              |
| **Architectural Style**        | [`architectural-style/`](./architectural-style/)   | Layered, Hexagonal, Onion, Clean, **EDA / EDD**, Microservices, REST                                  |
| **Architectural Pattern**      | [`architectural-pattern/`](./architectural-pattern/) | **Saga**, CQRS, Event Sourcing, Outbox/Inbox, Circuit Breaker, API Gateway, Strangler Fig             |
| **Enterprise Pattern**         | [`enterprise-pattern/`](./enterprise-pattern/)     | **Repository**, Unit of Work, Specification, Active Record, Data Mapper, Service Layer                |
| **Tactical DDD Pattern**       | [`ddd/tactical-patterns.md`](./ddd/tactical-patterns.md) | Aggregate, Entity, Value Object, Domain Event, Domain Service, Factory                            |
| **Strategic DDD Pattern**      | [`ddd/strategic-patterns.md`](./ddd/strategic-patterns.md) | Bounded Context, Ubiquitous Language, Context Map, Anti-Corruption Layer                          |
| **Design Pattern (GoF, 23)**   | [`design-pattern/`](./design-pattern/)             | Strategy, Observer, Decorator, Factory, Builder…                                                      |
| **Idiom / Language Technique** | (theo từng ngôn ngữ)                               | Generic Repository, RAII, Disposable, `IAsyncEnumerable<T>` streaming                                 |

**Quy tắc về scope:** càng lên cao thì phạm vi càng rộng (system → service → class → dòng code). GoF pattern nằm ở dưới cùng — chúng mô tả cách các class collaborate *bên trong một process*.

---

## Tra cứu nhanh — "X thuộc loại gì?"

| Khái niệm              | Category (Phân loại)                                            | Xem                                                       |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------- |
| **Saga**               | Architectural Pattern (distributed transaction)                  | [`architectural-pattern/saga-vi.md`](./architectural-pattern/saga-vi.md) |
| **DDD**                | Methodology (chứa nhiều sub-pattern tactical + strategic)        | [`ddd/README-vi.md`](./ddd/README-vi.md)                  |
| **EDA / EDD**          | Architectural Style                                              | [`architectural-style/event-driven-vi.md`](./architectural-style/event-driven-vi.md) |
| **CQRS**               | Architectural Pattern                                            | [`architectural-pattern/cqrs-vi.md`](./architectural-pattern/cqrs-vi.md) |
| **Event Sourcing**     | Architectural Pattern                                            | [`architectural-pattern/event-sourcing-vi.md`](./architectural-pattern/event-sourcing-vi.md) |
| **Outbox / Inbox**     | Architectural Pattern (messaging reliability)                    | [`architectural-pattern/outbox-vi.md`](./architectural-pattern/outbox-vi.md) |
| **Repository**         | Enterprise Pattern (cũng là Tactical DDD pattern)                | [`enterprise-pattern/repository-vi.md`](./enterprise-pattern/repository-vi.md) |
| **Generic Repository** | **Implementation idiom** — thường là anti-pattern trong DDD      | [`enterprise-pattern/repository-vi.md`](./enterprise-pattern/repository-vi.md) |
| **Unit of Work**       | Enterprise Pattern                                               | [`enterprise-pattern/unit-of-work-vi.md`](./enterprise-pattern/unit-of-work-vi.md) |
| **Aggregate**          | Tactical DDD Pattern                                             | [`ddd/tactical-patterns-vi.md`](./ddd/tactical-patterns-vi.md) |
| **Bounded Context**    | Strategic DDD Pattern                                            | [`ddd/strategic-patterns-vi.md`](./ddd/strategic-patterns-vi.md) |
| **Strategy / Observer / Decorator / …** | GoF Design Patterns                                 | [`design-pattern/`](./design-pattern/)                    |
| **N-Layer Architecture** | Architectural Style                                            | [`n-layer-architecture.md`](./n-layer-architecture.md)    |

---

## Folder Map

* **[`design-pattern/`](./design-pattern/)** — 23 GoF design pattern kèm sample C# .NET.
* **[`architectural-pattern/`](./architectural-pattern/)** — Pattern ở mức service / distributed system (Saga, CQRS, Event Sourcing, Outbox).
* **[`architectural-style/`](./architectural-style/)** — Lựa chọn cấu trúc toàn hệ thống (EDA, Hexagonal, Clean, Onion).
* **[`enterprise-pattern/`](./enterprise-pattern/)** — Pattern cho data access và application layer (Repository, Unit of Work, Specification).
* **[`ddd/`](./ddd/)** — Domain-Driven Design như một methodology, cùng các tactical và strategic pattern của nó.
* **[`concurrency/`](./concurrency/)** — Race condition, chiến lược locking, coordination distributed — concern concurrency cắt qua các folder khác.
* **Các file architectural-style deep-dive cấp top** (mỗi file 1 style):
  * [`n-layer-architecture.md`](./n-layer-architecture.md)
  * [`n-tier-architecture.md`](./n-tier-architecture.md)
  * [`clean-architecture.md`](./clean-architecture.md)
  * [`hexagonal-architecture.md`](./hexagonal-architecture.md)
  * [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)
  * [`architecture-comparison.md`](./architecture-comparison.md) — hướng dẫn lựa chọn

---

## Các tầng tương tác như thế nào trong 1 solution .NET thực tế

Một microservice production điển hình kết hợp pattern từ **nhiều tầng**:

* **Style** — Clean Architecture (dependency chỉ trỏ vào trong).
* **Architectural Pattern** — CQRS để tách read/write, Saga cho transaction xuyên service, Outbox để publish reliable.
* **Methodology** — DDD định hình domain model.
* **Enterprise Pattern** — Repository theo từng aggregate, Unit of Work qua `DbContext`.
* **GoF Pattern** — Strategy cho rule có thể thay đổi, Decorator cho caching, Observer cho domain event, Mediator cho command dispatch.

Không có cái nào thay thế cho cái nào — mỗi pattern giải một bài toán ở scale khác nhau.
