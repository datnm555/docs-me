# N-Tier Architecture — Explanation, Samples, Best Approach & Key Points

> A guide to **physical deployment architecture** in .NET — how an application is split across processes, machines, and network boundaries.

> **Companion docs:**
> * [`n-layer-architecture.md`](./n-layer-architecture.md) — **logical** code organization.
> * [`clean-architecture.md`](./clean-architecture.md) — principles-driven layered design.

---

## Quick Reference (What · Why · When · Where)

- **What** — Splitting an application across **N physical tiers** (processes / containers / machines) that communicate over a network. Classic 3-tier = browser/UI + application server + database.
- **Why** — Independent scaling per tier; security isolation (DB in a private subnet); fault isolation (cache outage ≠ app down); compliance boundaries; per-tier technology choice.
- **When** — You need to scale or harden one tier independently of others; regulatory requirements force network segmentation; multiple teams own different tiers.
- **Where** — *Physical* axis — orthogonal to N-Layer (logical) and Clean (principles). Any of those styles can run on 1, 2, 3, or N tiers. Often paired with N-Layer code organization inside the application tier.

---

## Table of Contents

1. [What Is N-Tier Architecture?](#1-what-is-n-tier-architecture)
2. [Tier vs. Layer — A Critical Distinction](#2-tier-vs-layer--a-critical-distinction)
3. [The Common Tier Topologies](#3-the-common-tier-topologies)
4. [Why Choose N-Tier?](#4-why-choose-n-tier)
5. [Trade-offs & Costs](#5-trade-offs--costs)
6. [Anatomy of a Classic 3-Tier .NET App](#6-anatomy-of-a-classic-3-tier-net-app)
7. [Sample — End-to-End 3-Tier in .NET](#7-sample--end-to-end-3-tier-in-net)
8. [Communication Between Tiers](#8-communication-between-tiers)
9. [Deployment Topology Examples](#9-deployment-topology-examples)
10. [Scaling Each Tier Independently](#10-scaling-each-tier-independently)
11. [Security: Network Segmentation, DMZ, Zero Trust](#11-security-network-segmentation-dmz-zero-trust)
12. [Reliability: Health, Resilience, Failure Modes](#12-reliability-health-resilience-failure-modes)
13. [Observability Across Tiers](#13-observability-across-tiers)
14. [Best Structure & Approach (Recommendations)](#14-best-structure--approach-recommendations)
15. [Anti-Patterns](#15-anti-patterns)
16. [Cloud Mapping — Azure & AWS](#16-cloud-mapping--azure--aws)
17. [N-Tier vs. Microservices](#17-n-tier-vs-microservices)
18. [Key Points — The Short List](#18-key-points--the-short-list)
19. [Checklist](#19-checklist)
20. [References](#20-references)

---

## 1. What Is N-Tier Architecture?

**N-Tier architecture** is a way of splitting an application across **N physical (or process) boundaries** — separate machines, containers, VMs, or processes — connected over a network. Each tier has a single responsibility and communicates with its neighbors through well-defined protocols.

Where **N-Layer** is *logical* (how the code is organized inside a single program), **N-Tier** is *physical* (where each piece of the program actually runs).

> Quick mental model: layers live in your IDE; tiers live in your deployment diagram.

---

## 2. Tier vs. Layer — A Critical Distinction

| Concept   | Type      | Boundary                  | Communication                          | Failure unit                |
| --------- | --------- | ------------------------- | -------------------------------------- | --------------------------- |
| **Layer** | Logical   | Project / namespace       | In-process method calls                | The whole process           |
| **Tier**  | Physical  | Network / process / VM    | HTTP / gRPC / TCP / queue              | A single tier (others survive) |

You can have **N layers in 1 tier** (monolith with clean internal structure) or **N tiers each containing its own layers** (distributed system).

---

## 3. The Common Tier Topologies

### 3.1 1-Tier (Single-Tier / Standalone)

Everything runs on one machine: UI + logic + data.
**Example:** A WinForms or WPF desktop app with embedded SQLite.

```
┌────────────────────────────────┐
│   Desktop App                  │
│   UI + Business + SQLite       │
└────────────────────────────────┘
```

### 3.2 2-Tier (Client/Server)

A "fat client" connects directly to a database server.
**Example:** WinForms app talking to SQL Server over TCP.

```
┌────────────┐                   ┌────────────┐
│  Client    │ ── SQL/TCP ──►   │  DB Server │
│ (UI + BL)  │                   └────────────┘
└────────────┘
```

**Problems:** business logic on every client → patching nightmare; DB credentials exposed to clients.

### 3.3 3-Tier (Classic Web Architecture)

The most common enterprise topology. Adds an **application server** between the client and the database.

```
┌────────┐    HTTP    ┌────────────┐   TCP    ┌────────────┐
│ Browser│──────────► │  Web/API   │ ───────► │  Database  │
└────────┘            │  Server    │          └────────────┘
                      └────────────┘
```

- **Presentation Tier** — browser / mobile / Blazor WASM.
- **Application Tier** — ASP.NET Core Web API, gRPC services, background workers.
- **Data Tier** — SQL Server, PostgreSQL, Cosmos DB.

### 3.4 4-Tier

Splits the application tier into **web tier** (request handling) and **business/services tier** (domain logic, often called the "app server").

```
Browser ─► Web Tier (API/BFF) ─► Services Tier ─► Data Tier
                                                      │
                                              optional cache (Redis)
```

### 3.5 N-Tier (5+)

Adds further tiers for caching, message brokers, search, identity, etc. Each can scale, fail, and be replaced independently.

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

## 4. Why Choose N-Tier?

| Benefit             | What you actually get                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------- |
| **Independent scaling** | Stateless API tier scales horizontally without scaling the DB.                          |
| **Security**        | DB never sees the public internet; client never sees DB credentials.                        |
| **Maintainability** | A tier can be rewritten or replaced (e.g., Node → .NET) without touching its neighbors.     |
| **Reliability**     | A failure in one tier (cache outage) can be isolated and degraded gracefully.               |
| **Specialization**  | Each tier can use the best tool (Redis for cache, Elasticsearch for search).                |
| **Compliance**      | Place data tier inside a regulated network segment (PCI, HIPAA, GDPR).                      |
| **Tech evolution**  | Replace your monolithic Web Forms front-end with a Blazor or React app without rewriting the API. |

---

## 5. Trade-offs & Costs

| Cost                   | Reality                                                                              |
| ---------------------- | ------------------------------------------------------------------------------------ |
| **Network latency**    | Every tier hop adds milliseconds — and a potential failure.                          |
| **Operational complexity** | More moving parts to deploy, configure, secure, monitor, back up.                |
| **Distributed failures**| Partial outages are now possible (and frequent).                                     |
| **Eventual consistency** | Caches, queues, and replicas drift from the primary store.                          |
| **Serialization overhead** | Marshalling DTOs at each boundary has CPU + bandwidth cost.                       |
| **Debugging difficulty** | Tracing a request across tiers needs correlation IDs and distributed tracing.       |

> Rule of thumb: don't split tiers for theory's sake. Split when you have a concrete reason — scale, security, ownership, or fault isolation.

---

## 6. Anatomy of a Classic 3-Tier .NET App

| Tier               | Typical .NET stack                                                                  | Runs on                                          |
| ------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Presentation**   | Blazor WASM, React/Angular SPA, MAUI, mobile, MVC views                            | User device / CDN / App Service / Static Web App |
| **Application/API**| ASP.NET Core Web API, minimal APIs, gRPC, SignalR, background workers (`IHostedService`) | App Service / AKS / EC2 / Functions          |
| **Data**           | SQL Server, PostgreSQL, Cosmos DB, MongoDB, Redis                                   | Managed DB service / dedicated VM                |

Common companions:
- **Reverse proxy / API gateway** (NGINX, YARP, Azure Front Door)
- **Identity provider** (Azure AD, Auth0, IdentityServer / Duende)
- **Message broker** (Azure Service Bus, RabbitMQ, Kafka)
- **Cache** (Redis, MemoryCache only for single-instance)
- **Search** (Elasticsearch, Azure AI Search)
- **Observability** (App Insights, OpenTelemetry → Grafana/Tempo/Loki)

---

## 7. Sample — End-to-End 3-Tier in .NET

### 7.1 The .sln structure

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
│   ├── Shop.Application/          ← (Layer inside the API tier)
│   │   ├── Orders/
│   │   └── Abstractions/
│   │
│   ├── Shop.Domain/               ← (Layer inside the API tier)
│   │   └── Orders/
│   │
│   ├── Shop.Infrastructure/       ← (Layer inside the API tier; talks to Data Tier)
│   │   ├── Persistence/
│   │   └── External/
│   │
│   └── Shop.Shared/               ← DTOs shared between Web (client) and Api
│       └── Contracts/
│
├── deploy/
│   ├── docker-compose.yml         ← Local 3-tier topology
│   ├── api.Dockerfile
│   ├── web.Dockerfile
│   └── nginx.conf
│
└── tests/
    └── …
```

> Note: `Shop.Application`, `Shop.Domain`, `Shop.Infrastructure` are **layers** *inside* the Application Tier. They become tiers only if you deploy them as separate processes.

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

### 7.4 Data Tier — connection from API to DB

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
            sql => sql.EnableRetryOnFailure()));      // built-in tier-hop resilience
    s.AddScoped<IOrderRepository, EfOrderRepository>();
    return s;
}
```

### 7.5 Local 3-tier topology with Docker Compose

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
    networks: [internal, data]   # NOT on the public network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=Strong#Passw0rd
    volumes: ["dbdata:/var/opt/mssql"]
    networks: [data]              # only API can reach it

networks:
  public:    {}
  internal:  { internal: true }
  data:      { internal: true }

volumes:
  dbdata:
```

Each tier is a separate container, on a separate network segment. The browser can reach **only** `web`; `web` can reach **only** `api`; `api` is the **only** thing that can reach `db`.

---

## 8. Communication Between Tiers

| Mechanism                     | When to use                                                                     | .NET stack                                   |
| ----------------------------- | ------------------------------------------------------------------------------- | -------------------------------------------- |
| **HTTP / REST (JSON)**        | Default for browser-to-API and most internal calls. Universal, observable.      | `HttpClient`, ASP.NET Core, OpenAPI          |
| **gRPC**                      | Service-to-service, low latency, strongly typed contracts, streaming.            | `Grpc.AspNetCore`, `Grpc.Net.Client`         |
| **GraphQL**                   | Aggregation BFF tier that needs flexible queries from heterogeneous backends.    | HotChocolate                                 |
| **SignalR / WebSockets**      | Push from server to client (notifications, dashboards).                          | `Microsoft.AspNetCore.SignalR`               |
| **Message bus (async)**       | Decouple producers/consumers; survive temporary outages.                         | MassTransit / NServiceBus on RabbitMQ / Azure SB |
| **Database replication**      | Spread reads to replicas; primary handles writes.                                | SQL AG / PostgreSQL streaming replication    |
| **Cache as side-tier**        | Hot reads served from Redis; primary store as fallback.                          | `StackExchange.Redis`, `HybridCache` (.NET 9)|

**Rules of thumb:**
- Synchronous HTTP/gRPC = simplest mental model, but propagates failures.
- Asynchronous messaging = resilience and decoupling, but eventual consistency.
- Internal tier-to-tier traffic should travel over mTLS or a private network, never the public internet.

---

## 9. Deployment Topology Examples

### 9.1 Small SaaS — single region

```
Internet ─► Cloudflare ─► App Service (Web+API combined) ─► Azure SQL
```

A 2-tier deployment that still has 3 logical layers in code.

### 9.2 Medium SaaS — 3 tiers

```
Internet ─► CDN ─► App Service (Blazor WASM static) ─► App Service (API) ─► Azure SQL
                                                              │
                                                           Redis
```

### 9.3 Enterprise — 4-tier with workers

```
Internet ─► WAF ─► API Gateway ─► Web/BFF cluster ─► Services cluster ─► SQL Cluster
                                                            │
                                                       Service Bus ─► Worker tier
```

### 9.4 Hybrid — on-prem data, cloud frontend

```
Cloud:    Web/API tier
            │  mTLS over VPN / ExpressRoute
On-prem:  Data tier (legacy SQL Server)
```

---

## 10. Scaling Each Tier Independently

| Tier         | Scale strategy                                                                       |
| ------------ | ------------------------------------------------------------------------------------ |
| Presentation | CDN edge caching; static hosting; client-side rendering offloads work from the API.  |
| Application  | **Horizontal**: stateless replicas behind a load balancer. Keep sessions externalized (Redis). |
| Data         | **Vertical**: bigger VM. **Read replicas** for read-heavy workloads. **Sharding** for write-heavy. |
| Cache        | Redis cluster with replicas; sentinel for failover.                                  |
| Workers      | Scale based on queue depth; competing-consumer pattern.                              |

**Statelessness is non-negotiable** for horizontal scaling of the API tier. If a request must hit a specific server (sticky session), you cannot scale freely.

---

## 11. Security: Network Segmentation, DMZ, Zero Trust

Each tier sits in its own **network segment** (VPC / VNet / subnet). Inter-tier traffic flows through **explicitly allowed** routes — usually:

```
Public  → Web/API  → Application  → Data
(DMZ)    (App subnet)              (Data subnet, no egress)
```

Best practices:

- **Database tier never sees the public internet.** Use private endpoints.
- **TLS in transit** between every tier (mTLS for service-to-service).
- **No credentials in code.** Use managed identities (Azure) or IAM roles (AWS) plus a secret store (Key Vault / Secrets Manager).
- **Web Application Firewall (WAF)** in front of the presentation tier.
- **Rate limiting & throttling** at the API tier (`AddRateLimiter` in .NET 7+).
- **Least privilege** for each tier's DB user (read-only for the read replica, etc.).
- **Audit logs** at every tier hop; centralize in SIEM.
- **Zero Trust** — authenticate every call between tiers, don't trust the network.

---

## 12. Reliability: Health, Resilience, Failure Modes

Each tier hop is a place a request can fail.

- **Health checks** per tier (`/health/live`, `/health/ready`).
- **Resilience policies** between tiers — `Microsoft.Extensions.Resilience` (.NET 8+) or Polly:
  - Retries with exponential backoff + jitter
  - Circuit breakers
  - Timeouts
  - Bulkhead isolation
- **Graceful degradation**: if cache tier is down, fall back to the data tier; if recommendation tier is down, hide that widget but show the rest of the page.
- **Idempotency keys** on commands so retries are safe.
- **Outbox pattern** to ensure DB + message bus stay consistent.

---

## 13. Observability Across Tiers

A single user request becomes N requests through N tiers — you need correlation.

- **OpenTelemetry** instrumentation in every tier; propagate `traceparent` HTTP headers.
- **Structured logging** with `Serilog` enriched with correlation ID + user/tenant.
- **Distributed tracing** UI: Jaeger, Tempo, Application Insights, Honeycomb.
- **Metrics per tier**: P99 latency, RPS, error rate, saturation (USE/RED).
- **Log all tier-hop failures** with the upstream + downstream tier name.

---

## 14. Best Structure & Approach (Recommendations)

1. **Decide tiers based on requirements, not fashion.** Each tier must earn its keep with a clear scaling, security, or ownership reason.
2. **Start with 3 tiers** (UI, API, DB). Add more (cache, workers, BFF) when a real bottleneck appears.
3. **Keep tier contracts explicit and versioned.** OpenAPI for REST, `.proto` for gRPC, schema registry for events.
4. **Stateless API tier.** Externalize sessions, caches, locks (Redis, DB, distributed lock manager).
5. **Place the DB in a private network segment.** No public IP, no exception.
6. **Use managed identities / IAM roles**, not connection strings with passwords.
7. **TLS everywhere, mTLS internally.** Rotate certs automatically.
8. **One responsibility per tier.** Don't put cache logic inside the DB tier or auth inside the data tier.
9. **Resilience policies between every tier hop.** Retry + timeout + circuit breaker are not optional.
10. **CancellationToken end-to-end.** Hung downstream calls otherwise starve threads upstream.
11. **Idempotent write endpoints** so retries don't double-charge / double-create.
12. **Outbox pattern** for any "DB write + message publish" combination.
13. **Separate read paths from write paths** at the data tier (replica or CQRS) when read load dominates.
14. **Deploy each tier independently.** No "atomic deploy" of all tiers — that's a monolith pretending to be N-Tier.
15. **Version every tier contract.** Roll forward; never break consumers without deprecation.
16. **Health endpoints per tier** wired to load-balancer probes and to your alerting.
17. **Use a service mesh or sidecars** (Linkerd / Istio / Dapr) when you have many internal tiers.
18. **Distributed tracing from day one.** Adding it later is painful.
19. **Single source of truth for config.** Azure App Configuration, AWS AppConfig, or environment variables sourced from a secret manager.
20. **Define SLO/SLI per tier** so you know which tier is the current bottleneck.

---

## 15. Anti-Patterns

| Anti-pattern                                | Why it hurts                                                                          |
| ------------------------------------------- | ------------------------------------------------------------------------------------- |
| **Chatty tier hops** (10 calls/request)      | Latency × number of hops. Aggregate at the BFF or denormalize.                        |
| **Distributed monolith**                     | Tiers must deploy together — you get the cost of distribution without the benefits.   |
| **Public DB**                                | DB exposed to internet, even with strong creds. Eventually leaks.                     |
| **Shared DB across tiers from different teams** | Becomes the integration point; couples teams; blocks releases.                    |
| **Stateful API tier**                        | Cannot scale horizontally; sticky sessions create hotspots.                           |
| **No retries / no timeouts**                 | One slow downstream takes down the whole stack via thread exhaustion.                 |
| **Aggressive retries** (no backoff/jitter)   | Turns a transient blip into a thundering herd that kills the downstream tier.         |
| **Synchronous chain across many tiers**      | Availability multiplies *downward*. Five 99.9% tiers in a row ≈ 99.5%.                |
| **Logging connection strings or PII**        | Compliance breach and credential theft vector.                                        |
| **One-size-fits-all DB**                     | A single SQL DB for cache, search, queue, and OLTP. Move workloads to fit-for-purpose stores. |
| **No correlation IDs**                       | Debugging across tiers becomes guesswork.                                             |
| **Cross-tier DB joins via linked servers**   | Hidden coupling; performance landmine.                                                |

---

## 16. Cloud Mapping — Azure & AWS

| Tier         | Azure                                              | AWS                                                |
| ------------ | -------------------------------------------------- | -------------------------------------------------- |
| Edge / CDN   | Front Door + CDN                                   | CloudFront                                         |
| Presentation | Static Web Apps / App Service / Blazor on Azure    | S3 + CloudFront / Amplify                          |
| API Gateway  | API Management / Front Door / YARP                 | API Gateway / ALB                                  |
| Application  | App Service / Container Apps / AKS / Functions     | ECS / EKS / Lambda / EC2                           |
| Cache        | Azure Cache for Redis                              | ElastiCache                                        |
| Messaging    | Service Bus / Event Hubs / Event Grid              | SQS / SNS / Kinesis / EventBridge                  |
| Data (SQL)   | Azure SQL DB / SQL Managed Instance / PostgreSQL Flexible | RDS / Aurora                                |
| Data (NoSQL) | Cosmos DB                                          | DynamoDB                                           |
| Search       | Azure AI Search                                    | OpenSearch                                         |
| Identity     | Entra ID (Azure AD) / Entra External ID            | Cognito / IAM Identity Center                       |
| Secrets      | Key Vault                                          | Secrets Manager / Parameter Store                   |
| Observability| Application Insights / Azure Monitor / Log Analytics | CloudWatch / X-Ray                              |

---

## 17. N-Tier vs. Microservices

They're related but **not the same**:

| Aspect              | N-Tier                                         | Microservices                                       |
| ------------------- | ---------------------------------------------- | --------------------------------------------------- |
| Split by            | **Technical role** (UI, app, DB)               | **Business capability** (Orders, Billing, Shipping) |
| Shared DB           | Yes — usually one DB shared across the app tier | No — each service owns its data                    |
| Deploy unit         | Whole application                              | One service                                         |
| Team ownership      | Often horizontal (UI team, DB team)            | Vertical (full-stack per service)                   |
| Failure isolation   | Moderate                                       | High                                                |
| Operational cost    | Moderate                                       | High                                                |
| Best for            | Cohesive product, single bounded context       | Many bounded contexts; many teams                   |

Both are **distributed systems**. Microservices are essentially many N-Tier apps stitched together at the contract boundary.

---

## 18. Key Points — The Short List

* **Tier = where it runs. Layer = how it's organized.** Different concepts; both useful.
* **Three tiers solve 90% of problems.** Add more only with a concrete reason.
* **Statelessness is the price of horizontal scale.**
* **Every tier hop is a place to fail.** Retry, timeout, and trace it.
* **Lock down the data tier.** Private network, managed identity, least privilege.
* **Versioned, explicit contracts** between tiers — OpenAPI / gRPC / event schema.
* **Deploy tiers independently** or you've built a distributed monolith.
* **Observe and SLO each tier**; otherwise you can't tell which one is failing.
* **N-Tier is structural. Microservices add organizational + bounded-context dimensions.**

---

## 19. Checklist

Before shipping an N-Tier system:

- [ ] Each tier has a documented purpose and a single responsibility.
- [ ] Network topology diagram exists and matches reality.
- [ ] DB tier is in a private subnet with no public IP.
- [ ] All inter-tier calls use TLS (mTLS internally where possible).
- [ ] Secrets are in a vault; no plaintext in config files.
- [ ] API tier is stateless; sessions and locks live in a shared store.
- [ ] Resilience policies (retry, timeout, circuit breaker) are in place at every hop.
- [ ] Health endpoints (`/health/live`, `/health/ready`) per tier; wired to load balancer.
- [ ] OpenTelemetry tracing propagates across all tiers.
- [ ] Structured logs include correlation IDs; centralized in a SIEM.
- [ ] CI deploys each tier independently.
- [ ] Tier contracts (OpenAPI / .proto / event schema) are versioned.
- [ ] Auto-scaling rules per tier with sane min/max.
- [ ] Backup + restore tested for the data tier; runbook documented.
- [ ] Failure-mode docs (what happens if cache is down? if DB primary fails over?).
- [ ] Cost per tier monitored.

---

## 20. References

* [Microsoft Learn — N-tier architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
* [Microsoft Learn — N-tier on Azure (virtual machines)](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/n-tier/n-tier-sql-server)
* [Microsoft Learn — Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
* [AWS — Three-tier architecture overview](https://docs.aws.amazon.com/whitepapers/latest/serverless-multi-tier-architectures-api-gateway-lambda/three-tier-architecture-overview.html)
* [Stackify — What is N-Tier Architecture?](https://stackify.com/n-tier-architecture/)
* [Martin Fowler — Tradeoffs of microservices vs. monoliths](https://martinfowler.com/articles/microservices.html)
* [The Twelve-Factor App](https://12factor.net/)
