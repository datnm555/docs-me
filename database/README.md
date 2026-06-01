# Databases — Comparison & Per-DB Deep Dives

> Practical, opinionated notes on the four mainstream database engines you'll meet in production: **PostgreSQL**, **MySQL**, **SQL Server**, and **MongoDB**. Each per-DB doc focuses on **strengths/weaknesses + concurrency and consistency model** — the things that actually decide whether a DB is the right fit for a workload. The comparison doc is the decision guide on top.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — Per-DB deep dives covering the four mainstream engines, plus a side-by-side comparison and decision guide. Each per-DB doc covers: strengths, weaknesses, concurrency/consistency model, schema/data model, operational characteristics, .NET integration, when to choose, common pitfalls.
- **Why** — Database choice is high-cost and hard to reverse. Most "DB X vs Y" debates miss the actual decision dimensions (concurrency model, scaling axis, schema flexibility). These docs aim to give you the dimensions, not opinions.
- **When** — Designing a new service. Considering migration. Building a polyglot persistence architecture. Before any "which DB?" meeting.
- **Where** — Top-level `database/` folder, language-agnostic on the principles. Code samples lean **C#** since this repo is .NET-focused, but the concurrency/consistency notes apply regardless of language. Cross-links into `dot-net/docs/enterprise-pattern/data-access.md` (patterns on top of the DB) and `dot-net/docs/concurrency/race-conditions.md` (how isolation levels interact with concurrent code).
- **How** — Read the per-DB doc for whichever DBs you're considering. Then read the comparison for the final pick. Don't reason from "we know X" alone — workload fit matters more than familiarity at decision time, though familiarity is a real cost saver.

---

## Index

| Doc                                                | Focus                                                                                  |
| -------------------------------------------------- | -------------------------------------------------------------------------------------- |
| [`postgresql.md`](./postgresql.md)                  | Open-source RDBMS default. MVCC, rich type system, JSONB, PostGIS, extensions.         |
| [`mysql.md`](./mysql.md)                            | Most-deployed open-source RDBMS. InnoDB MVCC + next-key locks. Vitess for sharding.    |
| [`sql-server.md`](./sql-server.md)                  | Microsoft RDBMS. Lock-based + RCSI for MVCC. Temporal tables, columnstore, T-SQL.       |
| [`mongodb.md`](./mongodb.md)                        | Document DB. BSON, sharding native, replica sets, write/read concerns, change streams.  |
| [`comparison.md`](./comparison.md)                  | Side-by-side matrix, decision tree, use-case → DB mapping, migration considerations.    |

### Read order

1. **`comparison.md`** for the big picture.
2. The per-DB doc for whatever you're seriously considering.
3. The other per-DB docs for the rejected candidates — knowing what you didn't pick (and why) is the foundation of an honest decision record.

---

## How These Docs Are Structured

Every per-DB doc follows the same skeleton:

1. **Quick Reference** (5 bullets: What / Why / When / Where / How).
2. **Strengths** — what it's genuinely great at.
3. **Weaknesses** — what it's bad at, and what you'll fight.
4. **Concurrency & Consistency Model** — MVCC vs lock-based, isolation levels, replication. The most important section.
5. **Schema & Data Model** — what good schema looks like for this engine.
6. **Operational Characteristics** — scaling axis, backup, monitoring, tooling.
7. **.NET Integration** — drivers, EF Core providers, performance notes.
8. **When to Choose / When Not** — explicit guidance.
9. **Pitfalls** — the surprises that bite production.
10. **References** — books, official docs, blogs worth reading.

This makes per-DB comparison easy: read section 4 across all four DBs to compare concurrency models, section 6 to compare ops, and so on.

---

## Cross-References Into the Rest of the Repo

* **[`../dot-net/docs/enterprise-pattern/data-access.md`](../dot-net/docs/enterprise-pattern/data-access.md)** — patterns *on top of* the DB choice: Lazy / Eager loading, Identity Map, Active Record vs. Data Mapper, Specification.
* **[`../dot-net/docs/enterprise-pattern/repository.md`](../dot-net/docs/enterprise-pattern/repository.md)** — Repository as an abstraction over the DB; when generic Repository is an anti-pattern.
* **[`../dot-net/docs/enterprise-pattern/unit-of-work.md`](../dot-net/docs/enterprise-pattern/unit-of-work.md)** — UoW backed by `DbContext` / `IClientSessionHandle`.
* **[`../dot-net/docs/concurrency/race-conditions.md`](../dot-net/docs/concurrency/race-conditions.md)** — race conditions and how DB isolation levels prevent (or don't prevent) them.
* **[`../dot-net/docs/architectural-pattern/outbox.md`](../dot-net/docs/architectural-pattern/outbox.md)** — Outbox depends on transactional same-DB writes; CDC-based outbox depends on the DB's binlog / WAL / change-stream feature.
* **[`../dot-net/docs/architectural-pattern/event-sourcing.md`](../dot-net/docs/architectural-pattern/event-sourcing.md)** — Event Sourcing on top of any DB; specialized engines (EventStoreDB) sit in a separate category.
* **[`../books/building-microservices-sam-newman.md`](../books/building-microservices-sam-newman.md)** — "Database per service" and polyglot persistence.
* **[`../books/domain-driven-design-eric-evans.md`](../books/domain-driven-design-eric-evans.md)** — aggregate design influences how you'd model in document vs. relational stores.

---

## Note on Scope

These docs intentionally do **not** cover:

* **Specialized stores** — Redis (cache/streams), Elasticsearch (search), ClickHouse / DuckDB (OLAP columnar), Cassandra / ScyllaDB (wide-column distributed), Neo4j (graph), Pinecone / Weaviate (vector). They occupy different decision spaces.
* **Wire-compatible re-implementations** — Cosmos DB's "Mongo API" / "PostgreSQL API" / "Cassandra API" are not the engines they look like; the comparison doc has a warning about Cosmos Mongo specifically.
* **Embedded engines** — SQLite, DuckDB-as-embedded, LiteDB. Different use case (single-process / desktop / mobile / test).
* **Per-version feature minutiae** — these docs aim for stable principles, not version-specific changelogs.

If demand arises, those can become their own folder later. For now: the four most-likely-to-be-on-your-shortlist engines.
