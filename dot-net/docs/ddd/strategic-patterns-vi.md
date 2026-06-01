# Strategic DDD Patterns

> Các pattern **cấp hệ thống** của Domain-Driven Design. Chúng trả lời: *boundary đặt ở đâu, vocabulary nào dùng, các context liên hệ ra sao?* Đây là nửa DDD trả công cho mình ngay cả khi bạn không adopt tactical pattern nào.

> 🇻🇳 Phiên bản tiếng Việt. English: [`strategic-patterns.md`](./strategic-patterns.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Nửa cấp hệ thống của DDD: **Ubiquitous Language** (một vocabulary per context), **Bounded Context** (boundary explicit cho một model), **Context Map** (diagram cách context relate), **Anti-Corruption Layer** (dịch khi model không fit), cộng 9 quan hệ context-mapping (Partnership, Shared Kernel, Customer/Supplier, Conformist, ACL, Open Host Service, Published Language, Separate Ways, Big Ball of Mud).
- **Tại sao** — Tactical pattern work bên trong *một* model. Moment bạn có hai team, nhiều service, hoặc legacy system, bạn có nhiều model sẽ conflict. Strategic design đặt tên boundary và quan hệ giữa chúng.
- **Khi nào** — Hệ thống multi-team; công ty với terminology share nghĩa khác cho department khác; integration với legacy hoặc third-party; quyết định boundary microservices.
- **Ở đâu** — Trên tactical pattern; inform cả *service nào* build (một per Bounded Context) và *cách* chúng integrate (Context Map). Pair với modeling microservices của Sam Newman.

---

## Tại sao strategic đến trước

Trong app nhỏ, bạn có thể làm tactical DDD (Aggregate, Value Object) mà không cần strategic. Nhưng khi hai team ship cùng nhau — hoặc một hệ thống vượt qua một model — strategic DDD thành phao cứu sinh:

* Hai team dùng từ **"Customer"** với nghĩa khác nhau → integration bug mãi mãi.
* Một model khổng lồ bao trùm cả công ty → không ai hiểu hết; thay đổi nhỏ gây vỡ ở chỗ xa.
* Không có anti-corruption layer ở biên legacy → idea legacy thấm vào hệ thống mới.

Strategic DDD làm boundary **rõ ràng, có tên, có chủ ý**.

---

## Index

1. [Ubiquitous Language](#1-ubiquitous-language)
2. [Bounded Context](#2-bounded-context)
3. [Context Map](#3-context-map)
4. [Anti-Corruption Layer (ACL)](#4-anti-corruption-layer-acl)
5. [Các quan hệ Context-Mapping khác](#5-các-quan-hệ-context-mapping-khác)
6. [Tổng hợp](#6-tổng-hợp)

---

## 1. Ubiquitous Language

**Ubiquitous Language** là vocabulary chung được **mọi người** làm việc trên một Bounded Context sử dụng — domain expert, developer, tester, PM.

* Trong conversation.
* Trong ticket và doc.
* Trong code: tên class, tên method, tên biến.
* Trong test: scenario description.

Nếu business gọi là "Policy" mà code gọi là `InsuranceContract`, mỗi conversation có ma sát phiên dịch. Mỗi lần dịch là cơ hội cho hiểu lầm.

### Thực tế trông như thế nào

| Domain expert nói…               | Code tệ                            | Code tốt                                  |
| -------------------------------- | --------------------------------- | ---------------------------------------- |
| "Underwrite a policy"            | `policies.Update(status = "OK")`  | `policy.Underwrite(underwriter, today)`  |
| "The claim is in adjudication"   | `claim.StatusCode = 3`            | `claim.IsInAdjudication`                 |
| "Settle the claim"               | `claim.Finalize()`                | `claim.Settle(amount, settledOn)`         |

Code trở thành **specification thực thi được** — đọc được bởi domain expert.

### Quy tắc

* **Một Ubiquitous Language cho mỗi Bounded Context.** "Customer" có thể có nghĩa khác trong Sales vs. Support — và *điều đó đúng*.
* **Duy trì glossary** cho mỗi context. Một README trong mỗi service là đủ.
* **Refactor tên không nương tay** khi vocabulary business thay đổi.

---

## 2. Bounded Context

**Bounded Context** là **boundary rõ ràng bên trong đó một model và ngôn ngữ cụ thể nhất quán**. Ngoài boundary, các từ đó có thể có nghĩa khác.

```
┌─────────────────────────┐   ┌─────────────────────────┐
│   Sales Context         │   │   Support Context        │
│                         │   │                          │
│   Customer              │   │   Customer               │
│     • Name              │   │     • Name               │
│     • LeadScore         │   │     • OpenTickets        │
│     • LastQuotedAt      │   │     • SatisfactionScore  │
└─────────────────────────┘   └─────────────────────────┘
```

Đây là **hai khái niệm khác nhau** chia sẻ một nhãn. Ép chúng vào một class `Customer` chung — xuyên các phòng ban — là lỗi enterprise kinh điển. Model không phục vụ ai tốt.

### Cách nhận diện Bounded Context

* **Boundary ngôn ngữ** — chỗ nào cùng một từ thay đổi nghĩa?
* **Boundary team** — mỗi team sở hữu cái gì từ đầu đến cuối?
* **Boundary business capability** — Sales, Billing, Shipping, Inventory là chia điển hình.
* **Boundary subdomain** — Core / Supporting / Generic (xem Evans).

### Biểu hiện trong .NET

Thường **một microservice = một Bounded Context**. Trong modular monolith, **một module = một Bounded Context**. Cả hai trường hợp:

```
SalesContext.csproj            SupportContext.csproj
 ├── Customer.cs                 ├── Customer.cs
 ├── Quote.cs                    ├── Ticket.cs
 └── ...                         └── ...
```

Hai class `Customer`, trong namespace/project khác nhau. **Đây đúng**, không phải lỗi cần dedupe.

---

## 3. Context Map

**Context Map** là sơ đồ các **Bounded Context trong hệ thống và cách chúng liên hệ**. Đây là artifact giá trị nhất trong strategic DDD.

```
        ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
        │   Sales      │────────▶│   Billing    │────────▶│   Shipping   │
        │  (Core)      │  U/D    │  (Supporting)│   U/D   │  (Supporting)│
        └──────────────┘         └──────────────┘         └──────────────┘
                ▲                                                ▲
                │ Customer/Supplier                              │
                │                                                │ ACL
        ┌──────────────┐                                ┌────────────────┐
        │   CRM        │                                │  Legacy WMS    │
        │ (Conformist) │                                │  (Big Ball of  │
        └──────────────┘                                │   Mud)         │
                                                       └────────────────┘
```

Mỗi mũi tên được **gắn nhãn** loại quan hệ. Không phải tuỳ chọn — *loại* quan hệ định hình thiết kế integration.

---

## 4. Anti-Corruption Layer (ACL)

Khi phải integrate với context mà model **không fit với của bạn** — legacy system, third-party API, partner mà domain thật sự khác — build một **Anti-Corruption Layer**.

```
┌────────────────────┐     ACL      ┌──────────────────────┐
│   Context của bạn  │  ◀─────────▶ │   Context của họ      │
│   Model sạch       │  dịch        │   Model lộn xộn/khác   │
└────────────────────┘              └──────────────────────┘
```

ACL **dịch** giữa hai model. Đây là bảo vệ một chiều: model lộn xộn **không được phép** vào code của bạn.

### C# Sample

```csharp
// Domain model sạch của bạn
public sealed record Address(string Street, string City, string PostalCode, string Country);

// Legacy SOAP API expose monstrosity này
public class LegacyAddressDto
{
    public string? AddressLine1 { get; set; }
    public string? AddressLine2 { get; set; } // đôi khi chứa city + postal trộn lẫn
    public string? CityField   { get; set; }
    public string? ZipPostCode { get; set; }
    public int     CountryCode { get; set; } // ISO code dạng số
}

// ACL — dịch giữa hai thế giới và cô lập sự lộn xộn
public interface IAddressProvider
{
    Task<Address> GetAsync(Guid customerId, CancellationToken ct);
}

public sealed class LegacyAddressAcl : IAddressProvider
{
    private readonly LegacyCustomerClient _legacy;
    private readonly ICountryCodeMapper _countries;

    public LegacyAddressAcl(LegacyCustomerClient legacy, ICountryCodeMapper countries)
    { _legacy = legacy; _countries = countries; }

    public async Task<Address> GetAsync(Guid customerId, CancellationToken ct)
    {
        var dto = await _legacy.GetCustomerAddressAsync(customerId.ToString(), ct);

        var street = string.Join(", ",
            new[] { dto.AddressLine1, dto.AddressLine2 }
                .Where(s => !string.IsNullOrWhiteSpace(s)));

        return new Address(
            Street:     street,
            City:       dto.CityField ?? "",
            PostalCode: dto.ZipPostCode ?? "",
            Country:    _countries.IsoCodeFromLegacy(dto.CountryCode));
    }
}
```

**Điểm chính:** phần còn lại của application dùng `Address` sạch. Sự lộn xộn chỉ sống **trong** `LegacyAddressAcl`. Nếu legacy system thay đổi, chỉ class này thay đổi.

ACL thường là một **service** trong microservices hoặc một **namespace/project chuyên dụng** trong monolith.

---

## 5. Các quan hệ Context-Mapping khác

Evans đặt tên cho nhiều quan hệ giữa các context. Biết chúng cho bạn vocabulary để quyết định integration.

| Quan hệ                          | Ý nghĩa                                                                                       |
| -------------------------------- | --------------------------------------------------------------------------------------------- |
| **Partnership**                  | Hai team thành công hoặc thất bại cùng nhau; hợp tác chặt trên model chung.                    |
| **Shared Kernel**                | Một subset nhỏ của model được share giữa các context. Chi phí điều phối — dùng sparingly.       |
| **Customer / Supplier**          | Upstream cung cấp; downstream consume. Downstream có tiếng nói trong ưu tiên của upstream.    |
| **Conformist**                   | Downstream chấp nhận model upstream nguyên xi, không dịch. Rẻ nhưng coupling bạn.              |
| **Anti-Corruption Layer (ACL)**  | Downstream dịch model upstream để tự bảo vệ. Dùng khi model thật sự khác.                       |
| **Open Host Service**            | Upstream cung cấp protocol/API có document cho nhiều consumer downstream.                      |
| **Published Language**           | Ngôn ngữ chung có document tốt dùng giữa các context (vd. event schema registry).               |
| **Separate Ways**                | Hai context không integrate chút nào. Đôi khi là lựa chọn đúng.                                 |
| **Big Ball of Mud**              | Context không có model rõ. Tốt nhất xử lý bằng ACL giữa bạn và nó.                              |

Cặp quan trọng nhất cho công việc mới: **Open Host Service + Published Language** cho upstream; **Customer/Supplier hoặc ACL** cho downstream.

---

## 6. Tổng hợp

Một strategic DDD thật cho nền tảng e-commerce:

```
   ┌────────────────┐ U/D  ┌────────────────┐ U/D  ┌────────────────┐
   │   Catalog      │─────▶│   Pricing      │─────▶│   Sales        │
   │   (Generic)    │      │   (Core)       │      │   (Core)       │
   └────────────────┘      └────────────────┘      └────────────────┘
                                                           │
                                                           │ Customer/Supplier
                                                           ▼
   ┌────────────────┐ ACL  ┌────────────────┐  U/D ┌────────────────┐
   │   Legacy SAP   │◀─────│   Billing      │─────▶│  Fulfillment    │
   │  (Big Ball of  │      │   (Supporting) │      │  (Supporting)   │
   │     Mud)       │      └────────────────┘      └────────────────┘
   └────────────────┘
```

Mỗi hộp:

* Có **Ubiquitous Language riêng**.
* Có **model riêng** — `Customer` trong Sales không cùng shape với trong Billing.
* **Deploy độc lập** (microservice) hoặc **modify độc lập** (monolith module).
* **Giao tiếp qua contract rõ ràng** — event, API call, ACL.

Các context **Core** (Pricing, Sales) được đầu tư tactical-DDD mạnh. **Generic** (Catalog — gần như off-the-shelf) và **Supporting** (Billing, Fulfillment — quan trọng nhưng không khác biệt) chỉ cần modeling đủ.

---

## Anti-Pattern thường gặp

* **Model "common" chung** — một class `User`, `Customer`, `Product` dùng khắp công ty. Couple mọi team với mọi team khác.
* **"Enterprise Data Model" tập trung** — cùng vấn đề ở tầng data.
* **Boundary ngầm** — boundary "tồn tại trong đầu chúng tôi". Chúng không tồn tại.
* **Không anti-corruption khi integrate với legacy** — idea legacy bò vào code mới và bạn thừa kế mớ hỗn độn.
* **Microservices không có Bounded Context** — bạn nhận chi phí vận hành phân tán mà không có lợi ích thiết kế.

---

## TL;DR

* **Ubiquitous Language** — cùng từ, cùng code, cùng conversation.
* **Bounded Context** — boundary rõ ràng cho một model.
* **Context Map** — bản vẽ của tất cả context và cách chúng liên hệ.
* **Anti-Corruption Layer** — dịch giữa các context không fit với nhau.

Bạn có thể lấy 80% giá trị của DDD chỉ từ 4 idea này, kể cả khi không viết một Aggregate nào.

---

## Tham khảo

* Eric Evans — *Domain-Driven Design* (2003). Part IV (nửa strategic — thường bị bỏ qua, giá trị nhất).
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Chương 2–4.
* Vlad Khononov — *Learning Domain-Driven Design* (2021). Cách tiếp cận hiện đại, dễ tiếp cận.
* DDD Crew — [github.com/ddd-crew](https://github.com/ddd-crew) — template Context Map, hướng dẫn EventStorming.
