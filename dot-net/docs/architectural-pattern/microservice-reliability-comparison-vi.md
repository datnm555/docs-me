# Microservice Reliability Patterns — So sánh & Hướng dẫn Chọn

> Bản đồ các reliability pattern trong folder này — Saga, 2PC, TCC, Outbox, Inbox, Idempotency, Retry/Backoff/DLQ, Circuit Breaker, Bulkhead, Timeout — sắp xếp theo **vấn đề chúng giải** và **cách compose**. Không cái nào là alternative riêng lẻ; service production dùng stack nhiều layer của chúng.

> 🇻🇳 Phiên bản tiếng Việt. English: [`microservice-reliability-comparison.md`](./microservice-reliability-comparison.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Hướng dẫn chọn và decision matrix cho việc chọn đúng reliability pattern cho đúng failure mode trong microservice. Cross-reference mọi pattern trong `dot-net/docs/architectural-pattern/` cộng với câu hỏi kinh điển "thế còn 2PC?".
- **Tại sao** — Các pattern này hay bị nhầm là alternative của nhau. Không phải. Saga không "thay" Idempotency; Circuit Breaker không "thay" Retry. Biết mỗi cái thuộc đâu giúp bạn không over-engineer hay bỏ trống 1 layer.
- **Khi nào** — Design microservice mới; review postmortem ("layer nào fail?"); quyết định thêm pattern mới hay sửa cái có; giải thích trade-off với team.
- **Ở đâu** — `dot-net/docs/architectural-pattern/` — mọi entry trong [Index](./README.md#index) được reference ở đây. Cross-link cũng vào `architectural-style/event-driven.md` và `ddd/`.
- **Như thế nào** — Xác định failure mode trước (commit coordination? at-least-once delivery? downstream chậm?). Match nó với pattern layer dưới. Hầu hết service cần ít nhất 1 pattern từ mỗi layer.

---

## Bốn Layer của Reliability

Mọi microservice nói chuyện với microservice khác phải trả lời **bốn** câu hỏi độc lập. Outage thường xảy ra vì 1 trong 4 câu bị bỏ trống.

```
┌───────────────────────────────────────────────────────────────────────────┐
│ Layer 4 — Synchronous resilience                                          │
│   "Đừng để downstream chậm giết ta."                                       │
│   ► Timeout · Circuit Breaker · Bulkhead                                  │
├───────────────────────────────────────────────────────────────────────────┤
│ Layer 3 — Messaging resilience                                            │
│   "Đừng để transient failure hay poison message kẹt ta."                  │
│   ► Retry · Backoff+Jitter · DLQ                                          │
├───────────────────────────────────────────────────────────────────────────┤
│ Layer 2 — Delivery semantics                                              │
│   "Làm cho duplicate delivery vô hại."                                     │
│   ► Idempotency · Inbox                                                   │
├───────────────────────────────────────────────────────────────────────────┤
│ Layer 1 — Write coordination                                              │
│   "Làm state nhất quán xuyên service."                                     │
│   ► Saga · TCC · (2PC, gần như tránh) · Outbox · Event Sourcing           │
└───────────────────────────────────────────────────────────────────────────┘
```

Các layer **độc lập**. Bạn có Saga hoàn hảo vẫn mất data nếu service tham gia không có Outbox. Outbox + Inbox hoàn hảo vẫn dính thundering-herd outage nếu consumer không có backoff. Mỗi layer fail riêng và phải xử lý riêng.

---

## Bản đồ Pattern theo Vấn đề

### Layer 1 — Coordinate write xuyên service

| Vấn đề                                                                           | Pattern                                            | Tại sao                                                                                   |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Atomic write xuyên service với strict ACID                                       | **2PC** ([`two-phase-commit-tcc.md`](./two-phase-commit-tcc.md)) | Chỉ khả thi trong scenario single-vendor / same-process. Block khi failure. **Tránh cho microservice.** |
| Long-running workflow với rollback ngữ nghĩa                                     | **Saga** ([`saga.md`](./saga.md))                  | Local TX + compensating action. Default cho distributed transaction trong microservice.   |
| Business model "reserve rồi confirm" (travel, ticketing)                         | **TCC** ([`two-phase-commit-tcc.md`](./two-phase-commit-tcc.md)) | API 3 pha cho mỗi service. Nhẹ hơn 2PC, chặt hơn Saga.                                  |
| Atomic (1) write DB và (2) publish message                                        | **Outbox** ([`outbox.md`](./outbox.md))            | Dual-write problem. Bắt buộc cho at-least-once messaging.                                 |
| Cần audit trail hoàn hảo; cần rebuild state từ history                            | **Event Sourcing** ([`event-sourcing.md`](./event-sourcing.md)) | Event là source of truth, publish free, audit free. Trade-off lớn.                    |

### Layer 2 — Delivery semantics

| Vấn đề                                                                           | Pattern                                            | Tại sao                                                                                   |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| HTTP retry gây duplicate side effect (`POST /payments`)                          | **Idempotency Key** ([`idempotency.md`](./idempotency.md)) | Server dedup theo key do client cung cấp.                                          |
| Message broker at-least-once redeliver cùng message                              | **Inbox** ([`outbox.md`](./outbox.md))             | Consumer record message ID đã xử lý. Đối xứng với Outbox.                                |

Hai cái này là cùng 1 ý tưởng áp dụng ở 2 boundary khác: HTTP edge vs. message consumer. Không có chúng, mọi retry ở Layer 3 là duplicate bug.

### Layer 3 — Messaging resilience (async)

| Vấn đề                                                                           | Pattern                                            | Tại sao                                                                                   |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Transient downstream failure (rate limit, 5xx, network blip)                     | **Retry + Exponential Backoff + Jitter** ([`resilience-messaging.md`](./resilience-messaging.md)) | Sống sót blip mà không thundering herd. |
| Message mãi không xử lý được (validation, schema, bug)                            | **Dead Letter Queue** ([`resilience-messaging.md`](./resilience-messaging.md)) | Park poison, page on-call, re-drive sau khi fix.                  |

### Layer 4 — Synchronous resilience (sync)

| Vấn đề                                                                           | Pattern                                            | Tại sao                                                                                   |
| -------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Downstream chậm giữ thread / connection của ta                                   | **Timeout** ([`resilience-sync.md`](./resilience-sync.md))    | Hard deadline. Bắt buộc cho mọi cross-boundary call.                                |
| Downstream fail liên tục — ngừng gọi                                              | **Circuit Breaker** ([`resilience-sync.md`](./resilience-sync.md)) | Fail-fast + half-open probe. Cặp với fallback.                                  |
| 1 dependency chậm làm cạn cả service                                              | **Bulkhead** ([`resilience-sync.md`](./resilience-sync.md))   | Pool thread / connection riêng cho mỗi dependency.                                   |

---

## Decision Matrix

### "Chọn 1 transaction strategy"

| Nếu vấn đề là…                                                               | Dùng                                | Đừng dùng                       |
| ----------------------------------------------------------------------------- | ----------------------------------- | ------------------------------- |
| 1 service, 2 local DB                                                         | Local TX hoặc MSDTC                 | Saga (overkill)                 |
| Nhiều service, eventual consistency OK                                        | **Saga**                            | 2PC, TCC                        |
| Nhiều service, business *vốn* nghĩa "reserve rồi confirm"                    | **TCC**                             | Saga thuần (mất reservation semantics) |
| Nhiều service, phải strict ACID                                               | Đánh giá lại boundary; hoặc 2PC trong cùng vendor | Greenfield 2PC xuyên service |
| Domain audit-heavy, rebuild history quan trọng                                 | **Event Sourcing** + Saga           | Saga thuần (mất history)         |

### "Chọn reliability primitive" (mọi service đều cần)

| Failure mode                                       | Sync (HTTP/gRPC)            | Async (queue/topic)                                |
| -------------------------------------------------- | --------------------------- | -------------------------------------------------- |
| Duplicate request / message                        | Idempotency Key             | Inbox                                              |
| Transient failure                                  | Retry + backoff + jitter    | Retry + backoff + jitter                           |
| Persistent failure                                 | Circuit Breaker + fallback  | DLQ + alert + re-drive                             |
| Slowness                                           | Timeout, Bulkhead           | Visibility timeout / lease                         |
| Atomic write + publish                             | n/a                         | **Outbox**                                         |
| Coordinator fail giữa workflow                     | Stateless retry             | Durable saga state (event-sourced hoặc DB-persisted) |

### "Chọn đúng pattern khi X cám dỗ bạn"

| Bạn bị cám dỗ…                                                | Nhưng bạn thật sự cần…                                          |
| ------------------------------------------------------------- | --------------------------------------------------------------- |
| Dùng 2PC xuyên microservice                                   | Saga (hoặc nghĩ lại boundary)                                    |
| Chỉ publish broker sau `SaveChangesAsync`                      | Outbox                                                           |
| Chỉ check "đã xử lý chưa?" trong memory                       | Inbox (durable, transactional)                                   |
| Retry mãi đến khi work                                         | Cap + backoff + DLQ                                              |
| Catch all exception rồi retry                                  | Phân loại transient vs terminal, DLQ terminal                    |
| Set `HttpClient.Timeout = TimeSpan.FromMinutes(5)`             | Per-attempt timeout thật size theo p99 downstream                |
| Thêm circuit breaker vì call thỉnh thoảng fail                  | Confirm có fallback trước; không thì breaker chỉ đổi error      |
| Share `HttpClient` giữa Pricing và Catalog                     | Named client riêng cho mỗi downstream + bulkhead                |

---

## Cách các Pattern Compose — E-Commerce Checkout End-to-End

1 use case "Place Order" duy nhất chạy qua mọi layer:

```
                          ┌─────────────────────────────────────────────┐
                          │  Checkout Service                           │
                          │                                             │
HTTP  POST /orders ──────►│  [Idempotency middleware] (Layer 2)         │
   Idempotency-Key: K     │      │                                      │
                          │      ▼                                      │
                          │  Validate cart                              │
                          │      │                                      │
                          │      ▼                                      │
                          │  Call Pricing  (Layer 4: Timeout +          │
                          │      │          Circuit Breaker + Bulkhead) │
                          │      ▼                                      │
                          │  Start Saga (Layer 1)                        │
                          │      │                                      │
                          │      ▼                                      │
                          │  Local TX:                                  │
                          │    • Save Order                             │
                          │    • Save Outbox row OrderPlaced (Layer 1)  │
                          │      │                                      │
                          │      ▼                                      │
                          │  Outbox Publisher ── publish → [Broker]     │
                          └─────────────────────────────────────────────┘
                                                            │
                                                            │ at-least-once
                                                            ▼
                          ┌─────────────────────────────────────────────┐
                          │  Payment Service                            │
                          │      │                                      │
                          │      ▼                                      │
                          │  [Inbox check] (Layer 2)                    │
                          │      │                                      │
                          │      ▼                                      │
                          │  Charge qua Stripe (Layer 4 trên Stripe call)│
                          │      │ fail 429? → Retry+backoff+jitter (Layer 3)
                          │      │ fail terminal? → DLQ (Layer 3)        │
                          │      ▼                                      │
                          │  Local TX:                                  │
                          │    • Save Payment                           │
                          │    • Save Inbox row                         │
                          │    • Save Outbox row PaymentCompleted        │
                          └─────────────────────────────────────────────┘
                                                            │
                                                            ▼
                                                     [Broker] … saga continues
```

Cái gì bảo vệ mỗi bước:

* **Layer 1** — Outbox đảm bảo `Order` và event `OrderPlaced` được ghi atomic. Saga sequence `OrderPlaced → ChargePayment → ReserveStock → ShipOrder` với compensation nếu step fail.
* **Layer 2** — Idempotency middleware đảm bảo client retry `POST /orders` không bị double-charge. Inbox trong PaymentService dedup `OrderPlaced` redelivered.
* **Layer 3** — Retry+backoff+jitter xử lý Stripe rate-limit. DLQ bắt response Stripe sẽ không bao giờ thành công (card metadata invalid).
* **Layer 4** — Timeout cap mỗi call Pricing/Stripe. Circuit Breaker mở nếu Pricing liên tục chết → checkout fallback giá last-known. Bulkhead cô lập Pricing client để chậm của nó không cạn Stripe call.

Bỏ 1 layer = 1 lớp outage:

| Bỏ đi                  | Loại outage                                                                  |
| ---------------------- | ---------------------------------------------------------------------------- |
| Outbox                 | Mất event — Order saved nhưng downstream không biết. Im lặng.                |
| Idempotency / Inbox    | Duplicate charge, duplicate order.                                            |
| Retry+backoff          | 1 blip = mất message. Không jitter = thundering herd lúc phục hồi.            |
| DLQ                    | 1 poison message kẹt partition.                                               |
| Timeout                | Pricing chậm → thread checkout cạn → 503 storm.                              |
| Circuit Breaker        | Call lặp lại Pricing chết → timeout 5s lặp lại đau UX.                       |
| Bulkhead               | Pricing chậm leak vào Stripe call latency.                                    |

---

## Anti-Pattern: Lỗ Hổng Nhìn Thấy

* **Dùng 1 pattern thay layer khác.** "Có Retry rồi, không cần Circuit Breaker." Retry không breaker là thundering herd.
* **"Distributed transaction khó, thôi không coordinate."** Mất order không phải "trade-off." Là bug. Dùng Saga + Outbox.
* **Choreography saga + at-least-once + không idempotent.** Duplicate sẽ xảy ra. Luôn luôn.
* **Không timeout vì "thêm sau."** "Sau" là postmortem.
* **DLQ không alert.** Black hole. Bug invisible đến khi khách phàn nàn.
* **Idempotency key là hash request body.** Whitespace khác trông như key khác. Dùng UUID tường minh client cung cấp.

---

## Tham khảo

* Chris Richardson — *Microservices Patterns* (2018). Map kinh điển các pattern này vào microservice.
* Michael Nygard — *Release It!* (2nd ed.). Stability Patterns — Timeout, Circuit Breaker, Bulkhead, Fail Fast.
* Sam Newman — *Building Microservices* (2nd ed.). Chapter 12 (Resilience), Chapter 6 (Workflow).
* Gregor Hohpe, Bobby Woolf — *Enterprise Integration Patterns* (2003). Inbox, Outbox, Idempotent Receiver, Dead Letter Channel.
* Pat Helland — [*Life beyond Distributed Transactions: an Apostate's Opinion*](https://queue.acm.org/detail.cfm?id=3025012). Tại sao ta ngừng dùng 2PC.
* [microservices.io/patterns](https://microservices.io/patterns/) — catalog pattern cross-reference xuyên suốt.
