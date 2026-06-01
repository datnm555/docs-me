# MongoDB

> Document database định nghĩa NoSQL cho cả 1 thế hệ. Schema linh hoạt, data shape denormalized, scale-out ngang qua sharding, và query language design cho JSON. Mongo hiện đại (4.0+) thêm multi-document transaction, thu hẹp khoảng cách với DB quan hệ — nhưng use case tối ưu vẫn là **data dạng document được truy cập theo document boundary**, không phải data quan hệ bị ép vào BSON.

> 🇻🇳 Phiên bản tiếng Việt. English: [`mongodb.md`](./mongodb.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Document database phân tán. Lưu document **BSON** (JSON nhị phân) trong **collection**. Không schema cố định mặc định (dù có JSON Schema validation tùy chọn). Scale ngang native qua **sharding**. Replica set cho HA + scale read.
- **Tại sao** — Khi data dạng document (mỗi entity được read/write như đơn vị, có field biến đổi, array lồng sâu), Mongo khớp shape. Sharding built-in, không phải bolt-on. Replica set operationally đơn giản hơn hầu hết HA quan hệ.
- **Khi nào** — Data catalog / CMS với field không đồng nhất. Profile user. Stream event/activity. Bất cứ đâu design quan hệ tạo hàng chục table join cho 1 entity logic. **Không** đúng cho analytic quan hệ cao hay workload transactional cross-document chặt.
- **Ở đâu** — MongoDB Atlas (managed, dominant trong cloud), self-host (Community / Enterprise), Azure Cosmos DB Mongo API (engine khác bên dưới — *không phải* Mongo thật). Từ .NET: official **MongoDB.Driver**.
- **Như thế nào** — Model **theo query**, không theo entity. Denormalize cái đọc cùng nhau. Embedded document cho 1:few; reference cho 1:many cardinality bounded; collection + lookup cho many-to-many thật. Chọn write/read concern phù hợp per operation. Plan shard key trước khi đụng tường (đổi đau).

---

## Điểm mạnh

* **Schema linh hoạt.** Thêm field không cần migration. Shape không đồng nhất cùng tồn tại trong 1 collection.
* **Document model khớp entity thế giới thực** khi entity *là* document (article, order với item, user với preference embedded).
* **Sharding native.** Scale ngang write built-in, không phải "thêm Citus rồi viết lại app."
* **Replica set.** Multi-replica, failover tự động, read preference configurable. Đơn giản hơn semi-sync MySQL hay streaming PG + Patroni.
* **Aggregation pipeline.** Pipeline staged cho transformation phức tạp — gần LINQ / SQL CTE hơn MapReduce truyền thống.
* **Atlas** (managed) thực sự tốt — backup, monitoring, search, vector, encryption-at-rest tích hợp hết.
* **Change Stream** — CDC native. Subscribe insert/update không cần tool ngoài.

## Điểm yếu

* **ACID cross-document mới.** Multi-document transaction thêm ở 4.0 (replica set), 4.2 (sharded). Được nhưng đắt — dùng tiết kiệm, ưu tiên atomicity single-document.
* **Không join (well, `$lookup`).** `$lookup` aggregation có nhưng đắt — Mongo design giả định bạn sẽ denormalize.
* **Schema linh hoạt cắt 2 chiều.** Không validation, drift tích lũy và query thành defensive. Dùng **JSON Schema validation** trên collection production.
* **Shard key gần như mãi mãi.** Chọn shard key tệ (cardinality thấp, monotonic tăng) tạo hot shard. Resharding có (5.0+) nhưng là operation, không phải đổi casual.
* **Ngốn memory.** WiredTiger cache muốn RAM tỷ lệ working set. Cluster mis-size degrade im lặng.
* **"Mongo API" của Cosmos DB không phải Mongo.** Storage engine khác, consistency model khác, cost model khác. Đừng nhầm.

---

## Concurrency & Consistency Model

### Atomicity single-document là nền tảng

Update **1 document** là atomic — bao gồm array bên trong. Đây là đòn bẩy Mongo cược: design schema sao cho operation thường chạm *1 document*, và bạn được semantic kiểu ACID miễn phí.

### Multi-document transaction

Có từ 4.0 (replica set) và 4.2 (sharded cluster). Dùng cho cross-document atomic write thật — nhưng đắt hơn single-document và có time limit (mặc định 60s). **Coi là ngoại lệ, không phải default.**

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

### Write Concern — "đã ghi" durable cỡ nào?

| Write Concern         | Nghĩa                                                                          |
| --------------------- | ------------------------------------------------------------------------------ |
| `w: 0`                | Fire and forget. **Không bao giờ dùng** ngoài workload telemetry.              |
| `w: 1` (default)       | Ack bởi primary. Primary fail trước replicate → mất data.                      |
| `w: "majority"`        | Ack bởi majority của replica set. **Default đúng** cho durability.              |
| `w: <n>`              | Chờ n node. Majority tường minh thường tốt hơn.                                  |
| `j: true`             | Chờ write journal (durable trên disk).                                          |

### Read Concern — "đọc" tươi cỡ nào?

| Read Concern         | Nghĩa                                                                            |
| -------------------- | -------------------------------------------------------------------------------- |
| `local` (default)     | Đọc từ node local, có thể chưa majority-acked. Nhanh, yếu nhất.                  |
| `available`          | Như `local` nhưng cho orphan read khi chunk migrate. Tránh nói chung.            |
| `majority`           | Chỉ trả data đã majority-committed. **Cặp với `w: majority` cho read-your-own-writes** ở nhiều topology. |
| `linearizable`       | Mạnh nhất — đảm bảo read phản ánh mọi majority-committed write trước nó. Primary-only, chậm. |
| `snapshot`           | Snapshot consistency thời điểm transaction.                                       |

### Read Preference

Read route vào đâu:

* `primary` (default) — strict consistency cho node đó.
* `primaryPreferred` — primary trước, fallback secondary nếu down.
* `secondary` / `secondaryPreferred` — scale read off replica. **Eventual** consistency.
* `nearest` — latency thấp nhất. Mix primary/secondary.

> Kết hợp `w: majority` + `readConcern: majority` + `readPreference: primary` cho bạn gần nhất với semantic "single-server" trong cluster phân tán. Default trừ khi có lý do không.

### Cơ chế Replica Set

* 3 node tối thiểu cho HA (hoặc 2 data + 1 arbiter).
* Primary handle write; secondary replicate oplog (WAL của Mongo).
* Primary fail, secondary elect primary mới trong ~10–30 giây qua consensus kiểu Raft.
* Secondary read-only có thể serve read (với trade-off consistency ở trên).

### Cơ chế Sharding

* Cluster có **shard** (mỗi shard là 1 replica set), **config server** (metadata), và **mongos** router (client connect vào).
* Document partition theo **shard key**.
* Strategy shard-key thường gặp:
  * **Hashed** (`{ _id: "hashed" }`) — phân phối đều, không range query.
  * **Ranged** trên field cardinality cao — support range query, rủi ro hot shard.
  * **Compound** (ví dụ `{ tenantId: 1, _id: 1 }`) — locality per tenant + spread.
* **Shard key monotonic tăng** (ví dụ timestamp thô) gây **hot shard cuối**. Dùng hashed hoặc compound với prefix.

---

## Schema & Data Model

Kỹ năng quan trọng nhất Mongo: **model quanh cách data được read.**

* **Embed** khi:
  * Child thuộc parent (line item của order, thread comment của post).
  * Cardinality 1:few.
  * Luôn đọc cùng nhau.
* **Reference** (lưu ID parent trong child) khi:
  * 1:many với tăng trưởng unbounded (user → activity log).
  * Child query độc lập.
  * Child to làm phình parent document.
* **Tránh** document > 1 MB. Limit cứng Mongo là 16 MB / document — nhưng performance degrade trước đó nhiều.

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

Dùng trong production. Linh hoạt cho prototype; production phải pin shape.

### Index

* `_id` được index tự động (B-tree trên primary).
* Single-field, compound, multikey (array), text, geo (2dsphere), hashed, partial, TTL.
* `explain()` plan trông giống plan SQL. Đọc nó.
* Index có cost trên write — quá nhiều index làm insert chậm và tốn RAM.

---

## Operational Characteristics

| Concern             | Thực tế                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Scale dọc           | Tốt, nhưng sweet spot là replica set medium-to-large với RAM đủ.                     |
| Scale ngang         | **Native qua sharding.** Khác biệt to nhất của Mongo.                                 |
| Backup              | Atlas: continuous + snapshot. Self-host: `mongodump`, snapshot từ secondary.          |
| Monitoring          | Atlas built-in; self-host: Prometheus exporter + MongoDB Ops Manager.                 |
| Schema migration    | Online — nhưng viết migration là script update document (không `ALTER`).             |
| Driver maturity     | Xuất sắc ở major language; .NET driver là first-class.                                |

---

## .NET Integration (ngắn)

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

* `MongoClient` **thread-safe và connection-pool** — register singleton.
* `IMongoCollection<T>` typed cho LINQ — nhưng là subset của LINQ; check cái gì được translate.
* High throughput: batch qua `InsertManyAsync` / `BulkWriteAsync`.
* `IClientSessionHandle` cho multi-document transaction.
* `[BsonId]`, `[BsonElement]`, `[BsonIgnoreIfNull]` attribute cho mapping tường minh; hoặc `BsonClassMap.RegisterClassMap` cho config tập trung.

---

## Chọn / Không chọn khi nào

**Chọn khi:**

* Data *là* dạng document — entity có field biến đổi và document boundary tự nhiên.
* Cần scale write ngang mà không invent lớp sharding.
* Catalog / CMS / activity stream / IoT telemetry / event log.
* Đang dùng Atlas và muốn experience managed đầy đủ với search + vector + analytic tích hợp.

**Đừng chọn khi:**

* Data vốn quan hệ (nhiều join, nhiều transaction cross-entity). DB quan hệ sẽ đơn giản hơn.
* Cần ACID transaction chặt xuyên nhiều entity by default. Mongo làm được, nhưng design tax có thật.
* Cần SQL giàu — `$lookup` + aggregation pipeline mạnh nhưng không phải SQL.
* Compliance / regulation yêu cầu feature mà "Mongo API" alternative của Atlas không có.

---

## Pitfall

* **Shard key chọn muộn** — shard key monotonic tạo hot shard; shard key cardinality thấp không split được. Plan sớm.
* **Default `w: 1`** coi là durable — không phải khi failover. Dùng `w: majority` cho mọi cái quan trọng.
* **Mix kiểu field** trong cùng field (`"price": 99` chỗ này, `"price": "99"` chỗ khác). Storage cho phép; query fail im lặng. Dùng schema validation.
* **Array tăng unbounded** (ví dụ thêm mọi comment vào document post mãi mãi) — document reallocate liên tục và index chậm. Cap array hoặc tách thành child collection.
* **Đọc từ secondary mong tươi** — secondary lag. Dùng `readConcern: majority` + `readPreference: primary` cho freshness.
* **`$lookup` trên collection to** — chậm hơn SQL join tương đương nhiều. Denormalize cái cần đọc cùng nhau.
* **Mongo API Cosmos DB ≠ MongoDB.** Consistency model khác, cost model khác (RU pricing), feature set khác. Test cụ thể.

---

## Tham khảo

* [MongoDB Manual](https://www.mongodb.com/docs/manual/) — official.
* Kyle Banker et al. — *MongoDB in Action* (2nd ed.).
* [MongoDB University](https://learn.mongodb.com/) — khóa free.
* [MongoDB Engineering Blog](https://www.mongodb.com/blog/channel/engineering-blog).
* [MongoDB .NET driver docs](https://www.mongodb.com/docs/drivers/csharp/current/) — cho C#.
