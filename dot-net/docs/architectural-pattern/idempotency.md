# Idempotency Pattern

> An operation is **idempotent** if invoking it once or a hundred times produces the same effect. In a distributed system where the network can drop responses and clients **must** retry, idempotency is the difference between one charge and ten.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — A guarantee that **repeated execution of the same logical operation has the same effect as executing it once**. Implemented via a client-supplied **idempotency key** + a server-side dedup store that maps `key → result`.
- **Why** — Networks are unreliable. A client that sees a timeout has no way to know whether the server processed the request. The only safe retry is one the server can recognize as a duplicate.
- **When** — Every **mutating** operation that crosses a network boundary: `POST /payments`, `PUT /orders/{id}`, any consumer of an at-least-once message broker (Kafka, RabbitMQ, SQS, Azure Service Bus).
- **Where** — At the API edge (middleware / action filter) and at every message-consumer side effect. The Inbox pattern is idempotency applied to async messaging.
- **How** — Client generates a unique key per business intent (UUID v4 / ULID); server stores `(key, response)` for a TTL (typically 24h); on retry, server returns the stored response **without re-executing**.

---

## The Problem

A naive `POST /payments` flow:

```
Client                    Server                Stripe
  │   POST /payments         │                     │
  ├─────────────────────────►│                     │
  │                          ├─── Charge $100 ────►│
  │                          │◄──── success ───────┤
  │   200 OK (lost in net)   │                     │
  │ ←✗──────────────────────│                     │
  │                          │                     │
  │   (timeout — retry!)     │                     │
  ├─────────────────────────►│                     │
  │                          ├─── Charge $100 ────►│  ← 💥 DUPLICATE
  │                          │◄──── success ───────┤
  │   200 OK                  │                     │
  │ ◄──────────────────────── │                     │
```

The user clicks "Pay" once. They get charged twice. There is **no way to fix this with retries alone** — the second request looks identical to a brand-new one.

---

## What Counts as Idempotent

Idempotency is **about effect, not output**. A call can return different timestamps and still be idempotent — what must not change is the **observable state of the system**.

| Operation              | Idempotent? | Why                                                      |
| ---------------------- | ----------- | -------------------------------------------------------- |
| `GET /orders/123`      | ✅ Yes       | Read-only.                                               |
| `DELETE /orders/123`   | ✅ Yes       | Order is gone; deleting again → still gone.              |
| `PUT /orders/123`      | ✅ Yes (if full replace) | Final state determined by the request body. |
| `POST /payments`       | ❌ No (by default) | Each call creates a new resource → duplicates.    |
| `POST /payments` with `Idempotency-Key: abc-123` | ✅ Yes | Server dedupes by key. |
| `UPDATE counter SET v = v + 1` | ❌ No | Repeats accumulate. |
| `UPDATE counter SET v = 5` | ✅ Yes | Absolute set, not delta. |

> **Rule of thumb:** an operation is idempotent when its target state is **absolute**, not **relative**. "Set balance = 100" is idempotent; "add 50" is not.

---

## The Idempotency Key Pattern

The HTTP convention popularized by Stripe:

1. **Client** generates a unique key per *business intent* — not per HTTP retry. UUID v4 or ULID.
2. **Client** sends it in a header: `Idempotency-Key: 8a7f3e2b-...`.
3. **Server** looks up the key in a dedup store.
4. **Server** either: returns the stored response (replay), or processes the request and stores the response (first call).
5. Entry expires after a TTL (typically 24h–7d).

```
First request:
  Client →  Idempotency-Key: K  →  Server
                                     ├─ store(K, "pending")
                                     ├─ execute business logic
                                     ├─ store(K, response)
                                     └─ return response

Retry:
  Client →  Idempotency-Key: K  →  Server
                                     ├─ found(K) → return stored response
                                     └─ (no business logic re-runs)
```

### Edge case: in-flight retries

If the client retries while the first request is still executing, naïve "check then insert" loses races. Use **atomic** semantics:

* DB: `INSERT ... ON CONFLICT DO NOTHING` and treat the conflict as "in-flight, return 409".
* Redis: `SET key value NX EX 86400` — succeeds only if the key didn't exist.

---

## C# Implementation — ASP.NET Core Middleware

```csharp
public sealed class IdempotencyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IIdempotencyStore _store;

    public IdempotencyMiddleware(RequestDelegate next, IIdempotencyStore store)
        => (_next, _store) = (next, store);

    public async Task InvokeAsync(HttpContext ctx)
    {
        if (!HttpMethods.IsPost(ctx.Request.Method) && !HttpMethods.IsPut(ctx.Request.Method))
        {
            await _next(ctx);
            return;
        }

        if (!ctx.Request.Headers.TryGetValue("Idempotency-Key", out var keyHeader)
            || !Guid.TryParse(keyHeader, out var key))
        {
            ctx.Response.StatusCode = StatusCodes.Status400BadRequest;
            await ctx.Response.WriteAsync("Idempotency-Key header required.");
            return;
        }

        var cached = await _store.TryGetAsync(key, ctx.RequestAborted);
        if (cached is { } resp)
        {
            ctx.Response.StatusCode = resp.StatusCode;
            await ctx.Response.WriteAsync(resp.Body, ctx.RequestAborted);
            return;
        }

        // Reserve the key — atomic. If another request reserved it, we're a concurrent retry.
        if (!await _store.TryReserveAsync(key, ttl: TimeSpan.FromHours(24), ctx.RequestAborted))
        {
            ctx.Response.StatusCode = StatusCodes.Status409Conflict;
            await ctx.Response.WriteAsync("Request in flight.");
            return;
        }

        // Capture the response so we can store it after the pipeline runs.
        var original = ctx.Response.Body;
        await using var buffer = new MemoryStream();
        ctx.Response.Body = buffer;

        await _next(ctx);

        buffer.Position = 0;
        var body = await new StreamReader(buffer).ReadToEndAsync();

        await _store.SaveAsync(key, new CachedResponse(ctx.Response.StatusCode, body), ctx.RequestAborted);

        buffer.Position = 0;
        await buffer.CopyToAsync(original);
    }
}

public record CachedResponse(int StatusCode, string Body);

public interface IIdempotencyStore
{
    Task<CachedResponse?> TryGetAsync(Guid key, CancellationToken ct);
    Task<bool> TryReserveAsync(Guid key, TimeSpan ttl, CancellationToken ct);
    Task SaveAsync(Guid key, CachedResponse response, CancellationToken ct);
}
```

---

## Real Project — Payment API

A payment service that charges via Stripe.

```csharp
[ApiController]
[Route("payments")]
public sealed class PaymentsController : ControllerBase
{
    private readonly IPaymentService _payments;

    [HttpPost]
    public async Task<IActionResult> Charge([FromBody] ChargeRequest req, CancellationToken ct)
    {
        // Idempotency-Key already validated + cached by middleware.
        // If we get here, this is a FIRST call for this key.
        var result = await _payments.ChargeAsync(req.OrderId, req.Amount, ct);
        return Ok(new { paymentId = result.Id, status = result.Status });
    }
}

public sealed class PaymentService
{
    private readonly IStripeClient _stripe;
    private readonly AppDbContext _db;

    public async Task<Payment> ChargeAsync(Guid orderId, decimal amount, CancellationToken ct)
    {
        // Defense in depth: forward the same idempotency key to Stripe.
        var key = $"order-{orderId}";

        var charge = await _stripe.Charges.CreateAsync(
            new ChargeCreateOptions { Amount = (long)(amount * 100), Currency = "usd" },
            new RequestOptions { IdempotencyKey = key },
            ct);

        var payment = new Payment(orderId, charge.Id, amount, charge.Status);
        _db.Payments.Add(payment);
        await _db.SaveChangesAsync(ct);
        return payment;
    }
}
```

Two layers of idempotency:

1. **Our API** dedupes by `Idempotency-Key` header so the client's retry doesn't even reach Stripe twice.
2. **Stripe** dedupes by the key we forward so even if a bug in our service bypassed (1), the charge still doesn't double.

> "Idempotency is a property of the **whole call chain**. Each network hop needs its own key — or it can't dedupe."

---

## Common Pitfalls

* **Reusing the same key for different intents** — "Pay $100" and "Pay $200" with the same key collide. The first wins. Generate a fresh key per business action.
* **Caching errors too aggressively** — caching a `500 Internal Server Error` blocks legitimate retries forever. Cache only `2xx` and certain `4xx` (e.g., validation errors). Let `5xx` re-execute on retry.
* **TTL too short** — clients with poor connectivity may retry hours later. 24h minimum.
* **Storing only the status code** — clients re-issuing the request expect the **same body** (e.g., the same `paymentId`). Cache the full response.
* **Hashing the request body as a key** — sounds clever, breaks the moment a field re-orders or whitespace differs. Use an explicit client-supplied key.
* **Forgetting at-least-once consumers** — message handlers need idempotency too. See the [Inbox](./outbox.md) pattern.

---

## References

* [Stripe — Designing robust APIs with idempotency](https://stripe.com/docs/api/idempotent_requests)
* [IETF draft: The Idempotency-Key HTTP Header](https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-06.html)
* Sam Newman — *Building Microservices* (2nd ed.), Chapter 12: Resilience.
* Chris Richardson — *Microservices Patterns*, Chapter 3.
