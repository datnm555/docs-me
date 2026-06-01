# MySQL

> The world's most-deployed open-source RDBMS. Battle-hardened, fast on read-heavy workloads, and the default for huge swathes of the LAMP/web world. Less featureful than PostgreSQL, but with rock-solid replication, a vast operator ecosystem, and clear performance characteristics. Read this and PostgreSQL side-by-side to choose between them.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — An open-source ACID-compliant RDBMS. **InnoDB** is the default storage engine (transactional, MVCC, row-level locking). Owned by Oracle; **MariaDB** is the community fork with a similar feature set and a different release cadence.
- **Why** — Decades of production usage have produced very mature replication, a huge tool ecosystem, and well-understood scaling patterns (read replicas, ProxySQL, Vitess). Lighter operational footprint than PostgreSQL for "vanilla" CRUD workloads.
- **When** — Large-scale OLTP, read-heavy web applications, multi-region deployments where mature replication matters, when your team already runs MySQL. Default choice in many companies for historical reasons.
- **Where** — Self-hosted (MySQL Community / Percona Server / MariaDB) or managed (AWS RDS / Aurora MySQL, GCP Cloud SQL, Azure Database for MySQL, PlanetScale, Vitess). From .NET: **MySqlConnector** + **EF Core** (`Pomelo.EntityFrameworkCore.MySql`).
- **How** — Pick **InnoDB** (default). Pick a recent version (8.0+). Decide replication topology early (single-primary + read replicas is 95% of deployments). Use ProxySQL or PlanetScale/Vitess for sharding when single-primary writes saturate.

---

## Strengths

* **Mature replication.** Async + semi-sync + group replication (multi-primary). Tools like Orchestrator/Percona Toolkit are well-established.
* **Read-replica scaling.** Trivial to scale reads by adding replicas; this is the canonical pattern.
* **Operational tooling.** ProxySQL, PMM, Vitess (sharding), gh-ost / pt-online-schema-change (online DDL).
* **Performance on simple queries.** Hash index + tightly-tuned InnoDB pages = very fast point lookups and primary-key joins.
* **Storage-engine pluggability.** InnoDB for OLTP, optional engines for column stores (MyRocks). Different from PostgreSQL's "one engine" approach.
* **MyISAM is dead** for transactional work — but mention it because legacy code still uses it. Migrate.

## Weaknesses

* **Smaller feature surface than PostgreSQL.** No fully-baked partial indexes, no native materialized views, weaker JSON ergonomics, fewer extension capabilities, no native arrays.
* **Schema migrations** can take exclusive locks on large tables. `ALTER TABLE` without `ALGORITHM=INPLACE, LOCK=NONE` is dangerous; tooling like `gh-ost` is often required.
* **Default behaviors used to be unsafe** (silent truncation, weak character handling). Modern defaults (`STRICT_TRANS_TABLES`, `utf8mb4`) are better — but check legacy configs.
* **Replication is logical / row-based** — replication lag during big writes / DDL is a real concern.
* **JSON support exists** but is less rich than PostgreSQL JSONB and slower at scale.

---

## Concurrency & Consistency Model

### InnoDB MVCC

Like PostgreSQL, InnoDB uses MVCC: readers see a snapshot, writes don't block reads of older versions. Old versions live in the **rollback segment** (undo log) rather than in the table itself — so InnoDB doesn't have the "VACUUM" problem PG does. The undo log is reclaimed when no transaction needs the old version.

### Isolation Levels

| Level                   | What it prevents                                                       | What it allows                                          |
| ----------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------- |
| **Read Uncommitted**    | Nothing                                                                | Dirty reads (rarely used)                               |
| **Read Committed**      | Dirty reads                                                            | Non-repeatable reads, phantom reads                     |
| **Repeatable Read** (default!) | Dirty reads, non-repeatable reads, **and most phantom reads** via next-key locking | Some write skew                |
| **Serializable**        | All anomalies                                                          | None (heavy locking)                                    |

> **Default = Repeatable Read** (different from Postgres's Read Committed default). Be careful when porting — long-running reads under RR hold versions and can balloon undo log.

### Locking — the big difference vs Postgres

InnoDB uses **next-key locks** (record + gap) by default at Repeatable Read. They prevent phantom reads but are aggressive — `UPDATE WHERE indexed_column = X` can lock more rows than you'd expect, sometimes whole gap ranges. Many "weird deadlock" stories are about gap locks.

* **Row locks** — straightforward `FOR UPDATE` semantics.
* **Gap locks** — lock the *space between* indexed values. Disabled at Read Committed.
* **Insert intention locks** — special gap lock signalling an upcoming insert.

If gap locks are biting you and you don't need strict phantom-read prevention, switch the session to **Read Committed**.

### Replication

* **Async (default).** Replicas can lag arbitrarily. The standard pattern.
* **Semi-synchronous.** Primary waits for at least one replica to ack the binlog before returning to the client. Safer; modest latency cost.
* **Group Replication / InnoDB Cluster.** Built-in multi-primary with automatic failover.
* **External tools.** Orchestrator, Percona XtraBackup + Pseudo-GTID, MaxScale.

> Reading off a replica = eventual consistency. The pattern: route reads with explicit lag tolerance ("any-replica" / "primary-only") at the application layer (ProxySQL, app-level router).

### Binlog — the underrated feature

Every change writes to the **binary log** in either statement, row, or mixed format. Row-based (default in 8.0) is what every modern CDC pipeline (Debezium, Maxwell) consumes. Outbox-via-CDC is a clean integration if you want to publish events without writing an outbox table.

---

## Schema & Data Model

* **InnoDB is index-organized**: rows are physically stored in primary-key order. Choose primary keys carefully — random UUIDs cause page splits and write amplification. Use a sortable ID (sequential int, ULID, or `UUIDv7`) for hot tables.
* **Secondary indexes contain the primary key**, not a physical row pointer. Wide primary keys → wide indexes.
* **Foreign keys work** but many large MySQL shops disable them — they can cause unexpected lock contention. A controversial choice; document it.
* **JSON column type** exists, query-able with `JSON_EXTRACT`/`->>`. Performance and indexing (via generated columns) are improving but lag PG JSONB.

---

## Operational Characteristics

| Concern             | Reality                                                                              |
| ------------------- | ------------------------------------------------------------------------------------ |
| Vertical scaling    | Excellent — single InnoDB primary scales to ~50k–100k write TPS with right tuning.   |
| Read scaling        | Best-in-class via replicas + ProxySQL.                                                |
| Horizontal write    | Vitess (YouTube's sharding layer) or PlanetScale (managed Vitess).                    |
| Online DDL          | `ALGORITHM=INPLACE` for many `ALTER`s; `gh-ost` / `pt-online-schema-change` for the rest. |
| Backup              | `mysqldump` (logical) or Percona XtraBackup (physical, hot).                          |
| Connection pooling  | Lighter per-connection footprint than PG; pool inside app or via ProxySQL.            |

---

## .NET Integration (short)

```csharp
// EF Core
services.AddDbContext<AppDbContext>(o =>
    o.UseMySql(connStr, ServerVersion.AutoDetect(connStr), mysql =>
    {
        mysql.EnableRetryOnFailure(maxRetryCount: 3);
        mysql.CharSet(CharSet.Utf8Mb4);     // always utf8mb4
    }));
```

* Use **`MySqlConnector`** (not the deprecated `MySql.Data`). It's async-correct and faster.
* Always `utf8mb4` charset / `utf8mb4_0900_ai_ci` collation (8.0+). `utf8` is a legacy 3-byte alias that misses emoji / many CJK chars.
* For bulk inserts, use `LOAD DATA LOCAL INFILE` or `MySqlBulkCopy`.

---

## When to Choose / When Not

**Choose when:**

* Your team already runs MySQL — operational familiarity wins.
* Read-heavy web workloads where the replica-scale-out pattern fits cleanly.
* You need a path to sharding (Vitess) without inventing one.
* You want lighter per-connection overhead than Postgres for very high concurrency.

**Don't choose when:**

* You need a richer SQL surface — PG's CTEs, JSONB, partial indexes, etc.
* You want PostGIS-grade GIS — MySQL has spatial types but it's not in the same league.
* You want analytical workloads with window/CTE complexity — go PG or ClickHouse.

---

## Pitfalls

* **Default isolation is Repeatable Read with gap locks** — surprising deadlocks. Switch to RC if gap locks aren't doing useful work.
* **UUIDv4 primary key** on InnoDB → write amplification + index bloat. Use UUIDv7 / ULID / sequential.
* **Migrations on hot tables** without online DDL tooling = downtime.
* **Replication lag during big batch writes** — readers off a replica see stale data for minutes.
* **`SELECT *`** through ORMs combined with InnoDB's wide-PK-in-every-index → bigger result sets than expected.
* **`utf8` vs `utf8mb4`** — old `utf8` is 3-byte, breaks on emoji. Always `utf8mb4`.

---

## References

* [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
* [High Performance MySQL (3rd ed.)](https://www.oreilly.com/library/view/high-performance-mysql/9781449332471/) — Schwartz / Zaitsev / Tkachenko. Still the canonical operations book.
* [Vitess documentation](https://vitess.io/)
* [PlanetScale Engineering Blog](https://planetscale.com/blog) — modern MySQL ops.
* [MySqlConnector docs](https://mysqlconnector.net/) — for .NET.
