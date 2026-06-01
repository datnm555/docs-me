# Object-Oriented Programming — Hướng dẫn đầy đủ

> Tham chiếu thực hành end-to-end về OOP: nó là gì, bốn trụ cột, các khái niệm cốt lõi, walkthrough dự án thật, và các điểm phải biết cho interview và thảo luận thiết kế. Ví dụ trong **C#**, nhưng các idea áp dụng cho Java, TypeScript, Kotlin, Python, và bất kỳ ngôn ngữ OO nào.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Tham chiếu thực hành end-to-end cho OOP: bốn trụ cột (Encapsulation, Abstraction, Inheritance, Polymorphism), construct cấp ngôn ngữ, walkthrough e-commerce checkout thật, và cheat sheet interview.
- **Tại sao** — OOP là paradigm default cho enterprise software vì nó quản lý dependency và hấp thụ change well. Các note này biến trụ cột trừu tượng thành vocabulary bạn có thể áp dụng trong code review.
- **Khi nào** — Trước khi học design pattern hoặc DDD; bất cứ khi nào team tranh luận inheritance vs. composition; refresh fundamental trước interview.
- **Ở đâu** — Language-agnostic (ví dụ trong C# nhưng idea áp dụng cho Java/TypeScript/Kotlin/Python). Foundation cho `principles/` (SOLID, GRASP) và `dot-net/docs/design-pattern/` (GoF 23).

---

## Mục lục

1. [OOP là gì?](#1-oop-là-gì)
2. [Tại sao OOP?](#2-tại-sao-oop)
3. [Bốn trụ cột (Tóm tắt)](./pillars-vi.md)
4. [Khái niệm cốt lõi (class, object, interface, …)](./concepts-vi.md)
5. [Walkthrough dự án thật — E-Commerce Checkout](./real-project-vi.md)
6. [Cheat Sheet các điểm quan trọng](./key-points-vi.md)
7. [OOP vs các Paradigm khác](#3-oop-vs-các-paradigm-khác)
8. [Khi nào OOP là công cụ sai](#4-khi-nào-oop-là-công-cụ-sai)
9. [Đọc thêm](#5-đọc-thêm)

---

## 1. OOP là gì?

**Object-Oriented Programming** là paradigm trong đó phần mềm được model như một collection các **object** đóng gói **data** (state) và **behavior** (method), tương tác qua **interface rõ ràng**, và được tổ chức quanh **abstraction** của các khái niệm thế giới thực hoặc domain.

Paradigm được xây trên **bốn trụ cột**:

| Trụ cột             | Nghĩa một dòng                                                                |
| ------------------- | ----------------------------------------------------------------------------- |
| **Encapsulation**   | Gom state và behavior; ẩn internal.                                            |
| **Abstraction**     | Expose **cái gì** object làm, ẩn **cách**.                                     |
| **Inheritance**     | Specialize một type có sẵn bằng cách thêm/override behavior.                   |
| **Polymorphism**    | Một interface, nhiều implementation được chọn tại runtime.                     |

→ Xem chi tiết trong [`pillars-vi.md`](./pillars-vi.md).

### 1.1 Ví dụ nhỏ

```csharp
public abstract class PaymentMethod
{
    public abstract Task<PaymentResult> ChargeAsync(decimal amount);
}

public class CreditCard : PaymentMethod
{
    public override Task<PaymentResult> ChargeAsync(decimal amount) { /* Stripe call */ }
}

public class PayPal : PaymentMethod
{
    public override Task<PaymentResult> ChargeAsync(decimal amount) { /* PayPal call */ }
}

// Caller không biết và không quan tâm nó có cái nào:
async Task Checkout(PaymentMethod m, decimal total) => await m.ChargeAsync(total);
```

Cả 4 trụ cột đều xuất hiện:
* **Encapsulation** — mỗi class ẩn API key, HTTP plumbing, retry của nó.
* **Abstraction** — base class chỉ khai báo `ChargeAsync`.
* **Inheritance** — `CreditCard` và `PayPal` extend `PaymentMethod`.
* **Polymorphism** — `Checkout` gọi đúng `ChargeAsync` dựa trên type runtime.

---

## 2. Tại sao OOP?

OOP trở thành dominant vì nó cho lời giải tốt cho 4 vấn đề muôn thuở của phần mềm:

| Vấn đề                            | Câu trả lời của OOP                                          |
| --------------------------------- | ----------------------------------------------------------- |
| **Quản lý độ phức tạp**           | Decompose thành các object nhỏ, có tên, tập trung.          |
| **Model thế giới thực**           | Khái niệm domain map sang class (Customer, Order, Invoice). |
| **Reuse không copy-paste**        | Inheritance, composition, và polymorphism.                  |
| **Sống sót qua thay đổi**         | Encapsulation cô lập thay đổi bên trong boundary.            |

OOP **không** phải cách duy nhất để giải — nhưng là paradigm phổ biến nhất trong enterprise software vì lý do.

---

## 3. OOP vs các Paradigm khác

| Paradigm          | Đơn vị cốt lõi       | Model thế giới như…                | Điểm mạnh                              |
| ----------------- | -------------------- | ---------------------------------- | -------------------------------------- |
| **OOP**           | Object               | Thing có state và behavior         | Hệ thống lớn, tiến hoá, stateful       |
| **Functional**    | Function             | Transformation của value           | Data pipeline, concurrency             |
| **Procedural**    | Procedure            | Chuỗi step                         | Script nhỏ, embedded                   |
| **Logic**         | Predicate            | Fact và rule                       | Giải constraint, AI                    |
| **Data-oriented** | Data thuần + behavior| Array, record, transform           | Game, high-performance, ECS            |

> Ngôn ngữ hiện đại là **multi-paradigm**. C# / Java / TypeScript đều dùng feature functional (LINQ / stream / map-reduce, record, pattern matching). Treat OOP như một công cụ — không phải tôn giáo.

---

## 4. Khi nào OOP là công cụ sai

* **Pure data transform** (CSV → JSON, aggregation). Style functional / pipeline thắng.
* **Script đơn giản** (dưới 200 dòng). Một function là đủ.
* **Inner loop nhạy performance** (game, signal processing). Data-oriented design vượt OO indirection.
* **Domain nặng toán** (statistics, ML). Vector / matrix function, không phải object.

Quy tắc hữu ích: **nếu "object" của bạn không bao giờ thay đổi state và có một method, chúng chỉ là function**. Đừng đóng lốt class.

---

## 5. Đọc thêm

* **Grady Booch** — *Object-Oriented Analysis and Design with Applications*
* **Bertrand Meyer** — *Object-Oriented Software Construction*
* **Eric Evans** — *Domain-Driven Design*
* **Martin Fowler** — *Refactoring*, *Patterns of Enterprise Application Architecture*
* **Robert C. Martin** — *Clean Code*, *Agile Software Development: Principles, Patterns, and Practices*
* **Joshua Bloch** — *Effective Java* (vẫn là vàng kể cả không-Java)

---

> **Bước tiếp:** đọc [`pillars-vi.md`](./pillars-vi.md) cho bốn trụ cột, [`concepts-vi.md`](./concepts-vi.md) cho các construct cấp ngôn ngữ, [`real-project-vi.md`](./real-project-vi.md) cho walkthrough dự án đầy đủ, và [`key-points-vi.md`](./key-points-vi.md) cho cheat sheet.
