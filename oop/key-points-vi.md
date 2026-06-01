# OOP Key Points — Cheat Sheet cho Interview & Design

> Tóm tắt cô đọng. Nếu chỉ nhớ một trang từ guide này, hãy chọn trang này. Mỗi dòng là một claim bạn phải **bảo vệ được bằng ví dụ** và **phản biện được bằng trade-off**.

> 🇻🇳 Phiên bản tiếng Việt. English: [`key-points.md`](./key-points.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Cheat sheet high-density của OO design wisdom: bốn trụ cột một dòng mỗi, essential class-design, inheritance vs. composition, abstract class vs. interface, value vs. entity vs. aggregate, SOLID 30-giây, OO smell, top interview question, và cái junior vs. senior đúng/sai.
- **Tại sao** — Mỗi dòng là *claim bạn nên defend được bằng ví dụ và counter được bằng trade-off*. Phần lớn "checklist" vô dụng; cái này được nghĩ cho review trước interview và argument code-review.
- **Khi nào** — Ngày trước interview; chuẩn bị lead design review; mentor developer sắp vi phạm một trong các rule này.
- **Ở đâu** — Distillation của `pillars-vi.md`, `concepts-vi.md`, `real-project-vi.md`, và `principles/`. Recall aid sau khi đã internalize doc dài hơn.

---

## Mục lục

1. [Bốn trụ cột — mỗi cái một dòng](#1-bốn-trụ-cột--mỗi-cái-một-dòng)
2. [Class Design — Cốt lõi](#2-class-design--cốt-lõi)
3. [Inheritance vs Composition](#3-inheritance-vs-composition)
4. [Abstract Class vs Interface](#4-abstract-class-vs-interface)
5. [Value vs Reference / Entity vs Value Object](#5-value-vs-reference--entity-vs-value-object)
6. [SOLID trong 30 giây](#6-solid-trong-30-giây)
7. [Code Smell OOP phổ biến](#7-code-smell-oop-phổ-biến)
8. [Mental Checklist trước khi viết class](#8-mental-checklist-trước-khi-viết-class)
9. [Top câu hỏi interview với câu trả lời ngắn gọn](#9-top-câu-hỏi-interview-với-câu-trả-lời-ngắn-gọn)
10. [Điều junior thường sai](#10-điều-junior-thường-sai)
11. [Điều senior quan tâm](#11-điều-senior-quan-tâm)

---

## 1. Bốn trụ cột — mỗi cái một dòng

| Trụ cột            | One-liner                                                  | Dấu hiệu trong code                    |
| ------------------ | ---------------------------------------------------------- | -------------------------------------- |
| **Encapsulation**  | Ẩn state đằng sau behavior.                                | Field `private`, `init` setter         |
| **Abstraction**    | Chọn cái gì expose; ẩn cái không liên quan.                | Interface, abstract class              |
| **Inheritance**    | "is-a" specialization của type có sẵn.                     | `class Dog : Animal`                   |
| **Polymorphism**   | Cùng call, behavior runtime khác.                          | `virtual / override`, interface        |

---

## 2. Class Design — Cốt lõi

* **Mặc định `sealed` + `private` + `readonly`.** Chỉ mở khi có lý do.
* **Constructor enforce invariant.** Object xây dở dang không nên tồn tại.
* **Public surface = behavior**, không phải data. Property OK; raw field thì không.
* **Method mô tả intent**, không phải implementation. `Order.MarkPaid()`, không phải `Order.SetStatus(2)`.
* **Một lý do để thay đổi.** Nếu tên class cần "And" để mô tả, hãy tách.
* **Nhỏ > Thông minh.** Ba class ngắn, đặt tên tốt hơn một class lắt léo.
* **Tell, Don't Ask.** Đừng with vào object để tự quyết định thay nó.
* **Fail ở construction.** Từ chối state invalid ngay lập tức, không phải sau.

---

## 3. Inheritance vs Composition

| Inheritance ("is-a")                               | Composition ("has-a")                               |
| -------------------------------------------------- | --------------------------------------------------- |
| Compile-time, vĩnh viễn                            | Runtime, có thể swap                                |
| Một parent (trong C# / Java)                       | Nhiều collaborator                                  |
| Coupling mạnh với base class                       | Coupling lỏng qua interface                         |
| Tốt nhất cho quan hệ is-a **ổn định, thật**        | Tốt nhất khi behavior **vary** hoặc **có thể tăng**|
| Rủi ro fragile-base-class                          | Nhiều file, nhiều wiring                            |

> **Mặc định composition.** Chỉ dùng inheritance khi *"X is-a Y"* thực sự đúng *và* ổn định.

---

## 4. Abstract Class vs Interface

| Chọn **abstract class** khi…              | Chọn **interface** khi…                          |
| ----------------------------------------- | ------------------------------------------------ |
| Có shared **state hoặc default**          | Cần pure **contract**                            |
| Muốn kiểm soát construction               | Nhiều capability có thể implement cùng lúc       |
| Hierarchy sẽ **đóng** và ổn định          | Contract trải qua implementer **không liên quan**|
| Có thể đáng tin nói "is-a"                | Có thể đáng tin nói "behaves-as"                 |

> Code hiện đại ưu tiên **interface + composition** cho flexibility, **abstract class** cho taxonomy thật với internal share.

---

## 5. Value vs Reference / Entity vs Value Object

### Value Object

* Không identity. Hai instance value bằng nhau là **tương đương**.
* **Immutable.** Mutate một sẽ ảnh hưởng mọi "logical copy".
* Ví dụ: `Money`, `Address`, `DateRange`, `PhoneNumber`, `Color`.

### Entity

* Có **identity** (`Id`) tồn tại qua thay đổi.
* Hai entity có thể có cùng data và vẫn khác nhau.
* Ví dụ: `Customer`, `Order`, `Account`.

### Aggregate

* Cluster entity + value object với một **root** duy nhất.
* Code ngoài chỉ nói chuyện với **root**, không bao giờ vào trong.
* Boundary của consistency: mọi invariant enforce trong một transaction.

---

## 6. SOLID trong 30 giây

| Chữ    | Nguyên lý                       | "Nếu quên…"                                                      |
| ------ | ------------------------------- | ---------------------------------------------------------------- |
| **S**  | Single Responsibility           | Sẽ có God class.                                                 |
| **O**  | Open/Closed                     | Mỗi feature mới sửa code cũ.                                     |
| **L**  | Liskov Substitution             | Subclass làm caller bất ngờ.                                     |
| **I**  | Interface Segregation           | Một thay đổi lan qua implementer không liên quan.                |
| **D**  | Dependency Inversion            | Concrete coupling làm test và swap không thể.                    |

Xem file đầy đủ tại [`../principles/solid-vi.md`](../principles/solid-vi.md).

---

## 7. Code Smell OOP phổ biến

| Smell                       | Báo hiệu gì                              |
| --------------------------- | ---------------------------------------- |
| **God Class**               | Vi phạm SRP; cohesion thấp.              |
| **Anemic Domain Model**     | Data và behavior bị tách rời.            |
| **Feature Envy**            | Method thuộc về class khác.              |
| **Train Wreck** (`a.b().c().d()`) | Law of Demeter bị phá.              |
| **Switch on Type**          | Thiếu polymorphism.                      |
| **Long Parameter List**     | Có thể thiếu một value object.           |
| **Primitive Obsession**     | String/int chỗ đáng có type.             |
| **Speculative Generality**  | Vi phạm YAGNI.                           |
| **Refused Bequest**         | Subclass không muốn cái nó thừa kế — is-a sai. |
| **Singletonitis**           | Global state ẩn; untestable.             |

---

## 8. Mental Checklist trước khi viết class

Trước khi thêm class mới, tự hỏi:

1. **Single responsibility của nó là gì?** (Đặt tên dưới dạng noun + role.)
2. **State nào nó sở hữu?** (Và cái nào nên derive?)
3. **Invariant nào nó đảm bảo?** (Enforce ở đâu?)
4. **Ai tạo nó?** (Factory / constructor / aggregate root.)
5. **Ai *không thể* thấy nó?** (Khoá bằng `internal` / `private`.)
6. **Thay đổi nào nó sẽ hấp thụ?** (Method ổn định vs hot spot.)
7. **Nó phụ thuộc cái gì?** (Inject interface; không bao giờ `new` service.)
8. **Test nó ra sao?** (Khó test — thiết kế chắc sai.)

Nếu không trả lời được 1-3 trong một câu mỗi, bạn chưa sẵn sàng viết class.

---

## 9. Top câu hỏi interview với câu trả lời ngắn gọn

**H: Khác nhau giữa abstraction và encapsulation?**
Encapsulation ẩn *cách*; abstraction quyết định *cái gì expose*. Encapsulation là cơ chế (access modifier); abstraction là lựa chọn thiết kế.

**H: Khi nào chọn composition over inheritance?**
Khi quan hệ không phải is-a ổn định, khi bạn sẽ override để "không làm gì", khi behavior cần vary runtime, hoặc khi muốn testable seam. Trong thực tế — gần như luôn luôn.

**H: Giải thích polymorphism không dùng từ "polymorphism".**
Hai object khác nhau phản hồi cùng một message theo cách riêng, và caller không cần biết nó có cái nào.

**H: Tại sao `instanceof` / `is X` thường là code smell?**
Nó encode kiến thức về type vào caller, chặn OCP. Thường là method virtual hoặc strategy bị thiếu.

**H: Value object là gì?**
Type immutable mà identity *chính là* value của nó — `Money(10, "USD")` tương đương bất kỳ `Money(10, "USD")` nào khác.

**H: Tại sao ưu tiên immutability?**
An toàn thread, predictability, semantics equality, dễ lý luận về state hơn.

**H: Khi nào OOP là lựa chọn sai?**
Pure data transformation, code nặng toán, script rất ngắn, inner loop nhạy performance — functional, procedural, hoặc data-oriented phù hợp hơn.

**H: OOP có phải về reuse?**
Không. OOP chủ yếu về **quản lý dependency và hấp thụ thay đổi**. Reuse là side effect, và thường bị đánh giá quá cao.

**H: Quan hệ giữa SOLID và OOP?**
SOLID là bộ nguyên lý biến code OO ad-hoc thành code OO *maintainable*. OOP cho phép chúng; SOLID kỷ luật chúng.

**H: Khác nhau giữa `interface` và `abstract class`?**
Interface = contract, không state. Abstract class = partial implementation với shared state. Ưu tiên interface; chỉ dùng abstract khi thật sự cần internal share.

---

## 10. Điều junior thường sai

* Treat inheritance như cơ chế reuse mặc định.
* Đặt mọi thứ `public` "phòng khi cần".
* Viết class anemic với `get; set;` khắp nơi, rồi đặt business rule trong service.
* Dùng `static` cho "tiện lợi", rồi chìm trong coupling ẩn.
* Model bằng primitive (`string customerId`, `decimal price`) thay vì value object.
* Lạm dụng `if (x is Y)` thay vì polymorphism.
* Lạm dụng polymorphism cho logic 2 trường hợp mà `if` thể hiện rõ hơn.
* Mock mọi thứ trong test, kể cả value object.
* Build extensibility cho variant không bao giờ đến.

---

## 11. Điều senior quan tâm

* **Coupling và cohesion** trên gần như mọi thứ khác.
* **Tính ổn định của abstraction** — cái dễ biến động sống đằng sau một port.
* **Tên survive refactor** — ngôn ngữ domain, không phải jargon technical.
* **Behavior trên data** — method, không phải property, định nghĩa API.
* **Boundary layer rõ ràng** — dependency rule là không thương lượng.
* **Testability như tín hiệu feedback thiết kế** — khó test, thiết kế sai.
* **Tính dễ xoá** — code dễ xoá là code dễ thay đổi.
* **Biết khi *không* áp dụng nguyên lý** — context > dogma.

---

## TL;DR

> **OOP không phải về object — nó về boundary. Encapsulation tạo chúng, abstraction đặt tên chúng, inheritance và composition sắp xếp chúng, polymorphism cho phép behavior vượt qua chúng an toàn. Bốn trụ cột tồn tại để quản lý *thay đổi*. Mọi nguyên lý trong folder doc này là heuristic để giữ những boundary đó vẽ rẻ và vẽ lại rẻ.**

Đọc theo thứ tự:
1. [`README-vi.md`](./README-vi.md) — orientation
2. [`pillars-vi.md`](./pillars-vi.md) — bốn trụ cột
3. [`concepts-vi.md`](./concepts-vi.md) — construct ngôn ngữ
4. [`real-project-vi.md`](./real-project-vi.md) — kết hợp tất cả
5. **File này** — takeaway

Rồi luyện. Lý thuyết rẻ; thiết kế là cơ bắp memory.
