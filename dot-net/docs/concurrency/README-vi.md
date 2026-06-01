# Concurrency trong .NET

> Note về concern programming concurrent và distributed cắt qua architectural style, pattern, và DI lifetime.

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Documentation cho concern concurrency trong .NET hiện đại: race condition, chiến lược locking, coordination async, consistency distributed, và lựa chọn design loại bỏ bug concurrency hoàn toàn.
- **Tại sao** — Bug concurrency là nguồn lớn nhất của failure production "work trên máy tôi". Chúng không fit gọn vào "architectural pattern" hoặc "design pattern" — chúng là concern *cross-cutting* với vocabulary riêng.
- **Khi nào** — Thiết kế path read-modify-write; review PR introduce shared state; debug failure intermittent của test; chọn giữa optimistic và pessimistic concurrency.
- **Ở đâu** — Cắt qua mọi layer. Cross-reference DI lifetime (`../enterprise-pattern/lifetimes-vi.md`), workflow consistency (`../architectural-pattern/saga-vi.md`), messaging reliable (`../architectural-pattern/outbox-vi.md`), và event-sourced aggregate (`../architectural-pattern/event-sourcing-vi.md`).

---

## Index

| File                                              | Topic                                                                                         |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [`race-conditions-vi.md`](./race-conditions-vi.md) | Giải pháp cho race condition qua bốn scope: in-process, single-DB, distributed, design-out.   |

🇬🇧 English: [`README.md`](./README.md)
