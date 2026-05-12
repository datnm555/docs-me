# Architectural Style trong .NET

> Architectural **style** định nghĩa **hình dạng tổng thể** của hệ thống — component được tổ chức ra sao, dependency trỏ về đâu, communication như thế nào. Style lớn hơn pattern và đặt luật chơi cho các pattern bên trong.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Style vs. Pattern vs. Design Pattern

| Khái niệm                  | Phạm vi                | Ví dụ                                                          |
| -------------------------- | ---------------------- | -------------------------------------------------------------- |
| **Architectural Style**    | Toàn hệ thống          | Layered, Hexagonal, Onion, Clean, **EDA**, Microservices, REST  |
| **Architectural Pattern**  | Một service / workflow | Saga, CQRS, Event Sourcing, Outbox, Circuit Breaker             |
| **GoF Design Pattern**     | Class / object         | Strategy, Observer, Decorator, …                               |

Hệ thống có **một hoặc hai style** (vd. "Clean Architecture + EDA"), dùng **nhiều pattern**, và chứa **rất nhiều design pattern**.

---

## Index

| Style                              | File / Doc                                                       |
| ---------------------------------- | ---------------------------------------------------------------- |
| **N-Layer (Layered)**              | [`n-layer-architecture.md`](./n-layer-architecture.md)            |
| **N-Tier (deployment vật lý)**     | [`n-tier-architecture.md`](./n-tier-architecture.md)              |
| **Clean Architecture**             | [`clean-architecture.md`](./clean-architecture.md)                |
| **Hexagonal (Ports & Adapters)**   | [`hexagonal-architecture.md`](./hexagonal-architecture.md)        |
| **Vertical Slice**                 | [`vertical-slice-architecture.md`](./vertical-slice-architecture.md) |
| **Clean Slice** (hybrid)           | [`clean-slice-architecture.md`](./clean-slice-architecture.md)    |
| **Hexagonal vs. Onion vs. Clean** *(so sánh)* | [`hexagonal-onion-clean-vi.md`](./hexagonal-onion-clean-vi.md) |
| **Hướng dẫn chọn style**           | [`architecture-comparison.md`](./architecture-comparison.md)      |
| **Five Architectures — So sánh & Mixing** | [`five-architectures-comparison-and-mixing.md`](./five-architectures-comparison-and-mixing.md) |
| **Event-Driven (EDA / EDD)**       | [`event-driven-vi.md`](./event-driven-vi.md)                      |
| Microservices                      | *(chưa mở rộng — xem sách của Chris Richardson)*                  |
| REST / RPC / gRPC                  | *(communication style)*                                          |
| Reactive                           | *(chưa mở rộng — xem ReactiveX, Akka.NET)*                        |

---

## Chọn Style như thế nào

Quyết định dựa trên **cái gì thay đổi nhiều nhất** và **tối ưu cho cái gì**:

| Nếu ưu tiên…                                                | Nghiêng về…                                |
| ----------------------------------------------------------- | ------------------------------------------ |
| Đơn giản, team nhỏ, app CRUD                                | **N-Layer** (Layered)                      |
| Domain testable, dễ thay infra, business rule phức tạp      | **Hexagonal / Clean / Onion**              |
| Coupling lỏng, scale subsystem độc lập                      | **Microservices** + **EDA**                |
| Real-time, audit-heavy, domain giàu lịch sử                 | **EDA + Event Sourcing**                   |
| High-throughput streaming, push-based                       | **Reactive**                               |

**Có thể kết hợp style:** một Clean Architecture service bên trong hệ thống EDA microservices là phổ biến và idiomatic.

---

## Cách style kết hợp với pattern

```
Style       :  Clean Architecture  +  EDA
                        │
                        ▼
Pattern     :  CQRS, Saga, Outbox, Repository
                        │
                        ▼
Design      :  Strategy, Decorator, Observer (events), Mediator
```

Lựa chọn ở mỗi tầng ràng buộc tầng kế tiếp.
