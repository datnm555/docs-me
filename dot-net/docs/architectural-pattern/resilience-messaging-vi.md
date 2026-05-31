# Messaging Resilience — Retry, Backoff, Jitter, DLQ

> Async messaging mặc định là **at-least-once**. Transient failure là chuyện thường. Consumer phải retry được mà không khuếch đại vấn đề, và phải có chỗ để xếp message thật sự không xử lý được.

> 🇻🇳 Phiên bản tiếng Việt. English: [`resilience-messaging.md`](./resilience-messaging.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Bốn primitive cùng nhau làm consumer chịu được transient failure: **Retry** (thử lại), **Exponential Backoff** (chờ lâu dần), **Jitter** (random hóa thời gian chờ để tản tải), **Dead Letter Queue** (để dành những message tự động không xử lý nổi).
- **Tại sao** — Broker redeliver. Downstream chập chờn. Không có backoff, retry biến thành thundering herd. Không có DLQ, một message lỗi block cả partition mãi mãi.
- **Khi nào** — Mọi async consumer (Kafka, RabbitMQ, SQS, Azure Service Bus, MassTransit). Đi cặp với [Idempotency](./idempotency.md) — retry mà không idempotent thì tạo duplicate.
- **Ở đâu** — Config phía consumer. Broker hoặc retry native (RabbitMQ delayed exchange, SQS visibility timeout, Service Bus auto-forwarding) hoặc bạn orchestrate qua library (MassTransit, Wolverine, Polly).
- **Như thế nào** — Cap số lần retry (thường 3–5). Delay = `base * 2^attempt` cộng jitter random `±50%`. Phân loại error là **transient** (retry) hay **terminal** (DLQ ngay). DLQ có alert và tool re-drive.

---

## Vấn đề

Một consumer xử lý payment event:

```csharp
public async Task Handle(PaymentReceived ev)
{
    await _email.SendAsync(ev.CustomerEmail, "Receipt...");  // gọi 3rd-party API
}
```

Ba mode lỗi nhìn giống hệt nhau với broker (message không được ack):

| Nguyên nhân                            | Nên retry?         | Bao lâu?                |
| -------------------------------------- | ------------------ | ----------------------- |
| Email API rate-limit (`429`)           | Có                 | Chờ rồi retry           |
| Email API down (`503`)                 | Có                 | Chờ lâu hơn, retry      |
| Bug — null reference trong code ta     | Không              | Sẽ fail mọi lần         |
| Bad payload — thiếu email khách hàng   | Không              | Sẽ fail mọi lần         |
| Network blip                           | Có                 | Retry ngay được         |

Không retry thông minh: 1 bad message bị redelivery mãi, block partition. Có retry **vô hạn** mà không backoff: 10k consumer đập downstream đang hồi phục và giết lại nó (thundering herd).

---

## Retry

### Retry ở đâu

| Layer                         | Cơ chế                                                       | Trade-off                                                             |
| ----------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------- |
| **Broker-native**             | RabbitMQ delayed exchange, SQS visibility timeout, Kafka pause | Survive consumer restart; durable; control thô                        |
| **In-process (Polly / MassTransit)** | Library wrap handler                                  | Control mịn; mất state khi crash nếu không cặp với broker             |
| **Application-level (state machine)** | Re-enqueue với attempt count trong payload           | Flexibility tối đa; bạn tự book-keep                                  |

Cho consumer at-least-once, **ưu tiên broker-native hoặc library-có-fallback-broker**. In-process retry một mình mất state khi crash.

### Phân loại failure

Đừng retry cái không hồi phục:

```csharp
public enum FailureMode { Transient, Terminal }

public static FailureMode Classify(Exception ex) => ex switch
{
    HttpRequestException { StatusCode: HttpStatusCode.TooManyRequests } => FailureMode.Transient,
    HttpRequestException { StatusCode: >= HttpStatusCode.InternalServerError } => FailureMode.Transient,
    TimeoutException => FailureMode.Transient,
    SocketException => FailureMode.Transient,

    ValidationException => FailureMode.Terminal,
    JsonException       => FailureMode.Terminal,
    ArgumentException   => FailureMode.Terminal,

    _ => FailureMode.Transient,  // default — thận trọng
};
```

Terminal failure đi **thẳng vào DLQ**. Đừng phí 5 retry cho 1 bug.

---

## Backoff & Jitter

### Fixed delay → thundering herd

```
attempt 1: chờ 1s
attempt 2: chờ 1s
attempt 3: chờ 1s
```

10k consumer fail cùng lúc → 10k retry đánh service đang hồi phục cùng giây → nó chết lại.

### Exponential backoff

```
attempt 1: chờ 1s
attempt 2: chờ 2s
attempt 3: chờ 4s
attempt 4: chờ 8s
attempt 5: chờ 16s
```

Tốt hơn — tổng thời gian chờ tăng nhanh, downstream có không khí thở. Nhưng client đồng bộ vẫn dồn vào cùng các mốc thời gian.

### Exponential backoff + jitter

```
delay = base * 2^attempt
jitter = random(0, delay * 0.5)        // "full jitter" hoặc "decorrelated jitter"
wait   = delay + jitter
```

Retry trải đều theo thời gian. Đây là mặc định của AWS SDK.

> **Quy tắc:** không bao giờ retry mà không có jitter ở scale lớn. Toán đơn giản: `N` client fail tại `t=0`, fixed backoff retry cùng dồn vào instant tiếp theo; jittered backoff trải qua delay window. Trải = phục hồi.

---

## Dead Letter Queue

Sau `N` retry (thường 3–5), message là **poison** — cả ta và downstream đều không làm cho nó chạy được.

```
                    fail N lần
   [Main Queue] ─────────────────► [Retry Queue] ─── delay ───┐
        ▲                                                       │
        │                                                       ▼
        └────────── delivery (re-enqueue) ────────────────  [Consumer]
                                                                │
                              không hồi phục sau N lần           │
                                                                ▼
                                                          [Dead Letter Queue]
                                                                │
                                                                ▼
                                                  ┌── Alert (PagerDuty/Slack)
                                                  ├── Inspection thủ công
                                                  └── Re-drive tool (replay về main)
```

DLQ là **operational tooling**, không chỉ là 1 flag config:

* **Alerting** — depth DLQ > 0 → page ai đó. DLQ im lặng là black hole.
* **Inspection** — UI / CLI xem payload, lý do fail, lịch sử retry.
* **Re-drive** — sau khi bug fix, push message về main queue.
* **Retention** — message DLQ cũng expire (mặc định 14 ngày ở hầu hết broker). Quá đó: điều tra hoặc archive.

---

## Implementation C# — MassTransit

MassTransit ship default hợp lý cho cả 4 primitive:

```csharp
services.AddMassTransit(x =>
{
    x.AddConsumer<PaymentReceivedConsumer>(cfg =>
    {
        cfg.UseMessageRetry(r => r
            .Exponential(
                retryLimit:    5,
                minInterval:   TimeSpan.FromSeconds(1),
                maxInterval:   TimeSpan.FromMinutes(5),
                intervalDelta: TimeSpan.FromSeconds(2))
            .Handle<HttpRequestException>()
            .Handle<TimeoutException>()
            .Ignore<ValidationException>());

        // Hết retry, MassTransit gửi vào queue `_error` (DLQ của nó).
    });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.UseDelayedMessageScheduler();   // cần cho delayed redelivery
        cfg.ConfigureEndpoints(ctx);
    });
});
```

DLQ poison-message tường minh:

```csharp
cfg.ReceiveEndpoint("payment-received", e =>
{
    e.UseMessageRetry(r => r.Exponential(...));
    e.UseInMemoryOutbox();              // dedup in-process
    e.DiscardFaultedMessages = false;    // giữ faulted msgs → queue _error
    e.ConfigureConsumer<PaymentReceivedConsumer>(ctx);
});
```

Jitter tay nếu library không có:

```csharp
public static TimeSpan WithJitter(TimeSpan baseDelay, double factor = 0.5)
{
    var jitterMs = Random.Shared.NextDouble() * baseDelay.TotalMilliseconds * factor;
    return baseDelay + TimeSpan.FromMilliseconds(jitterMs);
}
```

---

## Real Project — Email xác nhận Order

E-commerce. Sau khi `OrderPlaced` được publish, notification service email khách hàng. SendGrid thỉnh thoảng rate-limit; có khi email khách bounce (terminal).

```csharp
public sealed class OrderPlacedConsumer : IConsumer<OrderPlaced>
{
    private readonly ISendGridClient _sendgrid;
    private readonly AppDbContext _db;
    private readonly ILogger<OrderPlacedConsumer> _log;

    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        // Idempotency — inbox check
        if (await _db.Inbox.AnyAsync(i => i.Id == ctx.MessageId, ctx.CancellationToken))
            return;

        try
        {
            var email = BuildReceiptEmail(ctx.Message);
            var response = await _sendgrid.SendEmailAsync(email, ctx.CancellationToken);

            if ((int)response.StatusCode == 429)
                throw new HttpRequestException("Rate limited", null, HttpStatusCode.TooManyRequests);
            if ((int)response.StatusCode >= 500)
                throw new HttpRequestException("Provider error", null, HttpStatusCode.ServiceUnavailable);
            if ((int)response.StatusCode == 400)
                throw new ValidationException("Recipient không hợp lệ — sai format email");

            _db.Inbox.Add(new InboxMessage { Id = ctx.MessageId.Value, HandledOnUtc = DateTime.UtcNow });
            await _db.SaveChangesAsync(ct: ctx.CancellationToken);
        }
        catch (ValidationException ex)
        {
            // Terminal — log và KHÔNG re-throw để MassTransit không retry. Vào DLQ qua Fault<T>.
            _log.LogWarning(ex, "Order {OrderId} không hợp lệ — gửi DLQ", ctx.Message.OrderId);
            throw;   // policy MassTransit ignore ValidationException → thẳng _error
        }
    }
}
```

Pipeline:

* SendGrid 429 → `HttpRequestException` → retry với backoff (1s, 2s, 4s, 8s, 16s).
* SendGrid 503 → cùng path.
* Email khách sai format → `ValidationException` → không retry → DLQ.
* Bug trong code ta → exception unknown → policy retry default (transient) → 5 retry → DLQ.
* Hết retry → message vào queue `_error` → on-call được page → engineer review rồi re-drive hoặc discard.

---

## Pitfall thường gặp

* **Retry vô hạn** — không cap, 1 poison message ăn CPU mãi. Luôn cap 3–10 lần.
* **Không jitter** — ở scale lớn, retry đồng bộ giết service đang hồi phục. Luôn jitter.
* **Retry terminal error** — null ref và validation fail không tự sửa. Phân loại trước khi retry.
* **Không alert DLQ** — message rot im lặng trong dead letter trong khi incident chồng chất.
* **DLQ làm thùng tái chế** — re-drive mà không fix bug chỉ replay failure. Coi entry DLQ là bug cần triage.
* **Retry không idempotent** — broker redelivery + handler retry = duplicate side effect. Luôn cặp với [Idempotency](./idempotency.md) / Inbox.
* **Backoff đo bằng wall clock** — visibility timeout và lease window có clock riêng. Backoff phải fit trong đó nếu không sẽ bị redelivery **đồng thời**.

---

## Tham khảo

* [AWS Architecture Blog — Exponential backoff and jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
* [Polly resilience library](https://www.pollydocs.org/)
* [MassTransit — Exception handling](https://masstransit.io/documentation/concepts/exceptions)
* Sam Newman — *Building Microservices* (2nd ed.), Chapter 12.
* Michael Nygard — *Release It!* (2nd ed.), Stability Patterns.
