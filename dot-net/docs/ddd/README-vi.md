# Domain-Driven Design (DDD)

> DDD là một **methodology** để giải quyết phần mềm phức tạp, không phải design pattern. Đây là cách tư duy về thiết kế phần mềm sao cho code khớp với *domain* — bài toán business đang giải. Bên trong DDD có nhiều *pattern* (Aggregate, Repository, Bounded Context, …), nhưng **DDD tự nó không phải pattern**.

> 🇻🇳 Phiên bản tiếng Việt. English: [`README.md`](./README.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Domain-Driven Design như **methodology** — một cách thiết kế software xoay quanh domain business phức tạp, với hai nửa: **strategic** (Bounded Context, Context Map, Ubiquitous Language) và **tactical** (Aggregate, Entity, Value Object, Repository, Factory, Domain Event).
- **Tại sao** — Phức tạp trong software đến từ bản thân domain. DDD cho bạn vocabulary, pattern, và discipline để model phức tạp đó để code match cách business thực sự nghĩ — loại bỏ phần lớn bug do friction translation.
- **Khi nào** — Domain phức tạp và trung tâm với business (banking, insurance, logistics, healthcare); team hoặc service cần model share; integrating với legacy system mà model không fit với của bạn.
- **Ở đâu** — DDD ở cùng cấp với TDD/BDD/Agile. Áp dụng bên trong một hoặc nhiều Bounded Context; pair với Clean Architecture (các ring) và deploy microservices hoặc modular-monolith.

---

## DDD có phải Design Pattern không?

**Không.**

* **Design pattern** (GoF) = giải pháp tái sử dụng cho bài toán lặp lại ở cấp class.
* **DDD** = một discipline, một vocabulary, và một bộ practice để thiết kế phần mềm xoay quanh một domain phức tạp.

DDD ở cùng cấp với **TDD**, **BDD**, **Agile** — một cách làm việc. Bên trong DDD có:

* **Strategic pattern** — cách chia một hệ thống lớn: Bounded Context, Context Map, Ubiquitous Language, Anti-Corruption Layer.
* **Tactical pattern** — cách model bên trong một Bounded Context: Aggregate, Entity, Value Object, Domain Event, Domain Service, Repository, Factory.

Khi ai đó nói *"chúng tôi dùng DDD"*, họ có nghĩa là *"chúng tôi áp dụng practice DDD, và một subset các strategic + tactical pattern."*

---

## Hai nửa của DDD

### Strategic DDD — *làm sao chia hệ thống?*

Liên quan đến **bức tranh lớn**: boundary đặt ở đâu, ngôn ngữ chung là gì, các context giao tiếp ra sao?

* **Ubiquitous Language** — vocabulary chung giữa developer và domain expert.
* **Bounded Context** — boundary rõ ràng bên trong đó một model và ngôn ngữ của nó nhất quán.
* **Context Map** — sơ đồ các Bounded Context liên hệ ra sao (partnership, customer/supplier, conformist, anti-corruption layer, v.v.).
* **Anti-Corruption Layer (ACL)** — layer dịch ngăn không cho model của context khác làm ô nhiễm context của ta.

→ Xem [`strategic-patterns-vi.md`](./strategic-patterns-vi.md)

### Tactical DDD — *làm sao model bên trong một Bounded Context?*

Liên quan đến **code**: có object gì, trách nhiệm của chúng, enforce invariant ra sao.

* **Entity** — có identity tồn tại theo thời gian (`Order`, `Customer`).
* **Value Object** — immutable, equality theo giá trị (`Money`, `Email`, `Address`).
* **Aggregate / Aggregate Root** — boundary consistency; một root điều khiển truy cập phần còn lại.
* **Domain Event** — *"có gì đó đã xảy ra"* mà domain quan tâm (`OrderPlaced`).
* **Domain Service** — behavior domain không tự nhiên thuộc về entity.
* **Factory** — đóng gói construction phức tạp của aggregate.
* **Repository** — truy cập persistence cho aggregate root (xem [`../enterprise-pattern/repository-vi.md`](../enterprise-pattern/repository-vi.md)).

→ Xem [`tactical-patterns-vi.md`](./tactical-patterns-vi.md)

---

## DDD **không phải**

* **Không phải** Clean Architecture hay Onion. (Dù chúng kết hợp rất ăn ý.)
* **Không bắt buộc** với microservices. (Dù một Bounded Context cho mỗi service là default hữu ích.)
* **Không phải** "dùng Repository và Aggregate". Bạn có thể có chúng mà không làm DDD.
* **Không nói** về technology. Các tactical pattern là language-agnostic.

> *"Phần DDD không phải các pattern. Pattern thì dễ. Phần khó là strategic design."* — **Eric Evans**

---

## Ba trụ cột

1. **Tập trung vào core domain.** Đầu tư modeling vào nơi business tạo khác biệt; dùng giải pháp có sẵn cho phần còn lại (auth, payment, mailing).
2. **Khám phá model bằng cộng tác sáng tạo giữa domain expert và software practitioner.** Nói chuyện với domain expert là điều bắt buộc.
3. **Nói một Ubiquitous Language bên trong một Bounded Context rõ ràng.** Code, conversation, diagram, test — đều dùng cùng terms.

---

## Khi nào dùng DDD

✅ **Dùng DDD khi:**

* Domain **phức tạp** và **cốt lõi** với business (banking, insurance, logistics, healthcare, scheduling).
* Business logic khó vừa với CRUD model.
* Domain sẽ **tiến hoá** qua nhiều năm — đầu tư modeling dài hạn có lợi.
* Có thể tiếp cận **domain expert** sẵn sàng nói chuyện.

❌ **Không dùng DDD khi:**

* App CRUD. Stack controller → service → repository → DB là OK.
* Domain đã hiểu rõ và là commodity (admin tool cơ bản, site content).
* Không có domain expert để nói chuyện.
* Team không thoải mái với chi phí upfront và discipline.

---

## DDD + các khái niệm khác trong folder này

DDD là methodology — nó **kết hợp** với pattern và style ở chỗ khác:

| Khái niệm                               | Quan hệ với DDD                                                            |
| --------------------------------------- | -------------------------------------------------------------------------- |
| **Clean / Hexagonal / Onion**           | Style architectural đi cùng chuẩn. Giữ domain ở trung tâm.                  |
| **CQRS**                                | Thường áp dụng *bên trong* một Bounded Context.                            |
| **Event Sourcing**                      | Tùy chọn nhưng phù hợp tự nhiên — event hạng nhất trong DDD.                |
| **Microservices**                       | Một microservice thường = một Bounded Context.                              |
| **Saga**                                | Điều phối xuyên Bounded Context.                                           |
| **Repository**                          | Tactical DDD pattern (cũng là Fowler enterprise pattern).                  |
| **GoF Design Pattern**                  | Công cụ bạn dùng *bên trong* domain (Strategy, Factory, Visitor, …).        |

---

## Thứ tự đọc

1. **[`strategic-patterns-vi.md`](./strategic-patterns-vi.md)** — bắt đầu ở đây. Bounded Context và Ubiquitous Language là idea DDD giá trị nhất, dù bạn có adopt tactical pattern hay không.
2. **[`tactical-patterns-vi.md`](./tactical-patterns-vi.md)** — sau đó là pattern cấp code.
3. **[`../enterprise-pattern/repository-vi.md`](../enterprise-pattern/repository-vi.md)** — deep dive về Repository trong context DDD.
4. **[`../architectural-style/hexagonal-onion-clean-vi.md`](../architectural-style/hexagonal-onion-clean-vi.md)** — cách code DDD thường được tổ chức.

---

## Tham khảo

* Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003). Bản gốc.
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Companion thực hành.
* Vaughn Vernon — *Domain-Driven Design Distilled* (2016). Bản ngắn.
* Martin Fowler — [*DomainDrivenDesign*](https://martinfowler.com/tags/domain%20driven%20design.html).
* Steve Smith — *Domain-Driven Design Fundamentals* (Pluralsight).
