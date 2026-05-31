# Synchronous Resilience — Timeout, Circuit Breaker, Bulkhead

> Three patterns that stop a slow or failing downstream from taking down the caller. They apply to every synchronous boundary in a microservices system — HTTP, gRPC, database, third-party API. Together they implement the principle **"fail fast and stay alive."**

---

## Quick Reference (What · Why · When · Where · How)

- **What** — **Timeout** caps how long a single call may take. **Circuit Breaker** stops calling a downstream we already know is broken. **Bulkhead** isolates resources (thread pool, connection pool) so one bad dependency cannot starve the others.
- **Why** — Without them, a slow downstream chains into thread exhaustion, request queueing, and ultimately your service falling over while the actual problem is somewhere else. This is the classic **cascading failure**.
- **When** — Every synchronous call across a service boundary: HTTP, gRPC, DB, cache, external API. The "fewer than 50 ms in-process" rule does **not** apply — local calls don't need these patterns.
- **Where** — At the call site (HTTP client policy / gRPC interceptor / DB resilience strategy). In .NET: Polly v8 or `Microsoft.Extensions.Resilience` (Polly under the hood).
- **How** — Layer them: **Bulkhead** outer → **Timeout** middle → **Retry** → **Circuit Breaker** innermost. Tune from observed latency (p99 of healthy downstream + headroom), not guesses.

---

## The Problem — Cascading Failure

A checkout service that calls a pricing service synchronously:

```csharp
public async Task<Order> Checkout(Cart cart)
{
    var price = await _pricing.GetTotalAsync(cart);   // ← 5 s p99 normally
    return _orders.Place(cart, price);
}
```

When pricing slows from 100 ms → 30 s:

```
t=0s:    request 1 arrives → calls pricing → waits
t=2s:    request 2 arrives → calls pricing → waits
t=10s:   request 100 arrives → calls pricing → waits
         All 100 threads/connections held waiting on the slow downstream.
t=12s:   request 101 arrives → no thread/connection available → 503
         Checkout is down — even though it's the pricing service that's actually broken.
```

The downstream's problem became **our** problem. Bulkhead, Timeout, and Circuit Breaker break this chain.

---

## Timeout

The single most important pattern. Every cross-boundary call must have a hard deadline.

```csharp
using var cts = CancellationTokenSource.CreateLinkedTokenSource(callerCt);
cts.CancelAfter(TimeSpan.FromSeconds(2));
var response = await httpClient.GetAsync(url, cts.Token);
```

### How to pick the timeout

* Start from the downstream's **p99** in healthy operation.
* Add headroom (e.g. `p99 + 500 ms` or `p99 * 1.5`).
* **Caller's timeout < total time budget** — if your API has a 5 s SLA and you call three downstreams, each gets ≤ 1.5 s.
* If the downstream takes longer than your timeout, that's the downstream's bug. Don't widen the timeout — make the call async.

### Three different "timeouts" you must set

| Layer        | What it controls                                  | Common default trap                          |
| ------------ | ------------------------------------------------- | -------------------------------------------- |
| **Connect**  | Time to establish a TCP/TLS connection            | `Infinite` in many HTTP clients               |
| **Request**  | Time for the whole HTTP request/response          | `100 s` in `HttpClient` ← way too long       |
| **Total**    | End-to-end deadline including retries             | Often missing — retries can blow the SLA     |

Always set all three.

---

## Circuit Breaker

A circuit breaker has three states. The first time a call fails, the breaker is **Closed** (calls flow through). After enough failures, it **Opens** (calls fail fast without touching the downstream). After a cooldown, it goes **Half-Open** and lets one probe through to test recovery.

```
                  ┌────────────────────────────────────┐
                  │                                    │
                  ▼                                    │
            ┌──────────┐  failures ≥ threshold   ┌──────────┐
            │  Closed  │──────────────────────► │   Open    │
            │ (normal) │                        │ (fail fast)│
            └──────────┘ ◄──────────┐           └──────────┘
                  ▲                  │                │
        successes │                  │ failure         │ after cooldown
                  │                  │                 │
                  │            ┌─────┴────────┐        │
                  └───────────┤  Half-Open    │◄───────┘
                              │ (probe one)   │
                              └───────────────┘
```

| State        | Behavior                                                       |
| ------------ | -------------------------------------------------------------- |
| **Closed**   | Pass through. Track failure rate.                              |
| **Open**     | Reject immediately (no call to downstream). Return cached / fallback / `503`. |
| **Half-Open**| Allow a single trial call. Success → Closed. Failure → Open.   |

### When the breaker pays off

* Downstream is **flapping** — recovering from a deploy, GC pause, partition.
* You have a meaningful **fallback** — cached data, default value, degraded UX.
* The cost of a failed call is **high** — slow timeout, holds a thread, blocks customer flow.

When it does **not** help:

* Single bad request (validation error) — that's not a downstream health signal, don't trip the breaker.
* Per-request retries where the downstream is fine but the specific request is bad.

> **Tune by error rate, not absolute count.** "Trip after 5 failures" breaks under load (5 failures out of 5000 is fine; 5 out of 50 is not).

---

## Bulkhead

A bulkhead in a ship is a watertight wall — if one compartment floods, the rest stay dry. Same idea in code: isolate the resource pool used by each dependency.

```
Without bulkhead:                  With bulkhead:
┌──────────────────┐               ┌──────────────────┐
│   Shared pool    │               │  Pool: Pricing   │
│  (200 threads)   │               │  (20 threads)    │
│                  │               │                  │
│  Pricing → ████  │               │  Pricing → ████  │
│  Catalog → ██    │               ├──────────────────┤
│  Inventory → █   │               │  Pool: Catalog   │
└──────────────────┘               │  (20 threads)    │
                                    │  Catalog → ██    │
A slow Pricing eats               ├──────────────────┤
all threads — Catalog              │  Pool: Inventory │
also can't run.                    │  (20 threads)    │
                                    │  Inventory → █   │
                                    └──────────────────┘

                                    Pricing slowness only
                                    affects Pricing.
```

In .NET, the common bulkhead is a **per-downstream `HttpClient`** with its own connection pool + concurrency limiter. Polly v8 has `RateLimiterPolicy` and `ConcurrencyLimiter`.

```csharp
services.AddHttpClient("pricing", c => c.BaseAddress = new Uri("https://pricing/"))
    .AddStandardResilienceHandler(o =>
    {
        o.RateLimiter.PermitLimit  = 20;       // bulkhead: max 20 concurrent
        o.RateLimiter.QueueLimit   = 0;        // shed load instantly past the limit
    });
```

---

## Layering — Order Matters

When you stack these policies, the order around the call site changes behavior:

```
┌────────────────────────── Bulkhead (concurrency / queue limit) ──┐
│  ┌─────────────────────── Total timeout ─────────────────────┐   │
│  │  ┌──────────────────── Retry policy ──────────────────┐    │   │
│  │  │  ┌───────────────── Circuit breaker ───────────┐   │    │   │
│  │  │  │  ┌────────────── Per-attempt timeout ───┐  │   │    │   │
│  │  │  │  │      HTTP call to downstream         │  │   │    │   │
│  │  │  │  └──────────────────────────────────────┘  │   │    │   │
│  │  │  └──────────────────────────────────────────────┘   │    │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └───────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

* **Bulkhead outermost** — sheds load *before* a slow call can hold a thread.
* **Total timeout** — caps the entire pipeline including retries.
* **Retry** — wraps the breaker so a tripped breaker doesn't burn retries.
* **Per-attempt timeout innermost** — guards each individual call.

Reverse the order and you get pathologies (e.g., retry outside timeout → retry storm; circuit breaker outside retry → breaker counts retries as separate failures).

---

## C# — `Microsoft.Extensions.Resilience` (Polly v8)

```csharp
services.AddHttpClient<IPricingClient, PricingClient>(c =>
{
    c.BaseAddress = new Uri("https://pricing/");
    c.Timeout = TimeSpan.FromSeconds(5);    // total timeout (outermost net)
})
.AddResilienceHandler("pricing", b =>
{
    b.AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 20,
        QueueLimit  = 0,                    // bulkhead — fail fast past 20 in-flight
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
        FailureRatio     = 0.5,             // 50% failures in the window
        MinimumThroughput = 10,             // ...with at least 10 calls
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration    = TimeSpan.FromSeconds(15),
    });

    b.AddTimeout(TimeSpan.FromSeconds(2));  // per-attempt
});
```

---

## Real Project — Checkout depends on Pricing Service

E-commerce. Checkout calls Pricing synchronously to compute totals; Pricing occasionally lags during catalog re-indexing.

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
            // Breaker is Open — pricing is down. Degrade gracefully.
            _log.LogWarning("Pricing breaker open; using last-known price");
            quote = await _cache.GetLastKnownAsync(cart, ct)
                    ?? throw new CheckoutUnavailableException("Pricing temporarily unavailable.");
        }
        catch (TimeoutRejectedException)
        {
            _log.LogWarning("Pricing call timed out");
            quote = await _cache.GetLastKnownAsync(cart, ct)
                    ?? throw new CheckoutUnavailableException("Pricing temporarily unavailable.");
        }

        return _orders.Place(cart, quote);
    }
}
```

What changes under load:

* Pricing's p99 spikes from 100 ms to 30 s.
* Per-attempt timeout fires at 2 s → retry hits the same wall → after 3 retries the circuit breaker counts the failures.
* After ~10 failed calls in the 30 s window, breaker **Opens** → next 15 s of calls fail fast with `BrokenCircuitException` (no thread held, no downstream load).
* Checkout falls back to last-known cached price → user can still complete checkout (potentially with a "price refresh" reminder).
* After 15 s, breaker goes **Half-Open** → one probe call. If pricing recovered → Closed, normal flow resumes.

Bulkhead (concurrency = 20) ensures that even if checkout traffic spikes 100x while pricing is sick, only 20 concurrent Pricing calls exist — the other 80 fail fast and fall back.

---

## Common Pitfalls

* **No timeout** — the most common cause of cascading failure. `HttpClient` default is 100 s. *Always* override.
* **Trip threshold too aggressive** — circuit opens on random noise. Tune from real failure rate, not anxiety.
* **Trip threshold too loose** — by the time the breaker opens, threads are already starved. Faster trip + smaller bulkhead is usually better.
* **Retry inside circuit breaker counted as failures** — retry must be inside the breaker scope (one logical operation = one breaker decision), or layer order is wrong.
* **No fallback** — opening the breaker is supposed to enable a degraded response, not a different error. If you have nothing to fall back to, ask whether the call should be sync at all.
* **Shared pool for all downstreams** — every downstream shares one `HttpClient` / one pool. One slow downstream poisons all. Use a named `HttpClient` per downstream.
* **Bulkhead with infinite queue** — `QueueLimit = unbounded` defeats the purpose; requests pile up and time out anyway. Prefer `QueueLimit = 0` (instant rejection) or a tiny queue.

---

## References

* Michael Nygard — *Release It!* (2nd ed.), Stability Patterns (canonical reference).
* [Polly v8 documentation](https://www.pollydocs.org/)
* [.NET 8 — `Microsoft.Extensions.Resilience`](https://learn.microsoft.com/dotnet/core/resilience/)
* [AWS Builders' Library — Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
* Sam Newman — *Building Microservices* (2nd ed.), Chapter 12.
