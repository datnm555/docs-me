# Architectural Pattern trong .NET

> Các pattern hoạt động ở **mức service / distributed system** — to hơn class-level pattern của GoF, nhỏ hơn architectural style. Chúng giải quyết các bài toán lặp lại trong microservices, messaging, data consistency, và resilience.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## "Architectural Pattern" là gì?

* **Scope:** một service, một workflow xuyên nhiều service, hoặc một mảng infrastructure dùng chung.
* **Mức độ chi tiết:** to hơn class collaboration (GoF), nhỏ hơn một style toàn hệ thống (Hexagonal, EDA).
* **Nguồn chuẩn:** Chris Richardson — *Microservices Patterns* (2018); Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003); Microsoft — *Cloud Design Patterns*.

---

## Index

| Pattern               | Bài toán nó giải                                                                              | File                                                  |
| --------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Saga**              | Transaction phân tán xuyên nhiều microservices (không dùng 2PC).                              | [`saga-vi.md`](./saga-vi.md)                          |
| **CQRS**              | Tải đọc và ghi có shape và yêu cầu scaling khác nhau.                                          | [`cqrs-vi.md`](./cqrs-vi.md)                          |
| **Event Sourcing**    | Cần audit trail hoàn hảo và khả năng rebuild state từ lịch sử.                                | [`event-sourcing-vi.md`](./event-sourcing-vi.md)      |
| **Outbox / Inbox**    | Commit DB và publish message một cách atomic (giải bài "dual-write problem").                 | [`outbox-vi.md`](./outbox-vi.md)                      |

## Các Architectural Pattern phổ biến khác (chưa mở rộng)

* **Circuit Breaker** — fail fast khi downstream không khoẻ. Dùng Polly.
* **Retry / Timeout / Bulkhead** — primitive về resilience. Dùng Polly / `Microsoft.Extensions.Resilience`.
* **API Gateway / BFF** — entry point duy nhất cho từng loại client.
* **Strangler Fig** — thay thế dần legacy system.
* **Sidecar** — co-deploy các concern cross-cutting (Envoy, Dapr).
* **Materialized View** — pre-compute read model.
* **Idempotent Receiver** — xử lý duplicate an toàn (thường qua Inbox).

---

## Cách chúng liên hệ với nhau

```
                       ┌──────────────────────┐
                       │   Architectural      │
                       │      Style           │  Hexagonal / EDA / Microservices
                       └──────────┬───────────┘
                                  │ dùng (uses)
                                  ▼
                       ┌──────────────────────┐
                       │  Architectural       │
                       │      Pattern         │  Saga / CQRS / Outbox / Circuit Breaker
                       └──────────┬───────────┘
                                  │ implement bằng
                                  ▼
                       ┌──────────────────────┐
                       │  Enterprise +        │
                       │   GoF Patterns       │  Repository / Strategy / Decorator…
                       └──────────────────────┘
```

Một microservice dùng **EDA** (style) có thể implement **Saga** (pattern) dùng **Mediator + Strategy** (GoF) và **Outbox** (pattern) dựa trên **Repository** (enterprise pattern).
