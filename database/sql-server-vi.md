# SQL Server (Microsoft SQL Server)

> RDBMS flagship của Microsoft. Default cho shop enterprise .NET, integration chặt với stack Microsoft (Active Directory, Azure, Power BI), optimizer mature, và tooling giàu (SSMS, Profiler, Query Store). Mạnh ở mô hình "lock-based concurrency với MVCC tùy chọn" — triết lý concurrency khác Postgres/MySQL. Đắt, nhưng thường được nhiều hơn cái trả nếu bạn là tổ chức Microsoft-aligned.

> 🇻🇳 Phiên bản tiếng Việt. English: [`sql-server.md`](./sql-server.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — RDBMS thương mại của Microsoft. ACID, T-SQL, lock-based concurrency mặc định với **Read Committed Snapshot Isolation (RCSI)** tùy chọn cho behavior kiểu MVCC. Bao gồm columnstore index, in-memory OLTP (Hekaton), Always On Availability Groups, Service Broker, và integration với data platform Microsoft.
- **Tại sao** — Best-in-class cho enterprise stack Microsoft. Optimizer xuất sắc. Tooling (SSMS, Profiler, Query Store, Extended Events) polished nhất ngành. AlwaysOn và Failover Cluster Instance đã mature.
- **Khi nào** — Shop .NET / Windows-first. Enterprise đã trên stack Microsoft. Workload cần integration chặt với SSAS / SSRS / Power BI. Workload mixed OLTP + analytical qua columnstore.
- **Ở đâu** — On-prem (Standard / Enterprise edition), Azure SQL Database (PaaS), Azure SQL Managed Instance, AWS RDS for SQL Server, GCP Cloud SQL for SQL Server. Cũng chạy trên Linux và container từ 2017.
- **Như thế nào** — Từ .NET: **`Microsoft.Data.SqlClient`** + **EF Core** (`Microsoft.EntityFrameworkCore.SqlServer`). Bật **RCSI** trên DB mới (`ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON`) trừ khi có lý do cụ thể.

---

## Điểm mạnh

* **Optimizer.** Thường vượt peer trên query phức tạp nhiều join, nhiều CTE. Adaptive query plan, parameter sniffing diagnostics, Query Store cho plan history.
* **Tooling.** SSMS là DB IDE đầy đủ nhất ngành. Query Store, Extended Events, Profiler.
* **OLTP / OLAP mixed qua columnstore.** 1 DB có thể serve workload transactional + analytical (clustered + non-clustered columnstore).
* **High availability.** Always On Availability Group (multi-replica với sync/async + failover tự động), FCI, Database Mirroring (legacy).
* **In-memory OLTP (Hekaton).** Lock-free, latch-free in-memory table cho TPS cực cao — niche nhưng mạnh.
* **Stored procedure tooling.** T-SQL là procedural language thật. Debugger, profiler, intellisense — không RDBMS mainstream nào khác có support T-SQL level này.
* **Service Broker.** Message queue built-in với transactional semantic cho pattern kiểu outbox.

## Điểm yếu

* **Cost license.** Enterprise Edition đắt (per-core licensing). Cloud (Azure SQL) cạnh tranh nhưng Standard/Enterprise on-prem có thể chiếm dominantly cost phần cứng.
* **Locking-by-default** có thể confuse engineer từ Postgres/MySQL — bật RCSI để có read kiểu MVCC.
* **Schema management** ít mature hơn hệ sinh thái open-source (Flyway / Liquibase chạy được, nhưng EF Core migration hoặc DbUp là default).
* **Linux / container support** tốt nhưng Windows vẫn là home turf cho một số feature (Active Directory auth, một số HA config).
* **Vendor lock-in.** Vào dễ hơn ra.

---

## Concurrency & Consistency Model

### Hai triết lý concurrency trong 1 DB

SQL Server *độc nhất* khi ship **cả hai** mô hình concurrency lock-based kinh điển và option MVCC (RCSI / Snapshot Isolation). Chọn giữa chúng là quyết định cấp deployment.

* **Lock-based (default).** Reader block writer; writer block reader. `READPAST`, `NOLOCK`, `HOLDLOCK` hint nhan nhản trong code legacy.
* **RCSI** (`READ_COMMITTED_SNAPSHOT ON` ở cấp DB). Reader thấy snapshot — không read-write block ở Read Committed. **Bật cho DB mới.**
* **Snapshot Isolation** (riêng với RCSI). Isolation level snapshot thật opt-in per-transaction.

### Isolation Level

| Level                          | Chặn được                                                                                       | Note                                                                                        |
| ------------------------------ | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Read Uncommitted**           | Không gì                                                                                         | Chỉ dùng cho query monitoring không quan tâm correctness.                                    |
| **Read Committed** (default)   | Dirty read                                                                                       | Lock-based mặc định. Với RCSI bật, thành kiểu MVCC — reader thấy snapshot, không block.     |
| **Repeatable Read**             | Dirty + non-repeatable read                                                                      | Giữ shared lock suốt transaction. Đắt.                                                       |
| **Snapshot**                   | Dirty + non-repeatable + phantom read qua row versioning                                         | MVCC. Cần opt-in.                                                                            |
| **Serializable**               | Mọi thứ                                                                                          | Lock nặng. Range lock trên index.                                                            |

### Lock bạn sẽ gặp

* **Row, Page, Extent, Table lock** — escalation tự động (~5000 row lock trên 1 table → table lock). Watch nó.
* **Latch** vs **lock** — latch là primitive sync internal ngắn; lock là user-visible. Cả 2 có thể là nguồn contention.
* **Lock escalation threshold** tunable (`ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE)`).

### Replication / HA

* **Always On Availability Group** — multi-replica với sync và async replica, failover tự động. Câu chuyện HA hiện đại.
* **Failover Cluster Instance (FCI).** Shared-storage clustering — cũ hơn nhưng vẫn dùng.
* **Log shipping.** Backup transaction log định kỳ → restore trên standby. Đơn giản và đáng tin.
* **Read replica** trong AG có thể serve workload read-only.

> AG ở async mode nghĩa là replica read vẫn lag. Tag connection string (`ApplicationIntent=ReadOnly`) để route read.

---

## Schema & Data Model

* **Clustered index = table.** Table lưu vật lý theo thứ tự clustered index. Chọn cẩn thận — `IDENTITY INT` thường ổn; UUID không `NEWSEQUENTIALID()` gây fragmentation.
* **`IDENTITY` vs `SEQUENCE`** — `SEQUENCE` linh hoạt hơn và cross-table.
* **`HierarchyId`, `Geography`, `Geometry`** — native type cho cấu trúc cây và GIS.
* **Temporal table** — bảng system-versioned history built-in. `FOR SYSTEM_TIME AS OF '...'` cho audit query. Feature xuất sắc.
* **Columnstore index** — clustered hay non-clustered, cho compression 10x–100x và speed up query analytical.

---

## Operational Characteristics

| Concern             | Thực tế                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Scale dọc           | Xuất sắc — 1 server scale tới workload khổng lồ (Enterprise tier).                   |
| Scale read          | AG read replica; readable secondary.                                                  |
| Write ngang         | Sharding native limited (custom "data-dependent routing"). Dùng Azure SQL Hyperscale cho OLTP rất to. |
| Online operation    | Online index rebuild (Enterprise), online schema change (limited).                    |
| Backup              | Native `BACKUP DATABASE`; differential / log backup cho PITR.                         |
| Monitoring          | Query Store + Extended Events + dashboard SSMS.                                       |

---

## .NET Integration (ngắn)

```csharp
// EF Core
services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(connStr, sql =>
    {
        sql.EnableRetryOnFailure(maxRetryCount: 3);    // transient error (Azure SQL)
        sql.CommandTimeout(30);
    }));

// Raw / Dapper
await using var conn = new SqlConnection(connStr);
await conn.OpenAsync(ct);
var orders = await conn.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE Status IN @statuses",
    new { statuses = new[] { "Pending", "Paid" } });
```

* Dùng **`Microsoft.Data.SqlClient`** (client hiện đại) — không phải `System.Data.SqlClient` legacy.
* `SqlBulkCopy` là path insert nhanh kinh điển — nhanh gấp nhiều lần `INSERT` từng row.
* `MultipleActiveResultSets=True` (MARS) cho nested reader — nhưng có caveat; tránh nếu có thể.
* **Azure SQL throttle** — transient error là thường. Luôn bật retry-on-failure hoặc wrap Polly.

---

## Chọn / Không chọn khi nào

**Chọn khi:**

* Stack là Microsoft / .NET — tiết kiệm tuần làm integration.
* Cần temporal table / columnstore / Service Broker / Hekaton first-class.
* Complexity query benefit từ optimizer hạng top.
* Đã trả enterprise agreement Microsoft (có thể effectively free).

**Đừng chọn khi:**

* Cost dominant và không có lý do MS-stack — Postgres tương đương hoặc vượt cho hầu hết OLTP miễn phí.
* Muốn hệ sinh thái operator open-source rộng nhất (Postgres / MySQL thắng).
* Cần chạy combo hardware / OS exotic nhất — Linux chạy được nhưng Windows là "home".

---

## Pitfall

* **RCSI tắt mặc định** — nhiều dev giả định behavior MVCC. Bật RCSI hoặc bạn sẽ có reader-blocking bất ngờ.
* **`NOLOCK` khắp code legacy** — smell. Nghĩa là ai đó dính blocking và chọn dirty read thay vì fix root cause (thường: bật RCSI).
* **Parameter sniffing.** Plan của lần execute đầu được cache cho parameter rất khác sau đó → performance kinh khủng. Mitigation: `OPTION (RECOMPILE)`, `OPTIMIZE FOR UNKNOWN`, plan guide.
* **Surprise lock escalation** — bulk operation trên table to escalate thành table lock. Batch nhỏ hơn hoặc disable escalation trên hot table.
* **Clustered key `GUID` không `NEWSEQUENTIALID`** — địa ngục fragmentation.
* **Implicit conversion trong JOIN / WHERE** disable index seek (ví dụ so `VARCHAR` với `NVARCHAR`). Watch plan thật.
* **Tier Azure SQL DTU vs vCore** — DTU opaque; vCore cho reason về CPU/memory. Dùng vCore cho workload không tầm thường.

---

## Tham khảo

* [SQL Server Documentation](https://learn.microsoft.com/sql/sql-server/)
* [Brent Ozar's blog](https://www.brentozar.com/) — operations / performance.
* [SQLPerformance.com](https://sqlperformance.com/) — bài optimization sâu.
* Itzik Ben-Gan — *T-SQL Fundamentals*, *T-SQL Querying*. Reference T-SQL kinh điển.
* Kalen Delaney — *SQL Server Internals*. Internal engine.
* [Microsoft.Data.SqlClient docs](https://learn.microsoft.com/sql/connect/ado-net/microsoft-data-sqlclient) — cho .NET.
