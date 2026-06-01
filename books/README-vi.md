# Tóm tắt Sách

> Note cô đọng từ các sách software engineering có ảnh hưởng — các idea thiết yếu, ví dụ canonical, và take-away bạn có thể quote trong code review.

Mỗi tóm tắt có quan điểm: bắt **cái sách dạy**, **cách tác giả dạy** (running example), và **nguyên lý hoặc rule** chắt ra từ các chương. Không thay thế việc đọc sách — là *bản đồ navigation* và *aid recall*.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Map navigation cho tóm tắt sách software-engineering ảnh hưởng — idea, ví dụ, và rule distill thành note recall-friendly.
- **Tại sao** — Cho phép tìm sách đúng nhanh, recall argument chính trước interview hoặc design review, kéo ví dụ canonical vào discussion mà không re-read 500 trang.
- **Khi nào** — Khởi đầu một topic và cần orientation; revisit trước interview; dạy teammate; so sánh tác giả khác nhau về cùng idea.
- **Ở đâu** — Entry point cấp top vào `books/`; cross-link vào `dot-net/docs/`, `principles/`, và `oop/` cho content repo sâu hơn.

---

## Index

| Sách                                                                                   | Tác giả                          | Năm  | Tập trung                                                            | File                                                  |
| -------------------------------------------------------------------------------------- | -------------------------------- | ---- | -------------------------------------------------------------------- | ----------------------------------------------------- |
| **Head First Design Patterns** (2nd ed.)                                               | Eric Freeman & Elisabeth Robson  | 2020 | Các design pattern GoF dạy qua running example trong Java.            | [`head-first-design-patterns-vi.md`](./head-first-design-patterns-vi.md) |
| **Clean Architecture**                                                                 | Robert C. Martin (Uncle Bob)     | 2017 | Dependency Rule, diagram 4-ring, và cái gì tính là "detail".         | [`clean-architecture-robert-martin-vi.md`](./clean-architecture-robert-martin-vi.md) |
| **Building Microservices** (2nd ed.)                                                   | Sam Newman                       | 2021 | Định nghĩa, modeling, saga, deployment, testing, observability, và khi *không* dùng microservices. | [`building-microservices-sam-newman-vi.md`](./building-microservices-sam-newman-vi.md) |
| **Unit Testing: Principles, Practices, and Patterns**                                  | Vladimir Khorikov                | 2020 | Bốn trụ cột của good unit test, mock vs. stub, khi nào integration.   | [`unit-testing-vladimir-khorikov-vi.md`](./unit-testing-vladimir-khorikov-vi.md) |
| **Domain-Driven Design** ("Blue Book")                                                 | Eric Evans                       | 2003 | Ubiquitous Language, tactical pattern, Bounded Context, Context Map.  | [`domain-driven-design-eric-evans-vi.md`](./domain-driven-design-eric-evans-vi.md) |

### Thứ tự đọc đề xuất

Nếu bạn khởi đầu from scratch và muốn internalize canon, path coherent là:

1. **OOP fundamentals** → [`../oop/`](../oop/).
2. **Principles** (DRY, SOLID, GRASP) → [`../principles/`](../principles/).
3. **Head First Design Patterns** — pattern cấp class, made memorable.
4. **Clean Architecture** — rule cấu trúc tổ chức pattern đó.
5. **Domain-Driven Design** — cái gì đặt bên trong cấu trúc (cái *gì*).
6. **Unit Testing** — cách test cái bạn build well.
7. **Building Microservices** — chỉ khi có driver thật để split.

*(Có thể thêm tóm tắt sách khác khi folder grow.)*

---

## Cách dùng các tóm tắt này

* **Lần đầu** với một chủ đề? Đọc sách gốc. Tóm tắt sẽ make sense hơn sau đó.
* **Đã đọc** sách rồi? Tóm tắt là **aid recall** của bạn — đọc lại trước interview, design review, hoặc khi bắt đầu feature mới.
* **Dạy người khác?** Tóm tắt lift ra running example và code chính để bạn lean vào trong slide hoặc whiteboard.
* **Tra cứu một pattern?** Mỗi entry pattern trong tóm tắt theo cùng skeleton: intent → running example → code → key point → cạm bẫy.

---

## Cross-Reference với các Folder khác

* **GoF design pattern** với sample .NET / C#: [`../dot-net/docs/design-pattern/`](../dot-net/docs/design-pattern/)
* **Architectural style** (Clean, Hexagonal, Layered, v.v.): [`../dot-net/docs/architectural-style/`](../dot-net/docs/architectural-style/)
* **Architectural pattern** (Saga, CQRS, Event Sourcing, Outbox): [`../dot-net/docs/architectural-pattern/`](../dot-net/docs/architectural-pattern/)
* **Enterprise pattern** (Repository, UoW, IoC/DI): [`../dot-net/docs/enterprise-pattern/`](../dot-net/docs/enterprise-pattern/)
* **OO fundamentals** (bốn trụ cột): [`../oop/`](../oop/)
* **Nguyên lý software-engineering** (SOLID, GRASP, DRY): [`../principles/`](../principles/)

Các tóm tắt sách cite những cái này bất cứ khi nào lesson của sách reinforce hoặc contradict idea documented ở chỗ khác trong repo.
