# Synchronous Resilience — Timeout, Circuit Breaker, Bulkhead

> Ba pattern chặn downstream chậm/lỗi không kéo caller chết theo. Áp dụng cho mọi synchronous boundary trong microservice — HTTP, gRPC, database, third-party API. Cùng nhau implement nguyên tắc **"fail fast và sống sót."**

> 🇻🇳 Phiên bản tiếng Việt. English: [`resilience-sync.md`](./resilience-sync.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — **Timeout** giới hạn 1 call kéo dài bao lâu. **Circuit Breaker** ngừng gọi downstream đã biết là hỏng. **Bulkhead** cô lập resource (thread pool, connection pool) để 1 dependency tồi không vắt cạn các cái khác.
- **Tại sao** — Không có chúng, downstream chậm chuyển thành thread exhaustion, queue request, cuối cùng service ta đổ trong khi vấn đề thật ở chỗ khác. Đây là **cascading failure** kinh điển.
- **Khi nào** — Mọi synchronous call qua service boundary: HTTP, gRPC, DB, cache, external API. Quy tắc "in-process dưới 50 ms" **không** áp dụng — call cục bộ không cần.
- **Ở đâu** — Tại call site (HTTP client policy / gRPC interceptor / DB resilience strategy). Trong .NET: Polly v8 hoặc `Microsoft.Extensions.Resilience` (Polly under the hood).
- **Như thế nào** — Layer chúng: **Bulkhead** ngoài cùng → **Timeout** giữa → **Retry** → **Circuit Breaker** trong cùng. Tune từ latency thực tế (p99 của downstream khỏe + headroom), không đoán.

---

## Vấn đề — Cascading Failure

Service checkout gọi pricing service đồng bộ:

```csharp
public async Task<Order> Checkout(Cart cart)
{
    var price = await _pricing.GetTotalAsync(cart);   // ← bình thường p99 = 5 s
    return _orders.Place(cart, price);
}
```

Khi pricing chậm từ 100 ms → 30 s:

```
t=0s:    request 1 đến → gọi pricing → chờ
t=2s:    request 2 đến → gọi pricing → chờ
t=10s:   request 100 đến → gọi pricing → chờ
         Tất cả 100 thread/connection bị giữ chờ downstream chậm.
t=12s:   request 101 đến → không còn thread/connection → 503
         Checkout chết — dù lỗi thật ở pricing service.
```

Vấn đề của downstream thành vấn đề của **ta**. Bulkhead, Timeout, Circuit Breaker phá chuỗi này.

---

## Timeout

Pattern quan trọng nhất. Mọi cross-boundary call phải có hard deadline.

```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(callerCt);
cts.CancelAfter(TimeSpan.FromSeconds(2));
var response = await httpClient.GetAsync(url, cts.Token);
```

### Chọn timeout thế nào

* Start từ **p99** của downstream khi khỏe.
* Cộng headroom (ví dụ `p99 + 500 ms` hoặc `p99 * 1.5`).
* **Timeout caller < total time budget** — API ta có SLA 5 s và gọi 3 downstream, mỗi cái ≤ 1.5 s.
* Nếu downstream lâu hơn timeout của ta, đó là bug của downstream. Đừng nới timeout — chuyển sang async.

### Ba loại "timeout" phải set

| Layer        | Kiểm soát gì                                      | Trap default thường gặp                      |
| ------------ | ------------------------------------------------- | -------------------------------------------- |
| **Connect**  | Thời gian thiết lập TCP/TLS                       | `Infinite` ở nhiều HTTP client                |
| **Request**  | Thời gian cho cả HTTP request/response            | `100 s` trong `HttpClient` ← quá dài         |
| **Total**    | Deadline end-to-end gồm retry                     | Thường thiếu — retry có thể vỡ SLA           |

Luôn set cả 3.

---

## Circuit Breaker

Circuit breaker có 3 state. Lần đầu call fail, breaker **Closed** (call đi qua). Đủ fail thì **Open** (call fail fast không chạm downstream). Sau cooldown, **Half-Open** cho 1 probe chạy thử recovery.

```
                  ┌────────────────────────────────────┐
                  │                                    │
                  ▼                                    │
            ┌──────────┐  fail ≥ threshold       ┌──────────┐
            │  Closed  │──────────────────────► │   Open    │
            │ (normal) │                        │ (fail fast)│
            └──────────┘ ◄──────────┐           └──────────┘
                  ▲                  │                │
        success   │                  │ fail            │ sau cooldown
                  │                  │                 │
                  │            ┌─────┴────────┐        │
                  └───────────┤  Half-Open    │◄───────┘
                              │ (probe 1)     │
                              └───────────────┘
```

| State        | Hành vi                                                        |
| ------------ | -------------------------------------------------------------- |
| **Closed**   | Đi qua. Track tỉ lệ fail.                                       |
| **Open**     | Reject ngay (không gọi downstream). Trả cache / fallback / `503`. |
| **Half-Open**| Cho 1 trial call. Success → Closed. Fail → Open.                |

### Khi breaker đáng dùng

* Downstream **flapping** — đang phục hồi sau deploy, GC pause, partition.
* Có **fallback** ý nghĩa — cache data, default value, degraded UX.
* Chi phí 1 call fail **cao** — timeout chậm, giữ thread, block flow khách.

Khi **không** giúp:

* Single bad request (validation error) — không phải tín hiệu downstream sức khỏe, đừng trip.
* Per-request retry mà downstream khỏe nhưng request cụ thể sai.

> **Tune theo error rate, không theo absolute count.** "Trip sau 5 fail" hỏng dưới tải (5 fail / 5000 call thì ok; 5 / 50 thì không).

---

## Bulkhead

Bulkhead trong tàu là vách kín nước — 1 khoang ngập thì các khoang khác vẫn khô. Code cũng vậy: cô lập resource pool cho mỗi dependency.

```
Không bulkhead:                     Có bulkhead:
┌──────────────────┐               ┌──────────────────┐
│   Shared pool    │               │  Pool: Pricing   │
│  (200 thread)    │               │  (20 thread)     │
│                  │               │                  │
│  Pricing → ████  │               │  Pricing → ████  │
│  Catalog → ██    │               ├──────────────────┤
│  Inventory → █   │               │  Pool: Catalog   │
└──────────────────┘               │  (20 thread)     │
                                    │  Catalog → ██    │
Pricing chậm ăn                    ├──────────────────┤
hết thread — Catalog               │  Pool: Inventory │
cũng không chạy.                   │  (20 thread)     │
                                    │  Inventory → █   │
                                    └──────────────────┘

                                    Pricing chậm chỉ ảnh
                                    hưởng Pricing.
```

Trong .NET, bulkhead phổ biến là **`HttpClient` riêng cho mỗi downstream** với connection pool + concurrency limiter của riêng nó. Polly v8 có `RateLimiterPolicy` và `ConcurrencyLimiter`.

```csharp
services.AddHttpClient("pricing", c => c.BaseAddress = new Uri("https://pricing/"))
    .AddStandardResilienceHandler(o =>
    {
        o.RateLimiter.PermitLimit  = 20;       // bulkhead: tối đa 20 concurrent
        o.RateLimiter.QueueLimit   = 0;        // shed load ngay khi vượt
    });
```

---

## Layering — Thứ tự quan trọng

Khi xếp các policy này, thứ tự quanh call site đổi behavior:

```
┌────────────────────────── Bulkhead (concurrency / queue limit) ──┐
│  ┌─────────────────────── Total timeout ─────────────────────┐   │
│  │  ┌──────────────────── Retry policy ──────────────────┐    │   │
│  │  │  ┌───────────────── Circuit breaker ───────────┐   │    │   │
│  │  │  │  ┌────────────── Per-attempt timeout ───┐  │   │    │   │
│  │  │  │  │      HTTP call đến downstream         │  │   │    │   │
│  │  │  │  └──────────────────────────────────────┘  │   │    │   │
│  │  │  └──────────────────────────────────────────────┘   │    │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └───────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

* **Bulkhead ngoài cùng** — shed load *trước khi* call chậm giữ thread.
* **Total timeout** — cap toàn pipeline gồm retry.
* **Retry** — bọc breaker để breaker tripped không đốt retry.
* **Per-attempt timeout trong cùng** — bảo vệ từng call.

Đảo thứ tự sẽ ra pathology (ví dụ retry ngoài timeout → retry storm; circuit breaker ngoài retry → breaker đếm retry thành các fail riêng).

---

## C# — `Microsoft.Extensions.Resilience` (Polly v8)

```csharp
services.AddHttpClient<IPricingClient, PricingClient>(c =>
{
    c.BaseAddress = new Uri("https://pricing/");
    c.Timeout = TimeSpan.FromSeconds(5);    // total timeout (lưới ngoài cùng)
})
.AddResilienceHandler("pricing", b =>
{
    b.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 20,
        QueueLimit  = 0,                    // bulkhead — fail fast khi vượt 20 in-flight
    });

    b.AddTimeout(TimeSpan.FromSeconds(5));  // total

    b.AddRetry(new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        BackoffType      = DelayBackoffType.Exponential,
        UseJitter        = true,
    });

    b.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
    {
        FailureRatio     = 0.5,             // 50% fail trong window
        MinimumThroughput = 10,             // ...với ít nhất 10 call
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration    = TimeSpan.FromSeconds(15),
    });

    b.AddTimeout(TimeSpan.FromSeconds(2));  // per-attempt
});
```

---

## Real Project — Checkout phụ thuộc Pricing Service

E-commerce. Checkout gọi Pricing đồng bộ để tính total; Pricing đôi khi chậm khi catalog re-indexing.

```csharp
public sealed class CheckoutService
{
    private readonly IPricingClient _pricing;
    private readonly IFallbackPriceCache _cache;
    private readonly ILogger<CheckoutService> _log;

    public async Task<Order> CheckoutAsync(Cart cart, CancellationToken ct)
    {
        PriceQuote quote;
        try
        {
            quote = await _pricing.GetTotalAsync(cart, ct);
        }
        catch (BrokenCircuitException)
        {
            // Breaker Open — pricing chết. Degrade graceful.
            _log.LogWarning("Pricing breaker open; dùng last-known price");
            quote = await _cache.GetLastKnownAsync(cart, ct)
                    ?? throw new CheckoutUnavailableException("Pricing tạm thời không sẵn sàng.");
        }
        catch (TimeoutRejectedException)
        {
            _log.LogWarning("Pricing call timeout");
            quote = await _cache.GetLastKnownAsync(cart, ct)
                    ?? throw new CheckoutUnavailableException("Pricing tạm thời không sẵn sàng.");
        }

        return _orders.Place(cart, quote);
    }
}
```

Diễn biến dưới tải:

* p99 của Pricing tăng từ 100 ms → 30 s.
* Per-attempt timeout fire ở 2 s → retry đập tường tương tự → sau 3 retry circuit breaker đếm fail.
* Sau ~10 failed call trong window 30 s, breaker **Open** → 15 s tiếp theo call fail fast với `BrokenCircuitException` (không giữ thread, không tải downstream).
* Checkout fallback về cached price → user vẫn checkout được (kèm reminder "price refresh").
* Sau 15 s, breaker **Half-Open** → 1 probe. Pricing phục hồi → Closed, flow bình thường tiếp.

Bulkhead (concurrency = 20) đảm bảo dù checkout traffic tăng 100x lúc pricing ốm, chỉ có 20 Pricing call concurrent — 80 cái còn lại fail fast và fallback.

---

## Pitfall thường gặp

* **Không timeout** — nguyên nhân cascading failure phổ biến nhất. Default `HttpClient` là 100 s. *Luôn* override.
* **Trip threshold quá nhạy** — circuit mở vì noise. Tune từ failure rate thật, không từ lo lắng.
* **Trip threshold quá lỏng** — đến lúc breaker mở thì thread đã cạn. Trip nhanh hơn + bulkhead nhỏ hơn thường tốt hơn.
* **Retry trong circuit breaker bị tính như fail** — retry phải trong scope breaker (1 logical operation = 1 breaker decision), nếu không layer sai.
* **Không fallback** — mở breaker là để enable degraded response, không phải error khác. Không có gì để fallback thì hỏi xem call này có nên sync không.
* **Shared pool cho mọi downstream** — mọi downstream share 1 `HttpClient` / 1 pool. 1 downstream chậm đầu độc tất cả. Dùng named `HttpClient` riêng cho mỗi downstream.
* **Bulkhead với queue vô hạn** — `QueueLimit = unbounded` phá ý nghĩa; request chồng chất rồi cũng timeout. Ưu tiên `QueueLimit = 0` (reject ngay) hoặc queue nhỏ.

---

## Tham khảo

* Michael Nygard — *Release It!* (2nd ed.), Stability Patterns (reference kinh điển).
* [Polly v8 documentation](https://www.pollydocs.org/)
* [.NET 8 — `Microsoft.Extensions.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/)
* [AWS Builders' Library — Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
* Sam Newman — *Building Microservices* (2nd ed.), Chapter 12.
