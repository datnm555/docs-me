# Race Conditions — Solutions for .NET

> A practical guide to detecting and solving race conditions, ordered by **scope**: in-process, single-database, distributed, and design-out strategies. Examples in C# / .NET; the principles translate to Java, TypeScript, and Go.

> 🇻🇳 Vietnamese version: [`race-conditions-vi.md`](./race-conditions-vi.md)

---

## Quick Reference (What · Why · When · Where)

- **What** — A catalog of solutions for race conditions across four scopes: in-process synchronization (locks, atomics, channels), DB concurrency control (optimistic, pessimistic, isolation levels), distributed coordination (idempotency, distributed locks, sagas, outbox), and design strategies that eliminate races (immutability, single-writer, append-only).
- **Why** — Race conditions are the single most common source of "works on my machine" production bugs. Picking the wrong tool (a distributed lock when an atomic UPDATE would work, a `lock` block across an `await`) adds bugs faster than it removes them.
- **When** — Every time multiple threads/requests/services touch shared state; designing read-modify-write paths; reviewing PRs that introduce concurrency; debugging intermittent failures.
- **Where** — Cuts across all layers — domain (event-sourced aggregates), application (use-case orchestration), infrastructure (DB, broker), and deployment (sharding). Pair with `architectural-pattern/saga.md` (workflow consistency), `architectural-pattern/outbox.md` (publish reliability), and `enterprise-pattern/lifetimes.md` (DI captive-dependency races).

---

## Table of Contents

1. [What Is a Race Condition?](#1-what-is-a-race-condition)
2. [In-Process Solutions (Same App, Multiple Threads)](#2-in-process-solutions-same-app-multiple-threads)
3. [Database-Level Solutions (Single DB, Multiple Requests)](#3-database-level-solutions-single-db-multiple-requests)
4. [Distributed Solutions (Multiple Services / Nodes)](#4-distributed-solutions-multiple-services--nodes)
5. [Design Strategies That Eliminate Races](#5-design-strategies-that-eliminate-races)
6. [Common Pitfalls](#6-common-pitfalls)
7. [Decision Matrix — Picking the Right Tool](#7-decision-matrix--picking-the-right-tool)
8. [References](#8-references)

---

## 1. What Is a Race Condition?

A **race condition** occurs when the correctness of a program depends on the *timing or interleaving* of multiple concurrent operations accessing shared mutable state. The classic anatomy:

```csharp
// Two threads execute this concurrently
if (account.Balance >= amount)   // ← check
{
    account.Balance -= amount;   // ← act
}
```

Both threads can pass the check before either acts → the balance goes negative. This is the **TOCTOU** (Time-Of-Check to Time-Of-Use) pattern, the canonical race.

**Three ingredients of a race:**

1. **Shared mutable state** — a value visible to multiple flows of execution.
2. **Concurrent access** — overlapping in time (threads, requests, services).
3. **A check-then-act sequence** — the action depends on the prior check, but the check's truth can change before the action runs.

Remove any one of the three and the race disappears. Solutions either *serialize* access (locks, transactions, single-writer), *make the check-and-act atomic* (CAS, conditional UPDATE), or *eliminate shared mutable state* (immutability, append-only).

---

## 2. In-Process Solutions (Same App, Multiple Threads)

### 2.1 `lock` — the workhorse for sync code

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

* Always lock on a `private readonly object`, never `this` or a type (`typeof(...)`) — both leak to external callers and risk deadlocks.
* Keep the critical section **short** — no I/O, no callbacks, no method calls into unknown code.
* `lock` can't span `await` (see [Common Pitfalls](#6-common-pitfalls)).

### 2.2 `Interlocked` — atomic single-instruction operations

When the operation is one read-modify-write on an integer / reference, skip the lock:

```csharp
Interlocked.Increment(ref _counter);
Interlocked.Add(ref _total, amount);
Interlocked.Exchange(ref _ref, newRef);
Interlocked.CompareExchange(ref _state, newValue, expectedValue);
```

`CompareExchange` is the building block for **lock-free** algorithms. Use it sparingly — lock-free code is easy to write incorrectly.

### 2.3 `SemaphoreSlim` — locks for async code

`lock` doesn't support `await` inside. Use `SemaphoreSlim` with count 1 as an async-aware mutex:

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

For higher concurrency, set `initialCount` > 1 to throttle (e.g., max 10 in-flight HTTP calls).

### 2.4 Concurrent collections — instead of `Dictionary<>` + lock

```csharp
var cache = new ConcurrentDictionary<Guid, Order>();
var order = cache.GetOrAdd(id, key => LoadFromDb(key));
cache.AddOrUpdate(id, addValue: created, updateValueFactory: (k, old) => updated);
```

Other thread-safe collections: `ConcurrentQueue<T>`, `ConcurrentBag<T>`, `BlockingCollection<T>`, `ConcurrentStack<T>`.

### 2.5 `Channel<T>` — producer/consumer with backpressure

For pipelines instead of locks. The consumer reads in a single thread → no race:

```csharp
var channel = Channel.CreateBounded<WorkItem>(capacity: 100);

// Producers — many
await channel.Writer.WriteAsync(item, ct);

// Single consumer
await foreach (var item in channel.Reader.ReadAllAsync(ct))
{
    await Process(item);    // no race; only one thread reads
}
```

### 2.6 `ReaderWriterLockSlim` — many readers, rare writes

Only worth it when reads vastly outnumber writes. Otherwise, the bookkeeping overhead beats a plain `lock`.

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

### 2.7 `Lazy<T>` — thread-safe one-time initialization

The double-checked-locking pattern is famously easy to get wrong. Don't write it; use `Lazy<T>`:

```csharp
private static readonly Lazy<HttpClient> _client =
    new(() => new HttpClient(), LazyThreadSafetyMode.ExecutionAndPublication);
```

---

## 3. Database-Level Solutions (Single DB, Multiple Requests)

### 3.1 Optimistic concurrency — preferred default

Add a version column; the DB rejects the UPDATE if anyone else changed the row since you loaded it:

```csharp
public sealed class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    [Timestamp] public byte[] RowVersion { get; set; } = null!;   // EF Core
}

// SaveChangesAsync throws DbUpdateConcurrencyException if RowVersion changed
try
{
    order.Status = OrderStatus.Shipped;
    await db.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException)
{
    // Reload, reapply business decision, retry — or surface to user
}
```

For other DBs: `xmin` (PostgreSQL), `_etag` (Cosmos DB), `ROWVERSION` (SQL Server).

**Best for:** low-conflict scenarios. Cheap, no lock contention, fails fast.

### 3.2 Pessimistic locking — when collisions are frequent

Hold a DB lock on the row from SELECT to COMMIT:

```sql
-- Inside a transaction
SELECT * FROM Inventory WHERE ProductId = @id FOR UPDATE;
-- ... business logic, then UPDATE in same transaction ...
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

**Best for:** hot rows with high contention (popular SKU during a flash sale).

### 3.3 Atomic `UPDATE ... WHERE` — the cheapest fix

For most read-modify-write patterns, you don't need a lock — just put the precondition in the WHERE clause:

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

The DB engine guarantees the row is locked for the duration of the UPDATE — atomic check + change.

### 3.4 Unique constraints — let the DB say no

For "create only if not exists" semantics, define a unique index and treat the violation as success:

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
    // Another request already created it — that's fine, return the existing one
    return await db.Payments.SingleAsync(p => p.IdempotencyKey == key, ct);
}
```

### 3.5 Transaction isolation levels

Default is `READ COMMITTED` (SQL Server, Postgres). For tougher guarantees:

| Level                   | Prevents                                          | Cost                                  |
| ----------------------- | ------------------------------------------------- | ------------------------------------- |
| `READ COMMITTED`        | Dirty reads                                       | Default; cheap                        |
| `REPEATABLE READ`       | Dirty + non-repeatable reads                      | Holds shared locks longer             |
| `SERIALIZABLE`          | All three (dirty, non-repeatable, phantom)        | Highest contention; may deadlock      |
| `SNAPSHOT` (SQL Server) | All three, via row versioning                     | Storage overhead; safer than Serializable |

```csharp
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Serializable, ct);
```

**Rule of thumb:** start at default, raise only when an actual race appears. Serializable across many requests will deadlock under load.

---

## 4. Distributed Solutions (Multiple Services / Nodes)

### 4.1 Idempotency keys — the workhorse of distributed correctness

The client sends a unique `Idempotency-Key` header; the server records keys it has handled and short-circuits duplicates:

```csharp
[HttpPost("payments")]
public async Task<IActionResult> Charge(
    [FromHeader(Name = "Idempotency-Key")] string key,
    ChargePaymentCommand cmd,
    CancellationToken ct)
{
    var existing = await _idempotency.GetResultAsync(key, ct);
    if (existing is not null) return Ok(existing);   // already done

    var result = await _mediator.Send(cmd, ct);
    await _idempotency.RecordAsync(key, result, ct);
    return Ok(result);
}
```

Pair with the **Inbox pattern** for message consumers — see [`../architectural-pattern/outbox.md`](../architectural-pattern/outbox.md).

### 4.2 Optimistic concurrency on the event stream

For event-sourced aggregates, append with `expectedVersion`. The store rejects the append if anyone else added events since you loaded:

```csharp
var stream = await _store.LoadAsync(streamId, ct);
var order = Order.Rehydrate(stream.Events);

order.Submit();                                      // produces new events

try
{
    await _store.AppendAsync(streamId, order.PendingEvents, expectedVersion: stream.Version, ct);
}
catch (WrongExpectedVersionException)
{
    // Someone else appended first → reload, replay business decision, retry
}
```

Same idea as DB optimistic concurrency, but the version is the stream length, not a row column.

### 4.3 Distributed lock — only when necessary

A distributed lock is **expensive** (network round-trips, lease renewal, fencing-token bookkeeping) and **fragile** (network partitions can violate mutual exclusion). Use only when nothing else fits.

Redis with RedLock (Martin Kleppmann has [valid critiques](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — use Redis Enterprise / RedLockNet only for low-stakes locks):

```csharp
await using var redLock = await _redLockFactory.CreateLockAsync(
    resource: $"order:{orderId}",
    expiryTime: TimeSpan.FromSeconds(30),
    waitTime: TimeSpan.FromSeconds(5),
    retryTime: TimeSpan.FromMilliseconds(200));

if (redLock.IsAcquired)
{
    // Only one node holds the lock at a time
}
```

Stronger alternatives: **etcd**, **ZooKeeper**, Azure **blob lease**, AWS **DynamoDB conditional write**. All provide fencing tokens, which Redis basic does not.

### 4.4 Single-writer per partition — the most reliable answer

Instead of locking, **route all writes for one entity to one consumer**:

```
Kafka topic: orders.commands
  Partition key = OrderId
  Consumer group: 3 consumers, each owns ~⅓ of partitions
  → Two writes for the same OrderId always go to the same consumer
  → No concurrent writes to the same aggregate → no race
```

Same idea: **sharded services** (each service owns a tenant range), **Orleans grains** (one logical thread per virtual actor).

### 4.5 Saga + compensations — multi-service workflows

For workflows spanning multiple services, you can't take a distributed transaction. Use a Saga: each step is idempotent and has a compensating action. See [`../architectural-pattern/saga.md`](../architectural-pattern/saga.md).

### 4.6 Outbox + Inbox — exactly-once-effective messaging

For "save to DB and publish a message" without races: write the message to an outbox table *in the same DB transaction* as the business data; a separate process publishes. Consumers dedupe via an inbox. See [`../architectural-pattern/outbox.md`](../architectural-pattern/outbox.md).

### 4.7 CRDTs — for genuine multi-master writes

When multiple nodes must accept writes concurrently and merge later (offline-first apps, geo-distributed counters), **Conflict-free Replicated Data Types** mathematically guarantee convergence. Libraries: Yjs (JS), Automerge, Redis CRDT (Redis Enterprise).

---

## 5. Design Strategies That Eliminate Races

The cheapest race condition to fix is the one you never created.

### 5.1 Immutability

No shared mutable state → no race. C# records and `readonly struct`:

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other) => this with { Amount = Amount + other.Amount };
    // Returns a new Money. The original is never mutated. Thread-safe by construction.
}
```

### 5.2 Single-writer ownership (actor model)

One logical thread owns the state; everyone else sends messages. Orleans grains, Akka.NET actors, or a `Channel<T>` per entity:

```csharp
public sealed class OrderActor
{
    private readonly Channel<IOrderMessage> _mailbox = Channel.CreateUnbounded<IOrderMessage>();
    private Order _state;

    public Task<TResult> Send<TResult>(IOrderMessage<TResult> msg) => /* enqueue and await reply */;

    private async Task RunAsync(CancellationToken ct)
    {
        await foreach (var msg in _mailbox.Reader.ReadAllAsync(ct))
        {
            // Only one message processed at a time → no race
            _state = msg.Apply(_state);
        }
    }
}
```

### 5.3 Append-only event log

Never UPDATE; only INSERT events with monotonically increasing offsets. Conflicts on the same record are impossible by construction. This is the foundation of Event Sourcing.

### 5.4 Functional core / imperative shell

Pure functions can't have races — they have no state. Push business logic into pure functions; concentrate side effects in a thin shell:

```csharp
// Functional core — pure decision logic
public static OrderDecision DecideOnWithdrawal(Account a, decimal amount) =>
    a.Balance >= amount
        ? OrderDecision.Approve(a with { Balance = a.Balance - amount })
        : OrderDecision.Reject("Insufficient funds");

// Imperative shell — orchestrates the side effect; DB enforces concurrency
public async Task<Result> Withdraw(Guid accountId, decimal amount, CancellationToken ct)
{
    var rows = await _db.Database.ExecuteSqlInterpolatedAsync(
        $"UPDATE Accounts SET Balance = Balance - {amount} WHERE Id = {accountId} AND Balance >= {amount}", ct);
    return rows > 0 ? Result.Ok() : Result.Fail("Insufficient funds");
}
```

See [`../../../books/unit-testing-vladimir-khorikov.md`](../../../books/unit-testing-vladimir-khorikov.md) for the full functional-core pattern.

### 5.5 Per-request scope

In ASP.NET Core, DI's `Scoped` lifetime creates a fresh `DbContext` / handler per request. Two concurrent requests *don't share state by construction* — there's no race because there's no sharing. The only trap: holding a Scoped service inside a Singleton (the **captive dependency** bug — see [`../enterprise-pattern/lifetimes.md`](../enterprise-pattern/lifetimes.md)).

---

## 6. Common Pitfalls

| Pitfall                                          | What goes wrong                                                                                                             | Fix                                                                            |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **`lock` across `await`**                        | The `lock` body completes synchronously; `await` inside is undefined behavior. The compiler does *not* warn.                | `SemaphoreSlim` (async-aware) or restructure to keep async outside the critical section. |
| **Double-checked locking without `volatile`**    | Without a memory barrier, another thread can see a half-constructed object.                                                  | `Lazy<T>` or `LazyInitializer.EnsureInitialized`.                              |
| **`lock(this)` / `lock(typeof(X))`**             | Anyone who has a reference to the object/type can lock on it → external code can deadlock you.                              | `lock` on a `private readonly object _gate = new()`.                          |
| **Lock ordering deadlock**                       | Path A locks `_x` then `_y`; path B locks `_y` then `_x`. They'll deadlock under load.                                       | Always acquire locks in a consistent global order (e.g., by `Id`).             |
| **TOCTOU**                                       | `if (File.Exists(p)) File.Delete(p);` — file can vanish between the two calls.                                                | Atomic operation (try-delete-and-catch); avoid check-then-act with shared resources. |
| **Stale cache after concurrent write**           | Two threads load v1, both apply different mutations, the last one wins → first's change lost.                                | Optimistic concurrency with version; or read-and-write inside a transaction.   |
| **`Singleton` capturing `Scoped`**               | Singleton service holds a per-request `DbContext` reference → next request sees stale or disposed state.                     | Inject `IServiceScopeFactory`, create a scope per operation.                   |
| **Shared `HttpClient` is fine — shared `DbContext` is not** | `HttpClient` is thread-safe by design. `DbContext` is **not** — never share across threads.                       | One `DbContext` per request / scope.                                           |
| **`Task.Run` for "concurrency" in a request handler** | Forks the request thread but shares `DbContext` and `HttpContext` — both unsafe across threads.                          | Use `IServiceScopeFactory` to create a fresh scope inside the `Task.Run`.      |
| **Retrying without idempotency**                 | Distributed retry duplicates side effects (double charge, double email).                                                     | Idempotency keys + Outbox/Inbox.                                                |
| **Forgetting to handle `OperationCanceledException`** | A cancellation in the middle of a critical section leaks locks / leaves partial state.                                   | Always wrap critical sections in `try/finally` to release; pass `CancellationToken` deeply. |
| **Aggressive retries amplify races**             | All clients retry simultaneously after a transient failure → thundering herd.                                                 | Exponential backoff + jitter.                                                  |

---

## 7. Decision Matrix — Picking the Right Tool

| Scenario                                              | First tool to reach for                                                |
| ----------------------------------------------------- | ---------------------------------------------------------------------- |
| Counter / flag                                        | `Interlocked`                                                          |
| In-memory cache                                       | `ConcurrentDictionary` / `MemoryCache` / `HybridCache`                 |
| One-time lazy init                                    | `Lazy<T>`                                                              |
| Critical section in **async** code                    | `SemaphoreSlim`                                                        |
| Critical section in **sync** code                     | `lock` on a private object                                             |
| Producer-consumer pipeline                            | `Channel<T>`                                                           |
| Single-row UPDATE                                     | Atomic `UPDATE ... WHERE` with precondition                            |
| Aggregate write with low contention                   | Optimistic concurrency (`RowVersion` / event-stream version)           |
| Aggregate write with **high** contention              | Pessimistic `FOR UPDATE` / `WITH (UPDLOCK, ROWLOCK)` + short transaction|
| "Create only if not exists"                           | Unique constraint + handle duplicate-key exception                     |
| Multi-step business transaction across services       | Saga + Outbox + idempotency keys                                       |
| HTTP API safe to retry                                | Idempotency key header                                                 |
| "Only one worker does this across N nodes"            | Distributed lock (Redis RedLock / etcd / Azure blob lease)             |
| "Only one writer per entity across N nodes"           | Partition by entity id (Kafka partition / sharded service / Orleans grain) |
| Multi-master geo-distributed write                    | CRDT                                                                   |
| Anywhere you can                                      | Immutability, append-only, single-writer ownership                     |

### The 80/20 rule

Most production race conditions in modern .NET are solved with **four** tools:

1. **Optimistic concurrency on the DB** (RowVersion / ETag).
2. **Idempotency keys** at API edges and message consumers.
3. **`SemaphoreSlim`** for async critical sections.
4. **Appending events** instead of updating state.

Reach for **distributed locks** only when these four don't fit — and then verify the lock provider supports **fencing tokens** before trusting it for anything that matters.

---

## 8. References

* Brian Goetz et al. — *Java Concurrency in Practice* (2006). Still the canonical text; the principles translate directly to .NET.
* Stephen Cleary — *Concurrency in C# Cookbook* (2nd ed., 2019). Practical recipes for async, parallel, dataflow.
* Joseph Albahari — [*Threading in C#*](https://www.albahari.com/threading/) (free online book). The most-cited .NET concurrency reference.
* Martin Kleppmann — *Designing Data-Intensive Applications* (2017). Chapter 7 (Transactions) and Chapter 9 (Consistency and Consensus) for distributed-system race conditions.
* Martin Kleppmann — [*How to do distributed locking*](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) (2016). Why naïve Redis locks aren't safe; the case for fencing tokens.
* Microsoft Docs — [Concurrency conflicts in EF Core](https://learn.microsoft.com/en-us/ef/core/saving/concurrency).
* Microsoft Docs — [`System.Threading.Channels`](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels).
* Cross-reference in this repo: [`../architectural-pattern/saga.md`](../architectural-pattern/saga.md), [`../architectural-pattern/outbox.md`](../architectural-pattern/outbox.md), [`../architectural-pattern/event-sourcing.md`](../architectural-pattern/event-sourcing.md), [`../enterprise-pattern/lifetimes.md`](../enterprise-pattern/lifetimes.md), [`../../../books/unit-testing-vladimir-khorikov.md`](../../../books/unit-testing-vladimir-khorikov.md).
