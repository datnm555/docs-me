# Database Comparison — PostgreSQL · MySQL · SQL Server · MongoDB

> Side-by-side comparison and decision guide. Each DB is genuinely good at different things — the question is never "which is best?", but "which fits **this workload**?". Read each per-DB doc first; use this for the final pick.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — A decision guide across four mainstream databases: PostgreSQL, MySQL, SQL Server, MongoDB. Compares concurrency model, scalability, feature surface, .NET integration, and operational footprint.
- **Why** — Picking a DB is high-cost, hard-to-reverse. The wrong default ("we always use X") makes everything from query design to operations more painful for years.
- **When** — Starting a new service. Considering a migration. Building a multi-database microservices architecture and choosing a DB per service.
- **Where** — Top-level guide for the `database/` folder. Cross-references each per-DB deep dive and the .NET integration notes there.
- **How** — Identify the workload dimensions that actually matter (transactional? document-shaped? read-heavy?). Score the candidates on those dimensions. Don't reason from "we know X" alone — knowing X has weight, but doesn't override workload fit.

---

## Side-by-Side Matrix

| Dimension                    | **PostgreSQL**                       | **MySQL (InnoDB)**                   | **SQL Server**                                | **MongoDB**                                |
| ---------------------------- | ------------------------------------- | ------------------------------------ | --------------------------------------------- | ------------------------------------------ |
| **Data model**               | Relational + JSONB                    | Relational                            | Relational                                     | Document (BSON)                            |
| **License**                  | Free (PostgreSQL license)             | Free (GPL) / Commercial               | Commercial (Express free, tiered)              | SSPL / Atlas SaaS                          |
| **Concurrency model**        | MVCC (snapshot)                       | MVCC via InnoDB + next-key locks      | Lock-based default; **RCSI** for MVCC           | Single-document atomic + multi-doc TX (4.0+) |
| **Default isolation**        | Read Committed                        | Repeatable Read (with gap locks)      | Read Committed (lock-based unless RCSI)        | n/a (per-op concerns)                      |
| **Strongest isolation**      | Serializable (SSI)                    | Serializable                          | Snapshot / Serializable                        | Linearizable read + majority write         |
| **Vertical scaling**         | Excellent                              | Excellent                             | Excellent                                      | Good                                       |
| **Horizontal write scaling** | Citus (extension)                     | **Vitess / PlanetScale**              | Limited; Hyperscale on Azure                   | **Native sharding**                        |
| **Read replicas**            | Streaming / logical                   | Async / semi-sync / group              | AlwaysOn AG                                    | Replica set secondaries                    |
| **JSON support**             | **JSONB** (best in class)             | JSON (improving)                      | JSON (decent)                                  | Native (BSON)                              |
| **GIS**                      | **PostGIS** (best in class)           | Spatial (basic)                       | Geography / Geometry (good)                    | 2dsphere (decent)                          |
| **Full-text search**         | Built-in tsvector + extensions        | Built-in (decent)                      | Full-Text Search                                | Atlas Search (Lucene)                      |
| **CDC / change streams**     | Logical replication / Debezium        | **Binlog → Debezium / Maxwell**       | CDC / Change Tracking                          | **Change Streams** (native)                |
| **In-memory engine**         | UNLOGGED tables                       | NDB cluster (niche)                   | Hekaton (in-memory OLTP)                       | WiredTiger (cache-heavy)                   |
| **Columnar**                 | cstore_fdw / Citus columnar            | MyRocks / ClickHouse-as-companion     | **Columnstore index** (built-in)               | n/a (document)                             |
| **Cloud-native options**     | Aurora, AlloyDB, Supabase, Neon       | Aurora, PlanetScale, Vitess           | Azure SQL DB, Hyperscale, RDS                  | **Atlas** (dominant)                       |
| **.NET driver maturity**     | Npgsql (excellent)                    | MySqlConnector (excellent)            | Microsoft.Data.SqlClient (first-party)         | MongoDB.Driver (first-party)               |
| **EF Core provider**         | First-party (`Npgsql.EFCore`)         | Community (`Pomelo`)                  | First-party (Microsoft)                        | n/a (use driver / EF Mongo niche)          |
| **Operational footprint**    | Medium (vacuum, WAL tuning)           | Medium (binlog, replica lag)          | Medium-high (more knobs, more tooling)         | Medium (cluster + shard ops)               |
| **Schema flexibility**       | Strict + JSONB escape hatch           | Strict                                | Strict                                          | **Flexible by default** (validation optional) |
| **Best-fit workloads**       | "Default for everything"               | Read-heavy web / scale-out via Vitess  | .NET enterprises; mixed OLTP+analytical         | Document data; write scale-out             |

---

## Decision Tree

```
Is your data document-shaped?
  (entity = read/written as a unit, variable fields, deeply nested arrays)
├── Yes  → MongoDB
└── No
    │
    Are you in a Microsoft / .NET enterprise environment?
    ├── Yes
    │   └── Do you need temporal tables / columnstore / Service Broker /
    │       tight AD/SSAS/Power BI integration?
    │       ├── Yes → SQL Server
    │       └── No, cost matters → PostgreSQL
    │
    └── No
        │
        Do you need a path to horizontal write sharding without inventing it?
        ├── Yes → MySQL (Vitess / PlanetScale)
        └── No
            │
            Do you need rich SQL (CTE, JSONB, partial indexes, PostGIS, extensions)?
            ├── Yes → PostgreSQL
            └── No, your team already runs MySQL with ops in place → MySQL
```

This is a heuristic. Edge cases exist. Read the per-DB docs.

---

## Use Case → DB Mapping

### "Default OLTP service"

| Sub-context                                   | Pick                                |
| --------------------------------------------- | ----------------------------------- |
| Greenfield, no existing stack constraints     | **PostgreSQL**                       |
| Existing .NET / Microsoft shop                | **SQL Server** or **PostgreSQL**     |
| Team's operational muscle is already MySQL    | **MySQL**                            |

### "Multi-tenant SaaS"

* **PostgreSQL** — `tenant_id`-prefixed schemas or RLS. Citus when you outgrow a single primary.
* **MySQL** with Vitess — when tenant counts go large and per-tenant isolation is needed.
* **MongoDB** — when each tenant's data shape is heterogeneous.

### "Catalog / CMS / activity stream"

* **MongoDB** — variable fields, denormalized read patterns.
* **PostgreSQL** with JSONB — same shape benefits, plus relational queries.

### "Mixed OLTP + analytical"

* **SQL Server** columnstore indexes — same DB, two workloads.
* **PostgreSQL** + read replica running OLAP queries — works, with caveats.
* **PostgreSQL** + ClickHouse via Debezium — for genuinely analytical scale.

### "Event store / audit log"

* **PostgreSQL** — strong consistency, durable WAL, easy to query history.
* **MongoDB** — fine if events are document-shaped and you want sharding-by-aggregate.
* Specialized: EventStoreDB, Kafka + KSQL — when "event store" is the *only* job.

### "Geospatial workloads"

* **PostgreSQL + PostGIS** — best in class.
* **SQL Server** — decent native geography/geometry.
* **MongoDB** — 2dsphere indexes work for typical "find within radius" queries.

### "Vector / embeddings (RAG, semantic search)"

* **PostgreSQL + pgvector** — fully integrated with relational data.
* **MongoDB Atlas Vector Search** — fully managed.
* Specialized: Pinecone, Weaviate, Qdrant — when vector *is* the workload.

### "High-cardinality time-series"

* **PostgreSQL + TimescaleDB** — relational + time-series in one.
* Specialized: ClickHouse, InfluxDB, VictoriaMetrics — when ingest rate dominates.

---

## Cost — Beyond License Fees

License is the visible cost. **Operational cost** is usually larger:

| Cost type                                  | Postgres | MySQL    | SQL Server                    | Mongo                                 |
| ------------------------------------------ | -------- | -------- | ----------------------------- | ------------------------------------- |
| Software license                           | Free     | Free     | $$$$ (Enterprise) / Free (Express) | SSPL / Atlas $$                       |
| Skilled DBA / engineer availability        | High     | High     | High (.NET / enterprise)      | Medium-High (varies by region)         |
| Per-node memory overhead                   | High     | Medium   | High                          | High                                   |
| Connection cost (per-connection memory)    | High (mitigate w/ PgBouncer) | Low | Medium                | Low                                   |
| Migration / DDL operational cost           | Medium   | High (gh-ost often needed)   | Low (online ops in Enterprise) | Low (schemaless writes)               |
| Backup / restore (managed)                  | Easy on RDS/Aurora | Easy on RDS/Aurora | Easy on Azure SQL | Trivial on Atlas         |

Cloud-managed offerings (Aurora, Azure SQL, Atlas) flatten a lot of operational cost — but ramp price quickly.

---

## Migration Considerations

| From → To                  | Friction                                                                                       |
| -------------------------- | ---------------------------------------------------------------------------------------------- |
| MySQL → PostgreSQL         | Medium. SQL dialect differences, `AUTO_INCREMENT` → `SERIAL`/`IDENTITY`, replication differs. Tools: pgloader. |
| PostgreSQL → MySQL         | Higher. PG features (JSONB, CTE, partial indexes, PostGIS) often have no MySQL equivalent.      |
| SQL Server → PostgreSQL    | Medium-high. T-SQL → PL/pgSQL rewrites. Temporal tables, Service Broker need redesign. Tools: AWS DMS / Babelfish (T-SQL on PG). |
| Mongo → Postgres           | High if "Mongo as Postgres-with-JSON" — actually easy. Real Mongo (denormalized, $lookup) requires re-modeling. |
| Postgres → Mongo           | Almost always wrong. Re-model around aggregate boundaries first; only migrate when shape genuinely fits. |
| Any → managed (Atlas, RDS) | Low. Use logical replication / CDC for cutover.                                                |

---

## Anti-Patterns Across All Four

* **"We use X for everything."** A microservice architecture is the freedom to pick per-service; use it. Reach for the workload-fitted DB.
* **"It's just SQL — they're interchangeable."** They aren't. Concurrency models, isolation defaults, index limitations, and DDL semantics differ enough to bite at scale.
* **Ignoring the connection pool layer.** PG without PgBouncer, MySQL without ProxySQL, SQL Server's MARS without thought — every DB has connection ceilings. Plan for them.
* **Defaulting to maximum durability everywhere.** `synchronous_commit = on` / `w: majority` / `RCSI off + sync replica` cost throughput. Tune per-operation; not every write needs the strongest guarantee.
* **Treating "Cosmos DB Mongo API" as MongoDB.** It's a different engine with a Mongo wire protocol. Different consistency, different cost model, different feature set.
* **Picking a DB by Twitter / blog hype rather than your team's operational reality.** The DB you can operate is more valuable than the DB that's slightly better-fit on paper.

---

## See Also

* Per-DB deep dives: [PostgreSQL](./postgresql.md) · [MySQL](./mysql.md) · [SQL Server](./sql-server.md) · [MongoDB](./mongodb.md).
* Related: `../dot-net/docs/enterprise-pattern/data-access.md` — patterns *on top of* the DB choice (Repository, UoW, Lazy/Eager, etc.).
* Related: `../dot-net/docs/concurrency/race-conditions.md` — race conditions and how isolation levels prevent (or fail to prevent) them.

## References

* Martin Kleppmann — *Designing Data-Intensive Applications* (2017). The canonical book on cross-DB design trade-offs.
* AWS / GCP / Azure documentation for managed DB tiers — pricing and feature parity vary wildly.
* [Use The Index, Luke!](https://use-the-index-luke.com/) — Markus Winand. Language-agnostic index/query principles that apply across all the relational DBs above.
