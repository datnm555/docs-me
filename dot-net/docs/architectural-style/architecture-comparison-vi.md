# N-Layer vs. N-Tier vs. Clean Architecture — So sánh & Hướng dẫn quyết định

> So sánh side-by-side ba cách tiếp cận kiến trúc cover trong docs folder, với key point, ưu/nhược, và framework quyết định.

> **Source doc:**
> * [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md)
> * [`n-tier-architecture-vi.md`](./n-tier-architecture-vi.md)
> * [`clean-architecture-vi.md`](./clean-architecture-vi.md)

> 🇻🇳 Phiên bản tiếng Việt. English: [`architecture-comparison.md`](./architecture-comparison.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — So sánh ba bên và decision guide cho **N-Layer**, **N-Tier**, và **Clean** Architecture — mỗi cái là gì, ngồi trên trục nào, ưu, nhược, chọn cái nào khi nào.
- **Tại sao** — Ba cái này thường bị conflate (đặc biệt N-Layer ↔ N-Tier). So sánh chỉ chúng không phải alternative — trả lời câu hỏi khác và phần lớn app dùng *cả ba* cùng nhau.
- **Khi nào** — Khởi đầu hệ thống mới; giải thích khác biệt cho team; chọn default đúng cho app CRUD vs. domain phức tạp.
- **Ở đâu** — Đọc **trước** deep-dive per-style trong folder này. Cho version năm-bên bao gồm Hexagonal và Vertical Slice, xem `five-architectures-comparison-and-mixing-vi.md`.

---

## Mục lục

1. [TL;DR — Chúng sống trên trục khác nhau](#1-tldr--chúng-sống-trên-trục-khác-nhau)
2. [Định nghĩa một câu](#2-định-nghĩa-một-câu)
3. [Trục khái niệm mỗi cái ngồi](#3-trục-khái-niệm-mỗi-cái-ngồi)
4. [Bảng so sánh Side-by-Side](#4-bảng-so-sánh-side-by-side)
5. [So sánh trực quan](#5-so-sánh-trực-quan)
6. [Key Point mỗi cái](#6-key-point-mỗi-cái)
7. [Ưu điểm](#7-ưu-điểm)
8. [Nhược điểm](#8-nhược-điểm)
9. [Cách kết hợp trong app .NET thực](#9-cách-kết-hợp-trong-app-net-thực)
10. [Decision Matrix — Nên chọn cái nào?](#10-decision-matrix--nên-chọn-cái-nào)
11. [Hiểu lầm phổ biến](#11-hiểu-lầm-phổ-biến)
12. [Migration Path giữa chúng](#12-migration-path-giữa-chúng)
13. [Khuyến nghị cuối](#13-khuyến-nghị-cuối)
14. [Tham khảo](#14-tham-khảo)

---

## 1. TL;DR — Chúng sống trên trục khác nhau

> **Điều quan trọng nhất hiểu trước: ba cái này KHÔNG phải ba lựa chọn tương đương.**

| Approach              | Trục                                    | Hỏi                                        |
| --------------------- | --------------------------------------- | ------------------------------------------ |
| **N-Layer**           | Tổ chức code **logic**                  | *Code tổ chức ra sao?*                     |
| **N-Tier**            | Topology deploy **vật lý**              | *Mỗi mảnh chạy ở đâu?*                     |
| **Clean Architecture**| **Nguyên lý / triết lý** thiết kế       | *Dependency nào được phép?*                |

Bạn thường không chọn *một* trong ba. App .NET điển hình là **cả ba cùng lúc**: tổ chức thành N-Layer, deploy qua N-Tier, theo nguyên lý Clean Architecture.

Lựa chọn thật là **bạn nghiêm ngặt ra sao** trên mỗi trục.

---

## 2. Định nghĩa một câu

- **N-Layer** — Tổ chức code thành stack top-down các layer (Presentation → Business → Data) nơi mỗi layer chỉ nói chuyện với cái dưới.
- **N-Tier** — Tách application qua nhiều **process/máy/container vật lý** giao tiếp qua mạng.
- **Clean Architecture** — Thiết kế dựa trên nguyên lý nơi **domain ngồi ở trung tâm** và **mọi dependency trỏ vào trong**, treat UI, DB, và framework là detail có thể thay thế.

---

## 3. Trục khái niệm mỗi cái ngồi

```
                 Trục Principles
                       ▲
                       │   Clean Architecture
                       │   (triết lý: dependency rule)
                       │
                       │
    Trục Logic ────────┼────────► Trục Vật lý
       N-Layer         │              N-Tier
   (cấu trúc code)     │       (topology deploy)
```

- Một app có thể có **3 layer + 1 tier** (monolith tổ chức tốt).
- Một app có thể có **3 layer + 3 tier** (kiến trúc web kinh điển).
- Một app có thể có **Clean Architecture + 4 layer + 2 tier** (app .NET hiện đại).
- Microservice có thể có **Clean Architecture + 3 layer** per service + **nhiều tier** tổng.

---

## 4. Bảng so sánh Side-by-Side

| Chiều                        | **N-Layer**                                       | **N-Tier**                                                | **Clean Architecture**                                       |
| ---------------------------- | ------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| **Bản chất**                 | Pattern tổ chức code                              | Topology deploy                                           | Triết lý thiết kế + nguyên lý                                 |
| **Coined bởi**               | Practice ngành chung (1990s)                      | Practice ngành chung (1990s)                              | Robert C. Martin ("Uncle Bob"), sách 2012 / 2017             |
| **Loại boundary**            | Logic (project/namespace)                         | Vật lý (process/máy/network)                              | Logic (vòng đồng tâm)                                        |
| **Giao tiếp**                | Method call in-process                            | HTTP / gRPC / message bus / TCP                           | Method call in-process; boundary cross qua DTO               |
| **Hướng dependency**         | Trên → dưới                                       | Caller → tier callee                                      | Ngoài → trong (Dependency Rule, strict)                      |
| **Nguyên lý cốt lõi**        | Separation of concerns                            | Scale độc lập / security / availability                   | Dependency Inversion + Độc lập (UI, DB, framework)           |
| **Số "thứ"**                 | 3–6 layer                                         | 1–5+ tier                                                 | 4 vòng đồng tâm (Entity / Use Case / Adapter / Framework)    |
| **Vị trí Domain**            | Layer giữa                                        | Bên trong application tier                                | Vòng **trong cùng** (trung tâm)                              |
| **Trạng thái DB / UI**       | Layer như bất kỳ cái khác                          | Tier riêng                                                | "Detail" — giữ ở vòng ngoài cùng                             |
| **Coupling framework**       | Cho phép trong layer bất kỳ                       | Cho phép trong mỗi tier                                   | **Cấm** trong vòng trong; chỉ vòng ngoài cùng touch framework |
| **Độ phức tạp setup**        | Thấp                                              | Trung → Cao (network, deploy)                             | Trung → Cao (nhiều project, interface, DTO)                  |
| **Onboarding curve**         | Nhẹ (phần lớn dev đã biết)                        | Vừa (devops + distributed system)                         | Dốc hơn (DI, boundary, use case)                              |
| **Testability**              | Tốt (mock layer dưới)                             | Khó end-to-end (cần container/mock per tier)              | **Xuất sắc** (domain không dep external)                     |
| **Chi phí refactor**         | Trung bình (thay đổi cross-layer touch nhiều file) | Cao (đổi contract tier vỡ consumer)                       | Trung-Thấp (intent: đổi detail không đụng policy)            |
| **Tốt nhất cho**             | App CRUD nhỏ/vừa                                  | High-scale, secure, hoặc phân tán địa lý                  | Hệ thống sống lâu, domain phức tạp (DDD, banking, ERP)       |
| **Tệ nhất cho**              | Domain model phức tạp                              | App nhỏ (over-engineering)                                | Prototype tí xíu / CRUD thuần                                |

---

## 5. So sánh trực quan

### N-Layer (stack logic)

```
┌─────────────────────────┐
│  Presentation           │
├─────────────────────────┤
│  Application / Service  │
├─────────────────────────┤
│  Domain / Business      │
├─────────────────────────┤
│  Data Access            │
└─────────────────────────┘
   một process
```

### N-Tier (topology vật lý)

```
┌────────┐ HTTP ┌────────────┐ TCP ┌────────────┐
│Browser │─────►│  Web/API   │────►│  Database  │
└────────┘      │  Server    │     └────────────┘
                └────────────┘
   máy / container vật lý riêng
```

### Clean Architecture (vòng đồng tâm)

```
        ┌───────────────────────────────────────┐
        │  Frameworks & Drivers (UI/DB/Web)     │
        │  ┌─────────────────────────────────┐  │
        │  │  Interface Adapters             │  │
        │  │  ┌───────────────────────────┐  │  │
        │  │  │  Use Cases                │  │  │
        │  │  │  ┌─────────────────────┐  │  │  │
        │  │  │  │  Entities (Domain)  │  │  │  │
        │  │  │  └─────────────────────┘  │  │  │
        │  │  └───────────────────────────┘  │  │
        │  └─────────────────────────────────┘  │
        └───────────────────────────────────────┘
              dependency trỏ VÀO TRONG ─►
```

---

## 6. Key Point mỗi cái

### N-Layer — key point

- **Stack of layer** với hướng dependency strict top-down.
- Layer phổ biến trong .NET: **Presentation / Application / Domain / Infrastructure**.
- Mỗi layer thường là **project/assembly** riêng trong .sln.
- **Cấm skip layer** (không có shortcut controller → repository).
- Pair tự nhiên với **dependency injection** (compose ở startup).
- Cross-cutting concern (logging, validation, caching) handle qua DI hoặc middleware.

### N-Tier — key point

- **Tách vật lý**, không logic: mỗi tier chạy trong process/host riêng.
- Topology chuẩn là **3-tier**: Presentation → Application → Data.
- **Scale độc lập**, **cô lập security**, **cô lập fault** là động lực chính.
- Mỗi hop tier thêm **latency + một mode fail** → cần retry, timeout, circuit breaker.
- Application tier **stateless** là cần thiết cho scale ngang.
- **Data tier không bao giờ nên trên public internet**.
- Distributed tracing (OpenTelemetry) bắt buộc qua tier.

### Clean Architecture — key point

- **Dependency Rule** là rule tuyệt đối duy nhất: source code dependency chỉ trỏ vào trong.
- **Domain ở trung tâm**, không reference framework, ORM, hay UI.
- **Use Case** orchestrate Entity để fulfill mục tiêu application.
- **Interface Adapter** dịch giữa use case và technology bên ngoài.
- **Framework là detail** (database, UI, API ngoài cũng vậy).
- **Screaming Architecture**: cấu trúc folder thể hiện *hệ thống làm gì*, không *framework gì nó dùng*.
- Pair tự nhiên với **DDD**, **CQRS**, **MediatR**, **Result\<T\>** trong .NET.
- Driven bởi **SOLID** (đặc biệt Dependency Inversion) và **Component Principle** (REP, CCP, CRP, ADP, SDP, SAP).

---

## 7. Ưu điểm

### N-Layer

✅ **Quen thuộc với gần như mọi dev** — onboarding ngắn.
✅ **Nhanh setup** — minimal ceremony.
✅ **Tách trách nhiệm rõ** giữa UI / logic / data.
✅ **Dễ test** layer-by-layer với mock.
✅ **Thân thiện tooling** — Visual Studio template, EF scaffolding đều giả định style này.
✅ **Có thể refactor sang Clean** — bước đệm tự nhiên.

### N-Tier

✅ **Scale độc lập** mỗi tier (scale API không cần scale DB).
✅ **Boundary security mạnh** — DB trong subnet private, network segmentation.
✅ **Cô lập fault** — cache fail không hạ cả app.
✅ **Tự do tech-stack per tier** — Blazor front-end + .NET API + Postgres DB.
✅ **Thân thiện compliance** — dễ thoả PCI / HIPAA / GDPR với data tier cô lập.
✅ **Thay một tier không rewrite cái khác** (vd., swap UI framework).

### Clean Architecture

✅ **Testability tối đa** — domain thuần, không infrastructure.
✅ **Độc lập framework** — survive ASP.NET → Minimal API → framework kế.
✅ **Độc lập database** — swap SQL Server → Postgres → Cosmos không đụng domain.
✅ **Maintainability dài hạn** — bảo vệ policy khỏi detail biến động.
✅ **Hỗ trợ DDD tự nhiên** — domain model phong phú, value object, aggregate.
✅ **Ép good practice** — DI, interface segregation, single responsibility.
✅ **Refactor-friendly** — boundary cô lập tốt giảm blast radius.

---

## 8. Nhược điểm

### N-Layer

❌ **Rủi ro anemic domain** — entity dễ thành data bag.
❌ **Thay đổi cross-cutting touch nhiều file** qua layer.
❌ **Dễ drift** — controller reach trực tiếp vào repository ("skip layer").
❌ **Không có bảo vệ explicit cho domain** — EF Core entity thường leak khắp nơi.
❌ **Không scale tốt** khi domain phức tạp grow.
❌ **God service** xuất hiện khi một service xử lý quá nhiều use case.

### N-Tier

❌ **Network latency** ở mọi hop.
❌ **Phức tạp vận hành** — nhiều thứ phải deploy, monitor, secure, back up.
❌ **Fail distributed** — partial outage giờ có thể.
❌ **Eventual consistency** giữa tier (cache, replica).
❌ **Cost serialization** ở mọi boundary.
❌ **Debug khó hơn** — cần correlation ID + distributed tracing.
❌ **Over-engineer cho app nhỏ** — monolith 1-tier OK cho phần lớn.
❌ **Rủi ro distributed-monolith** — nhiều tier phải deploy cùng = tệ nhất cả hai thế giới.

### Clean Architecture

❌ **Nhiều project, nhiều interface, nhiều DTO** — setup nặng hơn.
❌ **Learning curve** — boundary, dependency inversion, port/adapter.
❌ **Rủi ro over-engineer** cho app CRUD đơn giản.
❌ **Nhiều code mapping** giữa layer (entity ↔ DTO ↔ view model).
❌ **Velocity khởi đầu chậm hơn** — trả công chỉ trên project sống lâu.
❌ **Có thể cảm giác "ceremony"** cho feature trivial.
❌ **Misapplication thường gặp** — dev theo folder layout nhưng phá Dependency Rule dù sao.

---

## 9. Cách kết hợp trong app .NET thực

Ba cái này thường **compose**, không phải chọn.

### Ví dụ 1 — SaaS nhỏ

- **Clean Architecture**: 4 project logic (Web, Application, Domain, Infrastructure).
- **N-Layer**: 4 project đó LÀ layer, với Dependency Rule enforce.
- **N-Tier**: deploy như **2 tier** — App Service + Azure SQL.

### Ví dụ 2 — SaaS Enterprise vừa

- **Clean Architecture**: cùng cấu trúc logic 4-layer per service.
- **N-Layer**: cùng cấu trúc project bên trong mỗi service.
- **N-Tier**: deploy như **4 tier** — CDN, API, Cache (Redis), DB (Azure SQL).

### Ví dụ 3 — Microservices Platform

- **Clean Architecture**: mỗi service có vòng Domain + Use Case + Adapter + Framework riêng.
- **N-Layer**: mỗi service nội bộ 4 layer.
- **N-Tier**: chục tier — gateway, service mesh, nhiều service, nhiều data store.

> Bài học: **adopt nguyên lý Clean Architecture, tổ chức qua N-Layer, deploy qua N-Tier**. Chúng là công cụ bổ sung ở ba cấp abstraction khác nhau.

---

## 10. Decision Matrix — Nên chọn cái nào?

Đọc hàng từ trên xuống. "Trả lời" cho bạn style cấu trúc. Cột "tier" cho topology deploy.

| Tình huống của bạn                                                    | Style kiến trúc                              | Số tier         |
| -------------------------------------------------------------------- | ------------------------------------------- | ---------------- |
| Prototype hackathon / spike                                          | Single-project N-Layer (hoặc 1 file)        | 1                |
| App admin CRUD nhỏ                                                   | N-Layer 3-layer                             | 2 (app + DB)     |
| App LOB nội bộ vừa                                                   | N-Layer 4-layer với ảnh hưởng Clean nhẹ     | 2–3              |
| Sản phẩm sống lâu với domain grow                                    | **Clean Architecture** + feature folder      | 3                |
| Banking, ERP, healthcare, logistics                                  | **Clean Architecture** + DDD + CQRS         | 3–5              |
| SaaS lưu lượng cao                                                   | Clean Architecture + API tier scale ngang   | 3–5              |
| Backend event-driven với worker                                      | Clean + Outbox pattern + worker tier riêng  | 4–5+             |
| Platform multi-team với bounded context                              | Microservices, mỗi cái với Clean Architecture | Nhiều          |
| Embedded / mobile / desktop, không network                           | N-Layer in-process                          | 1                |

### Chooser nhanh

- **Bạn có domain business phức tạp?** → Clean Architecture.
- **Cần scale phần độc lập?** → Thêm tier.
- **Mới với codebase hoặc .NET?** → Khởi đầu N-Layer; refactor sang Clean khi cần.
- **App nhỏ và sống ngắn?** → N-Layer là đủ.

---

## 11. Hiểu lầm phổ biến

| Hiểu lầm                                                            | Thực tế                                                                                                          |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| "N-Layer và N-Tier giống nhau."                                     | Là trục khác nhau (logic vs. vật lý). Thường bị confuse vì tên overlap.                                            |
| "Clean Architecture nghĩa là có 4 tên project cụ thể."              | Tên không quan trọng. Dependency Rule quan trọng. Có thể làm Clean Architecture trong một project — hoặc vi phạm qua 12. |
| "Clean Architecture cấm dùng EF Core."                              | Không. Cấm EF Core trong **Domain**. Vòng Infrastructure là *chính xác* nơi EF Core thuộc về.                    |
| "N-Tier nghĩa là microservices."                                    | Không. N-Tier là topology deploy; microservices là cách decompose theo business capability. Chúng overlap nhưng không bằng. |
| "Clean Architecture luôn tốt hơn."                                  | Không cho app CRUD nhỏ. Cost > benefit ở phía nhỏ.                                                               |
| "Nhiều layer = thiết kế tốt hơn."                                   | Nhiều layer = nhiều ma sát. Thêm layer chỉ khi có trách nhiệm rõ nó sở hữu.                                       |
| "Mỗi tier phải là VM riêng."                                        | Không. Container, process, hay kể cả host trên cùng VM tính. Boundary là process/network, không phải metal.       |
| "Clean Architecture và DDD giống nhau."                             | DDD tập trung modeling domain; Clean Architecture định nghĩa domain sống ở đâu. Chúng pair tốt nhưng giải vấn đề khác. |

---

## 12. Migration Path giữa chúng

### N-Layer thuần → Clean Architecture

1. Move business rule từ service *vào* entity.
2. Định nghĩa interface repository trong **Domain** (không Infrastructure).
3. Ngừng return EF entity từ controller; introduce DTO.
4. Thêm `Application/Abstractions` cho interface cross-cutting.
5. Thêm architecture test để enforce Dependency Rule.
6. Adopt MediatR (tuỳ chọn) cho dispatch use case.
7. Rename folder match business capability (Screaming Architecture).

### Monolith (1 tier) → N-Tier

1. Làm API stateless (externalize session sang Redis hoặc DB).
2. Move database sang server / managed service riêng.
3. Thêm health endpoint + load-balancer routing.
4. Introduce TLS / mTLS, network segmentation.
5. Thêm OpenTelemetry tracing.
6. Thêm resilience policy (retry, timeout, circuit breaker).
7. Deploy qua CI pipeline riêng per tier.

### Monolith N-Tier → Microservices

1. Xác định **bounded context** trong domain.
2. Extract một context một lần (pattern Strangler Fig).
3. Mỗi service mới phải sở hữu data — không DB share.
4. Lập async messaging (Service Bus / Kafka) cho event cross-context.
5. Thêm API gateway / BFF cho front-end.

---

## 13. Khuyến nghị cuối

> **Cho app .NET hiện đại 2026, default mạnh là:**
>
> 1. **Clean Architecture** như triết lý thiết kế.
> 2. Implement qua cấu trúc **4-layer N-Layer** (`Domain`, `Application`, `Infrastructure`, `Web`).
> 3. **Deploy qua 3 tier** (UI / API / DB).
> 4. **Folder feature vertical-slice** bên trong `Application` khi app có hơn vài use case.
> 5. Adopt **CQRS + MediatR + Result\<T\> + FluentValidation** khi use case nhân lên.
> 6. Thêm **microservices** chỉ khi bounded context và team boundary justify chi phí vận hành.

Cho app rất nhỏ hoặc sống ngắn, **N-Layer 3-layer thuần trong 2 tier** hoàn hảo — đừng over-engineer.

> Kiến trúc không được xếp hạng. Chúng trả lời câu hỏi khác nhau. Hệ thống .NET tuyệt trả lời cả ba: *code tổ chức ra sao?* (N-Layer), *nó chạy ở đâu?* (N-Tier), *dependency nào được phép?* (Clean).

---

## 14. Tham khảo

- [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md)
- [`n-tier-architecture-vi.md`](./n-tier-architecture-vi.md)
- [`clean-architecture-vi.md`](./clean-architecture-vi.md)
- Martin, Robert C. *Clean Architecture: A Craftsman's Guide to Software Structure and Design*, 2017.
- [Uncle Bob — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microsoft Learn — N-tier architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- [Microsoft Learn — Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [N-Layered vs Clean vs Vertical Slice (antondevtips)](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
