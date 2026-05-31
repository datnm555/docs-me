# Messaging Resilience — Retry, Backoff, Jitter, DLQ

> Async messaging is **at-least-once** by default. Transient failures are normal. A consumer must be able to retry without amplifying the problem and have a place to put messages it genuinely cannot handle.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — Four primitives that together make message consumers robust against transient failure: **Retry** (try again), **Exponential Backoff** (wait longer each time), **Jitter** (randomize the wait to spread load), and **Dead Letter Queue** (set aside what we can't fix automatically).
- **Why** — Brokers redeliver. Downstreams flap. Without backoff, retries become a thundering herd. Without DLQ, one bad message blocks the whole partition forever.
- **When** — Every async consumer (Kafka, RabbitMQ, SQS, Azure Service Bus, MassTransit). Pair with [Idempotency](./idempotency.md) — retry without idempotency causes duplicates.
- **Where** — Consumer-side configuration. Brokers either retry natively (RabbitMQ delayed exchange, SQS visibility timeout, Service Bus auto-forwarding) or you orchestrate via library (MassTransit, Wolverine, Polly).
- **How** — Cap retries (typically 3–5). Delay = `base * 2^attempt` with random jitter `±50%`. Mark errors as **transient** (retry) vs **terminal** (DLQ immediately). DLQ has alerts and a re-drive tool.

---

## The Problem

A consumer that processes payment events:

```csharp
public async Task Handle(PaymentReceived ev)
{
    await _email.SendAsync(ev.CustomerEmail, "Receipt...");  // calls a 3rd-party API
}
```

Three failure modes that look identical to the broker (the message wasn't acked):

| Cause                                  | Should retry?      | How long?               |
| -------------------------------------- | ------------------ | ----------------------- |
| Email API rate-limited (`429`)         | Yes                | Wait, then retry        |
| Email API down (`503`)                 | Yes                | Wait longer, retry      |
| Bug — null reference in our code       | No                 | Will fail every retry  |
| Bad payload — customer email missing   | No                 | Will fail every retry  |
| Network blip                           | Yes                | Immediate retry OK      |

Without smart retry: a single bad message gets redelivered forever, blocking the partition. With **infinite** retry of a recoverable failure but no backoff: 10k consumers hammer a recovering API and re-kill it (thundering herd).

---

## Retry

### Where retry happens

| Layer                         | Mechanism                                                    | Trade-off                                                             |
| ----------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------- |
| **Broker-native**             | RabbitMQ delayed exchange, SQS visibility timeout, Kafka pause | Survives consumer restart; durable; coarse control                    |
| **In-process (Polly / MassTransit)** | Library wraps the handler                              | Fine-grained control; lost on crash unless paired with broker         |
| **Application-level (state machine)** | Re-enqueue with attempt count in the payload          | Maximum flexibility; you own the bookkeeping                          |

For at-least-once consumers, **prefer broker-native or library-with-broker-fallback**. In-process retry alone loses state on crash.

### Classify the failure

Don't retry what won't recover:

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

    _ => FailureMode.Transient,  // default — be conservative
};
```

Terminal failures go **straight to DLQ**. Don't waste 5 retries on a bug.

---

## Backoff & Jitter

### Fixed delay → thundering herd

```
attempt 1: wait 1s
attempt 2: wait 1s
attempt 3: wait 1s
```

10k consumers fail at the same time → 10k retries hit the recovering service simultaneously → it dies again.

### Exponential backoff

```
attempt 1: wait 1s
attempt 2: wait 2s
attempt 3: wait 4s
attempt 4: wait 8s
attempt 5: wait 16s
```

Better — total wait grows quickly, downstream gets breathing room. But synchronized clients still pile up at the same moments.

### Exponential backoff + jitter

```
delay = base * 2^attempt
jitter = random(0, delay * 0.5)        // "full jitter" or "decorrelated jitter"
wait   = delay + jitter
```

Now retries spread across time. This is what AWS's SDK does by default.

> **Rule:** never retry without jitter at scale. The math: with `N` clients failing at `t=0`, fixed backoff retries all hit the next instant; jittered backoff spreads them across the delay window. Spread = recovery.

---

## Dead Letter Queue

After `N` retries (typically 3–5), the message is **poison** — neither us nor the downstream is going to make it work.

```
                    fails N times
   [Main Queue] ─────────────────► [Retry Queue] ─── delay ───┐
        ▲                                                       │
        │                                                       ▼
        └────────── delivery (re-enqueue) ────────────────  [Consumer]
                                                                │
                              cannot recover after N attempts   │
                                                                ▼
                                                          [Dead Letter Queue]
                                                                │
                                                                ▼
                                                  ┌── Alert (PagerDuty/Slack)
                                                  ├── Manual inspection
                                                  └── Re-drive tool (replay to main)
```

DLQ is **operational tooling**, not just a config flag:

* **Alerting** — depth of DLQ > 0 → page someone. A silent DLQ is a black hole.
* **Inspection** — a UI / CLI to view the payload, the failure reason, the retry history.
* **Re-drive** — once the bug is fixed, push messages back to the main queue.
* **Retention** — DLQ messages expire too (default 14d on most brokers). Beyond that, investigate or archive.

---

## C# Implementation — MassTransit

MassTransit ships sensible defaults for all four primitives:

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

        // After retries are exhausted, MassTransit sends to `_error` queue (its DLQ).
    });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.UseDelayedMessageScheduler();   // needed for delayed redelivery
        cfg.ConfigureEndpoints(ctx);
    });
});
```

For a poison-message DLQ explicitly:

```csharp
cfg.ReceiveEndpoint("payment-received", e =>
{
    e.UseMessageRetry(r => r.Exponential(...));
    e.UseInMemoryOutbox();              // de-dupe in-process
    e.DiscardFaultedMessages = false;    // keep faulted msgs → _error queue
    e.ConfigureConsumer<PaymentReceivedConsumer>(ctx);
});
```

Manual jitter if your library doesn't add it:

```csharp
public static TimeSpan WithJitter(TimeSpan baseDelay, double factor = 0.5)
{
    var jitterMs = Random.Shared.NextDouble() * baseDelay.TotalMilliseconds * factor;
    return baseDelay + TimeSpan.FromMilliseconds(jitterMs);
}
```

---

## Real Project — Order Confirmation Email

E-commerce. After `OrderPlaced` is published, a notification service emails the customer. SendGrid occasionally rate-limits us; sometimes the customer's email bounces (terminal).

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
                throw new ValidationException("Invalid recipient — bad email format");

            _db.Inbox.Add(new InboxMessage { Id = ctx.MessageId.Value, HandledOnUtc = DateTime.UtcNow });
            await _db.SaveChangesAsync(ct: ctx.CancellationToken);
        }
        catch (ValidationException ex)
        {
            // Terminal — log and DON'T re-throw, so MassTransit doesn't retry. Goes to DLQ via Fault<T>.
            _log.LogWarning(ex, "Invalid order {OrderId} — sending to DLQ", ctx.Message.OrderId);
            throw;   // MassTransit's policy ignores ValidationException → straight to _error
        }
    }
}
```

Pipeline behavior:

* SendGrid 429 → `HttpRequestException` → retry with backoff (1s, 2s, 4s, 8s, 16s).
* SendGrid 503 → same path.
* Customer email malformed → `ValidationException` → no retry → DLQ.
* Bug in our code → unknown exception → retry policy default (transient) → 5 retries → DLQ.
* All retries exhausted → message lands in `_error` queue → on-call paged → engineer reviews and re-drives or discards.

---

## Common Pitfalls

* **Infinite retry** — without a cap, one poison message can chew CPU forever. Always cap at 3–10 attempts.
* **No jitter** — at scale, synchronized retries kill the recovering service. Always jitter.
* **Retrying terminal errors** — null refs and validation failures won't fix themselves. Classify before retrying.
* **No DLQ alerting** — messages silently rot in the dead letter queue while incidents stack up.
* **DLQ as recycle bin** — re-driving without fixing the bug just replays the failure. Treat DLQ entries as bugs to triage.
* **Retry without idempotency** — broker redelivery + handler retry = duplicate side effects. Always pair with [Idempotency](./idempotency.md) / Inbox.
* **Backoff measured against wall clock** — visibility timeouts and lease windows have their own clocks. Make sure your backoff fits inside them or you'll get **concurrent** redelivery.

---

## References

* [AWS Architecture Blog — Exponential backoff and jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
* [Polly resilience library](https://www.pollydocs.org/)
* [MassTransit — Exception handling](https://masstransit.io/documentation/concepts/exceptions)
* Sam Newman — *Building Microservices* (2nd ed.), Chapter 12.
* Michael Nygard — *Release It!* (2nd ed.), Stability Patterns.
