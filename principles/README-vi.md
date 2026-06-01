# Nguyên Lý Software Engineering — Hướng dẫn đầy đủ

> Tham chiếu language-agnostic thực hành cho các nguyên lý mọi developer nên biết — từ **DRY** và **KISS** tới **SOLID**, **GRASP**, và xa hơn. Ví dụ trong C# / Java / TypeScript nhưng các idea áp dụng khắp nơi.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Tham chiếu language-agnostic cho nguyên lý software-engineering: **DRY**, **KISS**, **YAGNI**, **SoC** (core); **SOLID** (5 nguyên lý OO); **GRASP** (9 nguyên lý responsibility-assignment); cộng LoD, POLA, Fail Fast, Composition over Inheritance, Tell-Don't-Ask, và hơn.
- **Tại sao** — Nguyên lý là *guideline cho thiết kế tốt* — cái **tại sao** đằng sau mọi pattern và architecture. Internalize chúng và bạn sẽ reinvent nửa pattern theo nhu cầu; ignore chúng và kể cả pattern tốt nhất cũng không save hệ thống.
- **Khi nào** — Mọi design decision; code review ("nguyên lý nào cái này vi phạm?"); giải thích trade-off cho teammate junior; quyết định liệu áp dụng pattern hay không.
- **Ở đâu** — Trên pattern và architecture trong hierarchy. Foundation cho `oop/`, `dot-net/docs/design-pattern/`, `dot-net/docs/architectural-style/`, và `dot-net/docs/ddd/`.

---

## Mục lục

1. [Nguyên lý là gì?](#1-nguyên-lý-là-gì)
2. [Nguyên lý vs. Pattern vs. Architecture](#2-nguyên-lý-vs-pattern-vs-architecture)
3. [Catalog đầy đủ](#3-catalog-đầy-đủ)
4. [Nguyên lý cốt lõi (DRY, KISS, YAGNI, SoC)](./core-vi.md)
5. [SOLID — 5 nguyên lý OO](./solid-vi.md)
6. [GRASP — 9 nguyên lý gán trách nhiệm](./grasp-vi.md)
7. [Nguyên lý bổ sung (LoD, CoI, POLA, …)](./additional-vi.md)
8. [Cách áp dụng nguyên lý trong thực tế](#4-cách-áp-dụng-nguyên-lý-trong-thực-tế)
9. [Anti-Pattern: chuyện gì xảy ra khi bỏ qua](#5-anti-pattern)
10. [Đọc thêm](#6-đọc-thêm)

---

## 1. Nguyên lý là gì?

**Nguyên lý** là *quy tắc chung* hướng dẫn quyết định thiết kế. Khác với pattern (giải pháp cụ thể) hay architecture (cấu trúc cấp cao), nguyên lý là **giá trị hoặc heuristic** bạn áp dụng khi quyết định *cách* viết code.

Tính chất chính:

* **Phổ quát** — không phụ thuộc ngôn ngữ hay framework.
* **Dựa trên trade-off** — mọi nguyên lý có chi phí; áp dụng cứng nhắc gây hại.
* **Composable** — nguyên lý củng cố lẫn nhau (vd. SRP làm OCP dễ hơn).
* **Heuristic, không phải luật** — nguyên lý là mặc định, không phải contract.

> *"Nguyên lý là **tại sao**. Pattern là **cách**. Architecture là **ở đâu**."*

---

## 2. Nguyên lý vs. Pattern vs. Architecture

| Tầng             | Là gì                                                  | Ví dụ                                  |
| ---------------- | ------------------------------------------------------ | -------------------------------------- |
| **Nguyên lý**    | Quy tắc cho *thiết kế tốt*                             | DRY, SRP, Tell-Don't-Ask               |
| **Pattern**      | Template giải pháp có tên, tái sử dụng                  | Strategy, Factory, Repository          |
| **Architecture** | Style *cấu trúc* cấp cao của toàn hệ thống              | Clean Architecture, N-Layer, Hexagonal |

Pattern implement nguyên lý. Architecture tổ chức pattern. Nguyên lý phán xét tất cả.

---

## 3. Catalog đầy đủ

### 3.1 Nguyên lý cốt lõi (4)

| Acronym   | Tên                             | Ý định một dòng                                                       |
| --------- | ------------------------------- | --------------------------------------------------------------------- |
| **DRY**   | Don't Repeat Yourself           | Mỗi mẩu kiến thức có representation duy nhất, có thẩm quyền.          |
| **KISS**  | Keep It Simple, Stupid          | Ưu tiên giải pháp đơn giản nhất hoạt động.                            |
| **YAGNI** | You Aren't Gonna Need It        | Đừng build cho nhu cầu giả định tương lai.                            |
| **SoC**   | Separation of Concerns          | Tách chương trình thành phần mà mỗi phần xử lý một concern.            |

→ Xem [`core-vi.md`](./core-vi.md)

### 3.2 SOLID (5)

| Chữ     | Tên                                 | Ý định một dòng                                                              |
| ------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| **S**   | Single Responsibility Principle     | Class chỉ nên có **một lý do để thay đổi**.                                  |
| **O**   | Open/Closed Principle               | Mở cho extension, **đóng cho modification**.                                 |
| **L**   | Liskov Substitution Principle       | Subtype phải **có thể substitute** cho base type.                            |
| **I**   | Interface Segregation Principle     | Nhiều interface client-specific thắng một interface to.                       |
| **D**   | Dependency Inversion Principle      | Phụ thuộc **abstraction**, không concretion.                                  |

→ Xem [`solid-vi.md`](./solid-vi.md)

### 3.3 GRASP (9)

General Responsibility Assignment Software Patterns — bởi **Craig Larman**.

| # | Nguyên lý                  | Ý định một dòng                                                                |
| - | -------------------------- | ------------------------------------------------------------------------------ |
| 1 | **Information Expert**     | Gán trách nhiệm cho class có data cần để thực hiện.                            |
| 2 | **Creator**                | Class aggregate/dùng B nên tạo B.                                              |
| 3 | **Controller**             | Route system event tới class "controller" non-UI.                              |
| 4 | **Low Coupling**           | Tối thiểu dependency giữa các class.                                           |
| 5 | **High Cohesion**          | Giữ trách nhiệm liên quan cùng nhau; không liên quan tách ra.                  |
| 6 | **Polymorphism**           | Dùng operation polymorphic để xử lý variation theo type.                       |
| 7 | **Pure Fabrication**       | Phát minh class không có domain analog để giữ cohesion/coupling.               |
| 8 | **Indirection**            | Thêm component giữa để decouple hai bên.                                        |
| 9 | **Protected Variations**   | Bọc element bất ổn với interface ổn định.                                       |

→ Xem [`grasp-vi.md`](./grasp-vi.md)

### 3.4 Nguyên lý bổ sung

| Nguyên lý                              | Ý định một dòng                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------- |
| **LoD** — Law of Demeter               | Chỉ nói chuyện với bạn ngay sát.                                              |
| **Composition over Inheritance**       | Ưu tiên quan hệ "has-a" hơn "is-a".                                          |
| **Tell, Don't Ask**                    | Bảo object làm gì; đừng query state để quyết định thay nó.                    |
| **POLA** — Principle of Least Astonishment | Behavior nên match kỳ vọng trực giác của user.                            |
| **Fail Fast**                          | Phát hiện error ở thời điểm sớm nhất.                                         |
| **CoC** — Convention over Configuration | Default hợp lý thắng vô số nút configuration.                                |
| **Boy Scout Rule**                     | Để code sạch hơn so với khi bạn tìm thấy.                                     |
| **Encapsulation**                      | Ẩn internal; expose behavior.                                                |
| **Principle of Least Privilege**       | Cấp quyền truy cập tối thiểu cần thiết.                                       |
| **SoT** — Single Source of Truth       | Mỗi fact sống ở chính xác một nơi.                                            |

→ Xem [`additional-vi.md`](./additional-vi.md)

---

## 4. Cách áp dụng nguyên lý trong thực tế

Nguyên lý thường xung đột. Một số heuristic thực hành:

1. **Make it work, then make it right, then make it fast.** Đừng tối ưu SOLID ngày đầu.
2. **Trả chi phí abstraction chỉ khi use case thứ hai xuất hiện.** (DRY + YAGNI căng thẳng.)
3. **Coupling và cohesion vượt mọi metric khác.** Phần lớn "code tệ" thực ra là một trong hai cái này.
4. **Ưu tiên xoá hơn refactor.** Ít code thì obey mọi nguyên lý tự động.
5. **Rule of three.** Code trùng OK 2 lần; refactor ở lần thứ ba.

---

## 5. Anti-Pattern

Cái gì sai khi bạn bỏ qua nguyên lý:

| Smell                          | Vi phạm                                  |
| ------------------------------ | ----------------------------------------- |
| **God Class**                  | SRP, High Cohesion                        |
| **Copy-Paste Programming**     | DRY                                       |
| **Shotgun Surgery**            | SRP, Low Coupling                         |
| **Feature Envy**               | Information Expert, Tell-Don't-Ask        |
| **Speculative Generality**     | YAGNI, KISS                               |
| **Long Parameter List**        | SRP, ISP                                  |
| **Inappropriate Intimacy**     | Law of Demeter, Encapsulation             |
| **Refused Bequest**            | LSP                                       |
| **Switch on Type**             | OCP, Polymorphism                         |
| **Primitive Obsession**        | Encapsulation                             |

---

## 6. Đọc thêm

* Robert C. Martin — *Clean Code*, *Clean Architecture*, *Agile Software Development: Principles, Patterns, and Practices*
* Craig Larman — *Applying UML and Patterns* (GRASP)
* Andy Hunt & Dave Thomas — *The Pragmatic Programmer* (DRY, orthogonality)
* Martin Fowler — *Refactoring* (code smell), *Patterns of Enterprise Application Architecture*
* Steve McConnell — *Code Complete*
* Eric Evans — *Domain-Driven Design*

---

> **Tóm lại:** nguyên lý không phải rule để mù quáng tuân theo — chúng là lăng kính để đánh giá trade-off. Internalize chúng để bạn có thể phá *có ý thức* khi context yêu cầu.
