# MongoDB

> The document database that defined NoSQL for a generation. Flexible schema, denormalized data shapes, horizontal-scale-out via sharding, and a query language designed for JSON. Modern Mongo (4.0+) added multi-document transactions, narrowing the gap with relational DBs — but the optimal use case is still **document-shaped data accessed by document boundaries**, not relational data forced into BSON.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — A distributed document database. Stores **BSON** (binary JSON) documents in **collections**. No fixed schema by default (though optional JSON Schema validation exists). Native horizontal scaling via **sharding**. Replica sets provide HA + read scaling.
- **Why** — When your data is document-shaped (each entity is read/written as a unit, has variable fields, deeply nested arrays), Mongo matches the shape. Sharding is built-in, not bolted on. Replica sets are operationally simpler than most relational HA setups.
- **When** — Catalog / CMS data with heterogeneous fields. User profiles. Event/activity streams. Anywhere a relational design produces dozens of joined tables for one logical entity. **Not** the right pick for highly relational analytics or strict cross-document transactional workloads.
- **Where** — MongoDB Atlas (managed, dominant in cloud), self-host (Community / Enterprise), Azure Cosmos DB Mongo API (different engine under the hood — *not* real Mongo). From .NET: official **MongoDB.Driver**.
- **How** — Model **per query**, not per entity. Denormalize what's read together. Use embedded documents for 1:few; references for 1:many-with-bounded-cardinality; collections + lookups for true many-to-many. Pick the right write concern + read concern per operation. Plan shard key before hitting the wall (changing it is painful).

---

## Strengths

* **Flexible schema.** Add a field without a migration. Heterogeneous shapes coexist in the same collection.
* **Document model fits real-world entities** when the entity *is* a document (article, order with items, user with embedded preferences).
* **Native sharding.** Built-in horizontal write scaling, not "add Citus and rewrite your app."
* **Replica sets.** Multi-replica, automatic failover, configurable read preferences. Simpler than MySQL semi-sync or PG streaming + Patroni.
* **Aggregation pipeline.** A staged pipeline language for complex transformations — closer to LINQ / SQL CTE than to traditional MapReduce.
* **Atlas** (managed) is genuinely good — backup, monitoring, search, vector, encryption-at-rest all integrated.
* **Change Streams** — native CDC. Subscribe to inserts/updates without external tooling.

## Weaknesses

* **Cross-document ACID is recent.** Multi-document transactions added in 4.0 (replica set), 4.2 (sharded). Possible but expensive — use sparingly, prefer single-document atomicity.
* **No joins (well, `$lookup`).** Aggregation `$lookup` exists but is expensive — Mongo is designed assuming you'd denormalize instead.
* **Schema flexibility cuts both ways.** Without validation, drift accumulates and queries become defensive. Use **JSON Schema validation** on collections in production.
* **Shard key is forever-ish.** Choosing a bad shard key (low cardinality, monotonically increasing) creates hot shards. Resharding exists (5.0+) but it's an operation, not a casual change.
* **Memory hungry.** WiredTiger cache wants RAM proportional to working set. Mis-sized clusters degrade silently.
* **Cosmos DB's "Mongo API" is not Mongo.** Different storage engine, different consistency model, different cost model. Don't conflate.

---

## Concurrency & Consistency Model

### Single-document atomicity is the foundation

Updates to a **single document** are atomic — including arrays inside it. This is the lever Mongo bets on: design your schema so the common operation touches *one document*, and you get ACID-like semantics for free.

### Multi-document transactions

Available since 4.0 (replica sets) and 4.2 (sharded clusters). Use them for genuine cross-document atomic writes — but they cost more than single-document writes and have time limits (default 60s). **Treat them as the exception, not the default.**

```csharp
using var session = await client.StartSessionAsync(cancellationToken: ct);
session.StartTransaction();
try
{
    await orders.InsertOneAsync(session, order, cancellationToken: ct);
    await inventory.UpdateOneAsync(session, filter, dec, cancellationToken: ct);
    await session.CommitTransactionAsync(ct);
}
catch
{
    await session.AbortTransactionAsync(ct);
    throw;
}
```

### Write Concern — how durable is "written"?

| Write Concern         | Meaning                                                                        |
| --------------------- | ------------------------------------------------------------------------------ |
| `w: 0`                | Fire and forget. **Never use** outside of telemetry-like workloads.            |
| `w: 1` (default)       | Acked by primary. Primary failure before replication → data loss.              |
| `w: "majority"`        | Acked by a majority of the replica set. **The right default** for durability.   |
| `w: <n>`              | Wait for n nodes. Explicit majority is usually preferable.                      |
| `j: true`             | Wait for write to be journaled (durable on disk).                              |

### Read Concern — how fresh is "read"?

| Read Concern         | Meaning                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| `local` (default)     | Read from the local node, may not be majority-acked. Fast, weakest.              |
| `available`          | Like `local` but allows orphaned reads during chunk migrations. Avoid generally.  |
| `majority`           | Returns only data that's majority-committed. **Pair with `w: majority` for read-your-own-writes** in many topologies. |
| `linearizable`       | Strongest — guarantees the read reflects all majority-committed writes before it. Primary-only, slow. |
| `snapshot`           | Transaction-time snapshot consistency.                                            |

### Read Preference

Where the read is routed:

* `primary` (default) — strict consistency for that node.
* `primaryPreferred` — primary first, fall back to secondary if down.
* `secondary` / `secondaryPreferred` — scale read off replicas. **Eventual** consistency.
* `nearest` — lowest latency. Mix of primary/secondaries.

> Combining `w: majority` + `readConcern: majority` + `readPreference: primary` gets you the closest thing to "single-server" semantics in a distributed cluster. Defaults to use unless you have a reason not to.

### Replica Set Mechanics

* 3-node minimum for HA (or 2 data + 1 arbiter).
* Primary handles writes; secondaries replicate the oplog (Mongo's WAL equivalent).
* On primary failure, secondaries elect a new primary in ~10–30 seconds via Raft-like consensus.
* Read-only secondaries can serve reads (with the consistency trade-offs above).

### Sharding Mechanics

* Cluster has **shards** (each shard is itself a replica set), **config servers** (metadata), and **mongos** routers (clients connect to these).
* Documents are partitioned by **shard key**.
* Common shard-key strategies:
  * **Hashed** (`{ _id: "hashed" }`) — even distribution, no range queries.
  * **Ranged** on a high-cardinality field — supports range queries, risks hot shards.
  * **Compound** (e.g., `{ tenantId: 1, _id: 1 }`) — locality per tenant + spread.
* **Monotonically increasing shard keys** (e.g., raw timestamp) cause **hot last shard**. Use hashed or compound with a prefix.

---

## Schema & Data Model

The single most important Mongo skill: **model around how the data is read.**

* **Embed** when:
  * The child is owned by the parent (order's line items, comment thread for a post).
  * 1:few cardinality.
  * Always read together.
* **Reference** (store the parent ID inside the child) when:
  * 1:many with unbounded growth (user → activity log).
  * The child is queried independently.
  * The child is large enough to bloat the parent document.
* **Avoid** documents > 1 MB. Mongo's hard limit is 16 MB per document — but performance degrades well before that.

### Schema validation

```javascript
db.createCollection("orders", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["customerId", "items", "total"],
    properties: {
      customerId: { bsonType: "objectId" },
      total:      { bsonType: "decimal", minimum: 0 },
    }
  }}
});
```

Use this in production. The flexibility is for prototyping; production should pin the shape.

### Indexes

* `_id` is indexed automatically (B-tree on primary).
* Single-field, compound, multikey (array), text, geo (2dsphere), hashed, partial, TTL.
* `explain()` plans look like SQL plans. Read them.
* Indexes cost on write — too many indexes slow inserts and consume RAM.

---

## Operational Characteristics

| Concern             | Reality                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Vertical scaling    | Good, but the sweet spot is medium-to-large replica sets with adequate RAM.           |
| Horizontal scaling  | **Native via sharding.** Mongo's biggest differentiator.                              |
| Backup              | Atlas: continuous + snapshots. Self-host: `mongodump`, snapshot from a secondary.     |
| Monitoring          | Atlas built-in; self-host: Prometheus exporters + MongoDB Ops Manager.                |
| Schema migrations    | Online — but you write the migration as a script that updates documents (no `ALTER`). |
| Driver maturity     | Excellent in major languages; the .NET driver is first-class.                         |

---

## .NET Integration (short)

```csharp
// Registration
services.AddSingleton<IMongoClient>(_ => new MongoClient(connStr));
services.AddScoped(s => s.GetRequiredService<IMongoClient>().GetDatabase("appdb"));

// Usage
public sealed class OrderRepository
{
    private readonly IMongoCollection<Order> _orders;

    public OrderRepository(IMongoDatabase db) =>
        _orders = db.GetCollection<Order>("orders");

    public Task PlaceAsync(Order o, CancellationToken ct) =>
        _orders.InsertOneAsync(o, cancellationToken: ct);

    public Task<List<Order>> FindByCustomerAsync(Guid customerId, CancellationToken ct) =>
        _orders.Find(o => o.CustomerId == customerId).ToListAsync(ct);
}
```

* `MongoClient` is **thread-safe and connection-pooling** — register as singleton.
* Strongly-typed `IMongoCollection<T>` lets you use LINQ — but it's a subset of LINQ; check what gets translated.
* For high throughput, batch via `InsertManyAsync` / `BulkWriteAsync`.
* `IClientSessionHandle` for multi-document transactions.
* `[BsonId]`, `[BsonElement]`, `[BsonIgnoreIfNull]` attributes for explicit mapping; or `BsonClassMap.RegisterClassMap` for centralized config.

---

## When to Choose / When Not

**Choose when:**

* Your data *is* document-shaped — entities have variable fields and natural document boundaries.
* You need horizontal write scaling without inventing a sharding layer.
* Catalog / CMS / activity streams / IoT telemetry / event logs.
* You're using Atlas and want a fully-managed experience with integrated search + vector + analytics.

**Don't choose when:**

* The data is intrinsically relational (lots of joins, lots of cross-entity transactions). A relational DB will be simpler.
* You need strict ACID transactions across many entities by default. Mongo can, but the design tax is real.
* You need rich SQL — `$lookup` + aggregation pipeline is powerful but not SQL.
* Compliance / regulation requires features Atlas's "Mongo API" alternatives don't have.

---

## Pitfalls

* **Shard key picked late** — monotonic shard keys create hot shards; low-cardinality shard keys can't split. Plan early.
* **`w: 1` defaults** treated as durable — they aren't on failover. Use `w: majority` for anything that matters.
* **Mixing field types** in the same field (`"price": 99` here, `"price": "99"` there). Storage allows it; queries silently fail. Use schema validation.
* **Unbounded array growth** (e.g., adding all comments to a post document forever) — the document keeps reallocating and indexing slows. Cap arrays or split into a child collection.
* **Reading from secondary expecting freshness** — secondaries lag. Use `readConcern: majority` + `readPreference: primary` for freshness.
* **`$lookup` over big collections** — much slower than the equivalent SQL join. Denormalize what you need together.
* **Cosmos DB Mongo API ≠ MongoDB.** Different consistency model, different cost model (RU pricing), different feature set. Test specifically.

---

## References

* [MongoDB Manual](https://www.mongodb.com/docs/manual/) — official.
* Kyle Banker et al. — *MongoDB in Action* (2nd ed.).
* [MongoDB University](https://learn.mongodb.com/) — free courses.
* [MongoDB Engineering Blog](https://www.mongodb.com/blog/channel/engineering-blog).
* [MongoDB .NET driver docs](https://www.mongodb.com/docs/drivers/csharp/current/) — for C#.
