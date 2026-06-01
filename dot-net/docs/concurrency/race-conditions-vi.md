# Race Condition — Giải pháp cho .NET

> Guide thực hành phát hiện và giải race condition, sắp theo **scope**: in-process, single-database, distributed, và chiến lược design-out. Ví dụ trong C# / .NET; nguyên lý translate trực tiếp sang Java, TypeScript, và Go.

> 🇻🇳 Phiên bản tiếng Việt. English: [`race-conditions.md`](./race-conditions.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Catalog giải pháp cho race condition qua bốn scope: synchronization in-process (lock, atomic, channel), DB concurrency control (optimistic, pessimistic, isolation level), distributed coordination (idempotency, distributed lock, saga, outbox), và chiến lược design loại bỏ race (immutability, single-writer, append-only).
- **Tại sao** — Race condition là nguồn phổ biến nhất của bug production "work trên máy tôi". Pick tool sai (distributed lock khi atomic UPDATE đủ, `lock` block qua `await`) thêm bug nhanh hơn remove.
- **Khi nào** — Mỗi lần nhiều thread/request/service touch shared state; thiết kế path read-modify-write; review PR introduce concurrency; debug failure intermittent.
- **Ở đâu** — Cắt qua mọi layer — domain (event-sourced aggregate), application (orchestration use-case), infrastructure (DB, broker), và deployment (sharding). Pair với `architectural-pattern/saga-vi.md` (workflow consistency), `architectural-pattern/outbox-vi.md` (publish reliability), và `enterprise-pattern/lifetimes-vi.md` (race captive-dependency DI).

---

## Mục lục

1. [Race Condition là gì?](#1-race-condition-là-gì)
2. [Giải pháp In-Process (Cùng App, Nhiều Thread)](#2-giải-pháp-in-process-cùng-app-nhiều-thread)
3. [Giải pháp Database-Level (Một DB, Nhiều Request)](#3-giải-pháp-database-level-một-db-nhiều-request)
4. [Giải pháp Distributed (Nhiều Service / Node)](#4-giải-pháp-distributed-nhiều-service--node)
5. [Chiến lược Design Loại bỏ Race](#5-chiến-lược-design-loại-bỏ-race)
6. [Cạm bẫy phổ biến](#6-cạm-bẫy-phổ-biến)
7. [Decision Matrix — Chọn Tool Đúng](#7-decision-matrix--chọn-tool-đúng)
8. [Tham khảo](#8-tham-khảo)

---

## 1. Race Condition là gì?

**Race condition** xảy ra khi correctness của program phụ thuộc vào *timing hoặc interleaving* của nhiều operation concurrent truy cập shared mutable state. Anatomy kinh điển:

```csharp
// Hai thread thực thi cái này concurrent
if (account.Balance >= amount)   // ← check
{
    account.Balance -= amount;   // ← act
}
```

Cả hai thread có thể pass check trước khi cái nào act → balance thành âm. Đây là pattern **TOCTOU** (Time-Of-Check to Time-Of-Use), race canonical.

**Ba thành phần của race:**

1. **Shared mutable state** — value visible với nhiều flow execution.
2. **Truy cập concurrent** — overlap về thời gian (thread, request, service).
3. **Sequence check-then-act** — action phụ thuộc check trước, nhưng truth của check có thể đổi trước khi action chạy.

Remove một trong ba và race biến mất. Giải pháp hoặc *serialize* truy cập (lock, transaction, single-writer), *make check-and-act atomic* (CAS, conditional UPDATE), hoặc *loại bỏ shared mutable state* (immutability, append-only).

---

## 2. Giải pháp In-Process (Cùng App, Nhiều Thread)

### 2.1 `lock` — workhorse cho code sync

```csharp
public sealed class BankAccount
{
    private readonly object _gate = new();
    private decimal _balance;

    public bool Withdraw(decimal amount)
    {
        lock (_gate)
        {
            if (_balance < amount) return false;
            _balance -= amount;
            return true;
        }
    }
}
```

* Luôn lock trên `private readonly object`, không bao giờ `this` hoặc type (`typeof(...)`) — cả hai leak ra caller external và risk deadlock.
* Giữ critical section **ngắn** — không I/O, không callback, không call method vào code unknown.
* `lock` không span `await` (xem [Cạm bẫy phổ biến](#6-cạm-bẫy-phổ-biến)).

### 2.2 `Interlocked` — atomic single-instruction operation

Khi operation là một read-modify-write trên integer / reference, skip lock:

```csharp
Interlocked.Increment(ref _counter);
Interlocked.Add(ref _total, amount);
Interlocked.Exchange(ref _ref, newRef);
Interlocked.CompareExchange(ref _state, newValue, expectedValue);
```

`CompareExchange` là building block cho thuật toán **lock-free**. Dùng sparingly — code lock-free dễ viết sai.

### 2.3 `SemaphoreSlim` — lock cho code async

`lock` không support `await` bên trong. Dùng `SemaphoreSlim` với count 1 như mutex async-aware:

```csharp
private readonly SemaphoreSlim _sem = new(initialCount: 1, maxCount: 1);

public async Task<Order> LoadOrCreateAsync(Guid id, CancellationToken ct)
{
    await _sem.WaitAsync(ct);
    try
    {
        if (_cache.TryGetValue(id, out var existing)) return existing;
        var order = await _db.Orders.FindAsync(new object[] { id }, ct);
        _cache[id] = order;
        return order;
    }
    finally { _sem.Release(); }
}
```

Cho concurrency cao hơn, set `initialCount` > 1 để throttle (vd., max 10 HTTP call in-flight).

### 2.4 Concurrent collection — thay vì `Dictionary<>` + lock

```csharp
var cache = new ConcurrentDictionary<Guid, Order>();
var order = cache.GetOrAdd(id, key => LoadFromDb(key));
cache.AddOrUpdate(id, addValue: created, updateValueFactory: (k, old) => updated);
```

Collection thread-safe khác: `ConcurrentQueue<T>`, `ConcurrentBag<T>`, `BlockingCollection<T>`, `ConcurrentStack<T>`.

### 2.5 `Channel<T>` — producer/consumer với backpressure

Cho pipeline thay vì lock. Consumer đọc trong single thread → không race:

```csharp
var channel = Channel.CreateBounded<WorkItem>(capacity: 100);

// Producer — nhiều
await channel.Writer.WriteAsync(item, ct);

// Single consumer
await foreach (var item in channel.Reader.ReadAllAsync(ct))
{
    await Process(item);    // không race; chỉ một thread đọc
}
```

### 2.6 `ReaderWriterLockSlim` — nhiều reader, rare writer

Chỉ đáng khi read vượt write nhiều. Nếu không, overhead bookkeeping beat plain `lock`.

```csharp
private readonly ReaderWriterLockSlim _rw = new();

public T Read()
{
    _rw.EnterReadLock();
    try { return _value; } finally { _rw.ExitReadLock(); }
}

public void Write(T newValue)
{
    _rw.EnterWriteLock();
    try { _value = newValue; } finally { _rw.ExitWriteLock(); }
}
```

### 2.7 `Lazy<T>` — initialization one-time thread-safe

Pattern double-checked-locking famously dễ sai. Đừng viết; dùng `Lazy<T>`:

```csharp
private static readonly Lazy<HttpClient> _client =
    new(() => new HttpClient(), LazyThreadSafetyMode.ExecutionAndPublication);
```

---

## 3. Giải pháp Database-Level (Một DB, Nhiều Request)

### 3.1 Optimistic concurrency — default ưu tiên

Thêm version column; DB reject UPDATE nếu ai khác đã đổi row từ khi bạn load:

```csharp
public sealed class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    [Timestamp] public byte[] RowVersion { get; set; } = null!;   // EF Core
}

// SaveChangesAsync throw DbUpdateConcurrencyException nếu RowVersion đã đổi
try
{
    order.Status = OrderStatus.Shipped;
    await db.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException)
{
    // Reload, reapply business decision, retry — hoặc surface tới user
}
```

Cho DB khác: `xmin` (PostgreSQL), `_etag` (Cosmos DB), `ROWVERSION` (SQL Server).

**Tốt nhất cho:** scenario low-conflict. Rẻ, không lock contention, fail nhanh.

### 3.2 Pessimistic locking — khi collision thường xuyên

Giữ DB lock trên row từ SELECT tới COMMIT:

```sql
-- Trong transaction
SELECT * FROM Inventory WHERE ProductId = @id FOR UPDATE;
-- ... business logic, rồi UPDATE trong cùng transaction ...
COMMIT;
```

EF Core (SQL Server):
```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
var inv = await db.Inventory
    .FromSqlInterpolated($"SELECT * FROM Inventory WITH (UPDLOCK, ROWLOCK) WHERE ProductId = {id}")
    .SingleAsync(ct);
inv.Stock -= 1;
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

**Tốt nhất cho:** hot row với contention cao (SKU phổ biến trong flash sale).

### 3.3 Atomic `UPDATE ... WHERE` — fix rẻ nhất

Cho phần lớn pattern read-modify-write, không cần lock — chỉ đặt precondition vào WHERE clause:

```sql
UPDATE Accounts
   SET Balance = Balance - @amount
 WHERE Id = @id
   AND Balance >= @amount;
```

```csharp
var rowsAffected = await db.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Accounts SET Balance = Balance - {amount} WHERE Id = {id} AND Balance >= {amount}", ct);

if (rowsAffected == 0)
    return Result.Fail("Insufficient funds or account not found");
```

DB engine guarantee row được lock trong khoảng UPDATE — atomic check + change.

### 3.4 Unique constraint — để DB nói no

Cho semantics "create only if not exists", define unique index và treat violation như success:

```sql
CREATE UNIQUE INDEX ux_payments_idempotency_key ON Payments(IdempotencyKey);
```

```csharp
try
{
    db.Payments.Add(new Payment { IdempotencyKey = key, ... });
    await db.SaveChangesAsync(ct);
}
catch (DbUpdateException ex) when (IsUniqueViolation(ex))
{
    // Request khác đã tạo — OK, return cái có sẵn
    return await db.Payments.SingleAsync(p => p.IdempotencyKey == key, ct);
}
```

### 3.5 Transaction isolation level

Default là `READ COMMITTED` (SQL Server, Postgres). Cho guarantee mạnh hơn:

| Level                   | Ngăn                                              | Cost                                       |
| ----------------------- | ------------------------------------------------- | ------------------------------------------ |
| `READ COMMITTED`        | Dirty read                                        | Default; rẻ                                |
| `REPEATABLE READ`       | Dirty + non-repeatable read                        | Giữ shared lock lâu hơn                    |
| `SERIALIZABLE`          | Cả ba (dirty, non-repeatable, phantom)            | Contention cao nhất; có thể deadlock       |
| `SNAPSHOT` (SQL Server) | Cả ba, qua row versioning                         | Overhead storage; an toàn hơn Serializable |

```csharp
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Serializable, ct);
```

**Quy tắc:** khởi đầu default, raise chỉ khi race thật xuất hiện. Serializable qua nhiều request sẽ deadlock dưới tải.

---

## 4. Giải pháp Distributed (Nhiều Service / Node)

### 4.1 Idempotency key — workhorse của distributed correctness

Client gửi header `Idempotency-Key` unique; server record key đã handled và short-circuit duplicate:

```csharp
[HttpPost("payments")]
public async Task<IActionResult> Charge(
    [FromHeader(Name = "Idempotency-Key")] string key,
    ChargePaymentCommand cmd,
    CancellationToken ct)
{
    var existing = await _idempotency.GetResultAsync(key, ct);
    if (existing is not null) return Ok(existing);   // đã làm rồi

    var result = await _mediator.Send(cmd, ct);
    await _idempotency.RecordAsync(key, result, ct);
    return Ok(result);
}
```

Pair với **Inbox pattern** cho message consumer — xem [`../architectural-pattern/outbox-vi.md`](../architectural-pattern/outbox-vi.md).

### 4.2 Optimistic concurrency trên event stream

Cho aggregate event-sourced, append với `expectedVersion`. Store reject append nếu ai khác đã thêm event từ khi bạn load:

```csharp
var stream = await _store.LoadAsync(streamId, ct);
var order = Order.Rehydrate(stream.Events);

order.Submit();                                      // produce event mới

try
{
    await _store.AppendAsync(streamId, order.PendingEvents, expectedVersion: stream.Version, ct);
}
catch (WrongExpectedVersionException)
{
    // Ai khác đã append trước → reload, replay business decision, retry
}
```

Cùng idea với DB optimistic concurrency, nhưng version là độ dài stream, không phải row column.

### 4.3 Distributed lock — chỉ khi cần thiết

Distributed lock **đắt** (network round-trip, lease renewal, fencing-token bookkeeping) và **fragile** (network partition có thể vi phạm mutual exclusion). Chỉ dùng khi không gì khác fit.

Redis với RedLock (Martin Kleppmann có [critique hợp lệ](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — dùng Redis Enterprise / RedLockNet chỉ cho lock low-stake):

```csharp
await using var redLock = await _redLockFactory.CreateLockAsync(
    resource: $"order:{orderId}",
    expiryTime: TimeSpan.FromSeconds(30),
    waitTime: TimeSpan.FromSeconds(5),
    retryTime: TimeSpan.FromMilliseconds(200));

if (redLock.IsAcquired)
{
    // Chỉ một node giữ lock tại một thời điểm
}
```

Alternative mạnh hơn: **etcd**, **ZooKeeper**, Azure **blob lease**, AWS **DynamoDB conditional write**. Tất cả cung cấp fencing token, mà Redis basic không.

### 4.4 Single-writer per partition — câu trả lời reliable nhất

Thay vì locking, **route mọi write cho một entity tới một consumer**:

```
Kafka topic: orders.commands
  Partition key = OrderId
  Consumer group: 3 consumer, mỗi cái own ~⅓ partition
  → Hai write cho cùng OrderId luôn đi cùng consumer
  → Không write concurrent tới cùng aggregate → không race
```

Cùng idea: **sharded service** (mỗi service own tenant range), **Orleans grain** (một logical thread per virtual actor).

### 4.5 Saga + compensation — workflow multi-service

Cho workflow span nhiều service, không thể take distributed transaction. Dùng Saga: mỗi step idempotent và có compensating action. Xem [`../architectural-pattern/saga-vi.md`](../architectural-pattern/saga-vi.md).

### 4.6 Outbox + Inbox — messaging exactly-once-effective

Cho "save DB và publish message" không race: write message vào outbox table *trong cùng DB transaction* với business data; process riêng publish. Consumer dedupe qua inbox. Xem [`../architectural-pattern/outbox-vi.md`](../architectural-pattern/outbox-vi.md).

### 4.7 CRDT — cho multi-master write genuine

Khi nhiều node phải accept write concurrent và merge sau (app offline-first, counter geo-distributed), **Conflict-free Replicated Data Type** mathematically guarantee convergence. Library: Yjs (JS), Automerge, Redis CRDT (Redis Enterprise).

---

## 5. Chiến lược Design Loại bỏ Race

Race condition rẻ nhất để fix là cái bạn không bao giờ tạo.

### 5.1 Immutability

Không shared mutable state → không race. C# record và `readonly struct`:

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other) => this with { Amount = Amount + other.Amount };
    // Trả về Money mới. Original không bao giờ mutate. Thread-safe by construction.
}
```

### 5.2 Single-writer ownership (actor model)

Một logical thread own state; mọi người khác gửi message. Orleans grain, Akka.NET actor, hoặc `Channel<T>` per entity:

```csharp
public sealed class OrderActor
{
    private readonly Channel<IOrderMessage> _mailbox = Channel.CreateUnbounded<IOrderMessage>();
    private Order _state;

    public Task<TResult> Send<TResult>(IOrderMessage<TResult> msg) => /* enqueue và await reply */;

    private async Task RunAsync(CancellationToken ct)
    {
        await foreach (var msg in _mailbox.Reader.ReadAllAsync(ct))
        {
            // Chỉ một message process tại một thời điểm → không race
            _state = msg.Apply(_state);
        }
    }
}
```

### 5.3 Append-only event log

Không bao giờ UPDATE; chỉ INSERT event với offset monotonically increasing. Conflict trên cùng record là impossible by construction. Đây là foundation của Event Sourcing.

### 5.4 Functional core / imperative shell

Pure function không thể có race — chúng không state. Push business logic vào pure function; concentrate side effect ở shell mỏng:

```csharp
// Functional core — pure decision logic
public static OrderDecision DecideOnWithdrawal(Account a, decimal amount) =>
    a.Balance >= amount
        ? OrderDecision.Approve(a with { Balance = a.Balance - amount })
        : OrderDecision.Reject("Insufficient funds");

// Imperative shell — orchestrate side effect; DB enforce concurrency
public async Task<Result> Withdraw(Guid accountId, decimal amount, CancellationToken ct)
{
    var rows = await _db.Database.ExecuteSqlInterpolatedAsync(
        $"UPDATE Accounts SET Balance = Balance - {amount} WHERE Id = {accountId} AND Balance >= {amount}", ct);
    return rows > 0 ? Result.Ok() : Result.Fail("Insufficient funds");
}
```

Xem [`../../../books/unit-testing-vladimir-khorikov-vi.md`](../../../books/unit-testing-vladimir-khorikov-vi.md) cho pattern functional-core đầy đủ.

### 5.5 Scope per-request

Trong ASP.NET Core, lifetime `Scoped` của DI tạo `DbContext` / handler tươi per request. Hai request concurrent *không share state by construction* — không race vì không share. Bẫy duy nhất: giữ Scoped service bên trong Singleton (bug **captive dependency** — xem [`../enterprise-pattern/lifetimes-vi.md`](../enterprise-pattern/lifetimes-vi.md)).

---

## 6. Cạm bẫy phổ biến

| Cạm bẫy                                          | Gì sai                                                                                                                       | Fix                                                                            |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **`lock` qua `await`**                           | Body `lock` complete synchronously; `await` bên trong là undefined behavior. Compiler *không* warn.                          | `SemaphoreSlim` (async-aware) hoặc restructure giữ async ngoài critical section. |
| **Double-checked locking không `volatile`**      | Không có memory barrier, thread khác có thể thấy object half-constructed.                                                    | `Lazy<T>` hoặc `LazyInitializer.EnsureInitialized`.                            |
| **`lock(this)` / `lock(typeof(X))`**             | Ai có reference tới object/type có thể lock trên nó → code external có thể deadlock bạn.                                     | `lock` trên `private readonly object _gate = new()`.                          |
| **Lock ordering deadlock**                       | Path A lock `_x` rồi `_y`; path B lock `_y` rồi `_x`. Sẽ deadlock dưới tải.                                                  | Luôn acquire lock theo global order nhất quán (vd., theo `Id`).                |
| **TOCTOU**                                       | `if (File.Exists(p)) File.Delete(p);` — file có thể biến mất giữa hai call.                                                   | Atomic operation (try-delete-and-catch); tránh check-then-act với shared resource. |
| **Cache stale sau write concurrent**             | Hai thread load v1, cả hai apply mutation khác, last one wins → change đầu mất.                                              | Optimistic concurrency với version; hoặc read-and-write trong transaction.     |
| **`Singleton` capture `Scoped`**                 | Service Singleton giữ reference `DbContext` per-request → request sau thấy state stale hoặc disposed.                        | Inject `IServiceScopeFactory`, tạo scope per operation.                        |
| **Share `HttpClient` OK — share `DbContext` không** | `HttpClient` thread-safe by design. `DbContext` **không** — không bao giờ share qua thread.                                  | Một `DbContext` per request / scope.                                          |
| **`Task.Run` cho "concurrency" trong request handler** | Fork request thread nhưng share `DbContext` và `HttpContext` — cả hai unsafe qua thread.                                  | Dùng `IServiceScopeFactory` tạo scope tươi bên trong `Task.Run`.               |
| **Retry không có idempotency**                   | Distributed retry duplicate side effect (charge đúp, email đúp).                                                              | Idempotency key + Outbox/Inbox.                                                |
| **Quên handle `OperationCanceledException`**     | Cancellation giữa critical section leak lock / để state partial.                                                              | Luôn wrap critical section trong `try/finally` để release; pass `CancellationToken` sâu. |
| **Retry aggressive amplify race**                | Mọi client retry đồng thời sau transient failure → thundering herd.                                                           | Exponential backoff + jitter.                                                  |

---

## 7. Decision Matrix — Chọn Tool Đúng

| Scenario                                              | Tool đầu reach                                                          |
| ----------------------------------------------------- | ---------------------------------------------------------------------- |
| Counter / flag                                        | `Interlocked`                                                          |
| Cache in-memory                                       | `ConcurrentDictionary` / `MemoryCache` / `HybridCache`                 |
| Init lazy one-time                                    | `Lazy<T>`                                                              |
| Critical section trong code **async**                 | `SemaphoreSlim`                                                        |
| Critical section trong code **sync**                  | `lock` trên private object                                             |
| Pipeline producer-consumer                            | `Channel<T>`                                                           |
| UPDATE single row                                     | Atomic `UPDATE ... WHERE` với precondition                             |
| Write aggregate với contention thấp                   | Optimistic concurrency (`RowVersion` / version event-stream)           |
| Write aggregate với contention **cao**                | Pessimistic `FOR UPDATE` / `WITH (UPDLOCK, ROWLOCK)` + transaction ngắn|
| "Create only if not exists"                           | Unique constraint + handle exception duplicate-key                     |
| Transaction business multi-step qua service           | Saga + Outbox + idempotency key                                        |
| HTTP API safe để retry                                | Header idempotency key                                                 |
| "Chỉ một worker làm cái này qua N node"               | Distributed lock (Redis RedLock / etcd / Azure blob lease)             |
| "Chỉ một writer per entity qua N node"                | Partition theo entity id (Kafka partition / sharded service / Orleans grain) |
| Write multi-master geo-distributed                    | CRDT                                                                   |
| Bất cứ đâu có thể                                     | Immutability, append-only, single-writer ownership                     |

### Quy tắc 80/20

Phần lớn race condition production trong .NET hiện đại được giải bằng **bốn** tool:

1. **Optimistic concurrency trên DB** (RowVersion / ETag).
2. **Idempotency key** ở API edge và message consumer.
3. **`SemaphoreSlim`** cho async critical section.
4. **Append event** thay vì update state.

Reach **distributed lock** chỉ khi bốn cái này không fit — và sau đó verify lock provider support **fencing token** trước khi trust nó cho cái gì quan trọng.

---

## 8. Tham khảo

* Brian Goetz et al. — *Java Concurrency in Practice* (2006). Vẫn là text canonical; nguyên lý translate trực tiếp sang .NET.
* Stephen Cleary — *Concurrency in C# Cookbook* (2nd ed., 2019). Recipe thực hành cho async, parallel, dataflow.
* Joseph Albahari — [*Threading in C#*](https://www.albahari.com/threading/) (sách online miễn phí). Reference concurrency .NET được trích dẫn nhiều nhất.
* Martin Kleppmann — *Designing Data-Intensive Applications* (2017). Chương 7 (Transaction) và Chương 9 (Consistency và Consensus) cho race condition distributed-system.
* Martin Kleppmann — [*How to do distributed locking*](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) (2016). Tại sao naïve Redis lock không safe; case cho fencing token.
* Microsoft Docs — [Concurrency conflict trong EF Core](https://learn.microsoft.com/en-us/ef/core/saving/concurrency).
* Microsoft Docs — [`System.Threading.Channels`](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels).
* Cross-reference trong repo này: [`../architectural-pattern/saga-vi.md`](../architectural-pattern/saga-vi.md), [`../architectural-pattern/outbox-vi.md`](../architectural-pattern/outbox-vi.md), [`../architectural-pattern/event-sourcing-vi.md`](../architectural-pattern/event-sourcing-vi.md), [`../enterprise-pattern/lifetimes-vi.md`](../enterprise-pattern/lifetimes-vi.md), [`../../../books/unit-testing-vladimir-khorikov-vi.md`](../../../books/unit-testing-vladimir-khorikov-vi.md).
