# Năm Kiến Trúc — So sánh & Cách Kết Hợp

> Nhìn side-by-side năm kiến trúc cover trong docs folder, cộng guide thực hành **kết hợp** chúng trong app .NET thực.
>
> Nửa đầu so sánh; nửa sau cho biết mix nào work, mix nào không, và cách chọn.

> **Source doc:**
> * [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md)
> * [`n-tier-architecture-vi.md`](./n-tier-architecture-vi.md)
> * [`clean-architecture-vi.md`](./clean-architecture-vi.md)
> * [`vertical-slice-architecture-vi.md`](./vertical-slice-architecture-vi.md)
> * [`hexagonal-architecture-vi.md`](./hexagonal-architecture-vi.md)
> * [`architecture-comparison-vi.md`](./architecture-comparison-vi.md) — so sánh ba-bên trước đó

> 🇻🇳 Phiên bản tiếng Việt. English: [`five-architectures-comparison-and-mixing.md`](./five-architectures-comparison-and-mixing.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — So sánh side-by-side năm architectural style trong folder này (**N-Layer, N-Tier, Clean, Vertical Slice, Hexagonal**) cộng guide thực hành **mix** chúng — combination nào work, cái nào không, và cách pick.
- **Tại sao** — App thật kết hợp nhiều style (vd., Clean + Vertical Slice + 3-tier). Doc này cho bạn combination nào canonical, cái nào anti-mix, và cách migrate giữa chúng.
- **Khi nào** — Pick stack cho app mới; audit hệ thống hiện có để đặt tên cho cái bạn thực sự có; plan migration từ N-Layer sang Clean Slice.
- **Ở đâu** — File so sánh comprehensive nhất trong `architectural-style/`. Pair với deep-dive per-style cho detail và `architecture-comparison-vi.md` cho so sánh ba-bên đơn giản hơn.

---

## Mục lục

### Part A — So sánh

1. [TL;DR — Chúng sống trên trục khác nhau](#1-tldr--chúng-sống-trên-trục-khác-nhau)
2. [Định nghĩa một câu](#2-định-nghĩa-một-câu)
3. [Trục mỗi cái ngồi](#3-trục-mỗi-cái-ngồi)
4. [Bảng Side-by-Side lớn](#4-bảng-side-by-side-lớn)
5. [Visual Recap](#5-visual-recap)
6. [Key Point per kiến trúc](#6-key-point-per-kiến-trúc)
7. [Matrix Ưu điểm](#7-matrix-ưu-điểm)
8. [Matrix Nhược điểm](#8-matrix-nhược-điểm)

### Part B — Mix Kiến trúc

9. [Tại sao Mix?](#9-tại-sao-mix)
10. [Mental Model Mix — Trực giao vs. Cạnh tranh](#10-mental-model-mix--trực-giao-vs-cạnh-tranh)
11. [Compatibility Matrix](#11-compatibility-matrix)
12. [Combination kinh điển (Recipe)](#12-combination-kinh-điển-recipe)
13. [Anti-Mix — Combination cần tránh](#13-anti-mix--combination-cần-tránh)
14. [Mix bên trong vs. xuyên Bounded Context](#14-mix-bên-trong-vs-xuyên-bounded-context)
15. [Decision Flow — Chọn Stack của bạn](#15-decision-flow--chọn-stack-của-bạn)
16. [Recipe từng bước cho app thực](#16-recipe-từng-bước-cho-app-thực)
17. [Migration Path giữa các Mix](#17-migration-path-giữa-các-mix)
18. [Khuyến nghị cuối cho .NET 2026](#18-khuyến-nghị-cuối-cho-net-2026)
19. [Tham khảo](#19-tham-khảo)

---

# PART A — SO SÁNH

## 1. TL;DR — Chúng sống trên trục khác nhau

> **Điểm quan trọng nhất: năm cái này không phải năm lựa chọn thay thế nhau.** Chúng trả lời câu hỏi khác nhau.

| Kiến trúc           | Trục                                        | Hỏi                                                        |
| ------------------- | ------------------------------------------- | ---------------------------------------------------------- |
| **N-Layer**         | **Tổ chức logic** (horizontal)              | *Code tổ chức trong project/folder ra sao?*                |
| **Vertical Slice**  | **Tổ chức logic** (vertical)                | *Code tổ chức theo feature ra sao?*                        |
| **N-Tier**          | **Deploy vật lý**                           | *Mỗi mảnh chạy ở đâu?*                                     |
| **Clean**           | **Nguyên lý thiết kế** (prescriptive)       | *Dependency nào được phép? Strict ra sao?*                 |
| **Hexagonal**       | **Nguyên lý thiết kế** (cũ hơn, ít prescriptive) | *Core cô lập khỏi technology bên ngoài ra sao?*       |

Bạn chọn **một** điểm trên mỗi trục. App thực là *combination*, không phải một kiến trúc đơn.

---

## 2. Định nghĩa một câu

- **N-Layer** — Stack layer horizontal (Presentation → Business → Data), dependency top-down.
- **N-Tier** — Nhiều process/máy vật lý giao tiếp qua mạng.
- **Clean Architecture** — Vòng đồng tâm (Entity ← Use Case ← Adapter ← Framework) với dependency **inward** strict; framework và DB là "detail".
- **Vertical Slice** — Tổ chức theo feature, không layer; một folder per use case.
- **Hexagonal (Ports & Adapters)** — Core bao quanh bởi port; mọi hệ thống bên ngoài plug vào qua adapter; core không phụ thuộc gì bên ngoài.

---

## 3. Trục mỗi cái ngồi

```
                          Trục PRINCIPLES
                                ▲
                                │ ┌─────────────────┐
                                │ │ Clean (strict)  │
                                │ │ Hexagonal       │
                                │ └─────────────────┘
                                │
                                │
   ────── Trục LOGIC ───────────┼───────── Trục VẬT LÝ ──────
   ┌───────────────┐            │            ┌─────────────┐
   │ N-Layer       │            │            │ N-Tier      │
   │ Vertical Slice│            │            │             │
   └───────────────┘            │            └─────────────┘
                                │
```

- **N-Layer** và **Vertical Slice** là *thay thế trên trục logic* (chọn một hoặc hybrid).
- **Clean** và **Hexagonal** là *neighbor trên trục principles* — Hexagonal là tiền thân khái niệm; Clean là tổng hợp prescriptive hơn.
- **N-Tier** là *trực giao* với cả bốn — bất kỳ cái nào có thể deploy 1, 2, 3, hay N tier.

---

## 4. Bảng Side-by-Side lớn

| Chiều                      | **N-Layer**                       | **Vertical Slice**                          | **N-Tier**                                     | **Clean**                                                       | **Hexagonal**                                |
| -------------------------- | --------------------------------- | ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------- |
| **Năm / Origin**           | Practice ngành 1990s              | ~2018 (Jimmy Bogard)                        | Practice ngành 1990s                           | Sách 2012 / 2017 (Uncle Bob)                                    | ~2005 (Alistair Cockburn)                    |
| **Trục**                   | Logic (horizontal)                | Logic (vertical, theo feature)              | Vật lý                                         | Principles (ring + use case)                                    | Principles (port + adapter)                  |
| **Loại Boundary**          | Project / namespace               | Folder per feature                          | Process / network                              | Vòng đồng tâm                                                    | Port (interface)                             |
| **Giao tiếp**              | Method call in-process            | In-process qua MediatR                      | HTTP / gRPC / queue                            | In-process, DTO qua ring                                         | Qua interface port                            |
| **Hướng dependency**       | Trên → dưới                       | Không qua slice                             | Tier caller → tier callee                      | Ngoài → trong (Dependency Rule)                                  | Adapter → Core (luôn vào trong)               |
| **Vị trí Domain**          | Layer giữa                        | Share hoặc per-slice                        | Bên trong app tier                             | Vòng trong cùng (trung tâm)                                      | Bên trong hexagon                             |
| **Coupling Framework**     | Cho phép layer nào                | Cho phép bên trong slice                    | Cho phép trong mỗi tier                        | Cấm trong vòng trong                                              | Cấm trong core; chỉ adapter                   |
| **Cách Test**              | Mock layer dưới                   | Test slice end-to-end                       | Khó E2E qua tier                               | Test domain thuần + integration boundary                         | **In-memory fake adapter** (không mock)       |
| **Độ phức tạp Setup**      | Thấp                              | Thấp                                        | Trung-Cao                                      | Trung-Cao                                                        | Trung-Cao                                    |
| **Onboarding curve**       | Dễ                                | Dễ                                          | Vừa                                            | Dốc                                                              | Vừa-Dốc                                      |
| **Cost Refactor**          | Trung (thay đổi cross-layer)      | Thấp (per-slice)                            | Cao (contract tier)                            | Trung-Thấp (ring cô lập)                                          | Trung-Thấp (core cô lập)                      |
| **Tốt nhất cho**           | CRUD nhỏ/vừa                      | Nhiều feature độc lập                       | Scale, security, cô lập fault                  | Domain phức tạp sống lâu                                          | App multi-channel, integration nặng           |
| **Không fit**              | Domain phức tạp                   | Invariant domain nặng                       | App nhỏ                                        | Prototype tí xíu / CRUD thuần                                     | CRUD tí xíu; team mới với DI                  |
| **Boilerplate**            | Cao (file cross-layer)            | Thấp                                        | Cao (infra cross-tier)                         | Cao (nhiều project, interface, DTO)                              | Trung-Cao (port + adapter per tech)           |
| **Mức prescriptive**       | Lỏng                              | Lỏng                                        | Lỏng                                           | **Rất prescriptive**                                              | Lỏng (chỉ rule boundary)                      |

---

## 5. Visual Recap

```
N-LAYER (stack horizontal)               VERTICAL SLICE (per-feature)
─────────────────────────                 ─────────────────────────────
┌───────────────────────┐                 ┌─────┬─────┬─────┬─────┐
│  Presentation         │                 │Place│Get  │List │Cancel│
├───────────────────────┤                 │Order│Order│Order│Order │
│  Application          │                 │ R V │ R V │ R V │ R V │
├───────────────────────┤                 │ H E │ H E │ H E │ H E │
│  Domain               │                 └─────┴─────┴─────┴─────┘
├───────────────────────┤
│  Infrastructure       │
└───────────────────────┘


N-TIER (vật lý)                           CLEAN (đồng tâm)                HEXAGONAL (port & adapter)
─────────────────                          ──────────────────              ────────────────────────────
                                                                            Driving     Driven
┌───┐  HTTP ┌────┐  TCP ┌────┐                ┌────────────────┐           ┌──┐         ┌──┐
│UI │──────►│API │─────►│DB  │                │  Frameworks    │           │HTTP◄─port─►│DB│
└───┘       └────┘      └────┘                │ ┌────────────┐ │           ├──┤  CORE   ├──┤
                                              │ │ Adapters   │ │           │CLI◄─port─►│SMTP│
                                              │ │┌──────────┐│ │           ├──┤         ├──┤
                                              │ ││Use Cases ││ │           │TST◄─port─►│MQ│
                                              │ ││┌────────┐││ │           └──┘         └──┘
                                              │ │││Entities│││ │
                                              │ ││└────────┘││ │
                                              │ │└──────────┘│ │
                                              │ └────────────┘ │
                                              └────────────────┘
```

---

## 6. Key Point per kiến trúc

### N-Layer
- Stack layered top-down, in-process.
- Cấm "skip layer".
- Quen thuộc; template Visual Studio mặc định khuyến khích.
- Rủi ro: anemic domain, god service.

### Vertical Slice
- Một folder per use case; mọi thứ sống cùng nhau.
- "Tối thiểu coupling giữa slice, tối đa coupling trong slice." (J. Bogard)
- Pair với MediatR + pipeline behavior + FluentValidation.
- Tolerate trùng lặp; refactor qua rule of three.

### N-Tier
- Tách vật lý qua process/máy.
- Scale độc lập, cô lập security, cô lập fault.
- Mỗi hop tier = latency + mode fail mới.
- Resilience, distributed tracing, mTLS bắt buộc.

### Clean Architecture
- **Dependency Rule** không thương lượng: dependency chỉ trỏ vào trong.
- Domain ở trung tâm, framework ở rìa.
- Database và UI là "detail".
- Pair tự nhiên với DDD, CQRS, MediatR.

### Hexagonal (Ports & Adapters)
- Tiền thân khái niệm của Onion và Clean.
- **Port** thuộc core; **adapter** implement chúng.
- Đối xứng: driving adapter vào, driven adapter ra.
- Killer feature: **fake adapter** in-memory cho test Core instant.

---

## 7. Matrix Ưu điểm

| Lợi ích                            | N-Layer | Vertical Slice | N-Tier | Clean | Hexagonal |
| ---------------------------------- | :-----: | :------------: | :----: | :---: | :-------: |
| Dễ học                              | ✅     | ✅            | —      | —     | —         |
| Delivery feature đầu nhanh         | ✅     | ✅            | —      | —     | —         |
| Cohesion cao per feature           | —      | ✅✅          | —      | ✅    | ✅        |
| Coupling thấp giữa feature         | —      | ✅✅          | —      | ✅    | ✅        |
| Dễ xoá feature                     | —      | ✅✅          | —      | —     | —         |
| Scale độc lập                      | —      | —             | ✅✅  | —     | —         |
| Cô lập security/fault              | —      | —             | ✅✅  | —     | —         |
| Độc lập framework                  | —      | —             | —      | ✅✅  | ✅✅      |
| Độc lập DB                         | —      | —             | —      | ✅✅  | ✅✅      |
| Nhiều driving channel              | —      | —             | ✅     | ✅    | ✅✅      |
| Domain unit-testable thuần         | —      | —             | —      | ✅✅  | ✅✅      |
| Replace tech per adapter            | —      | ✅            | ✅     | ✅    | ✅✅      |
| Pair với DDD                       | —      | —             | —      | ✅✅  | ✅✅      |
| Default thân thiện tooling         | ✅✅   | ✅            | ✅     | —     | —         |

---

## 8. Matrix Nhược điểm

| Nhược                                  | N-Layer | Vertical Slice | N-Tier | Clean | Hexagonal |
| --------------------------------------- | :-----: | :------------: | :----: | :---: | :-------: |
| Thay đổi cross-layer đụng nhiều file    | ✅     | —              | —      | ✅    | ✅        |
| Rủi ro anemic domain                    | ✅     | ✅             | —      | —     | —         |
| Trùng lặp code qua feature             | —      | ✅             | —      | —     | —         |
| Boilerplate nặng / nhiều project       | —      | —              | ✅     | ✅    | ✅        |
| Learning curve dốc                      | —      | —              | ✅     | ✅    | ✅        |
| Network latency mọi hop                | —      | —              | ✅✅   | —     | —         |
| Mode fail distributed                   | —      | —              | ✅✅   | —     | —         |
| Gánh nặng eventual consistency          | —      | —              | ✅✅   | —     | —         |
| Khó enforce cơ học                     | —      | ✅             | ✅     | —     | —         |
| Over-engineer cho app nhỏ              | —      | —              | ✅     | ✅    | ✅        |
| Rủi ro abstraction leaky                | ✅     | ✅             | —      | ✅    | ✅        |

---

# PART B — MIX KIẾN TRÚC

## 9. Tại sao Mix?

> Không kiến trúc đơn nào trả lời mọi câu hỏi. App thực cần tổ chức code (logic), quyết định nơi chạy (vật lý), và định nghĩa dependency nào được phép (principles). **Mix là rule, không phải exception.**

Năm kiến trúc fit với nhau vì chúng trả lời câu hỏi khác nhau:

- **Cái gì đi đâu trong codebase?** → N-Layer **hoặc** Vertical Slice (hoặc cả hai).
- **Chạy ở đâu vật lý?** → N-Tier (luôn — kể cả "1 tier" là lựa chọn).
- **Ai được phép phụ thuộc ai?** → Clean (hoặc Hexagonal).

App .NET hiện đại điển hình chọn một option từ mỗi hàng.

---

## 10. Mental Model Mix — Trực giao vs. Cạnh tranh

| Cặp                           | Quan hệ         | Có thể dùng cả hai?                                                           |
| ----------------------------- | --------------- | ---------------------------------------------------------------------------- |
| **N-Layer ↔ Vertical Slice**  | **Cạnh tranh**  | Thường *một hoặc cái khác*. Hybrid có thể bên trong shell Clean ngoài.       |
| **Clean ↔ Hexagonal**         | **Đồng nghĩa**  | Cùng essence. Chọn một vocabulary. Dùng cả hai tên cho cùng code OK.         |
| **Clean ↔ N-Layer**           | **Bổ sung**     | Có — Clean là *principles*, N-Layer là *implementation*.                     |
| **Clean ↔ Vertical Slice**    | **Bổ sung**     | Có — hybrid **"Clean Slice"** nổi tiếng.                                     |
| **Hexagonal ↔ N-Layer**       | **Bổ sung**     | Có — layer *bên trong* core hexagon.                                          |
| **Hexagonal ↔ Vertical Slice**| **Bổ sung**     | Có — slice *bên trong* core hexagon.                                          |
| **N-Tier ↔ bất kỳ**           | **Trực giao**   | **Luôn có.** N-Tier về deployment; cái khác về code.                          |

> **Quy tắc:**
> - **Chọn một** từ {N-Layer, Vertical Slice}.
> - **Chọn một** từ {Clean, Hexagonal} làm tập nguyên lý (hoặc skip và không có nguyên lý).
> - **Chọn số tier** dựa trên nhu cầu scale/security.

---

## 11. Compatibility Matrix

| Kết hợp                                   | Tương thích  | Ghi chú                                                              |
| ----------------------------------------- | :-----------: | -------------------------------------------------------------------- |
| N-Layer + N-Tier                          | ✅ ✅        | Mix enterprise kinh điển. Mỗi tier giữ N layer.                       |
| Clean + N-Layer                           | ✅ ✅        | Default .NET hiện đại. Layer = 4 vòng đồng tâm.                       |
| Clean + Vertical Slice ("Clean Slice")    | ✅ ✅        | Layout .NET hiện đại được khuyến nghị nhất 2025/2026.                 |
| Clean + N-Tier                            | ✅ ✅        | Code tổ chức Clean; tier xử lý scale & security.                      |
| Hexagonal + Vertical Slice                | ✅ ✅        | Slice sống bên trong core hexagon.                                    |
| Hexagonal + N-Tier                        | ✅ ✅        | Mỗi tier có thể là hexagon riêng (rất phổ biến trong microservices).  |
| Hexagonal + N-Layer                       | ✅           | Layer tổ chức core; port + adapter bound rìa.                         |
| Clean + Hexagonal (cả hai)                | ⚠️           | Trùng lặp — cùng essence, hai vocabulary. Dùng một tên.               |
| N-Layer + Vertical Slice                  | ⚠️           | Conflict trừ khi split: layer cho share, slice cho feature.           |
| Vertical Slice + N-Tier                   | ✅           | Slice trong một tier; tier connect qua network.                        |
| Clean + Hexagonal + Vertical Slice + N-Tier | ✅         | App hiện đại thực — nhưng đảm bảo mỗi cái pulling weight.             |

> ✅ ✅ = kinh điển, ✅ = work, ⚠️ = có thể nhưng watch redundancy/conflict.

---

## 12. Combination kinh điển (Recipe)

### 🔥 Recipe 1 — "Classic Enterprise" (Default pre-2018)

**Mix:** N-Layer + N-Tier.

```
Presentation (Web tier)
 └── Application (App tier)
      └── Domain (App tier)
           └── Data Access (App tier ↔ DB tier)
```

- **Dùng khi:** hệ thống legacy, ERP / CRM / LOB truyền thống.
- **Không dùng khi:** domain phong phú hoặc thay đổi thường xuyên.

### 🔥 Recipe 2 — "Modern Clean" (Default hiện tại)

**Mix:** Clean + N-Layer + N-Tier.

```
src/
├── Domain/            ← Entity + Value Object (trong cùng)
├── Application/       ← Use case + abstraction
├── Infrastructure/    ← EF Core + adapter
└── Web/               ← ASP.NET Core (ngoài cùng)

Deploy qua 3 tier: UI / API / DB
```

- **Dùng khi:** app business vừa-lớn với lifespan dài.
- **Best fit:** banking, healthcare, logistics, SaaS platform.

### ⭐ Recipe 3 — "Clean Slice" (Khuyến nghị 2026)

**Mix:** Clean + Vertical Slice + N-Tier.

```
src/
├── Domain/                           ← Vòng trong cùng Clean
├── Application/
│   ├── Features/                     ← Vertical slice sống ở đây
│   │   ├── Orders/
│   │   │   ├── PlaceOrder/
│   │   │   └── CancelOrder/
│   │   └── Customers/
│   └── Abstractions/                 ← Port / interface
├── Infrastructure/                   ← Adapter (EF, SMTP, MQ)
└── Web/                              ← Endpoint, một per feature
```

- Clean enforce **dependency rule** bên ngoài.
- Vertical Slice cho **cohesion feature** bên trong.
- Deploy qua **3 tier** (UI / API / DB).

> Default mạnh cho app .NET mới 2026. Best of both worlds.

### ⭐ Recipe 4 — "Hexagonal Slice"

**Mix:** Hexagonal + Vertical Slice + N-Tier.

```
src/
├── Core/                             ← Hexagon
│   ├── Features/                     ← Slice
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderRequest.cs
│   │   │   ├── PlaceOrderResponse.cs
│   │   │   └── PlaceOrderUseCase.cs
│   │   └── CancelOrder/…
│   ├── Domain/
│   └── Ports/                        ← IOrderRepository, IEmailSender
├── Adapters.Web/
├── Adapters.Persistence.Ef/
├── Adapters.Email.SendGrid/
└── Adapters.Messaging.ServiceBus/
```

- Giống Clean Slice nhưng với **project ports + adapter explicit**.
- **Dùng khi:** app phải support **nhiều driving channel** (HTTP + CLI + scheduled job) hoặc integration external nặng.

### ⭐ Recipe 5 — "Microservices Hexagon"

**Mix:** Hexagonal + Vertical Slice + Nhiều N-Tier (microservices).

- Mỗi microservice là **Hexagon** nhỏ với Core, port, adapter riêng.
- **Bên trong** mỗi service, dùng Vertical Slice.
- **Xuyên** service, giao tiếp qua messaging async hoặc gRPC.

```
Order Service (hexagon)  ─┐
Billing Service (hexagon) ─┤  Service Bus / gRPC
Shipping Service (hexagon)─┘
```

- **Dùng khi:** nhiều team sở hữu bounded context; nhu cầu scale vary per service.
- **Cost:** overhead vận hành cao nhất — chỉ justify ở scale hoặc team boundary.

### Recipe 6 — "Vertical Slice First" (Prototype greenfield)

**Mix:** Vertical Slice + N-Tier (2 tier).

```
src/
├── Api/
│   ├── Features/
│   │   ├── PlaceOrder/
│   │   ├── ListProducts/
│   │   └── …
│   └── Program.cs
└── Domain/                  ← tuỳ chọn, entity share
```

- **Dùng khi:** API mới nhỏ, MVP, hoặc project hackathon.
- Skip Clean/Hexagonal tới khi phức tạp xuất hiện.
- **Migration path:** introduce vòng Clean chỉ khi domain matures.

### Recipe 7 — "Plain N-Layer 2 Tier" (App nhỏ hợp lý)

**Mix:** N-Layer + N-Tier (2 tier).

```
src/
├── Web/
├── Business/
└── Data/
```

- **Dùng khi:** admin tool nội bộ CRUD, portal vứt đi.
- Đủ tốt. Đừng over-engineer.

---

## 13. Anti-Mix — Combination cần tránh

| Anti-mix                                                             | Tại sao tệ                                                                                |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Clean + Hexagonal naming cả hai** trong cùng codebase              | Trùng lặp. Cùng essence, hai vocabulary → team confuse. Chọn một và stick.                |
| **Vertical Slice nhưng giữ folder `Services/` và `Repositories/`**   | Bạn tái tạo N-Layer cạnh slice → coupling cross-cutting khắp nơi.                          |
| **N-Tier không stateless ở API tier**                                | Sticky session giết scale ngang — bạn trả cost không nhận lợi ích.                        |
| **Clean Architecture với EF entity trong Domain**                    | Vi phạm Dependency Rule. Bạn "có Clean" chỉ trên giấy.                                    |
| **N-Tier với DB share qua service**                                  | Distributed monolith. Tệ nhất cả hai thế giới.                                            |
| **Hexagonal nơi port sống trong project adapter**                    | Đảo mũi tên dependency — adapter "sở hữu" contract.                                       |
| **Vertical Slice với "shared service" gọi từ mọi slice**             | Đánh bại slice isolation. Dùng **entity share** hoặc **event**, không service share.       |
| **Mix N-Layer (horizontal) và Vertical Slice không rule**            | Một số feature touch layer; khác là slice; không nhất quán. Chọn một hoặc define split.    |
| **Adopt Clean cho admin CRUD tí xíu**                                | Ceremony > value. Dùng N-Layer thuần.                                                     |
| **Microservices cho startup 2-team**                                 | Phức tạp vận hành lấn value. Modular monolith trước.                                       |

---

## 14. Mix bên trong vs. xuyên Bounded Context

Refinement hữu ích: **bounded context khác có thể dùng kiến trúc khác**.

- Context **domain phức tạp** (vd., `Pricing`, `Underwriting`) → Clean Architecture + DDD.
- Context **CRUD-heavy** (vd., `Catalog admin`, `User profile`) → N-Layer hoặc Vertical Slice thuần.
- Context **integration-heavy** (vd., `External Sync`) → Hexagonal với nhiều adapter.
- Context **report/read** → CQRS read model với raw SQL / Dapper, ignore phần lớn pattern.

> **Đừng áp đặt một kiến trúc cho mọi phần của hệ thống. Shape đúng phụ thuộc cost thay đổi trong *góc đó* của codebase.**

---

## 15. Decision Flow — Chọn Stack của bạn

```
                App tí xíu / sống ngắn?
                            │
              ┌──── có ─────┴──── không ──┐
              │                          │
   N-Layer thuần (3 layer,               Domain phức tạp /
   1–2 tier, không Clean,                sống lâu?
   không MediatR ceremony)                   │
                                ┌────── có ──┴── không ──────┐
                                │                            │
                       Clean (hoặc Hexagonal)        Nhiều feature độc lập?
                       + DDD nơi nó pay                       │
                                │               ┌──── có ────┴── không ──┐
                                │                │                       │
                                ▼          Vertical Slice            N-Layer
                       Nhiều driving         + Clean shell           + N-Tier
                       channel?              + N-Tier
                            │
                  ┌─── có ──┴── không ──┐
                  │                      │
              Hexagonal            Clean (Clean Slice)
              + Vertical Slice     + N-Tier
              + N-Tier
                            │
                  Nhiều bounded context
                  + team độc lập?
                            │
                  ┌─── có ──┴── không ──┐
                  │                      │
              Microservices         Stay monolithic /
              Hexagon per service   modular monolith
```

---

## 16. Recipe từng bước cho app thực

### 🛠 "Admin CRUD nội bộ" (1 dev, 3-month lifespan)

```
Pick: N-Layer (3 layer) + N-Tier (2 tier)
Skip: Clean, Hexagonal, Vertical Slice
Use:  ASP.NET Core MVC + EF Core + SQL Server
```

### 🛠 "API SaaS Greenfield" (4 dev, horizon nhiều năm)

```
Pick: Clean Slice (Clean + Vertical Slice) + N-Tier (3 tier)
Stack: ASP.NET Core minimal API + MediatR + FluentValidation + EF Core + Redis
Tests: WebApplicationFactory + Testcontainers + ArchUnitNET
```

### 🛠 "Platform banking enterprise" (15 dev, hàng thập kỷ)

```
Pick: Hexagonal + Clean (ring bên trong hexagon) + Vertical Slice + Microservices
       (mỗi microservice là hexagon riêng)
Stack: ASP.NET Core + gRPC + Service Bus + SQL Server + Redis + Kafka
Discipline: DDD bounded context, CQRS, event sourcing trên write side,
            Outbox pattern, distributed tracing, mTLS giữa tier
```

### 🛠 "Modernize monolith WebForms legacy"

```
Hôm nay:  N-Layer (4 layer) + N-Tier (3 tier)
Step 1:   Strangler Fig — bọc legacy đằng sau facade Clean
Step 2:   Introduce vòng Clean cho feature mới
Step 3:   Move feature mới sang Vertical Slice trong shell Clean
Step 4:   Extract bounded context khi mỗi cái có thể đứng riêng
```

---

## 17. Migration Path giữa các Mix

```
N-Layer thuần                                          Microservices Hexagon
       │                                                       ▲
       │                                                       │
       ▼                                                       │
N-Layer + Clean (monolith hiện đại)                            │
       │                                                       │
       ▼                                                       │
Clean Slice (Clean + Vertical Slice)                           │
       │                                                       │
       ▼                                                       │
Hexagonal Slice (thêm port/adapter project explicit)           │
       │                                                       │
       ▼                                                       │
Modular Monolith (bounded context trong một process)           │
       │                                                       │
       └──────────────────────────────────────────────────────►┘
                       (extract theo bounded context)
```

Mỗi bước là **refactor bounded**, không phải rewrite. Move theo thứ tự:

1. **Plain → Clean:** introduce project `Domain` và `Application`; move logic từ service vào entity.
2. **Clean → Clean Slice:** rearrange `Application` theo folder feature.
3. **Clean Slice → Hexagonal Slice:** extract assembly adapter; define port cạnh slice.
4. **Hexagonal Slice → Modular Monolith:** split thành module với boundary module enforce bởi `internal` và project reference.
5. **Modular Monolith → Microservices:** extract module thành service riêng chỉ khi boundary team và scale justify.

**Đừng skip step.** Đi thẳng từ N-Layer thuần sang microservices là mode fail đắt nhất trong ngành.

---

## 18. Khuyến nghị cuối cho .NET 2026

> **Stack khởi đầu mặc định cho app .NET mới:**
>
> 1. **Clean Architecture** như triết lý thiết kế.
> 2. **Folder feature Vertical Slice** bên trong `Application/Features/`.
> 3. **Deploy 3-tier** (UI / API / DB).
> 4. **MediatR + FluentValidation + Result\<T\>** cho use-case layer.
> 5. **EF Core** trong `Infrastructure/`, không bao giờ nơi khác.
> 6. **Architecture test** (NetArchTest / ArchUnitNET) trong CI để enforce Dependency Rule.
> 7. Thêm **project adapter Hexagonal** nếu app phải support nhiều driving channel (HTTP + CLI + worker + test).
> 8. **Microservices** *chỉ* khi bounded context và team boundary đòi hỏi.

Cho app nhỏ hoặc sống ngắn, **N-Layer 3-layer thuần trong 2 tier** là lựa chọn đúng. Đừng over-engineer.

> Năm kiến trúc không được xếp hạng. Chúng trả lời câu hỏi khác nhau. Hệ thống .NET tuyệt trả lời tất cả:
> - *Code tổ chức ra sao?* → N-Layer **hoặc** Vertical Slice
> - *Chạy ở đâu?* → N-Tier
> - *Dependency nào được phép?* → Clean **hoặc** Hexagonal

Chọn một từ mỗi hàng. Đó là stack của bạn.

---

## 19. Tham khảo

- [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md)
- [`n-tier-architecture-vi.md`](./n-tier-architecture-vi.md)
- [`clean-architecture-vi.md`](./clean-architecture-vi.md)
- [`vertical-slice-architecture-vi.md`](./vertical-slice-architecture-vi.md)
- [`hexagonal-architecture-vi.md`](./hexagonal-architecture-vi.md)
- [`architecture-comparison-vi.md`](./architecture-comparison-vi.md) — so sánh ba-bên trước đó
- Robert C. Martin — *Clean Architecture* (2017)
- Alistair Cockburn — [Hexagonal Architecture (Ports & Adapters)](https://alistair.cockburn.us/hexagonal-architecture)
- Jimmy Bogard — [Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
- Microsoft Learn — [N-tier architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- Microsoft Learn — [Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
