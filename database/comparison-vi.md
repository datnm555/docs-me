# So sánh Database — PostgreSQL · MySQL · SQL Server · MongoDB

> So sánh side-by-side và hướng dẫn chọn. Mỗi DB thực sự giỏi ở thứ khác nhau — câu hỏi không bao giờ là "cái nào tốt nhất?", mà là "cái nào fit **workload này**?". Đọc từng per-DB doc trước; dùng cái này cho quyết định cuối.

> 🇻🇳 Phiên bản tiếng Việt. English: [`comparison.md`](./comparison.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Hướng dẫn chọn xuyên 4 database mainstream: PostgreSQL, MySQL, SQL Server, MongoDB. So sánh concurrency model, scalability, feature surface, .NET integration, và footprint operational.
- **Tại sao** — Chọn DB cost cao, khó đảo ngược. Default sai ("ta luôn dùng X") làm mọi thứ từ design query tới operation đau khổ trong nhiều năm.
- **Khi nào** — Bắt đầu service mới. Cân nhắc migration. Build kiến trúc microservice nhiều DB và chọn DB per service.
- **Ở đâu** — Hướng dẫn top-level cho folder `database/`. Cross-reference mọi per-DB deep dive và note .NET integration ở đó.
- **Như thế nào** — Xác định dimension workload thực sự quan trọng (transactional? document-shaped? read-heavy?). Score candidate trên dimension đó. Đừng reason từ "ta biết X" một mình — biết X có trọng số, nhưng không override workload fit.

---

## Ma trận Side-by-Side

| Tiêu chí                     | **PostgreSQL**                       | **MySQL (InnoDB)**                   | **SQL Server**                                | **MongoDB**                                |
| ---------------------------- | ------------------------------------- | ------------------------------------ | --------------------------------------------- | ------------------------------------------ |
| **Data model**               | Relational + JSONB                    | Relational                            | Relational                                     | Document (BSON)                            |
| **License**                  | Free (PostgreSQL license)             | Free (GPL) / Thương mại               | Thương mại (Express free, theo tier)           | SSPL / Atlas SaaS                          |
| **Concurrency model**        | MVCC (snapshot)                       | MVCC qua InnoDB + next-key lock       | Lock-based default; **RCSI** cho MVCC          | Single-document atomic + multi-doc TX (4.0+) |
| **Isolation default**        | Read Committed                        | Repeatable Read (với gap lock)        | Read Committed (lock-based trừ khi RCSI)       | n/a (per-op concern)                       |
| **Isolation mạnh nhất**       | Serializable (SSI)                    | Serializable                          | Snapshot / Serializable                        | Linearizable read + majority write         |
| **Scale dọc**                | Xuất sắc                              | Xuất sắc                              | Xuất sắc                                       | Tốt                                        |
| **Scale write ngang**        | Citus (extension)                     | **Vitess / PlanetScale**              | Limited; Hyperscale trên Azure                 | **Sharding native**                        |
| **Read replica**             | Streaming / logical                   | Async / semi-sync / group              | AlwaysOn AG                                    | Replica set secondary                      |
| **Support JSON**             | **JSONB** (tốt nhất class)            | JSON (đang cải thiện)                 | JSON (ổn)                                       | Native (BSON)                              |
| **GIS**                      | **PostGIS** (tốt nhất class)          | Spatial (cơ bản)                       | Geography / Geometry (tốt)                     | 2dsphere (ổn)                              |
| **Full-text search**         | Built-in tsvector + extension          | Built-in (ổn)                          | Full-Text Search                                | Atlas Search (Lucene)                      |
| **CDC / change stream**      | Logical replication / Debezium        | **Binlog → Debezium / Maxwell**       | CDC / Change Tracking                          | **Change Stream** (native)                 |
| **In-memory engine**         | UNLOGGED table                        | NDB cluster (niche)                   | Hekaton (in-memory OLTP)                       | WiredTiger (cache nặng)                    |
| **Columnar**                 | cstore_fdw / Citus columnar            | MyRocks / ClickHouse companion         | **Columnstore index** (built-in)               | n/a (document)                             |
| **Cloud-native options**     | Aurora, AlloyDB, Supabase, Neon       | Aurora, PlanetScale, Vitess           | Azure SQL DB, Hyperscale, RDS                  | **Atlas** (dominant)                       |
| **.NET driver maturity**     | Npgsql (xuất sắc)                     | MySqlConnector (xuất sắc)             | Microsoft.Data.SqlClient (first-party)         | MongoDB.Driver (first-party)               |
| **EF Core provider**         | First-party (`Npgsql.EFCore`)         | Community (`Pomelo`)                  | First-party (Microsoft)                        | n/a (dùng driver / EF Mongo niche)         |
| **Footprint operational**    | Trung (vacuum, WAL tune)              | Trung (binlog, replica lag)           | Trung-cao (nhiều knob, nhiều tool)             | Trung (cluster + shard ops)                |
| **Schema flexibility**       | Strict + JSONB escape hatch           | Strict                                | Strict                                          | **Flexible mặc định** (validation tùy chọn) |
| **Workload fit best**        | "Default cho mọi thứ"                 | Web read-heavy / scale-out qua Vitess  | .NET enterprise; OLTP+analytic mixed            | Data document; scale-out write             |

---

## Decision Tree

```
Data có dạng document không?
  (entity = read/write như đơn vị, field biến đổi, array lồng sâu)
├── Có  → MongoDB
└── Không
    │
    Bạn trong môi trường Microsoft / .NET enterprise?
    ├── Có
    │   └── Cần temporal table / columnstore / Service Broker /
    │       integration AD/SSAS/Power BI chặt?
    │       ├── Có → SQL Server
    │       └── Không, cost quan trọng → PostgreSQL
    │
    └── Không
        │
        Cần path scale write ngang không phải invent?
        ├── Có → MySQL (Vitess / PlanetScale)
        └── Không
            │
            Cần SQL giàu (CTE, JSONB, partial index, PostGIS, extension)?
            ├── Có → PostgreSQL
            └── Không, team đã chạy MySQL với ops sẵn → MySQL
```

Đây là heuristic. Có edge case. Đọc per-DB doc.

---

## Use Case → DB Mapping

### "OLTP service mặc định"

| Sub-context                                   | Chọn                                |
| --------------------------------------------- | ----------------------------------- |
| Greenfield, không ràng buộc stack có sẵn      | **PostgreSQL**                       |
| Shop .NET / Microsoft có sẵn                  | **SQL Server** hoặc **PostgreSQL**   |
| Operational muscle của team đã là MySQL       | **MySQL**                            |

### "Multi-tenant SaaS"

* **PostgreSQL** — schema prefix `tenant_id` hoặc RLS. Citus khi vượt single primary.
* **MySQL** với Vitess — khi số tenant lớn và cần isolation per tenant.
* **MongoDB** — khi shape data mỗi tenant không đồng nhất.

### "Catalog / CMS / activity stream"

* **MongoDB** — field biến đổi, pattern read denormalized.
* **PostgreSQL** với JSONB — cùng lợi ích shape, cộng query quan hệ.

### "OLTP + analytical mixed"

* **SQL Server** columnstore index — cùng DB, 2 workload.
* **PostgreSQL** + read replica chạy query OLAP — chạy được, có caveat.
* **PostgreSQL** + ClickHouse qua Debezium — cho scale analytic thật.

### "Event store / audit log"

* **PostgreSQL** — consistency mạnh, WAL durable, dễ query history.
* **MongoDB** — ổn nếu event dạng document và muốn shard-theo-aggregate.
* Specialized: EventStoreDB, Kafka + KSQL — khi "event store" là job *duy nhất*.

### "Workload Geospatial"

* **PostgreSQL + PostGIS** — tốt nhất class.
* **SQL Server** — geography/geometry native ổn.
* **MongoDB** — index 2dsphere work cho query "find trong radius" thường.

### "Vector / embedding (RAG, semantic search)"

* **PostgreSQL + pgvector** — tích hợp đầy đủ với data quan hệ.
* **MongoDB Atlas Vector Search** — managed đầy đủ.
* Specialized: Pinecone, Weaviate, Qdrant — khi vector *là* workload.

### "Time-series cardinality cao"

* **PostgreSQL + TimescaleDB** — quan hệ + time-series trong 1.
* Specialized: ClickHouse, InfluxDB, VictoriaMetrics — khi ingest rate dominant.

---

## Cost — Ngoài Phí License

License là cost nhìn thấy. **Cost operational** thường to hơn:

| Loại cost                                  | Postgres | MySQL    | SQL Server                    | Mongo                                 |
| ------------------------------------------ | -------- | -------- | ----------------------------- | ------------------------------------- |
| License software                           | Free     | Free     | $$$$ (Enterprise) / Free (Express) | SSPL / Atlas $$                       |
| DBA / engineer skilled availability        | Cao      | Cao      | Cao (.NET / enterprise)       | Trung-Cao (tùy region)                |
| Memory overhead per-node                   | Cao      | Trung    | Cao                           | Cao                                   |
| Cost connection (memory per-connection)    | Cao (mitigate với PgBouncer) | Thấp | Trung                | Thấp                                  |
| Cost operational migration / DDL           | Trung    | Cao (cần gh-ost)              | Thấp (online ops Enterprise) | Thấp (write schemaless)               |
| Backup / restore (managed)                  | Dễ trên RDS/Aurora | Dễ trên RDS/Aurora | Dễ trên Azure SQL | Trivial trên Atlas       |

Managed cloud (Aurora, Azure SQL, Atlas) làm phẳng nhiều cost operational — nhưng giá leo nhanh.

---

## Migration Considerations

| Từ → Đến                  | Friction                                                                                       |
| -------------------------- | ---------------------------------------------------------------------------------------------- |
| MySQL → PostgreSQL         | Trung. Khác biệt dialect SQL, `AUTO_INCREMENT` → `SERIAL`/`IDENTITY`, replication khác. Tool: pgloader. |
| PostgreSQL → MySQL         | Cao hơn. Feature PG (JSONB, CTE, partial index, PostGIS) thường không có MySQL tương đương.   |
| SQL Server → PostgreSQL    | Trung-cao. T-SQL → PL/pgSQL rewrite. Temporal table, Service Broker cần redesign. Tool: AWS DMS / Babelfish (T-SQL trên PG). |
| Mongo → Postgres           | Cao nếu "Mongo như Postgres-với-JSON" — thực ra dễ. Mongo thật (denormalized, $lookup) cần re-modeling. |
| Postgres → Mongo           | Gần như luôn sai. Re-model quanh aggregate boundary trước; chỉ migrate khi shape thật sự fit.  |
| Bất kỳ → managed (Atlas, RDS) | Thấp. Dùng logical replication / CDC cho cutover.                                            |

---

## Anti-Pattern Xuyên Cả Bốn

* **"Ta dùng X cho mọi thứ."** Kiến trúc microservice là tự do chọn per-service; dùng nó. Reach DB fit workload.
* **"Cũng là SQL — chúng interchangeable."** Không. Concurrency model, default isolation, hạn chế index, semantic DDL khác đủ để cắn ở scale.
* **Bỏ qua tầng connection pool.** PG không PgBouncer, MySQL không ProxySQL, MARS SQL Server không suy nghĩ — mọi DB có trần connection. Plan cho nó.
* **Default durability tối đa khắp nơi.** `synchronous_commit = on` / `w: majority` / `RCSI off + sync replica` cost throughput. Tune per-operation; không phải mọi write cần guarantee mạnh nhất.
* **Coi "Cosmos DB Mongo API" là MongoDB.** Là engine khác với Mongo wire protocol. Consistency khác, cost model khác, feature set khác.
* **Chọn DB theo hype Twitter / blog thay vì thực tế operational team.** DB bạn operate được giá trị hơn DB fit tốt hơn trên giấy chút.

---

## Xem thêm

* Per-DB deep dive: [PostgreSQL](./postgresql-vi.md) · [MySQL](./mysql-vi.md) · [SQL Server](./sql-server-vi.md) · [MongoDB](./mongodb-vi.md).
* Liên quan: `../dot-net/docs/enterprise-pattern/data-access-vi.md` — pattern *trên đầu* lựa chọn DB (Repository, UoW, Lazy/Eager, v.v.).
* Liên quan: `../dot-net/docs/concurrency/race-conditions-vi.md` — race condition và cách isolation level chặn (hoặc không chặn) chúng.

## Tham khảo

* Martin Kleppmann — *Designing Data-Intensive Applications* (2017). Sách kinh điển về trade-off design cross-DB.
* Documentation AWS / GCP / Azure cho tier DB managed — pricing và feature parity khác nhau dữ.
* [Use The Index, Luke!](https://use-the-index-luke.com/) — Markus Winand. Nguyên tắc index/query language-agnostic áp dụng cho mọi DB quan hệ ở trên.
