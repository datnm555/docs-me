# Building Microservices — Tóm tắt

> **Sam Newman** — *Building Microservices: Designing Fine-Grained Systems*, 2nd edition, O'Reilly, 2021. Edition 1 năm 2015.

> **Tại sao sách này quan trọng:** definition microservices được trích dẫn nhiều nhất trong ngành đến từ sách này. Edition 2 là rewrite lớn incorporate ~6 năm lesson post-hype — khi *không* dùng microservices, phức tạp vận hành, vấn đề ownership data, testing pyramid, và vai trò của bounded context như tool slice đúng. Sách có opinion, thực hành, và tránh hype.

> **Tóm tắt này cover:** definition của Newman, nguyên lý, modeling quanh bounded context, splitting monolith, communication style, saga vs. orchestration, deployment, testing, observability, security, resilience, scaling, và thực tế tổ chức (Conway's Law). Code example là của tôi, illustrative.

> 🇻🇳 Phiên bản tiếng Việt. English: [`building-microservices-sam-newman.md`](./building-microservices-sam-newman.md)

---

## Mục lục

1. [Về cuốn sách](#1-về-cuốn-sách)
2. [Định nghĩa của Newman](#2-định-nghĩa-của-newman)
3. [Nguyên lý chính](#3-nguyên-lý-chính)
4. [Nhóm Chương 1 — Foundation](#4-nhóm-chương-1--foundation)
5. [Nhóm Chương 2 — Modeling và Splitting](#5-nhóm-chương-2--modeling-và-splitting)
6. [Nhóm Chương 3 — Communication](#6-nhóm-chương-3--communication)
7. [Nhóm Chương 4 — Workflow: Saga, Orchestration vs. Choreography](#7-nhóm-chương-4--workflow-saga-orchestration-vs-choreography)
8. [Nhóm Chương 5 — Build & Deployment](#8-nhóm-chương-5--build--deployment)
9. [Nhóm Chương 6 — Testing](#9-nhóm-chương-6--testing)
10. [Nhóm Chương 7 — Observability](#10-nhóm-chương-7--observability)
11. [Nhóm Chương 8 — Security](#11-nhóm-chương-8--security)
12. [Nhóm Chương 9 — Resilience](#12-nhóm-chương-9--resilience)
13. [Nhóm Chương 10 — Scaling](#13-nhóm-chương-10--scaling)
14. [Nhóm Chương 11 — User Interface](#14-nhóm-chương-11--user-interface)
15. [Nhóm Chương 12 — Organization & Conway's Law](#15-nhóm-chương-12--organization--conways-law)
16. [Câu hỏi khó — Có nên dùng Microservices?](#16-câu-hỏi-khó--có-nên-dùng-microservices)
17. [Take-Away bức tranh lớn](#17-take-away-bức-tranh-lớn)
18. [Sách này fit Repo ra sao](#18-sách-này-fit-repo-ra-sao)
19. [Khuyến nghị thứ tự đọc](#19-khuyến-nghị-thứ-tự-đọc)
20. [Hiểu lầm phổ biến](#20-hiểu-lầm-phổ-biến)
21. [Tham khảo](#21-tham-khảo)

---

## 1. Về cuốn sách

* **Tiêu đề:** *Building Microservices: Designing Fine-Grained Systems*
* **Tác giả:** Sam Newman (consultant độc lập, ex-ThoughtWorks)
* **NXB:** O'Reilly Media
* **Edition:** 1st (2015), **2nd (2021, version canonical đọc hôm nay)**
* **Độ dài:** ~600 trang
* **Audience:** architect, tech lead, senior engineer đã biết build monolith và quyết định liệu (hoặc cách) split một.

Edition 2 sober hơn nhiều so với edition 1. Sách 2015 publish trong đỉnh hype microservices; edition 2021 thêm chương substantial về khi **không** dùng microservices, và advice xuyên suốt là: *khởi đầu với modular monolith; extract microservices khi pain thật.*

---

## 2. Định nghĩa của Newman

> **Microservices là service deployable độc lập modeled quanh business domain. Chúng giao tiếp với nhau qua mạng, và như architecture choice offer nhiều option để giải vấn đề bạn có thể face.**

Ba cụm từ chịu lực:

1. **Deployable độc lập** — property không thương lượng duy nhất. Nếu không thể deploy service mà không coordinate với team khác, bạn có *distributed monolith*, không phải microservices.
2. **Modeled quanh business domain** — service align với bounded context (DDD). Không phải "auth service" — "customer onboarding service".
3. **Giao tiếp qua mạng** — vì thế, mọi hệ thống microservices là hệ thống phân tán, và vấn đề distributed-system (partial failure, latency, consistency) giờ là vấn đề của bạn.

Microservices **không phải**:
* Không phải "service nhỏ" — kích thước là hệ quả, không phải mục tiêu.
* Không phải "stateless function" — service thường own state; chỉ không share.
* Không phải "mỗi team chọn ngôn ngữ riêng" — heterogeneity có cost thật.
* Không phải "giống SOA" — share lineage, nhưng microservices thêm independent deployability như hard requirement.

---

## 3. Nguyên lý chính

Newman list các nguyên lý recurring trong sách. Chúng reinforce nhau:

| Nguyên lý                                       | Nghĩa                                                                         |
| ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **Modeled quanh business domain**               | Slice service theo bounded context, không technical layer.                    |
| **Văn hoá automation**                          | Nếu không thể fully automate build, test, và deploy, microservices sẽ crush bạn. |
| **Ẩn detail implementation internal**           | Mỗi service own data của nó. **Không database share.** Period.                |
| **Decentralize mọi thứ**                        | Quyết định thuộc về team own service — ngôn ngữ, datastore, cadence deploy.   |
| **Deploy độc lập**                              | Đây là rule định nghĩa microservices.                                          |
| **Consumer trước**                              | Nhu cầu consumer drive API. Backwards compatibility là job architect.         |
| **Cô lập failure**                              | Bulkhead, circuit breaker, timeout, graceful degradation.                     |
| **Observable cao**                              | Log, metric, distributed trace, correlation ID mọi nơi.                       |

Ba cái này thường bị vi phạm bởi team *nghĩ* họ có microservices:

* **Ẩn detail implementation internal.** Database share **là** leak của implementation. Nếu hai service join table, chúng không phải service riêng.
* **Deploy độc lập.** Nếu release process của bạn nói "deploy A, rồi B, rồi C, theo thứ tự đó, sau khi coordinate với team D," bạn không có microservices.
* **Decentralize mọi thứ.** Team trung tâm own "messaging system" hoặc "database access library" thành bottleneck synchronous qua mọi team.

---

## 4. Nhóm Chương 1 — Foundation

### Microservices mua gì cho bạn

Lợi ích thật, khi dùng appropriately:

* **Deployable độc lập** — release nhỏ, targeted giảm blast radius.
* **Scale độc lập** — scale bottleneck service ngang mà không scale toàn monolith.
* **Heterogeneity technology** — pick tool đúng per service (search service trong Rust, checkout trong C#, notification trong Node).
* **Robustness** — partial failure, handle tốt, giảm scope outage tổng.
* **Autonomy team** — Conway's Law: service nhỏ align với team nhỏ.
* **Reuse qua composition** — capability generic (notification, auth, audit) thành platform service.

### Microservices cost gì cho bạn

Cost thật, thường bị underestimate:

* **Distributed system khó.** Latency, partial failure, eventual consistency, retry, idempotency — vấn đề của bạn giờ.
* **Maturity vận hành bắt buộc.** CI/CD, observability, secret management, mTLS, service mesh — required, không tuỳ chọn.
* **Local development đau đớn** khi 12 service phải chạy cùng để test một feature.
* **Refactor cross-service** 10× khó hơn refactor cross-module trong monolith.
* **End-to-end testing** khó, chậm, flaky.

> Newman explicit: **nếu bạn có thể build modular monolith well, làm cái đó trước.** Microservices giải vấn đề scaling/team/deployment, không phải vấn đề modeling.

---

## 5. Nhóm Chương 2 — Modeling và Splitting

### Modeling quanh bounded context

Newman là DDD advocate. Quyết định architectural quan trọng nhất là **vẽ đường giữa service ở đâu**. Câu trả lời của sách: **bounded context** (Evans).

Bounded context là phần của domain nơi từ có một nghĩa. *"Customer"* trong Sales không phải *"Customer"* trong Support; chúng có attribute khác và lifecycle khác. Mỗi cái thành bounded context riêng — và mỗi cái nhận service riêng.

### Hai axis coupling

Trong và giữa bounded context, hai property quan trọng:

* **Coupling** — mức độ hai service phụ thuộc nhau. Coupling thấp tốt.
* **Cohesion** — bao nhiêu behavior liên quan sống cùng nhau bên trong một service. Cohesion cao tốt.

Microservices vi phạm chúng — high coupling, low cohesion — là *distributed monolith* và kết hợp cái tệ nhất cả hai thế giới.

### Cấp coupling (ít tệ tới tệ nhất)

Newman rank các form coupling:

| Form                       | Là gì                                                                      | Verdict           |
| -------------------------- | -------------------------------------------------------------------------- | ----------------- |
| **Domain coupling**        | Một service dùng functionality của khác (nhu cầu cross-context hợp lệ)     | Chấp nhận được    |
| **Pass-through coupling**  | Service pass data sang khác, không transform                               | Smell             |
| **Common coupling**         | Service share state (cùng DB table, cùng global resource)                 | Tệ                |
| **Content coupling**       | Một service reach vào internal của khác                                    | Tệ nhất           |

Fix khi coupling thành content coupling: vẽ lại bounded context.

### Splitting monolith

Cả chương về *cách* split. Bài học:

* **Đừng split big-bang.** Dùng pattern **Strangler Fig** — wrap legacy với facade, peel một bounded context một lần.
* **Database trước, service sau.** Database thường là phần khó nhất. Sách outline pattern: shared DB → schema per service → database per service.
* **Parallel run** trước cutover — chạy cũ và mới parallel, so output.
* **Pattern Branch by Abstraction** cho phép introduce implementation mới an toàn đằng sau interface.
* **Đừng refactor vì refactor.** Split khi có driver thật: scaling, ownership, deploy độc lập, security boundary.

```csharp
// Strangler Fig — facade route giữa legacy và new
public sealed class CheckoutFacade : ICheckoutService
{
    private readonly ILegacyCheckout _legacy;
    private readonly INewCheckoutService _new;
    private readonly IFeatureFlag _flags;

    public CheckoutFacade(ILegacyCheckout legacy, INewCheckoutService @new, IFeatureFlag flags)
    {
        _legacy = legacy; _new = @new; _flags = flags;
    }

    public Task<CheckoutResult> Checkout(CheckoutRequest req, CancellationToken ct) =>
        _flags.IsEnabled("checkout-v2", req.CustomerId)
            ? _new.Checkout(req, ct)
            : _legacy.Checkout(req, ct);
}
```

---

## 6. Nhóm Chương 3 — Communication

### Hai style (per chương)

Newman tổ chức communication inter-service theo hai axis — **synchronous/asynchronous** và **request-response/event-driven**:

|                | Request-response                        | Event-driven                                     |
| -------------- | --------------------------------------- | ------------------------------------------------ |
| **Synchronous** | REST, gRPC                              | (hiếm — call-and-wait-for-event lạ)             |
| **Asynchronous** | Message-queue request/reply             | Pub/sub event (form phổ biến nhất)               |

### Khi nào dùng cái nào

| Style                       | Dùng khi                                                         | Tránh khi                                                          |
| --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Synchronous request-response** (REST/gRPC) | Read query đơn giản; flow user-facing cần feedback ngay. | Chuỗi 5+ service; fail của service nào phá chuỗi.                 |
| **Asynchronous event-driven** | Decouple producer khỏi consumer; workflow chạy dài.            | Khi caller thực sự cần result synchronous.                       |

### REST vs. gRPC (và khác)

Sách cho ranking thực dụng của Newman — không phải "cái nào best", mà "cái nào appropriate":

* **REST/JSON** — default cho API external và phần lớn call internal. Tooling universal, debuggable.
* **gRPC** — contract strongly-typed, protocol binary, streaming. Tốt cho service-to-service internal.
* **GraphQL** — BFF aggregation; không phải transport per se.
* **SOAP** — legacy, vẫn thấy trong enterprise.

### Backwards compatibility

Nguồn bug lớn nhất trong hệ thống microservices là **break consumer**. Rule của Newman:

1. **Liberal trong cái accept, strict trong cái emit** (Postel's Law).
2. **Thêm field, đừng đổi.** Remove hoặc rename field break consumer silent.
3. **Version explicit** — `/v1/orders`, `/v2/orders` — hoặc dùng header. Đừng break caller không có deprecation period.
4. **Consumer-driven contract test** (CDC) catch break *trước khi* deploy. Sách strongly recommend Pact hoặc tương tự.

---

## 7. Nhóm Chương 4 — Workflow: Saga, Orchestration vs. Choreography

Cả chương về workflow business cross-service. Thách thức cơ bản: **trong microservices, không thể dùng distributed ACID transaction.** 2PC qua service operationally fragile.

### Saga

> Workflow business chạy dài model như **chuỗi local transaction**, mỗi cái trong một service, với **compensating transaction** để undo work khi cái gì downstream fail.

Hai flavor:

| Style              | Cách work                                                                          | Lợi                                              | Hại                                                |
| ------------------ | ---------------------------------------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------- |
| **Orchestration**  | Coordinator trung tâm ("saga orchestrator") command mỗi service theo trình tự.     | View workflow tập trung; logic explicit.         | Orchestrator có thể thành god service.            |
| **Choreography**   | Service emit event; service khác subscribe và act. Không có brain trung tâm.       | Decoupled; resilient; scale tốt về tổ chức.      | Workflow implicit, scattered qua service; debug khó. |

Newman không prescribe winner. Anh ấy nói:
* Dùng **choreography** cho workflow đơn giản, ổn định, mostly-linear.
* Dùng **orchestration** cho workflow phức tạp với branch, retry, và step human-in-the-loop.

> Cross-reference: [`../dot-net/docs/architectural-pattern/saga-vi.md`](../dot-net/docs/architectural-pattern/saga-vi.md) — deep-dive .NET-specific (với code MassTransit).

### Compensating action là action *business*

Compensation không phải database `ROLLBACK`. Là **reversal mức business** — refund, cancel, recall. Nhiều operation thế giới thực *không thể* thực sự undone (không thể un-charge credit card, có thể refund). Thiết kế compensation ép bạn nghĩ về *failure nghĩa là gì với business*.

### Idempotency bắt buộc

Trong thế giới at-least-once delivery, mọi command phải **idempotent** — nhận lại cùng command produce cùng result, không side-effect đúp. Sách pair saga với **Outbox / Inbox pattern** cho delivery reliable.

> Cross-reference: [`../dot-net/docs/architectural-pattern/outbox-vi.md`](../dot-net/docs/architectural-pattern/outbox-vi.md).

---

## 8. Nhóm Chương 5 — Build & Deployment

### Một repo per service vs. monorepo

Newman famously even-handed:

* **Repo per service** — boundary ownership sạch, CI đơn giản, không noise refactor cross-service. Default cho phần lớn team.
* **Monorepo** — refactor cross-service dễ hơn, commit atomic qua service, khó enforce độc lập service. Work tốt ở công ty rất lớn với tooling mạnh (Google, Meta).

Lựa chọn quan trọng ít hơn *discipline*. Monorepo với discipline tệ produce service couple chặt phải deploy cùng — distributed monolith với extra steps.

### CI/CD

Bắt buộc. Bar của Newman:

* Mỗi commit trên `main` trigger build.
* Build artifact là **the** artifact — cùng artifact deploy staging và prod.
* Test tự động chạy mỗi commit.
* Deploy production push tự động (hoặc với promotion single-click).
* Mỗi service có pipeline riêng.

### Tuỳ chọn deployment

Sách sweep landscape — physical host, VM, container, Kubernetes, serverless — và land trên:

> Default **container trên Kubernetes** cho service stateful hoặc chạy dài; cân nhắc **serverless** (Lambda/Function) cho workload event-driven, bursty.

### Pattern deployment

* **Blue/green** — hai environment; cut over instant.
* **Canary** — gradually shift traffic sang version mới.
* **Feature toggle** — deploy code dark, enable per-user.
* **Parallel run** — khi stake cao (financial, regulatory), chạy new và old parallel và compare.

---

## 9. Nhóm Chương 6 — Testing

Sách update testing pyramid của Mike Cohn cho microservices:

```
                      /\
                     /E2E\          (ít, chậm, flaky)
                    /------\
                   /Contract\        (consumer-driven contract test)
                  /----------\
                 /Integration \      (một service + DB / broker thật)
                /--------------\
               /     Unit       \    (nhiều, nhanh, isolated)
              /__________________\
```

### Bias của Newman

* **Unit test** nên là majority. Nhanh, isolated, không infrastructure.
* **Integration test** hit DB thật qua full stack của service (dùng Testcontainers) catch bug thật không có flakiness của E2E.
* **Contract test** (Pact-style) thay thế phần lớn test integration cross-service. Chạy trong CI của mỗi service; không ai cần spin up service khác.
* **End-to-end test** chậm, flaky, đắt. Minimize chúng — giữ vài scenario critical-path, chấp nhận chúng break thường xuyên.

### Tranh luận "no end-to-end testing"

Newman tóm tắt position controversial của một số team: skip E2E entirely. Compromise của sách:

* Bạn có lẽ không thể skip E2E entirely.
* Nhưng mỗi test E2E là tax. Thêm chỉ khi scenario cụ thể không thể cover bằng contract + integration test.
* Chạy E2E trong pipeline riêng, không block deploy.

> Cross-reference: [`unit-testing-vladimir-khorikov-vi.md`](./unit-testing-vladimir-khorikov-vi.md) — cho "good" unit test trông như thế nào trước khi viết hàng trăm.

---

## 10. Nhóm Chương 7 — Observability

Trong hệ thống phân tán, bạn không thể reproduce phần lớn bug locally. Cách duy nhất hiểu hệ thống trong production là **observing** nó. Ba trụ cột:

| Trụ cột              | Trả lời gì                                                                | Tool example                              |
| -------------------- | ------------------------------------------------------------------------- | ------------------------------------------ |
| **Log**              | Chuyện gì xảy ra? (context-rich, per-request)                            | Serilog + Loki/Elasticsearch                |
| **Metric**           | Hệ thống perform ra sao? (time-series, aggregated)                       | Prometheus, Datadog, App Insights           |
| **Distributed trace** | Request đi đâu? (một trace per request, span service)                   | OpenTelemetry → Jaeger / Tempo / Honeycomb |

### Correlation ID và trace propagation

Mỗi request inbound nhận **correlation ID** propagate qua mọi downstream call (HTTP header, message header). Log và trace tất cả carry nó. Khi cái gì fail, grep ID và xem toàn flow request.

```csharp
// Middleware inbound — set hoặc read correlation ID
app.Use(async (ctx, next) =>
{
    var correlationId = ctx.Request.Headers["X-Correlation-ID"].FirstOrDefault()
                        ?? Guid.NewGuid().ToString();
    ctx.Response.Headers["X-Correlation-ID"] = correlationId;
    using (LogContext.PushProperty("CorrelationId", correlationId))
        await next();
});
```

### Advice observability của Newman

* **Khởi đầu với structured logging + correlation ID.** Dễ nhất, payoff ngay lớn nhất.
* **Thêm OpenTelemetry distributed tracing kế.** Một khi có một trace per request, debug cross-service stop being guesswork.
* **Metric cho SLO / SLI.** Pick vài key indicator (latency P99, error rate, saturation) và alert chúng.
* **Sample, đừng log mọi thứ.** Ở 10,000 request/sec, full logging infeasible.

---

## 11. Nhóm Chương 8 — Security

### Threat model shift

Microservices có **far more network surface area** so với monolith. Sách sweep landscape:

* **Authentication** — caller là ai? (OAuth 2 / OIDC, mutual TLS, signed JWT)
* **Authorization** — caller được phép gì? (Centralized vs. decentralized; service-to-service vs. user-on-behalf-of)
* **Confidentiality** — TLS in transit, encryption at rest, secret management (Key Vault, AWS Secrets Manager).
* **Integrity** — signed message để service downstream trust payload.

### Zero Trust

Newman endorse **Zero Trust** — đừng trust mạng. Mọi call service-to-service authenticate và authorize, kể cả trong cluster. mTLS qua service mesh (Istio, Linkerd) làm cái này transparent.

### Đừng roll crypto của riêng

Sách emphatic: dùng library well-supported, không bao giờ encryption tự build. Identity provider (Keycloak, Auth0, Entra ID) handle phần khó; bạn wire chúng.

---

## 12. Nhóm Chương 9 — Resilience

> Distributed system là cái mà fail của một computer bạn không biết tồn tại có thể render computer của bạn unusable.
> — Leslie Lamport (quoted approvingly)

### Failure mode

Trong monolith, thứ fail; trong microservices, thứ fail **partial**. Bạn phải explicit design cho:

* **Timeout** — mọi network call có timeout, period. Setting default (Java's "no timeout") chết người.
* **Retry với exponential backoff và jitter** — không bao giờ retry chặt; sẽ gây thundering herd.
* **Circuit breaker** — khi downstream unhealthy, stop call; check periodically; resume khi healthy.
* **Bulkhead** — thread pool hoặc connection pool riêng để một downstream chậm không starve cái khác.
* **Fallback / graceful degradation** — nếu recommendation service down, ẩn widget; đừng fail page.
* **Idempotency** — mọi command phải safe nhận hai lần.
* **Dead-letter queue** — khi message không xử lý được sau N attempt, park nó cho human inspection.

```csharp
// Polly resilience pipeline (.NET)
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(30),
    })
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();
```

### Chaos engineering

Newman recommend — một khi hệ thống mature — **deliberate inject failure** trong production. Netflix's Chaos Monkey là ví dụ canonical. Tool: Gremlin, Chaos Mesh, AWS FIS.

---

## 13. Nhóm Chương 10 — Scaling

### Bốn axis scaling (AKF Scale Cube, attributed AKF Partners)

| Axis              | Chiến lược                                                     | Ví dụ                                            |
| ----------------- | -------------------------------------------------------------- | ------------------------------------------------ |
| **X — Horizontal duplication** | Nhiều instance của cùng service đằng sau load balancer | Chạy 5 copy của API; round-robin traffic         |
| **Y — Functional split**       | Split theo business capability                          | Extract billing khỏi monolith                     |
| **Z — Data partitioning (sharding)** | Split data theo attribute                         | Customer A–M shard 1, N–Z shard 2                |

Hệ thống thật kết hợp cả ba. Microservices cho bạn Y (functional split) miễn phí; X và Z vẫn là tool bạn chọn per service.

### Caching như tool scaling

* **Client-side caching** — browser, mobile app. HTTP caching header.
* **CDN** — edge caching cho content static và semi-static.
* **Server-side cache** (Redis, Memcached) — hot read serve từ memory.
* **Request collapsing** — coalesce duplicate concurrent request ở cache layer.
* **Cache invalidation** — Newman quote Phil Karlton: *"Chỉ có hai thứ khó trong Computer Science: cache invalidation và naming things."* Treat invalidation như feature, không phải afterthought.

---

## 14. Nhóm Chương 11 — User Interface

Sách push back idea UI là monolith đơn trên microservices.

### Backends for Frontends (BFF)

**BFF** là layer mỏng per loại client (web, iOS, Android) aggregate và shape data từ nhiều backend service cho client cụ thể đó. Team web own BFF của họ; team mobile own BFF của họ.

Tại sao: API đơn try phục vụ mọi client kết thúc phục vụ không ai tốt. Mobile muốn ít round-trip và payload nhỏ; web có thể handle chattiness nhiều hơn.

### Micro-frontend

Newman cautious. **Micro-frontend** (split UI theo team, mỗi cái own vertical slice bao gồm UI) work khi team đủ lớn và discipline cao. Cho team nhỏ, overhead vận hành vượt benefit.

---

## 15. Nhóm Chương 12 — Organization & Conway's Law

> **Conway's Law:** *Bất kỳ tổ chức nào thiết kế hệ thống sẽ produce design mà cấu trúc là copy của cấu trúc giao tiếp tổ chức.*

Maxim được trích dẫn nhiều nhất của sách. Implication:

* Nếu team tổ chức theo technical layer (UI team, backend team, DBA team), bạn sẽ produce layered monolith.
* Nếu team tổ chức theo business capability (Orders team, Billing team, Shipping team), bạn sẽ produce service-oriented architecture *dù bạn plan hay không*.
* **Inverse Conway maneuver** — restructure team *trước* sang shape bạn muốn hệ thống lấy.

### Team hai-pizza

Heuristic Bezos: team đúng size nếu hai pizza có thể feed nó (~6–8 người). Newman endorse team nhỏ, autonomous, cross-functional own service end-to-end (build, deploy, operate, on-call).

### Evolutionary Architect

Chương cuối. View của Newman về role architect:

* **Vision** — articulate technical direction.
* **Empathy** — hiểu impact quyết định lên team làm việc.
* **Collaboration** — architect không phải lone genius; họ là teammate.
* **Adaptability** — architecture phải evolve. Từ chối revisit quyết định là tội.
* **Autonomy** — cho team càng nhiều freedom càng tốt; intervene chỉ khi quan trọng.
* **Govern** — set rule of the road (security, observability, deploy standard), rồi tránh đi.

---

## 16. Câu hỏi khó — Có nên dùng Microservices?

Edition 2 thêm chương specifically về cái này. Flowchart của Newman, distilled:

* **Bạn ở greenfield với team nhỏ và domain unclear?** → **Build modular monolith.** Microservices first là cách đắt nhất discover boundary domain.
* **Bạn có vấn đề team-size hoặc deploy-frequency với monolith hiện có?** → Cân nhắc extract bounded context. Khởi đầu với context *độc lập nhất*.
* **Bạn có vấn đề scaling?** → Try vertical scaling, caching, read replica trước. Microservices là tool *cuối*, không phải đầu.
* **Bạn có nhiều team đụng nhau?** → Microservices giúp, nhưng modularization cũng. Try module trước.
* **Bạn có maturity vận hành (CI/CD, observability, container platform, on-call)?** → Nếu không, microservices sẽ đau hơn giúp.

**Bài học thực hành lớn nhất của sách:** path đúng cho *phần lớn* tổ chức là **modular monolith → extract microservices selectively**. Đi microservices-first vẫn là, trong 2021, cách reliable nhất ship distributed disaster.

---

## 17. Take-Away bức tranh lớn

1. **Deployable độc lập là property định nghĩa.** Nếu không thể, bạn không có microservices.
2. **Model service quanh bounded context**, không technical layer.
3. **Mỗi service own data của nó.** Không database share. Period.
4. **Dùng saga, không distributed transaction.** Pair chúng với idempotency và Outbox.
5. **Backwards compatibility là job architect.** Thêm field, không bao giờ break consumer.
6. **Testing pyramid vẫn áp dụng** — unit + integration + contract + chút E2E.
7. **Observability bắt buộc.** Log + metric + distributed trace từ ngày một.
8. **Resilience là property design.** Timeout, retry, circuit breaker, bulkhead — không phải afterthought.
9. **Conway's Law là thật.** Team structure shape hệ thống. Plan tổ chức như plan architecture.
10. **Microservices là tradeoff.** Đừng adopt cho resume; adopt khi pain *không* có chúng vượt cost vận hành.

---

## 18. Sách này fit Repo ra sao

| Topic trong sách                        | Doc repo                                                                                            |
| --------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Bounded Context (modeling)              | [`../dot-net/docs/ddd/strategic-patterns-vi.md`](../dot-net/docs/ddd/strategic-patterns-vi.md)      |
| Saga (workflow)                         | [`../dot-net/docs/architectural-pattern/saga-vi.md`](../dot-net/docs/architectural-pattern/saga-vi.md) |
| Outbox / Inbox (messaging reliable)     | [`../dot-net/docs/architectural-pattern/outbox-vi.md`](../dot-net/docs/architectural-pattern/outbox-vi.md) |
| CQRS (split read/write)                 | [`../dot-net/docs/architectural-pattern/cqrs-vi.md`](../dot-net/docs/architectural-pattern/cqrs-vi.md) |
| Event Sourcing                           | [`../dot-net/docs/architectural-pattern/event-sourcing-vi.md`](../dot-net/docs/architectural-pattern/event-sourcing-vi.md) |
| Event-Driven Architecture (style)       | [`../dot-net/docs/architectural-style/event-driven-vi.md`](../dot-net/docs/architectural-style/event-driven-vi.md) |
| Chiến lược Testing                       | [`unit-testing-vladimir-khorikov-vi.md`](./unit-testing-vladimir-khorikov-vi.md)                    |

---

## 19. Khuyến nghị thứ tự đọc

* **Lần đầu:** khởi đầu với chương 1–3 (foundation + modeling). Rồi nhảy tới bất kỳ pain bạn có hiện tại.
* **Đã chạy microservices:** chương về workflow, resilience, observability, security pay off ngay.
* **Đang cân nhắc microservices:** đọc Chương 1 (definition + benefit/cost) và chương "Should you?" (flowchart Newman) — chỉ chúng thường save tổ chức khỏi adoption sai.

---

## 20. Hiểu lầm phổ biến

| Hiểu lầm                                                            | Thực tế                                                                                                |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| "Microservices là service nhỏ."                                      | Size là hệ quả của modeling bounded context, không phải mục tiêu. Một số service lớn hơn cái khác.    |
| "Microservices fix mọi vấn đề monolith."                            | Microservices thêm vấn đề riêng. Hệ thống module hoá kém thành distributed mess.                       |
| "Mỗi team pick ngôn ngữ riêng."                                      | Cho phép, nhưng đắt — bạn nhân tooling, observability, hiring, và concern runtime.                    |
| "Hai service có thể share database để giữ đơn giản."                 | Không thể. Đó không phải hai service; là một.                                                          |
| "Eventually-consistent nghĩa là mất data."                          | Không — nghĩa là chấp nhận window ngắn cache lag source of truth. Done right, không mất data.          |
| "Microservices = REST."                                             | Microservices là lựa chọn cấu trúc; REST là một protocol. gRPC, event, và async messaging cũng valid.  |
| "Distributed transaction / 2PC fix consistency."                    | 2PC operationally fragile và hiếm khi work qua store heterogeneous. Dùng saga.                        |

---

## 21. Tham khảo

* Sam Newman — *Building Microservices: Designing Fine-Grained Systems* (2nd ed.), O'Reilly, 2021.
* Sam Newman — *Monolith to Microservices*, O'Reilly, 2019 (volume companion về *cách* extract).
* Eric Evans — *Domain-Driven Design*, 2003 (concept bounded-context Newman build trên).
* Chris Richardson — *Microservices Patterns*, Manning, 2018 (catalog pattern có tên bao gồm Saga, Outbox, API Gateway, Strangler Fig).
* Martin Fowler — [microservices.io](https://microservices.io/) và [martinfowler.com/microservices](https://martinfowler.com/microservices/).
* Cross-reference trong repo này: [`../dot-net/docs/architectural-pattern/`](../dot-net/docs/architectural-pattern/), [`../dot-net/docs/architectural-style/event-driven-vi.md`](../dot-net/docs/architectural-style/event-driven-vi.md), [`../dot-net/docs/ddd/strategic-patterns-vi.md`](../dot-net/docs/ddd/strategic-patterns-vi.md).
