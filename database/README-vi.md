# Database — So sánh & Deep Dive theo từng DB

> Note thực dụng, opinionated về 4 engine database mainstream bạn sẽ gặp ở production: **PostgreSQL**, **MySQL**, **SQL Server**, và **MongoDB**. Mỗi per-DB doc tập trung **điểm mạnh/yếu + concurrency và consistency model** — thứ thực sự quyết định DB có fit workload không. Comparison doc là hướng dẫn chọn trên đầu.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — Per-DB deep dive cover 4 engine mainstream, cộng so sánh side-by-side và hướng dẫn chọn. Mỗi per-DB doc cover: điểm mạnh, điểm yếu, concurrency/consistency model, schema/data model, operational characteristics, .NET integration, khi nào chọn, pitfall thường gặp.
- **Tại sao** — Chọn DB cost cao và khó đảo ngược. Hầu hết debate "DB X vs Y" miss decision dimension thực (concurrency model, scaling axis, schema flexibility). Doc này nhắm cho bạn dimension, không phải opinion.
- **Khi nào** — Design service mới. Cân nhắc migration. Build kiến trúc polyglot persistence. Trước mọi meeting "DB nào?".
- **Ở đâu** — Folder top-level `database/`, language-agnostic về principle. Code sample lean **C#** vì repo .NET-focused, nhưng note concurrency/consistency apply bất kể language. Cross-link vào `dot-net/docs/enterprise-pattern/data-access-vi.md` (pattern trên đầu DB) và `dot-net/docs/concurrency/race-conditions-vi.md` (isolation level tương tác với code concurrent).
- **Như thế nào** — Đọc per-DB doc cho DB đang cân nhắc. Rồi đọc comparison cho quyết định cuối. Đừng reason từ "ta biết X" một mình — workload fit quan trọng hơn quen thuộc lúc decision, dù quen thuộc là cost saver thật.

---

## Index

| Doc                                                | Tập trung                                                                              |
| -------------------------------------------------- | -------------------------------------------------------------------------------------- |
| [`postgresql-vi.md`](./postgresql-vi.md)            | RDBMS open-source default. MVCC, hệ type giàu, JSONB, PostGIS, extension.              |
| [`mysql-vi.md`](./mysql-vi.md)                      | RDBMS open-source được deploy nhiều nhất. InnoDB MVCC + next-key lock. Vitess cho shard. |
| [`sql-server-vi.md`](./sql-server-vi.md)            | RDBMS Microsoft. Lock-based + RCSI cho MVCC. Temporal table, columnstore, T-SQL.       |
| [`mongodb-vi.md`](./mongodb-vi.md)                  | Document DB. BSON, sharding native, replica set, write/read concern, change stream.    |
| [`comparison-vi.md`](./comparison-vi.md)            | Matrix side-by-side, decision tree, use-case → DB mapping, migration considerations.   |

### Thứ tự đọc

1. **`comparison-vi.md`** cho bức tranh lớn.
2. Per-DB doc cho cái nghiêm túc cân nhắc.
3. Per-DB doc khác cho candidate đã loại — biết cái không chọn (và tại sao) là nền tảng decision record trung thực.

---

## Cấu trúc Doc

Mỗi per-DB doc theo skeleton giống nhau:

1. **Quick Reference** (5 bullet: Cái gì / Tại sao / Khi nào / Ở đâu / Như thế nào).
2. **Điểm mạnh** — cái nó thực sự giỏi.
3. **Điểm yếu** — cái nó tệ, và cái bạn sẽ đánh nhau.
4. **Concurrency & Consistency Model** — MVCC vs lock-based, isolation level, replication. Section quan trọng nhất.
5. **Schema & Data Model** — schema tốt cho engine này trông thế nào.
6. **Operational Characteristics** — scaling axis, backup, monitoring, tooling.
7. **.NET Integration** — driver, EF Core provider, note performance.
8. **Chọn / Không chọn khi nào** — guidance tường minh.
9. **Pitfall** — bất ngờ cắn production.
10. **Tham khảo** — sách, doc official, blog đáng đọc.

Cấu trúc này làm so sánh per-DB dễ: đọc section 4 xuyên cả 4 DB để so concurrency model, section 6 để so ops, v.v.

---

## Cross-References Vào Phần Khác Của Repo

* **[`../dot-net/docs/enterprise-pattern/data-access-vi.md`](../dot-net/docs/enterprise-pattern/data-access-vi.md)** — pattern *trên đầu* lựa chọn DB: Lazy / Eager loading, Identity Map, Active Record vs. Data Mapper, Specification.
* **[`../dot-net/docs/enterprise-pattern/repository-vi.md`](../dot-net/docs/enterprise-pattern/repository-vi.md)** — Repository là abstraction trên DB; khi nào generic Repository là anti-pattern.
* **[`../dot-net/docs/enterprise-pattern/unit-of-work-vi.md`](../dot-net/docs/enterprise-pattern/unit-of-work-vi.md)** — UoW backed bởi `DbContext` / `IClientSessionHandle`.
* **[`../dot-net/docs/concurrency/race-conditions-vi.md`](../dot-net/docs/concurrency/race-conditions-vi.md)** — race condition và cách isolation level DB chặn (hoặc không chặn) chúng.
* **[`../dot-net/docs/architectural-pattern/outbox-vi.md`](../dot-net/docs/architectural-pattern/outbox-vi.md)** — Outbox phụ thuộc write transactional cùng-DB; outbox dựa CDC phụ thuộc feature binlog / WAL / change-stream của DB.
* **[`../dot-net/docs/architectural-pattern/event-sourcing-vi.md`](../dot-net/docs/architectural-pattern/event-sourcing-vi.md)** — Event Sourcing trên đầu bất kỳ DB; engine specialized (EventStoreDB) ở category riêng.
* **[`../books/building-microservices-sam-newman-vi.md`](../books/building-microservices-sam-newman-vi.md)** — "Database per service" và polyglot persistence.
* **[`../books/domain-driven-design-eric-evans-vi.md`](../books/domain-driven-design-eric-evans-vi.md)** — aggregate design ảnh hưởng cách bạn model trong store document vs. quan hệ.

---

## Note về Scope

Doc này cố ý **không** cover:

* **Store specialized** — Redis (cache/stream), Elasticsearch (search), ClickHouse / DuckDB (OLAP columnar), Cassandra / ScyllaDB (wide-column distributed), Neo4j (graph), Pinecone / Weaviate (vector). Chúng chiếm decision space khác.
* **Re-implementation wire-compatible** — "Mongo API" / "PostgreSQL API" / "Cassandra API" của Cosmos DB không phải engine chúng giống; comparison doc có warning về Cosmos Mongo cụ thể.
* **Engine embedded** — SQLite, DuckDB-as-embedded, LiteDB. Use case khác (single-process / desktop / mobile / test).
* **Chi tiết feature per-version** — doc này nhắm principle ổn định, không phải changelog version-specific.

Nếu có demand, mấy cái đó có thể thành folder riêng sau. Hiện tại: 4 engine có khả năng cao nhất trong shortlist của bạn.
