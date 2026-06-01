# SQL Server (Microsoft SQL Server)

> Microsoft's flagship RDBMS. The default for .NET enterprise shops, with tight integration into the Microsoft stack (Active Directory, Azure, Power BI), a mature optimizer, and rich tooling (SSMS, Profiler, Query Store). Strong on the "lock-based concurrency with optional MVCC" model — a different concurrency philosophy than Postgres/MySQL. Expensive, but you often get more than you pay for if you're a Microsoft-aligned organization.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — Microsoft's commercial RDBMS. ACID, T-SQL, lock-based concurrency by default with optional **Read Committed Snapshot Isolation (RCSI)** to give MVCC-like behavior. Includes columnstore indexes, in-memory OLTP (Hekaton), Always On Availability Groups, Service Broker, and integration with the broader Microsoft data platform.
- **Why** — Best-in-class for Microsoft-stack enterprises. The optimizer is excellent. Tooling (SSMS, Profiler, Query Store, Extended Events) is the most polished in the industry. AlwaysOn and Failover Cluster Instances are mature.
- **When** — .NET / Windows-first shops. Enterprises already on the Microsoft stack. Workloads requiring tight integration with SSAS / SSRS / Power BI. Mixed OLTP + analytical workloads via columnstore.
- **Where** — On-prem (Standard / Enterprise editions), Azure SQL Database (PaaS), Azure SQL Managed Instance, AWS RDS for SQL Server, GCP Cloud SQL for SQL Server. Also runs on Linux and containers since 2017.
- **How** — From .NET: **`Microsoft.Data.SqlClient`** + **EF Core** (`Microsoft.EntityFrameworkCore.SqlServer`). Turn on **RCSI** on new databases (`ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON`) unless you have a specific reason not to.

---

## Strengths

* **Optimizer.** Routinely outperforms peers on complex multi-join, multi-CTE queries. Adaptive query plans, parameter sniffing diagnostics, Query Store for plan history.
* **Tooling.** SSMS is the most full-featured DB IDE in the industry. Query Store, Extended Events, Profiler.
* **Mixed OLTP / OLAP via columnstore.** A single DB can serve transactional + analytical workloads (clustered + non-clustered columnstore).
* **High availability.** Always On Availability Groups (multi-replica with sync/async + automatic failover), FCI, Database Mirroring (legacy).
* **In-memory OLTP (Hekaton).** Lock-free, latch-free in-memory tables for extreme TPS — niche but powerful.
* **Stored procedure tooling.** T-SQL is a real procedural language. Debugger, profiler, intellisense — none of the other mainstream RDBMS has this level of T-SQL support.
* **Service Broker.** Built-in messaging queue with transactional semantics for outbox-like patterns.

## Weaknesses

* **Licensing cost.** Enterprise Edition is expensive (per-core licensing). Cloud (Azure SQL) is competitive but Standard/Enterprise on-prem can dominate hardware cost.
* **Locking-by-default** can confuse engineers coming from Postgres/MySQL — turn on RCSI to get MVCC-style reads.
* **Schema management** is less mature than the open-source ecosystem (Flyway / Liquibase work, but EF Core migrations or DbUp are the default).
* **Linux / container support** is good but Windows is still the home turf for some features (Active Directory authentication, certain HA configurations).
* **Vendor lock-in.** Easier to start than to leave.

---

## Concurrency & Consistency Model

### Two concurrency philosophies in one DB

SQL Server is *unique* in shipping **both** a classical lock-based concurrency model and an MVCC option (RCSI / Snapshot Isolation). Choosing between them is a deployment-level decision.

* **Lock-based (default).** Readers block writers; writers block readers. `READPAST`, `NOLOCK`, `HOLDLOCK` hints abound in legacy code.
* **RCSI** (`READ_COMMITTED_SNAPSHOT ON` at DB level). Readers see a snapshot — no read-write blocking under Read Committed. **Turn this on for new DBs.**
* **Snapshot Isolation** (separate from RCSI). True snapshot isolation level you opt into per-transaction.

### Isolation Levels

| Level                          | What it prevents                                                                                | Notes                                                                                       |
| ------------------------------ | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Read Uncommitted**           | Nothing                                                                                          | Use only for monitoring queries you don't care about correctness.                            |
| **Read Committed** (default)   | Dirty reads                                                                                      | Lock-based by default. With RCSI on, becomes MVCC-style — readers see snapshots, no blocking. |
| **Repeatable Read**             | Dirty reads + non-repeatable reads                                                               | Holds shared locks for duration of transaction. Expensive.                                    |
| **Snapshot**                   | Dirty + non-repeatable + phantom reads via row versioning                                        | MVCC. Requires opt-in.                                                                       |
| **Serializable**               | Everything                                                                                       | Lock-heavy. Range locks on indexes.                                                          |

### Locks you'll meet

* **Row, Page, Extent, Table locks** — escalation happens automatically (~5000 row locks on one table → table lock). Watch for it.
* **Latches** vs **locks** — latches are short-lived internal sync primitives; locks are user-visible. Both can be contention sources.
* **Lock escalation thresholds** are tunable (`ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE)`).

### Replication / HA

* **Always On Availability Groups** — multi-replica with sync and async replicas, automatic failover. The modern HA story.
* **Failover Cluster Instances (FCI).** Shared-storage clustering — older but still used.
* **Log shipping.** Periodic transaction log backup → restore on standby. Simple and reliable.
* **Read replicas** in AG can serve read-only workloads.

> AGs in async mode still mean replica reads can lag. Tag connection strings (`ApplicationIntent=ReadOnly`) to route reads.

---

## Schema & Data Model

* **Clustered index = the table.** Tables are stored physically in clustered-index order. Choose carefully — `IDENTITY INT` is usually fine; UUIDs without `NEWSEQUENTIALID()` cause fragmentation.
* **`IDENTITY` vs `SEQUENCE`** — `SEQUENCE` is more flexible and cross-table.
* **`HierarchyId`, `Geography`, `Geometry`** — native types for tree structures and GIS.
* **Temporal tables** — built-in system-versioned history tables. `FOR SYSTEM_TIME AS OF '...'` for audit queries. Excellent feature.
* **Columnstore indexes** — clustered or non-clustered, give 10x–100x compression and analytical query speedup.

---

## Operational Characteristics

| Concern             | Reality                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Vertical scaling    | Excellent — single server scales to massive workloads (Enterprise tier).             |
| Read scaling        | AG read replicas; readable secondaries.                                              |
| Horizontal write    | Limited native sharding (custom "data-dependent routing"). Use Azure SQL Hyperscale for huge OLTP. |
| Online operations   | Online index rebuild (Enterprise), online schema changes (limited).                  |
| Backup              | Native `BACKUP DATABASE`; differential / log backups for PITR.                       |
| Monitoring          | Query Store + Extended Events + SSMS dashboards.                                     |

---

## .NET Integration (short)

```csharp
// EF Core
services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(connStr, sql =>
    {
        sql.EnableRetryOnFailure(maxRetryCount: 3);    // transient errors (Azure SQL)
        sql.CommandTimeout(30);
    }));

// Raw / Dapper
await using var conn = new SqlConnection(connStr);
await conn.OpenAsync(ct);
var orders = await conn.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE Status IN @statuses",
    new { statuses = new[] { "Pending", "Paid" } });
```

* Use **`Microsoft.Data.SqlClient`** (the modern client) — not the legacy `System.Data.SqlClient`.
* `SqlBulkCopy` is the canonical fast-insert path — orders of magnitude faster than per-row `INSERT`.
* `MultipleActiveResultSets=True` (MARS) for nested readers — but it has caveats; avoid if you can.
* **Azure SQL throttles** — transient errors are normal. Always enable retry-on-failure or wrap with Polly.

---

## When to Choose / When Not

**Choose when:**

* Your stack is Microsoft / .NET — you'll save weeks of integration work.
* You need first-class temporal tables / columnstore / Service Broker / Hekaton.
* The complexity of your queries benefits from a top-tier optimizer.
* You're already paying for Microsoft enterprise agreements (it might be effectively free).

**Don't choose when:**

* Cost is dominant and you have no MS-stack reason — Postgres equals or exceeds it for most OLTP for free.
* You want the broadest open-source operator ecosystem (Postgres / MySQL win).
* You need to run on the most exotic hardware / OS combos — Linux works but Windows is "home".

---

## Pitfalls

* **RCSI off by default** — many devs assume MVCC behavior. Turn RCSI on or you'll have unexpected reader-blocking.
* **`NOLOCK` everywhere in legacy code** — a smell. Means someone hit blocking and chose dirty reads instead of fixing the root cause (often: enable RCSI).
* **Parameter sniffing.** First execution's plan gets cached for subsequent very-different parameters → terrible performance. Mitigations: `OPTION (RECOMPILE)`, `OPTIMIZE FOR UNKNOWN`, plan guides.
* **Lock escalation surprise** — bulk operations on large tables escalate to table locks. Batch in smaller chunks or disable escalation on hot tables.
* **`GUID` clustered keys without `NEWSEQUENTIALID`** — fragmentation hell.
* **Implicit conversions in JOINs / WHEREs** disable index seeks (e.g., comparing `VARCHAR` to `NVARCHAR`). Watch the actual plan.
* **Azure SQL DTU vs vCore tiers** — DTU is opaque; vCore lets you reason about CPU/memory. Use vCore for non-trivial workloads.

---

## References

* [SQL Server Documentation](https://learn.microsoft.com/sql/sql-server/)
* [Brent Ozar's blog](https://www.brentozar.com/) — operations / performance.
* [SQLPerformance.com](https://sqlperformance.com/) — deep-dive optimization articles.
* Itzik Ben-Gan — *T-SQL Fundamentals*, *T-SQL Querying*. Canonical T-SQL reference.
* Kalen Delaney — *SQL Server Internals*. Engine internals.
* [Microsoft.Data.SqlClient docs](https://learn.microsoft.com/sql/connect/ado-net/microsoft-data-sqlclient) — for .NET.
