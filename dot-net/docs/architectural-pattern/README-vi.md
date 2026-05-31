# Architectural Pattern trong .NET

> Các pattern hoạt động ở **mức service / distributed system** — to hơn class-level pattern của GoF, nhỏ hơn architectural style. Chúng giải quyết các bài toán lặp lại trong microservices, messaging, data consistency, và resilience.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Pattern hoạt động ở mức service / distributed-system — Saga, CQRS, Event Sourcing, Outbox/Inbox, cộng note ngắn về Circuit Breaker, API Gateway, Strangler Fig, Sidecar.
- **Tại sao** — Giải các vấn đề lặp lại của microservices: distributed transaction, tách read/write, publish-and-save atomic, resilience.
- **Khi nào** — Build hoặc split hệ thống microservices; introduce messaging event-driven; cần workflow cross-service reliable; thay 2PC.
- **Ở đâu** — Giữa architectural style (Hexagonal, EDA — `architectural-style/`) và pattern cấp class (GoF — `design-pattern/`). Thường implement dùng Repository (`enterprise-pattern/`) và DDD aggregate (`ddd/`).

---

## "Architectural Pattern" là gì?

* **Scope:** một service, một workflow xuyên nhiều service, hoặc một mảng infrastructure dùng chung.
* **Mức độ chi tiết:** to hơn class collaboration (GoF), nhỏ hơn một style toàn hệ thống (Hexagonal, EDA).
* **Nguồn chuẩn:** Chris Richardson — *Microservices Patterns* (2018); Gregor Hohpe & Bobby Woolf — *Enterprise Integration Patterns* (2003); Microsoft — *Cloud Design Patterns*.

---

## Index

### Distributed transaction & coordinate state

| Pattern               | Bài toán nó giải                                                                              | File                                                  |
| --------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Saga**              | Transaction phân tán xuyên nhiều microservices (không dùng 2PC).                              | [`saga-vi.md`](./saga-vi.md)                          |
| **2PC / TCC**         | Cross-service write strong-consistency — và tại sao microservice tránh 2PC.                   | [`two-phase-commit-tcc-vi.md`](./two-phase-commit-tcc-vi.md) |
| **CQRS**              | Tải đọc và ghi có shape và yêu cầu scaling khác nhau.                                          | [`cqrs-vi.md`](./cqrs-vi.md)                          |
| **Event Sourcing**    | Cần audit trail hoàn hảo và khả năng rebuild state từ lịch sử.                                | [`event-sourcing-vi.md`](./event-sourcing-vi.md)      |
| **Outbox / Inbox**    | Commit DB và publish message một cách atomic (giải bài "dual-write problem").                 | [`outbox-vi.md`](./outbox-vi.md)                      |

### Delivery semantics & resilience

| Pattern               | Bài toán nó giải                                                                              | File                                                  |
| --------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Idempotency**       | Retry an toàn — thực thi lặp lại = cùng effect. Nền tảng cho mọi resilience pattern khác.     | [`idempotency-vi.md`](./idempotency-vi.md)            |
| **Retry / Backoff / Jitter / DLQ** | Transient failure ở async consumer — và park message không thể xử lý.            | [`resilience-messaging-vi.md`](./resilience-messaging-vi.md) |
| **Timeout / Circuit Breaker / Bulkhead** | Sync call xuyên boundary — fail fast và sống sót.                          | [`resilience-sync-vi.md`](./resilience-sync-vi.md)    |

### Hướng dẫn chọn

| Doc                                         | Dùng khi                                                                                       |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| [So sánh Reliability Microservice](./microservice-reliability-comparison-vi.md) | Bạn đang chọn *những* pattern nào dùng, hoặc audit service xem thiếu layer nào. |

## Các Architectural Pattern phổ biến khác (chưa mở rộng)

* **API Gateway / BFF** — entry point duy nhất cho từng loại client.
* **Strangler Fig** — thay thế dần legacy system.
* **Sidecar** — co-deploy các concern cross-cutting (Envoy, Dapr).
* **Materialized View** — pre-compute read model.

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
