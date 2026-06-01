# PostgreSQL

> The open-source RDBMS that "just works" for most workloads. Strict relational + JSON-friendly + serious about correctness. MVCC done well, the richest type system of any mainstream DB, and a concurrency model that scales reads cleanly. If you have no specific reason to pick something else, **pick this**.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — A free, open-source, ACID-compliant relational database with **MVCC concurrency**, strong SQL standard support, native JSON/JSONB, GIS, full-text search, and an extension model that bolts on time-series (TimescaleDB), vector search (pgvector), and more.
- **Why** — Among open-source RDBMS, PostgreSQL has the best combination of correctness, feature surface, and operational maturity. MVCC means readers never block writers. JSONB blurs the SQL/NoSQL boundary without losing transactions.
- **When** — Default choice for new services. Strong fit when you need transactions, complex queries, joins, constraints, **and** semi-structured fields in the same store. Excellent for analytical workloads up to several TB.
- **Where** — OLTP services, multi-tenant SaaS, geospatial systems, search-with-structure (jsonb + GIN), event stores (with care), embeddings/vector via pgvector. Less ideal for write throughput beyond ~100k tx/sec without sharding (Citus).
- **How** — Run on managed services (AWS RDS / Aurora Postgres, GCP Cloud SQL / AlloyDB, Azure Database for PostgreSQL, Supabase, Neon, Crunchy). Self-host is fine but vacuum + WAL ops require attention. From .NET: **Npgsql** + **EF Core** (`Microsoft.EntityFrameworkCore.PostgreSQL`).

---

## Strengths

* **MVCC concurrency.** Readers never block writers. Every transaction sees a consistent snapshot. No reader locks for normal queries.
* **Standards-compliant SQL.** Window functions, CTEs (including recursive), `LATERAL`, materialized views, partial / expression / covering indexes, custom types, custom aggregates, custom operators.
* **JSONB.** Indexed, queryable, schemaless-when-you-want. Fully transactional alongside relational columns.
* **Extensions.** PostGIS (spatial), TimescaleDB (time-series), pgvector (embeddings), pg_partman (partitioning), pglogical (logical replication), Citus (sharding). Most are first-class production code.
* **Crash safety.** Synchronous commit + WAL; survives hardware loss without data loss when configured correctly.
* **Cost.** Free. No per-core licensing.

## Weaknesses

* **Write scaling beyond a single primary** is harder than systems built for it. You can read-replicate easily; you cannot multi-primary write without sharding (Citus) or app-level partitioning.
* **VACUUM** required to reclaim space from MVCC tuples. Misconfigured autovacuum on large/hot tables is the classic Postgres outage.
* **Replication lag** on async replicas — your read replica may be seconds behind the primary.
* **Connection cost.** Each connection is a process; ~10MB+ memory. PgBouncer is virtually mandatory above a few hundred connections.
* **DDL is mostly transactional but can take strong locks.** `ALTER TABLE` on a hot table needs care (use `CONCURRENTLY` for indexes; tools like `pg_repack` for big rewrites).

---

## Concurrency & Consistency Model

### MVCC in one paragraph

Every row has hidden `xmin` (creating transaction) and `xmax` (deleting transaction) columns. A transaction sees a row if `xmin` is committed before its snapshot and `xmax` is either null or after the snapshot. Updates are implemented as `delete + insert` — the old version remains until `VACUUM` collects it. **Result:** readers never block writers and writers never block readers (except on the same row update).

### Isolation Levels

| Level                   | What it prevents                                                      | What it allows                                          | Lock cost     |
| ----------------------- | --------------------------------------------------------------------- | ------------------------------------------------------- | ------------- |
| **Read Uncommitted**    | (Treated as Read Committed in PG — dirty reads never possible)         | n/a                                                     | None          |
| **Read Committed** (default) | Dirty reads                                                       | Non-repeatable reads, phantom reads                     | Row-level only |
| **Repeatable Read**     | Dirty reads, non-repeatable reads, **phantom reads** (PG goes stronger than SQL standard via snapshot isolation) | Some write skew                | Snapshot      |
| **Serializable**        | All anomalies. SSI = Serializable Snapshot Isolation.                  | None (but may abort with `40001`)                       | Predicate-locks |

**Practical recipe:**

* Read Committed (default) — most CRUD. Cheap.
* Repeatable Read — multi-statement reads that must agree.
* Serializable — financial / counter logic where write skew matters. Retry on `serialization_failure (40001)`.

### Locks you'll actually meet

* `FOR UPDATE` — row-level lock for "select then update" workflows. Default for `SELECT ... FOR UPDATE`.
* `FOR UPDATE SKIP LOCKED` — the canonical work-queue pattern. Workers grab unlocked rows and process them.
* Advisory locks (`pg_advisory_lock`) — application-defined mutexes outside the row model.

### Replication

* **Streaming replication** (physical): WAL shipped byte-for-byte. Replicas are read-only; can fail over.
* **Logical replication**: per-table publish/subscribe; lets the subscriber be a different version or different schema. Cross-DB CDC entry point.
* **Sync vs async commit** is per-transaction tunable (`synchronous_commit`).

> **Default replication is eventual on the replica.** If you read from a replica immediately after writing to the primary, you may not see your write. Use the primary for read-your-writes, or use `synchronous_commit = remote_apply` (slow).

---

## Schema & Data Model

* Start fully normalized. Add denormalization (materialized views, JSONB rollups, generated columns) only after you see the query pattern.
* `JSONB` columns are a powerful escape hatch for sparse / variable shapes — **but** treat them as a typed contract; don't let arbitrary client JSON in.
* Generated columns (`GENERATED ALWAYS AS ... STORED`) let you index a computed value (e.g. `lower(email)`).
* Partitioning (range / list / hash) helps with very large tables — but adds operational complexity. Default to plain tables until they hurt.

---

## Operational Characteristics

| Concern             | Reality                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Vertical scaling    | Excellent — single primary handles tens of thousands of write TPS on modern hardware. |
| Horizontal write    | Requires Citus or application sharding.                                              |
| Read replicas       | Trivial to add; pick one for OLAP / async reads.                                     |
| Backup              | `pg_basebackup` + WAL archive → point-in-time recovery.                              |
| Schema migrations   | Tooling: Flyway, Liquibase, EF Core migrations. Strong locks on `ALTER TABLE`.       |
| Connection pooling  | PgBouncer (transaction-pooling mode is the common default).                          |

---

## .NET Integration (short)

```csharp
// EF Core
services.AddDbContext<AppDbContext>(o =>
    o.UseNpgsql(connStr, npg =>
    {
        npg.EnableRetryOnFailure(maxRetryCount: 3);   // transient retry for serialization aborts
        npg.MapEnum<OrderStatus>("order_status");      // native enum mapping
    }));

// Raw / Dapper
await using var conn = new NpgsqlConnection(connStr);
var rows = await conn.QueryAsync<Order>(
    "SELECT * FROM orders WHERE status = ANY(@statuses)",
    new { statuses = new[] { "Pending", "Paid" } });
```

* `Npgsql` supports native arrays, `jsonb`, enums, `uuid`, `interval`, `inet`.
* For high-throughput inserts, use binary `COPY` via `NpgsqlBinaryImporter` — orders of magnitude faster than per-row `INSERT`.
* Pair with PgBouncer in transaction-pooling mode; **disable prepared statements** (`No Reset On Close=true; Max Auto Prepare=0`) when behind PgBouncer.

---

## When to Choose / When Not

**Choose when:**

* You want one DB to handle relational + JSON + GIS + full-text + analytics without bolting on three more services.
* Read-heavy workload — MVCC + read replicas scale this easily.
* Need real transactions across rich data types.

**Don't choose when:**

* Sustained write throughput > ~100k tx/sec on a single shardless DB.
* Pure document workload with no relational queries — Mongo or DynamoDB may be simpler.
* Hard real-time analytics across petabytes — that's ClickHouse / BigQuery territory.

---

## Pitfalls

* **Bloat from never vacuuming hot rows** — autovacuum settings tuned for the table size, not server-default.
* **Long-running transactions** prevent vacuum and pile up MVCC garbage. Watch `pg_stat_activity` for `state = idle in transaction`.
* **`ORDER BY` without index** at scale — Postgres will sort in memory and spill to disk silently. Add the index.
* **Connection storms** — without PgBouncer, a Node/.NET app can open thousands of connections and OOM the server.
* **Reading from replica expecting strict consistency** — see replication note above. Read your own writes on the primary.
* **`SELECT *` over JSONB** with no projection — sends huge payloads. Project the keys you need with `->`.

---

## References

* [PostgreSQL documentation](https://www.postgresql.org/docs/current/) — the manual is excellent.
* [Postgres Weekly](https://postgresweekly.com/) — newsletter.
* Markus Winand — *Use The Index, Luke!* — language-agnostic index/query tuning.
* Laurenz Albe — *PostgreSQL Server Programming* / Cybertec blog.
* [Npgsql documentation](https://www.npgsql.org/) — for .NET.
