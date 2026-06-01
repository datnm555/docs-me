# Concurrency in .NET

> Notes on concurrent and distributed programming concerns that cut across architectural styles, patterns, and DI lifetimes.

---

## Quick Reference (What · Why · When · Where)

- **What** — Documentation for concurrency concerns in modern .NET: race conditions, locking strategies, async coordination, distributed consistency, and the design choices that eliminate concurrency bugs entirely.
- **Why** — Concurrency bugs are the single largest source of "works on my machine" production failures. They don't fit neatly into "architectural pattern" or "design pattern" — they're a *cross-cutting* concern with its own vocabulary.
- **When** — Designing read-modify-write paths; reviewing PRs that introduce shared state; debugging intermittent test failures; choosing between optimistic and pessimistic concurrency.
- **Where** — Cuts across every layer. Cross-references DI lifetimes (`../enterprise-pattern/lifetimes.md`), workflow consistency (`../architectural-pattern/saga.md`), reliable messaging (`../architectural-pattern/outbox.md`), and event-sourced aggregates (`../architectural-pattern/event-sourcing.md`).

---

## Index

| File                                              | Topic                                                                                          |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| [`race-conditions.md`](./race-conditions.md)      | Solutions for race conditions across four scopes: in-process, single-DB, distributed, design-out. |

🇻🇳 Vietnamese: [`README-vi.md`](./README-vi.md)
