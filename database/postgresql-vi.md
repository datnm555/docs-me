# PostgreSQL

> RDBMS open-source mà "chạy ngon" cho hầu hết workload. Relational chặt chẽ + thân thiện JSON + nghiêm túc về correctness. MVCC làm tốt, hệ type giàu nhất trong các DB mainstream, và concurrency model scale read sạch sẽ. Nếu không có lý do cụ thể chọn cái khác, **chọn cái này**.

> 🇻🇳 Phiên bản tiếng Việt. English: [`postgresql.md`](./postgresql.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Relational DB miễn phí, open-source, ACID, với **MVCC concurrency**, support SQL standard mạnh, JSON/JSONB native, GIS, full-text search, và extension model bolt-on time-series (TimescaleDB), vector search (pgvector), v.v.
- **Tại sao** — Trong RDBMS open-source, PostgreSQL có combo correctness + feature surface + operational maturity tốt nhất. MVCC nghĩa là reader không bao giờ block writer. JSONB làm mờ ranh giới SQL/NoSQL mà không mất transaction.
- **Khi nào** — Default cho service mới. Hợp khi cần transaction, query phức tạp, join, constraint, **và** field semi-structured trong cùng store. Tốt cho analytical workload tới vài TB.
- **Ở đâu** — OLTP service, multi-tenant SaaS, geospatial, search-with-structure (jsonb + GIN), event store (cẩn thận), embedding/vector qua pgvector. Kém lý tưởng cho write throughput vượt ~100k tx/sec mà không shard (Citus).
- **Như thế nào** — Chạy trên managed service (AWS RDS / Aurora Postgres, GCP Cloud SQL / AlloyDB, Azure Database for PostgreSQL, Supabase, Neon, Crunchy). Self-host được nhưng vacuum + WAL ops cần chú ý. Từ .NET: **Npgsql** + **EF Core** (`Microsoft.EntityFrameworkCore.PostgreSQL`).

---

## Điểm mạnh

* **MVCC concurrency.** Reader không bao giờ block writer. Mỗi transaction thấy snapshot nhất quán. Không reader lock cho query thường.
* **SQL standards-compliant.** Window function, CTE (cả recursive), `LATERAL`, materialized view, partial / expression / covering index, custom type, custom aggregate, custom operator.
* **JSONB.** Indexable, queryable, schemaless-khi-cần. Hoàn toàn transactional cùng với relational column.
* **Extension.** PostGIS (spatial), TimescaleDB (time-series), pgvector (embedding), pg_partman (partitioning), pglogical (logical replication), Citus (sharding). Phần lớn là production code first-class.
* **Crash safety.** Synchronous commit + WAL; sống sót mất phần cứng không mất data khi config đúng.
* **Chi phí.** Free. Không per-core licensing.

## Điểm yếu

* **Scale write vượt single primary** khó hơn các system thiết kế cho nó. Read-replicate dễ; multi-primary write không được mà không shard (Citus) hay partitioning ở app.
* **VACUUM** cần để reclaim space từ MVCC tuple. Autovacuum config sai trên table to/hot là outage Postgres kinh điển.
* **Replication lag** ở async replica — read replica có thể chậm vài giây so primary.
* **Cost connection.** Mỗi connection 1 process; ~10MB+ memory. PgBouncer gần như bắt buộc khi vượt vài trăm connection.
* **DDL chủ yếu transactional nhưng có thể giữ lock mạnh.** `ALTER TABLE` trên hot table cần cẩn thận (dùng `CONCURRENTLY` cho index; tool như `pg_repack` cho rewrite to).

---

## Concurrency & Consistency Model

### MVCC trong 1 đoạn

Mỗi row có cột ẩn `xmin` (transaction tạo) và `xmax` (transaction xóa). Transaction thấy row nếu `xmin` committed trước snapshot của nó và `xmax` hoặc null hoặc sau snapshot. Update implement là `delete + insert` — version cũ vẫn còn đến khi `VACUUM` thu. **Kết quả:** reader không block writer, writer không block reader (trừ update cùng 1 row).

### Isolation Level

| Level                   | Chặn được                                                              | Cho phép                                                | Cost lock     |
| ----------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------- | ------------- |
| **Read Uncommitted**    | (PG coi như Read Committed — không bao giờ dirty read)                  | n/a                                                     | Không         |
| **Read Committed** (default) | Dirty read                                                        | Non-repeatable read, phantom read                       | Row-level only |
| **Repeatable Read**     | Dirty read, non-repeatable read, **phantom read** (PG mạnh hơn standard SQL qua snapshot isolation) | Một số write skew              | Snapshot      |
| **Serializable**        | Mọi anomaly. SSI = Serializable Snapshot Isolation.                     | Không (nhưng có thể abort với `40001`)                  | Predicate-lock |

**Công thức thực tế:**

* Read Committed (default) — CRUD đa số. Rẻ.
* Repeatable Read — multi-statement read phải agree.
* Serializable — logic financial / counter nơi write skew quan trọng. Retry trên `serialization_failure (40001)`.

### Lock bạn sẽ gặp thực tế

* `FOR UPDATE` — row-level lock cho workflow "select rồi update". Default cho `SELECT ... FOR UPDATE`.
* `FOR UPDATE SKIP LOCKED` — pattern work-queue kinh điển. Worker grab row chưa lock và xử lý.
* Advisory lock (`pg_advisory_lock`) — mutex do app định nghĩa ngoài row model.

### Replication

* **Streaming replication** (physical): WAL ship byte-for-byte. Replica read-only; có thể fail over.
* **Logical replication**: publish/subscribe per-table; subscriber có thể khác version hay khác schema. Entry point cho cross-DB CDC.
* **Sync vs async commit** tunable per-transaction (`synchronous_commit`).

> **Replication default eventual ở replica.** Đọc từ replica ngay sau khi ghi vào primary có thể không thấy write của bạn. Dùng primary cho read-your-writes, hoặc `synchronous_commit = remote_apply` (chậm).

---

## Schema & Data Model

* Bắt đầu normalize đầy đủ. Thêm denormalization (materialized view, JSONB rollup, generated column) sau khi thấy query pattern.
* Cột `JSONB` là escape hatch mạnh cho shape thưa / biến đổi — **nhưng** coi như typed contract; đừng cho JSON tùy ý của client vào.
* Generated column (`GENERATED ALWAYS AS ... STORED`) cho phép index giá trị computed (ví dụ `lower(email)`).
* Partitioning (range / list / hash) giúp table rất lớn — nhưng thêm complexity ops. Default là plain table cho đến khi đau.

---

## Operational Characteristics

| Concern             | Thực tế                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Scale dọc           | Xuất sắc — 1 primary xử lý vài chục nghìn write TPS trên phần cứng hiện đại.         |
| Write ngang         | Cần Citus hoặc app sharding.                                                          |
| Read replica        | Thêm dễ; chọn 1 cho OLAP / async read.                                                |
| Backup              | `pg_basebackup` + WAL archive → point-in-time recovery.                              |
| Schema migration    | Tool: Flyway, Liquibase, EF Core migration. Lock mạnh trên `ALTER TABLE`.            |
| Connection pooling  | PgBouncer (transaction-pooling mode là default phổ biến).                            |

---

## .NET Integration (ngắn)

```csharp
// EF Core
services.AddDbContext<AppDbContext>(o =>
    o.UseNpgsql(connStr, npg =>
    {
        npg.EnableRetryOnFailure(maxRetryCount: 3);   // transient retry cho serialization abort
        npg.MapEnum<OrderStatus>("order_status");      // native enum mapping
    }));

// Raw / Dapper
await using var conn = new NpgsqlConnection(connStr);
var rows = await conn.QueryAsync<Order>(
    "SELECT * FROM orders WHERE status = ANY(@statuses)",
    new { statuses = new[] { "Pending", "Paid" } });
```

* `Npgsql` support native array, `jsonb`, enum, `uuid`, `interval`, `inet`.
* High-throughput insert: dùng binary `COPY` qua `NpgsqlBinaryImporter` — nhanh gấp nhiều lần `INSERT` từng row.
* Cặp với PgBouncer transaction-pooling mode; **disable prepared statement** (`No Reset On Close=true; Max Auto Prepare=0`) khi sau PgBouncer.

---

## Chọn / Không chọn khi nào

**Chọn khi:**

* Muốn 1 DB xử lý relational + JSON + GIS + full-text + analytic mà không phải dựng 3 service nữa.
* Workload read-heavy — MVCC + read replica scale dễ.
* Cần transaction thật xuyên data type giàu.

**Đừng chọn khi:**

* Write throughput bền vững > ~100k tx/sec trên 1 DB không shard.
* Workload document thuần không relational query — Mongo hay DynamoDB đơn giản hơn.
* Analytic real-time xuyên petabyte — đó là lãnh thổ ClickHouse / BigQuery.

---

## Pitfall

* **Bloat từ không vacuum row hot** — autovacuum setting tune theo table size, không phải default server.
* **Transaction chạy lâu** ngăn vacuum và chất đống MVCC garbage. Watch `pg_stat_activity` cho `state = idle in transaction`.
* **`ORDER BY` không có index** ở scale — Postgres sort in-memory rồi spill ra disk im lặng. Thêm index.
* **Connection storm** — không PgBouncer, app Node/.NET có thể mở hàng nghìn connection và OOM server.
* **Đọc từ replica mong strict consistency** — xem note replication. Read-your-writes thì đọc primary.
* **`SELECT *` qua JSONB** không project — gửi payload to. Project key cần với `->`.

---

## Tham khảo

* [PostgreSQL documentation](https://www.postgresql.org/docs/current/) — manual xuất sắc.
* [Postgres Weekly](https://postgresweekly.com/) — newsletter.
* Markus Winand — *Use The Index, Luke!* — index/query tuning language-agnostic.
* Laurenz Albe — *PostgreSQL Server Programming* / blog Cybertec.
* [Npgsql documentation](https://www.npgsql.org/) — cho .NET.
