# Idempotency Pattern

> Một operation là **idempotent** nếu gọi 1 lần hay 100 lần đều ra cùng một effect. Trong distributed system mà network có thể drop response và client **buộc phải** retry, idempotency là khác biệt giữa charge 1 lần và charge 10 lần.

> 🇻🇳 Phiên bản tiếng Việt. English: [`idempotency.md`](./idempotency.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Đảm bảo rằng **thực thi lặp lại cùng một logical operation cho effect giống hệt như thực thi 1 lần**. Implement qua **idempotency key** do client cung cấp + dedup store phía server map `key → result`.
- **Tại sao** — Network không tin được. Client thấy timeout không có cách nào biết server đã xử lý request hay chưa. Retry an toàn duy nhất là retry mà server nhận diện được là duplicate.
- **Khi nào** — Mọi operation **mutating** đi qua network boundary: `POST /payments`, `PUT /orders/{id}`, mọi consumer của at-least-once message broker (Kafka, RabbitMQ, SQS, Azure Service Bus).
- **Ở đâu** — Ở API edge (middleware / action filter) và ở mọi side effect của message consumer. Pattern Inbox chính là idempotency áp dụng cho async messaging.
- **Như thế nào** — Client tạo unique key cho mỗi business intent (UUID v4 / ULID); server lưu `(key, response)` trong TTL (thường 24h); khi retry, server trả response đã lưu mà **không re-execute**.

---

## Vấn đề

Flow `POST /payments` ngây thơ:

```
Client                    Server                Stripe
  │   POST /payments         │                     │
  ├─────────────────────────►│                     │
  │                          ├─── Charge $100 ────►│
  │                          │◄──── success ───────┤
  │   200 OK (mất trên net)  │                     │
  │ ←✗──────────────────────│                     │
  │                          │                     │
  │   (timeout — retry!)     │                     │
  ├─────────────────────────►│                     │
  │                          ├─── Charge $100 ────►│  ← 💥 DUPLICATE
  │                          │◄──── success ───────┤
  │   200 OK                  │                     │
  │ ◄──────────────────────── │                     │
```

User click "Pay" 1 lần. Bị charge 2 lần. **Không có cách nào sửa bằng retry** — request thứ 2 nhìn y hệt request mới hoàn toàn.

---

## Idempotent nghĩa là gì

Idempotency nói về **effect, không phải output**. Một call có thể trả timestamp khác nhau và vẫn idempotent — cái không được đổi là **observable state của hệ thống**.

| Operation              | Idempotent? | Tại sao                                                  |
| ---------------------- | ----------- | -------------------------------------------------------- |
| `GET /orders/123`      | ✅           | Read-only.                                               |
| `DELETE /orders/123`   | ✅           | Order đã xóa rồi; xóa nữa → vẫn không có.                |
| `PUT /orders/123`      | ✅ (nếu full replace) | State cuối quyết định bởi request body.         |
| `POST /payments`       | ❌ (mặc định) | Mỗi call tạo resource mới → duplicate.                  |
| `POST /payments` với `Idempotency-Key: abc-123` | ✅ | Server dedup theo key. |
| `UPDATE counter SET v = v + 1` | ❌    | Lặp lại thì cộng dồn.                                    |
| `UPDATE counter SET v = 5` | ✅        | Absolute set, không phải delta.                          |

> **Quy tắc:** một operation idempotent khi target state là **tuyệt đối**, không phải **tương đối**. "Set balance = 100" idempotent; "add 50" thì không.

---

## Pattern Idempotency Key

Convention HTTP do Stripe phổ biến hóa:

1. **Client** tạo unique key cho mỗi *business intent* — không phải mỗi HTTP retry. UUID v4 hoặc ULID.
2. **Client** gửi qua header: `Idempotency-Key: 8a7f3e2b-...`.
3. **Server** lookup key trong dedup store.
4. **Server** hoặc: trả response đã lưu (replay), hoặc xử lý request rồi lưu response (first call).
5. Entry expire sau TTL (thường 24h–7d).

```
Request đầu:
  Client →  Idempotency-Key: K  →  Server
                                     ├─ store(K, "pending")
                                     ├─ execute business logic
                                     ├─ store(K, response)
                                     └─ return response

Retry:
  Client →  Idempotency-Key: K  →  Server
                                     ├─ found(K) → return stored response
                                     └─ (không re-run business logic)
```

### Edge case: retry đồng thời

Nếu client retry khi request đầu vẫn đang chạy, "check then insert" ngây thơ thua race condition. Dùng **atomic** semantics:

* DB: `INSERT ... ON CONFLICT DO NOTHING` và coi conflict là "in-flight, return 409".
* Redis: `SET key value NX EX 86400` — chỉ thành công nếu key chưa tồn tại.

---

## Implementation C# — ASP.NET Core Middleware

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

        // Reserve key — atomic. Nếu request khác đã reserve, đây là retry đồng thời.
        if (!await _store.TryReserveAsync(key, ttl: TimeSpan.FromHours(24), ctx.RequestAborted))
        {
            ctx.Response.StatusCode = StatusCodes.Status409Conflict;
            await ctx.Response.WriteAsync("Request in flight.");
            return;
        }

        // Capture response để lưu sau khi pipeline chạy.
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

Service thanh toán charge qua Stripe.

```csharp
[ApiController]
[Route("payments")]
public sealed class PaymentsController : ControllerBase
{
    private readonly IPaymentService _payments;

    [HttpPost]
    public async Task<IActionResult> Charge([FromBody] ChargeRequest req, CancellationToken ct)
    {
        // Idempotency-Key đã được middleware validate + cache.
        // Đến đây nghĩa là FIRST call cho key này.
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
        // Defense in depth: forward cùng idempotency key sang Stripe.
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

Hai lớp idempotency:

1. **API của ta** dedup theo header `Idempotency-Key` để retry của client không đến Stripe 2 lần.
2. **Stripe** dedup theo key ta forward để dù bug trong service ta bypass (1), charge vẫn không bị double.

> "Idempotency là property của **toàn bộ call chain**. Mỗi network hop cần key riêng — không có thì không dedup được."

---

## Pitfall thường gặp

* **Reuse cùng key cho intent khác nhau** — "Pay $100" và "Pay $200" cùng key sẽ va. Cái đầu thắng. Tạo key mới cho mỗi business action.
* **Cache lỗi quá hăng** — cache `500 Internal Server Error` chặn retry hợp lệ mãi mãi. Chỉ cache `2xx` và một số `4xx` (như validation error). Để `5xx` re-execute khi retry.
* **TTL quá ngắn** — client mạng yếu có thể retry sau nhiều giờ. Tối thiểu 24h.
* **Chỉ store status code** — client retry mong response **giống y** (ví dụ cùng `paymentId`). Cache full response.
* **Hash request body làm key** — nghe khéo, vỡ ngay khi field re-order hoặc whitespace khác. Dùng explicit client-supplied key.
* **Quên consumer at-least-once** — message handler cũng cần idempotency. Xem pattern [Inbox](./outbox.md).

---

## Tham khảo

* [Stripe — Designing robust APIs with idempotency](https://stripe.com/docs/api/idempotent_requests)
* [IETF draft: The Idempotency-Key HTTP Header](https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-06.html)
* Sam Newman — *Building Microservices* (2nd ed.), Chapter 12: Resilience.
* Chris Richardson — *Microservices Patterns*, Chapter 3.
