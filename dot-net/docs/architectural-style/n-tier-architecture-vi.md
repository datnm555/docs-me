# N-Tier Architecture — Giải thích, Sample, Best Approach & Key Point

> Hướng dẫn về **kiến trúc deploy vật lý** trong .NET — cách application tách qua process, máy, và boundary mạng.

> **Companion doc:**
> * [`n-layer-architecture-vi.md`](./n-layer-architecture-vi.md) — tổ chức code **logic**.
> * [`clean-architecture-vi.md`](./clean-architecture-vi.md) — thiết kế layered driven bởi nguyên lý.

> 🇻🇳 Phiên bản tiếng Việt. English: [`n-tier-architecture.md`](./n-tier-architecture.md)

---

## Mục lục

1. [N-Tier Architecture là gì?](#1-n-tier-architecture-là-gì)
2. [Tier vs. Layer — Phân biệt quan trọng](#2-tier-vs-layer--phân-biệt-quan-trọng)
3. [Topology Tier phổ biến](#3-topology-tier-phổ-biến)
4. [Tại sao chọn N-Tier?](#4-tại-sao-chọn-n-tier)
5. [Trade-off & Chi phí](#5-trade-off--chi-phí)
6. [Anatomy của App 3-Tier .NET kinh điển](#6-anatomy-của-app-3-tier-net-kinh-điển)
7. [Sample — 3-Tier end-to-end trong .NET](#7-sample--3-tier-end-to-end-trong-net)
8. [Giao tiếp giữa các Tier](#8-giao-tiếp-giữa-các-tier)
9. [Ví dụ Topology Deploy](#9-ví-dụ-topology-deploy)
10. [Scale mỗi Tier độc lập](#10-scale-mỗi-tier-độc-lập)
11. [Security: Network Segmentation, DMZ, Zero Trust](#11-security-network-segmentation-dmz-zero-trust)
12. [Reliability: Health, Resilience, Failure Mode](#12-reliability-health-resilience-failure-mode)
13. [Observability xuyên Tier](#13-observability-xuyên-tier)
14. [Best Structure & Approach](#14-best-structure--approach)
15. [Anti-Pattern](#15-anti-pattern)
16. [Mapping Cloud — Azure & AWS](#16-mapping-cloud--azure--aws)
17. [N-Tier vs. Microservices](#17-n-tier-vs-microservices)
18. [Key Point — Danh sách ngắn](#18-key-point--danh-sách-ngắn)
19. [Checklist](#19-checklist)
20. [Tham khảo](#20-tham-khảo)

---

## 1. N-Tier Architecture là gì?

**N-Tier architecture** là cách tách application qua **N boundary vật lý (hoặc process)** — máy riêng, container, VM, hoặc process — kết nối qua mạng. Mỗi tier có một trách nhiệm và giao tiếp với neighbor qua protocol rõ ràng.

Nơi **N-Layer** là *logic* (cách code được tổ chức bên trong một program), **N-Tier** là *vật lý* (nơi mỗi mảnh program thực sự chạy).

> Mental model nhanh: layer sống trong IDE; tier sống trong diagram deploy.

---

## 2. Tier vs. Layer — Phân biệt quan trọng

| Khái niệm | Loại      | Boundary                  | Giao tiếp                              | Đơn vị fail                 |
| --------- | --------- | ------------------------- | -------------------------------------- | --------------------------- |
| **Layer** | Logic     | Project / namespace       | Method call in-process                  | Toàn process                |
| **Tier**  | Vật lý    | Mạng / process / VM       | HTTP / gRPC / TCP / queue              | Một tier (khác survive)     |

Bạn có thể có **N layer trong 1 tier** (monolith với cấu trúc nội bộ sạch) hoặc **N tier mỗi cái chứa layer riêng** (hệ thống phân tán).

---

## 3. Topology Tier phổ biến

### 3.1 1-Tier (Single-Tier / Standalone)

Mọi thứ chạy trên một máy: UI + logic + data.
**Ví dụ:** App desktop WinForms hoặc WPF với SQLite embedded.

```
┌────────────────────────────────┐
│   Desktop App                  │
│   UI + Business + SQLite       │
└────────────────────────────────┘
```

### 3.2 2-Tier (Client/Server)

"Fat client" kết nối trực tiếp với database server.
**Ví dụ:** App WinForms nói chuyện với SQL Server qua TCP.

```
┌────────────┐                   ┌────────────┐
│  Client    │ ── SQL/TCP ──►   │  DB Server │
│ (UI + BL)  │                   └────────────┘
└────────────┘
```

**Vấn đề:** business logic trên mỗi client → ác mộng patching; credential DB expose cho client.

### 3.3 3-Tier (Kiến trúc web kinh điển)

Topology enterprise phổ biến nhất. Thêm **application server** giữa client và database.

```
┌────────┐    HTTP    ┌────────────┐   TCP    ┌────────────┐
│ Browser│──────────► │  Web/API   │ ───────► │  Database  │
└────────┘            │  Server    │          └────────────┘
                      └────────────┘
```

- **Presentation Tier** — browser / mobile / Blazor WASM.
- **Application Tier** — ASP.NET Core Web API, gRPC service, background worker.
- **Data Tier** — SQL Server, PostgreSQL, Cosmos DB.

### 3.4 4-Tier

Tách application tier thành **web tier** (xử lý request) và **business/services tier** (domain logic, thường gọi "app server").

```
Browser ─► Web Tier (API/BFF) ─► Services Tier ─► Data Tier
                                                      │
                                              cache tuỳ chọn (Redis)
```

### 3.5 N-Tier (5+)

Thêm tier khác cho caching, message broker, search, identity. Mỗi cái scale, fail, replace độc lập.

```
        ┌──────────┐
Browser►│  CDN     │──┐
        └──────────┘  │
                      ▼
                  ┌──────────┐
                  │ Web/BFF  │──┐
                  └──────────┘  │
                                ▼
                            ┌──────────┐    ┌─────────┐
                            │ Services │◄──►│ Cache   │
                            └────┬─────┘    └─────────┘
                                 │
                                 ▼
                            ┌──────────┐    ┌─────────┐
                            │   DB     │◄──►│ Search  │
                            └──────────┘    └─────────┘
```

---

## 4. Tại sao chọn N-Tier?

| Lợi ích             | Thực sự nhận được                                                                            |
| ------------------- | ------------------------------------------------------------------------------------------- |
| **Scale độc lập**   | API tier stateless scale ngang không cần scale DB.                                           |
| **Security**        | DB không bao giờ thấy public internet; client không bao giờ thấy DB credential.              |
| **Maintainability** | Tier có thể rewrite hoặc replace (vd., Node → .NET) không đụng neighbor.                     |
| **Reliability**     | Fail một tier (cache outage) có thể được cô lập và degrade gracefully.                       |
| **Specialization**  | Mỗi tier dùng tool tốt nhất (Redis cho cache, Elasticsearch cho search).                     |
| **Compliance**      | Đặt data tier trong network segment regulated (PCI, HIPAA, GDPR).                            |
| **Tech evolution**  | Thay front-end Web Forms monolithic với Blazor hoặc React app không rewrite API.             |

---

## 5. Trade-off & Chi phí

| Chi phí                | Thực tế                                                                              |
| ---------------------- | ------------------------------------------------------------------------------------ |
| **Network latency**    | Mỗi hop tier thêm milliseconds — và fail tiềm năng.                                  |
| **Operational complexity** | Nhiều moving part phải deploy, configure, secure, monitor, back up.              |
| **Distributed failure**| Partial outage giờ có thể (và thường xuyên).                                          |
| **Eventual consistency** | Cache, queue, replica drift khỏi primary store.                                    |
| **Serialization overhead** | Marshal DTO ở mỗi boundary tốn CPU + bandwidth.                                  |
| **Debug khó**          | Trace request qua tier cần correlation ID và distributed tracing.                    |

> Quy tắc: đừng tách tier vì lý thuyết. Tách khi có lý do cụ thể — scale, security, ownership, hoặc isolation fail.

---

## 6. Anatomy của App 3-Tier .NET kinh điển

| Tier               | Stack .NET điển hình                                                                | Chạy trên                                          |
| ------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Presentation**   | Blazor WASM, React/Angular SPA, MAUI, mobile, MVC view                              | Thiết bị user / CDN / App Service / Static Web App |
| **Application/API**| ASP.NET Core Web API, minimal API, gRPC, SignalR, background worker (`IHostedService`) | App Service / AKS / EC2 / Functions          |
| **Data**           | SQL Server, PostgreSQL, Cosmos DB, MongoDB, Redis                                   | Managed DB service / VM riêng                     |

Companion phổ biến:
- **Reverse proxy / API gateway** (NGINX, YARP, Azure Front Door)
- **Identity provider** (Azure AD, Auth0, IdentityServer / Duende)
- **Message broker** (Azure Service Bus, RabbitMQ, Kafka)
- **Cache** (Redis, MemoryCache chỉ cho single-instance)
- **Search** (Elasticsearch, Azure AI Search)
- **Observability** (App Insights, OpenTelemetry → Grafana/Tempo/Loki)

---

## 7. Sample — 3-Tier end-to-end trong .NET

### 7.1 Cấu trúc .sln

```
Shop.sln
│
├── src/
│   ├── Shop.Web/                  ← Presentation Tier (Blazor WASM)
│   │   ├── Pages/
│   │   └── Program.cs
│   │
│   ├── Shop.Api/                  ← Application Tier (ASP.NET Core Web API)
│   │   ├── Controllers/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   ├── Shop.Application/          ← (Layer bên trong API tier)
│   │   ├── Orders/
│   │   └── Abstractions/
│   │
│   ├── Shop.Domain/               ← (Layer bên trong API tier)
│   │   └── Orders/
│   │
│   ├── Shop.Infrastructure/       ← (Layer bên trong API tier; nói chuyện với Data Tier)
│   │   ├── Persistence/
│   │   └── External/
│   │
│   └── Shop.Shared/               ← DTO share giữa Web (client) và Api
│       └── Contracts/
│
├── deploy/
│   ├── docker-compose.yml         ← Topology 3-tier local
│   ├── api.Dockerfile
│   ├── web.Dockerfile
│   └── nginx.conf
│
└── tests/
    └── …
```

> Lưu ý: `Shop.Application`, `Shop.Domain`, `Shop.Infrastructure` là **layer** *bên trong* Application Tier. Chúng thành tier chỉ nếu deploy như process riêng.

### 7.2 Presentation Tier — Blazor WASM client

```csharp
// Shop.Web/Pages/Orders.razor
@inject HttpClient Http

<button @onclick="LoadOrders">Refresh</button>
@foreach (var order in _orders)
{
    <div>@order.Id — @order.Total</div>
}

@code {
    private List<OrderDto> _orders = new();

    private async Task LoadOrders()
    {
        _orders = await Http.GetFromJsonAsync<List<OrderDto>>("/api/orders") ?? new();
    }
}
```

### 7.3 Application Tier — ASP.NET Core Web API

```csharp
// Shop.Api/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender _mediator;

    public OrdersController(ISender mediator) => _mediator = mediator;

    [HttpGet]
    public Task<IReadOnlyList<OrderDto>> List(CancellationToken ct) =>
        _mediator.Send(new ListOrdersQuery(), ct);

    [HttpPost]
    public async Task<IActionResult> Place(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var id = await _mediator.Send(cmd, ct);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }

    [HttpGet("{id:guid}")]
    public Task<OrderDto?> GetById(Guid id, CancellationToken ct) =>
        _mediator.Send(new GetOrderQuery(id), ct);
}
```

### 7.4 Data Tier — connection từ API tới DB

```jsonc
// Shop.Api/appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=db;Database=Shop;User Id=app;Password=...;TrustServerCertificate=true"
  }
}
```

```csharp
// Shop.Infrastructure/DependencyInjection.cs
public static IServiceCollection AddInfrastructure(this IServiceCollection s, IConfiguration cfg)
{
    s.AddDbContext<AppDbContext>(o =>
        o.UseSqlServer(cfg.GetConnectionString("Default"),
            sql => sql.EnableRetryOnFailure()));      // resilience tier-hop built-in
    s.AddScoped<IOrderRepository, EfOrderRepository>();
    return s;
}
```

### 7.5 Topology 3-tier local với Docker Compose

```yaml
# deploy/docker-compose.yml
services:
  web:
    build: { context: .., dockerfile: deploy/web.Dockerfile }
    ports: ["8080:80"]
    depends_on: [api]
    networks: [public, internal]

  api:
    build: { context: .., dockerfile: deploy/api.Dockerfile }
    environment:
      - ConnectionStrings__Default=Server=db;Database=Shop;User Id=app;Password=Pass#1
    depends_on: [db]
    networks: [internal, data]   # KHÔNG trên public network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=Strong#Passw0rd
    volumes: ["dbdata:/var/opt/mssql"]
    networks: [data]              # chỉ API có thể reach

networks:
  public:    {}
  internal:  { internal: true }
  data:      { internal: true }

volumes:
  dbdata:
```

Mỗi tier là container riêng, trên network segment riêng. Browser chỉ có thể reach `web`; `web` chỉ có thể reach `api`; `api` là thứ **duy nhất** có thể reach `db`.

---

## 8. Giao tiếp giữa các Tier

| Cơ chế                        | Dùng khi                                                                        | Stack .NET                                    |
| ----------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------- |
| **HTTP / REST (JSON)**        | Mặc định cho browser-to-API và phần lớn call nội bộ. Phổ quát, observable.      | `HttpClient`, ASP.NET Core, OpenAPI           |
| **gRPC**                      | Service-to-service, latency thấp, contract strongly typed, streaming.           | `Grpc.AspNetCore`, `Grpc.Net.Client`          |
| **GraphQL**                   | Tier aggregation BFF cần query flexible từ backend đa dạng.                     | HotChocolate                                  |
| **SignalR / WebSocket**       | Push từ server sang client (notification, dashboard).                            | `Microsoft.AspNetCore.SignalR`                |
| **Message bus (async)**       | Decouple producer/consumer; survive outage tạm.                                  | MassTransit / NServiceBus trên RabbitMQ / Azure SB |
| **Database replication**      | Spread read sang replica; primary handle write.                                  | SQL AG / PostgreSQL streaming replication     |
| **Cache như side-tier**       | Hot read serve từ Redis; primary store như fallback.                             | `StackExchange.Redis`, `HybridCache` (.NET 9) |

**Quy tắc:**
- HTTP/gRPC synchronous = mental model đơn giản, nhưng lan fail.
- Async messaging = resilience và decouple, nhưng eventual consistency.
- Traffic tier-to-tier nội bộ nên đi qua mTLS hoặc private network, không bao giờ public internet.

---

## 9. Ví dụ Topology Deploy

### 9.1 SaaS nhỏ — một region

```
Internet ─► Cloudflare ─► App Service (Web+API combine) ─► Azure SQL
```

Deploy 2-tier nhưng vẫn có 3 layer logic trong code.

### 9.2 SaaS trung — 3 tier

```
Internet ─► CDN ─► App Service (Blazor WASM static) ─► App Service (API) ─► Azure SQL
                                                              │
                                                           Redis
```

### 9.3 Enterprise — 4-tier với worker

```
Internet ─► WAF ─► API Gateway ─► Web/BFF cluster ─► Services cluster ─► SQL Cluster
                                                            │
                                                       Service Bus ─► Worker tier
```

### 9.4 Hybrid — data on-prem, frontend cloud

```
Cloud:    Web/API tier
            │  mTLS qua VPN / ExpressRoute
On-prem:  Data tier (legacy SQL Server)
```

---

## 10. Scale mỗi Tier độc lập

| Tier         | Chiến lược scale                                                                     |
| ------------ | ------------------------------------------------------------------------------------ |
| Presentation | Edge caching CDN; static hosting; client-side rendering offload work khỏi API.       |
| Application  | **Horizontal**: replica stateless đằng sau load balancer. Externalize session (Redis). |
| Data         | **Vertical**: VM to hơn. **Read replica** cho read-heavy. **Sharding** cho write-heavy. |
| Cache        | Redis cluster với replica; sentinel cho failover.                                    |
| Worker       | Scale theo queue depth; pattern competing-consumer.                                  |

**Statelessness là điều không thương lượng** cho scale ngang của API tier. Nếu request phải hit server cụ thể (sticky session), không thể scale tự do.

---

## 11. Security: Network Segmentation, DMZ, Zero Trust

Mỗi tier nằm trong **network segment riêng** (VPC / VNet / subnet). Traffic inter-tier flow qua route **được phép explicit** — thường:

```
Public  → Web/API  → Application  → Data
(DMZ)    (App subnet)              (Data subnet, không egress)
```

Best practice:

- **Data tier không bao giờ thấy public internet.** Dùng private endpoint.
- **TLS in transit** giữa mọi tier (mTLS cho service-to-service).
- **Không credential trong code.** Dùng managed identity (Azure) hoặc IAM role (AWS) cộng secret store (Key Vault / Secrets Manager).
- **Web Application Firewall (WAF)** trước presentation tier.
- **Rate limiting & throttling** ở API tier (`AddRateLimiter` trong .NET 7+).
- **Least privilege** cho user DB của mỗi tier (read-only cho read replica, v.v.).
- **Audit log** ở mỗi hop tier; tập trung trong SIEM.
- **Zero Trust** — authenticate mọi call giữa tier, đừng trust network.

---

## 12. Reliability: Health, Resilience, Failure Mode

Mỗi hop tier là chỗ request có thể fail.

- **Health check** per tier (`/health/live`, `/health/ready`).
- **Resilience policy** giữa tier — `Microsoft.Extensions.Resilience` (.NET 8+) hoặc Polly:
  - Retry với exponential backoff + jitter
  - Circuit breaker
  - Timeout
  - Bulkhead isolation
- **Graceful degradation**: nếu cache tier down, fallback data tier; nếu recommendation tier down, ẩn widget đó nhưng show phần còn lại.
- **Idempotency key** trên command để retry an toàn.
- **Outbox pattern** đảm bảo DB + message bus consistent.

---

## 13. Observability xuyên Tier

Một user request thành N request qua N tier — bạn cần correlation.

- **OpenTelemetry** instrumentation mọi tier; propagate header HTTP `traceparent`.
- **Structured logging** với `Serilog` enrich với correlation ID + user/tenant.
- **UI distributed tracing**: Jaeger, Tempo, Application Insights, Honeycomb.
- **Metric per tier**: P99 latency, RPS, error rate, saturation (USE/RED).
- **Log mọi fail tier-hop** với tên tier upstream + downstream.

---

## 14. Best Structure & Approach

1. **Quyết định tier dựa trên yêu cầu, không phải mốt.** Mỗi tier phải kiếm ăn được với lý do scale, security, hoặc ownership rõ.
2. **Khởi đầu 3 tier** (UI, API, DB). Thêm (cache, worker, BFF) khi bottleneck thật xuất hiện.
3. **Giữ contract tier explicit và có version.** OpenAPI cho REST, `.proto` cho gRPC, schema registry cho event.
4. **API tier stateless.** Externalize session, cache, lock (Redis, DB, distributed lock manager).
5. **Đặt DB trong network segment private.** Không public IP, không exception.
6. **Dùng managed identity / IAM role**, không phải connection string với password.
7. **TLS mọi nơi, mTLS nội bộ.** Rotate cert tự động.
8. **Một trách nhiệm per tier.** Đừng đặt logic cache trong DB tier hay auth trong data tier.
9. **Resilience policy giữa mọi hop tier.** Retry + timeout + circuit breaker không tuỳ chọn.
10. **CancellationToken end-to-end.** Downstream call treo nếu không sẽ làm cạn thread upstream.
11. **Idempotent write endpoint** để retry không double-charge / double-create.
12. **Outbox pattern** cho combination "DB write + message publish".
13. **Tách read path khỏi write path** ở data tier (replica hoặc CQRS) khi read load lớn.
14. **Deploy mỗi tier độc lập.** Không "atomic deploy" mọi tier — đó là monolith giả N-Tier.
15. **Version mọi contract tier.** Roll forward; không bao giờ break consumer không deprecate.
16. **Health endpoint per tier** wire vào load-balancer probe và alerting.
17. **Dùng service mesh hoặc sidecar** (Linkerd / Istio / Dapr) khi có nhiều tier nội bộ.
18. **Distributed tracing từ ngày một.** Thêm sau đau đớn.
19. **Single source of truth cho config.** Azure App Configuration, AWS AppConfig, hoặc env var từ secret manager.
20. **Định nghĩa SLO/SLI per tier** để biết tier nào bottleneck hiện tại.

---

## 15. Anti-Pattern

| Anti-pattern                                | Tại sao tệ                                                                            |
| ------------------------------------------- | ------------------------------------------------------------------------------------- |
| **Chatty tier hop** (10 call/request)        | Latency × số hop. Aggregate ở BFF hoặc denormalize.                                  |
| **Distributed monolith**                     | Tier phải deploy cùng nhau — bạn nhận chi phí phân tán không lợi ích.                |
| **Public DB**                                | DB expose ra internet, kể cả với cred mạnh. Cuối cùng leak.                          |
| **Shared DB qua tier từ team khác**         | Thành điểm integration; couple team; chặn release.                                    |
| **Stateful API tier**                        | Không scale ngang; sticky session tạo hotspot.                                        |
| **Không retry / không timeout**              | Một downstream chậm hạ cả stack qua cạn thread.                                       |
| **Retry aggressive** (không backoff/jitter)  | Biến blip transient thành thundering herd giết downstream tier.                       |
| **Chain synchronous qua nhiều tier**         | Availability nhân *xuống*. Năm tier 99.9% liên tiếp ≈ 99.5%.                          |
| **Log connection string hoặc PII**           | Vi phạm compliance và vector ăn cắp credential.                                       |
| **DB one-size-fits-all**                    | Một SQL DB cho cache, search, queue, OLTP. Move workload sang store fit-for-purpose.   |
| **Không correlation ID**                     | Debug qua tier thành đoán mò.                                                          |
| **Join DB cross-tier qua linked server**    | Coupling ẩn; landmine performance.                                                     |

---

## 16. Mapping Cloud — Azure & AWS

| Tier         | Azure                                              | AWS                                                |
| ------------ | -------------------------------------------------- | -------------------------------------------------- |
| Edge / CDN   | Front Door + CDN                                   | CloudFront                                         |
| Presentation | Static Web App / App Service / Blazor on Azure     | S3 + CloudFront / Amplify                          |
| API Gateway  | API Management / Front Door / YARP                 | API Gateway / ALB                                  |
| Application  | App Service / Container App / AKS / Function       | ECS / EKS / Lambda / EC2                           |
| Cache        | Azure Cache for Redis                              | ElastiCache                                        |
| Messaging    | Service Bus / Event Hub / Event Grid               | SQS / SNS / Kinesis / EventBridge                  |
| Data (SQL)   | Azure SQL DB / SQL Managed Instance / PostgreSQL Flexible | RDS / Aurora                                |
| Data (NoSQL) | Cosmos DB                                          | DynamoDB                                           |
| Search       | Azure AI Search                                    | OpenSearch                                         |
| Identity     | Entra ID (Azure AD) / Entra External ID            | Cognito / IAM Identity Center                       |
| Secret       | Key Vault                                          | Secrets Manager / Parameter Store                   |
| Observability| Application Insights / Azure Monitor / Log Analytics | CloudWatch / X-Ray                              |

---

## 17. N-Tier vs. Microservices

Liên quan nhưng **không giống nhau**:

| Khía cạnh           | N-Tier                                         | Microservices                                       |
| ------------------- | ---------------------------------------------- | --------------------------------------------------- |
| Tách theo           | **Vai trò technical** (UI, app, DB)             | **Business capability** (Order, Billing, Shipping)  |
| Shared DB           | Có — thường một DB share qua app tier           | Không — mỗi service sở hữu data                     |
| Deploy unit         | Toàn application                                | Một service                                          |
| Team ownership      | Thường horizontal (UI team, DB team)            | Vertical (full-stack per service)                    |
| Isolation fail      | Trung bình                                      | Cao                                                  |
| Cost vận hành       | Trung bình                                      | Cao                                                  |
| Tốt cho             | Sản phẩm cohesive, một bounded context           | Nhiều bounded context; nhiều team                    |

Cả hai là **hệ thống phân tán**. Microservices về cơ bản là nhiều app N-Tier stitch lại ở contract boundary.

---

## 18. Key Point — Danh sách ngắn

* **Tier = nơi chạy. Layer = cách tổ chức.** Khái niệm khác nhau; cả hai hữu ích.
* **Ba tier giải 90% bài toán.** Thêm chỉ với lý do cụ thể.
* **Statelessness là cái giá của scale ngang.**
* **Mỗi hop tier là chỗ fail.** Retry, timeout, trace nó.
* **Khoá data tier.** Network private, managed identity, least privilege.
* **Contract version, explicit** giữa tier — OpenAPI / gRPC / event schema.
* **Deploy tier độc lập** hoặc bạn đã build distributed monolith.
* **Observe và SLO mỗi tier**; nếu không không biết cái nào đang fail.
* **N-Tier là cấu trúc. Microservices thêm dimension tổ chức + bounded context.**

---

## 19. Checklist

Trước khi ship hệ thống N-Tier:

- [ ] Mỗi tier có mục đích document và một trách nhiệm.
- [ ] Diagram topology mạng tồn tại và match thực tế.
- [ ] DB tier trong private subnet không public IP.
- [ ] Mọi call inter-tier dùng TLS (mTLS nội bộ nếu có thể).
- [ ] Secret trong vault; không plaintext trong file config.
- [ ] API tier stateless; session và lock sống trong shared store.
- [ ] Resilience policy (retry, timeout, circuit breaker) ở mọi hop.
- [ ] Health endpoint (`/health/live`, `/health/ready`) per tier; wire vào load balancer.
- [ ] OpenTelemetry tracing propagate qua mọi tier.
- [ ] Structured log bao gồm correlation ID; tập trung trong SIEM.
- [ ] CI deploy mỗi tier độc lập.
- [ ] Contract tier (OpenAPI / .proto / event schema) có version.
- [ ] Auto-scaling rule per tier với min/max hợp lý.
- [ ] Backup + restore test cho data tier; runbook document.
- [ ] Doc failure-mode (chuyện gì xảy ra nếu cache down? primary DB failover?).
- [ ] Cost per tier được monitor.

---

## 20. Tham khảo

* [Microsoft Learn — N-tier architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
* [Microsoft Learn — N-tier on Azure (virtual machines)](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/n-tier/n-tier-sql-server)
* [Microsoft Learn — Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
* [AWS — Three-tier architecture overview](https://docs.aws.amazon.com/whitepapers/latest/serverless-multi-tier-architectures-api-gateway-lambda/three-tier-architecture-overview.html)
* [Stackify — What is N-Tier Architecture?](https://stackify.com/n-tier-architecture/)
* [Martin Fowler — Tradeoffs of microservices vs. monoliths](https://martinfowler.com/articles/microservices.html)
* [The Twelve-Factor App](https://12factor.net/)
