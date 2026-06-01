# MySQL

> RDBMS open-source được deploy nhiều nhất thế giới. Battle-hardened, nhanh trên workload read-heavy, và là default cho mảng lớn thế giới LAMP/web. Ít feature hơn PostgreSQL, nhưng replication rock-solid, hệ sinh thái tool khổng lồ, và đặc tính performance rõ ràng. Đọc cái này cạnh PostgreSQL để chọn giữa hai.

> 🇻🇳 Phiên bản tiếng Việt. English: [`mysql.md`](./mysql.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — RDBMS open-source ACID. **InnoDB** là storage engine mặc định (transactional, MVCC, row-level lock). Sở hữu bởi Oracle; **MariaDB** là fork cộng đồng với feature set tương tự và release cadence khác.
- **Tại sao** — Hàng chục năm sử dụng production tạo ra replication rất mature, hệ sinh thái tool to, và pattern scale dễ hiểu (read replica, ProxySQL, Vitess). Footprint operational nhẹ hơn PostgreSQL cho workload CRUD "vanilla".
- **Khi nào** — OLTP scale lớn, web app read-heavy, deploy multi-region nơi replication mature quan trọng, khi team đã chạy MySQL. Default ở nhiều công ty vì lý do lịch sử.
- **Ở đâu** — Self-hosted (MySQL Community / Percona Server / MariaDB) hoặc managed (AWS RDS / Aurora MySQL, GCP Cloud SQL, Azure Database for MySQL, PlanetScale, Vitess). Từ .NET: **MySqlConnector** + **EF Core** (`Pomelo.EntityFrameworkCore.MySql`).
- **Như thế nào** — Chọn **InnoDB** (default). Chọn version mới (8.0+). Quyết topology replication sớm (single-primary + read replica chiếm 95% deploy). Dùng ProxySQL hoặc PlanetScale/Vitess cho shard khi write single-primary bão hòa.

---

## Điểm mạnh

* **Replication mature.** Async + semi-sync + group replication (multi-primary). Tool như Orchestrator/Percona Toolkit đã chuẩn.
* **Scale read replica.** Thêm replica để scale read là pattern kinh điển.
* **Tooling operational.** ProxySQL, PMM, Vitess (sharding), gh-ost / pt-online-schema-change (online DDL).
* **Performance trên query đơn giản.** Hash index + InnoDB page tuned chặt = point lookup và primary-key join rất nhanh.
* **Storage-engine pluggable.** InnoDB cho OLTP, engine tùy chọn cho column store (MyRocks). Khác cách tiếp cận "một engine" của PostgreSQL.
* **MyISAM đã chết** cho transactional work — nhưng nhắc vì code legacy còn dùng. Migrate.

## Điểm yếu

* **Feature surface nhỏ hơn PostgreSQL.** Không có partial index hoàn chỉnh, không native materialized view, JSON ergonomics yếu hơn, ít extension hơn, không native array.
* **Schema migration** có thể giữ exclusive lock trên table to. `ALTER TABLE` không có `ALGORITHM=INPLACE, LOCK=NONE` là nguy hiểm; thường cần tool như `gh-ost`.
* **Default behavior xưa không an toàn** (silent truncation, character handling yếu). Default hiện đại (`STRICT_TRANS_TABLES`, `utf8mb4`) tốt hơn — nhưng check config legacy.
* **Replication là logical / row-based** — replication lag khi write/DDL to là concern thật.
* **JSON support có** nhưng kém JSONB PostgreSQL và chậm hơn ở scale.

---

## Concurrency & Consistency Model

### MVCC InnoDB

Giống PostgreSQL, InnoDB dùng MVCC: reader thấy snapshot, write không block read version cũ. Version cũ sống trong **rollback segment** (undo log) thay vì trong table — nên InnoDB không có vấn đề "VACUUM" như PG. Undo log reclaim khi không transaction nào cần version cũ.

### Isolation Level

| Level                   | Chặn được                                                              | Cho phép                                                |
| ----------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------- |
| **Read Uncommitted**    | Không gì                                                                | Dirty read (hiếm dùng)                                  |
| **Read Committed**      | Dirty read                                                              | Non-repeatable read, phantom read                       |
| **Repeatable Read** (default!) | Dirty read, non-repeatable read, **và phần lớn phantom read** qua next-key locking | Một số write skew                |
| **Serializable**        | Mọi anomaly                                                             | Không (lock nặng)                                       |

> **Default = Repeatable Read** (khác Postgres default Read Committed). Cẩn thận khi port — read chạy lâu dưới RR giữ version và phình undo log.

### Locking — khác biệt lớn so Postgres

InnoDB dùng **next-key lock** (record + gap) mặc định ở Repeatable Read. Chúng chặn phantom read nhưng aggressive — `UPDATE WHERE indexed_column = X` có thể lock nhiều row hơn bạn nghĩ, đôi khi cả gap range. Nhiều chuyện "deadlock kỳ lạ" là về gap lock.

* **Row lock** — semantic `FOR UPDATE` thẳng.
* **Gap lock** — lock *space giữa* các indexed value. Disable ở Read Committed.
* **Insert intention lock** — gap lock đặc biệt báo hiệu insert sắp đến.

Gap lock cắn bạn mà không cần strict phantom-read prevention, switch session sang **Read Committed**.

### Replication

* **Async (default).** Replica có thể lag tùy ý. Pattern chuẩn.
* **Semi-synchronous.** Primary chờ ít nhất 1 replica ack binlog trước khi trả client. An toàn hơn; chi phí latency vừa phải.
* **Group Replication / InnoDB Cluster.** Multi-primary built-in với failover tự động.
* **Tool ngoài.** Orchestrator, Percona XtraBackup + Pseudo-GTID, MaxScale.

> Đọc từ replica = eventual consistency. Pattern: route read với explicit lag tolerance ("any-replica" / "primary-only") ở tầng app (ProxySQL, router app-level).

### Binlog — feature ít được nói

Mọi thay đổi ghi vào **binary log** ở format statement, row, hoặc mixed. Row-based (default ở 8.0) là cái mọi CDC pipeline hiện đại (Debezium, Maxwell) consume. Outbox-qua-CDC là integration sạch nếu muốn publish event mà không ghi outbox table.

---

## Schema & Data Model

* **InnoDB là index-organized**: row lưu vật lý theo thứ tự primary key. Chọn primary key cẩn thận — UUID random gây page split và write amplification. Dùng ID sortable (sequential int, ULID, hoặc `UUIDv7`) cho table hot.
* **Secondary index chứa primary key**, không phải con trỏ row vật lý. Primary key rộng → index rộng.
* **Foreign key chạy được** nhưng nhiều shop MySQL to disable — chúng có thể gây lock contention bất ngờ. Lựa chọn gây tranh cãi; document nó.
* **JSON column type** có, query qua `JSON_EXTRACT`/`->>`. Performance và indexing (qua generated column) đang cải thiện nhưng vẫn sau PG JSONB.

---

## Operational Characteristics

| Concern             | Thực tế                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Scale dọc           | Xuất sắc — 1 InnoDB primary scale tới ~50k–100k write TPS với tuning đúng.           |
| Scale read          | Tốt nhất class qua replica + ProxySQL.                                                |
| Write ngang         | Vitess (lớp sharding của YouTube) hoặc PlanetScale (Vitess managed).                  |
| Online DDL          | `ALGORITHM=INPLACE` cho nhiều `ALTER`; `gh-ost` / `pt-online-schema-change` cho phần còn lại. |
| Backup              | `mysqldump` (logical) hoặc Percona XtraBackup (physical, hot).                        |
| Connection pooling  | Footprint per-connection nhẹ hơn PG; pool trong app hay qua ProxySQL.                 |

---

## .NET Integration (ngắn)

```csharp
// EF Core
services.AddDbContext<AppDbContext>(o =>
    o.UseMySql(connStr, ServerVersion.AutoDetect(connStr), mysql =>
    {
        mysql.EnableRetryOnFailure(maxRetryCount: 3);
        mysql.CharSet(CharSet.Utf8Mb4);     // luôn utf8mb4
    }));
```

* Dùng **`MySqlConnector`** (không phải `MySql.Data` deprecated). Async-correct và nhanh hơn.
* Luôn charset `utf8mb4` / collation `utf8mb4_0900_ai_ci` (8.0+). `utf8` là alias 3-byte legacy thiếu emoji / nhiều ký tự CJK.
* Bulk insert: `LOAD DATA LOCAL INFILE` hoặc `MySqlBulkCopy`.

---

## Chọn / Không chọn khi nào

**Chọn khi:**

* Team đã chạy MySQL — quen ops thắng.
* Workload web read-heavy nơi pattern replica-scale-out fit sạch.
* Cần path sharding (Vitess) mà không phải tự invent.
* Muốn per-connection overhead nhẹ hơn Postgres cho concurrency rất cao.

**Đừng chọn khi:**

* Cần SQL surface giàu — CTE, JSONB, partial index của PG, v.v.
* Muốn GIS hạng PostGIS — MySQL có spatial type nhưng không cùng league.
* Muốn analytic workload với complexity window/CTE — đi PG hoặc ClickHouse.

---

## Pitfall

* **Default isolation là Repeatable Read với gap lock** — deadlock bất ngờ. Switch sang RC nếu gap lock không làm việc hữu ích.
* **Primary key UUIDv4** trên InnoDB → write amplification + index bloat. Dùng UUIDv7 / ULID / sequential.
* **Migration trên hot table** không có tool online DDL = downtime.
* **Replication lag khi batch write to** — reader off replica thấy data cũ trong vài phút.
* **`SELECT *`** qua ORM + wide-PK-trong-mọi-index của InnoDB → result set to hơn dự kiến.
* **`utf8` vs `utf8mb4`** — `utf8` cũ là 3-byte, vỡ trên emoji. Luôn `utf8mb4`.

---

## Tham khảo

* [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
* [High Performance MySQL (3rd ed.)](https://www.oreilly.com/library/view/high-performance-mysql/9781449332471/) — Schwartz / Zaitsev / Tkachenko. Vẫn là sách operation kinh điển.
* [Vitess documentation](https://vitess.io/)
* [PlanetScale Engineering Blog](https://planetscale.com/blog) — MySQL ops hiện đại.
* [MySqlConnector docs](https://mysqlconnector.net/) — cho .NET.
