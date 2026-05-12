# Building Microservices — Summary

> **Sam Newman** — *Building Microservices: Designing Fine-Grained Systems*, 2nd edition, O'Reilly, 2021. First edition 2015.

> **Why this book matters:** the most-cited working definition of "microservices" in the industry comes from this book. The 2nd edition is a major rewrite that incorporates ~6 years of post-hype lessons — when to *not* use microservices, the operational complexity, the data-ownership problems, the testing pyramid, and the role of bounded contexts as the right slicing tool. The book is opinionated, practical, and avoids hype.

> **This summary covers:** Newman's definition, the principles, modeling around bounded contexts, splitting a monolith, communication styles, sagas vs. orchestration, deployment, testing, observability, security, resilience, scaling, and the organizational reality (Conway's Law). Code examples are mine, illustrative.

> 🇻🇳 Vietnamese version: [`building-microservices-sam-newman-vi.md`](./building-microservices-sam-newman-vi.md)

---

## Table of Contents

1. [About the Book](#1-about-the-book)
2. [Newman's Definition](#2-newmans-definition)
3. [Key Principles](#3-key-principles)
4. [Chapter Group 1 — Foundations](#4-chapter-group-1--foundations)
5. [Chapter Group 2 — Modeling and Splitting](#5-chapter-group-2--modeling-and-splitting)
6. [Chapter Group 3 — Communication](#6-chapter-group-3--communication)
7. [Chapter Group 4 — Workflow: Sagas, Orchestration vs. Choreography](#7-chapter-group-4--workflow-sagas-orchestration-vs-choreography)
8. [Chapter Group 5 — Build & Deployment](#8-chapter-group-5--build--deployment)
9. [Chapter Group 6 — Testing](#9-chapter-group-6--testing)
10. [Chapter Group 7 — Observability](#10-chapter-group-7--observability)
11. [Chapter Group 8 — Security](#11-chapter-group-8--security)
12. [Chapter Group 9 — Resilience](#12-chapter-group-9--resilience)
13. [Chapter Group 10 — Scaling](#13-chapter-group-10--scaling)
14. [Chapter Group 11 — User Interfaces](#14-chapter-group-11--user-interfaces)
15. [Chapter Group 12 — Organization & Conway's Law](#15-chapter-group-12--organization--conways-law)
16. [The Hard Question — Should You Use Microservices?](#16-the-hard-question--should-you-use-microservices)
17. [Big-Picture Take-Aways](#17-big-picture-take-aways)
18. [How This Book Fits the Repo](#18-how-this-book-fits-the-repo)
19. [Reading Order Recommendations](#19-reading-order-recommendations)
20. [Common Misreadings](#20-common-misreadings)
21. [References](#21-references)

---

## 1. About the Book

* **Title:** *Building Microservices: Designing Fine-Grained Systems*
* **Author:** Sam Newman (independent consultant, ex-ThoughtWorks)
* **Publisher:** O'Reilly Media
* **Editions:** 1st (2015), **2nd (2021, the canonical version to read today)**
* **Length:** ~600 pages
* **Audience:** architects, tech leads, senior engineers who already know how to build a monolith and are deciding whether (or how) to split one.

The 2nd edition is much more sober than the 1st. The 2015 book was published during peak microservices hype; the 2021 edition adds a substantial chapter on when **not** to use microservices, and the running advice throughout is: *start with a modular monolith; extract microservices when the pain is real.*

---

## 2. Newman's Definition

> **Microservices are independently deployable services modeled around a business domain. They communicate with each other via networks, and as an architecture choice offer many options for solving the problems you may face.**

Three load-bearing phrases:

1. **Independently deployable** — the single non-negotiable property. If you can't deploy a service without coordinating with other teams, you have a *distributed monolith*, not microservices.
2. **Modeled around a business domain** — services align with bounded contexts (DDD). Not "the auth service" — "the customer onboarding service".
3. **Communicate via networks** — therefore, every microservices system is a distributed system, and distributed-systems problems (partial failure, latency, consistency) are now your problems.

What microservices are **not**:
* Not "small services" — size is a consequence, not a goal.
* Not "stateless functions" — services often own state; they just don't share it.
* Not "every team gets their own programming language" — heterogeneity has real costs.
* Not "the same as SOA" — share lineage, but microservices add independent deployability as a hard requirement.

---

## 3. Key Principles

Newman lists the recurring principles in the book. They reinforce each other:

| Principle                                       | Meaning                                                                       |
| ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **Modeled around a business domain**            | Slice services along bounded contexts, not technical layers.                  |
| **Culture of automation**                       | If you can't fully automate build, test, and deploy, microservices will crush you. |
| **Hide internal implementation details**        | Each service owns its data. **No shared databases.** Period.                  |
| **Decentralize all the things**                 | Decisions belong with the team owning the service — language, datastore, deploy cadence. |
| **Deploy independently**                        | This is the rule that defines microservices.                                   |
| **Consumer first**                              | The consumer's needs drive the API. Backwards compatibility is the architect's job. |
| **Isolate failure**                              | Bulkheads, circuit breakers, timeouts, graceful degradation.                  |
| **Highly observable**                            | Logs, metrics, distributed traces, correlation IDs everywhere.                |

Three of these are routinely violated by teams that *think* they have microservices:

* **Hide internal implementation details.** A shared database **is** a leak of implementation. If two services join tables, they aren't separate services.
* **Deploy independently.** If your release process says "deploy A, then B, then C, in that order, after coordinating with team D," you don't have microservices.
* **Decentralize all the things.** A central team that owns "the messaging system" or "the database access library" becomes a synchronous bottleneck across all teams.

---

## 4. Chapter Group 1 — Foundations

### What microservices buy you

Real benefits, when used appropriately:

* **Independent deployability** — small, targeted releases reduce blast radius.
* **Independent scaling** — scale the bottleneck service horizontally without scaling the whole monolith.
* **Technology heterogeneity** — pick the right tool per service (a search service in Rust, a checkout in C#, a notification in Node).
* **Robustness** — partial failures, well-handled, reduce overall outage scope.
* **Team autonomy** — Conway's Law: small services align with small teams.
* **Reuse via composition** — generic capabilities (notification, auth, audit) become platform services.

### What microservices cost you

Real costs, often underestimated:

* **Distributed systems are hard.** Latency, partial failure, eventual consistency, retries, idempotency — all your problem now.
* **Operational maturity is mandatory.** CI/CD, observability, secret management, mTLS, service mesh — required, not optional.
* **Local development is painful** when 12 services must run together to test one feature.
* **Cross-service refactors** are 10× harder than cross-module refactors in a monolith.
* **End-to-end testing** is hard, slow, and flaky.

> Newman is explicit: **if you can build a well-modularized monolith, do that first.** Microservices solve scaling/team/deployment problems, not modeling problems.

---

## 5. Chapter Group 2 — Modeling and Splitting

### Modeling around bounded contexts

Newman is a DDD advocate. The single most important architectural decision is **where to draw the lines between services**. The book's answer: **bounded contexts** (Evans).

A bounded context is a part of the domain where words have one meaning. *"Customer"* in Sales is not *"Customer"* in Support; they have different attributes and different lifecycles. Each becomes its own bounded context — and each gets its own service.

### Two coupling axes

Within and between bounded contexts, two properties matter:

* **Coupling** — how strongly two services depend on each other. Low coupling is good.
* **Cohesion** — how much related behavior lives together inside one service. High cohesion is good.

Microservices that violate these — high coupling, low cohesion — are *distributed monoliths* and combine the worst of both worlds.

### Levels of coupling (least to worst)

Newman ranks coupling forms:

| Form                       | What it is                                                                | Verdict           |
| -------------------------- | ------------------------------------------------------------------------- | ----------------- |
| **Domain coupling**        | One service uses functionality of another (legitimate cross-context need)  | Acceptable        |
| **Pass-through coupling**  | A service passes data through to another, not transforming                | Smell             |
| **Common coupling**         | Services share state (same DB table, same global resource)                | Bad               |
| **Content coupling**       | One service reaches into the internals of another                          | Worst             |

The fix when coupling becomes content coupling: re-draw the bounded contexts.

### Splitting a monolith

A whole chapter on *how* to split. The lessons:

* **Don't split big-bang.** Use the **Strangler Fig** pattern — wrap legacy with a facade, peel off one bounded context at a time.
* **Database first, service second.** The database is usually the hardest part. The book outlines patterns: shared DB → schema per service → database per service.
* **Parallel run** before cutover — run old and new in parallel, compare outputs.
* **The Branch by Abstraction pattern** lets you introduce a new implementation safely behind an interface.
* **Don't refactor for refactor's sake.** Split when you have a real driver: scaling, ownership, deploy independence, security boundary.

```csharp
// Strangler Fig — facade that routes between legacy and new
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

## 6. Chapter Group 3 — Communication

### Two styles (per chapter)

Newman organizes inter-service communication along two axes — **synchronous/asynchronous** and **request-response/event-driven**:

|                | Request-response                        | Event-driven                                     |
| -------------- | --------------------------------------- | ------------------------------------------------ |
| **Synchronous** | REST, gRPC                              | (rare — call-and-wait-for-event is unusual)     |
| **Asynchronous** | Message-queue request/reply             | Pub/sub events (the most common form)            |

### When to use which

| Style                       | Use when                                                         | Avoid when                                                       |
| --------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Synchronous request-response** (REST/gRPC) | Simple read queries; user-facing flows that need immediate feedback. | Chains of 5+ services; failure of any service breaks the chain. |
| **Asynchronous event-driven** | Decoupling producers from consumers; long-running workflows.    | When the caller really needs a synchronous result.              |

### REST vs. gRPC (and others)

The book gives Newman's pragmatic ranking — not "what's best", but "what's appropriate":

* **REST/JSON** — the default for external APIs and most internal calls. Universal tooling, debuggable.
* **gRPC** — strongly-typed contracts, binary protocol, streaming. Good for internal service-to-service.
* **GraphQL** — aggregation BFFs; not a transport per se.
* **SOAP** — legacy, still seen in enterprise.

### Backwards compatibility

The single largest source of bugs in microservices systems is **breaking a consumer**. Newman's rules:

1. **Be liberal in what you accept, strict in what you emit** (Postel's Law).
2. **Add fields, don't change them.** Removing or renaming fields breaks consumers silently.
3. **Version explicitly** — `/v1/orders`, `/v2/orders` — or use a header. Don't break callers without a deprecation period.
4. **Consumer-driven contract tests** (CDC) catch breakage *before* you deploy. The book strongly recommends Pact or similar.

---

## 7. Chapter Group 4 — Workflow: Sagas, Orchestration vs. Choreography

A whole chapter on cross-service business workflows. The fundamental challenge: **in microservices, you cannot use distributed ACID transactions.** 2PC across services is operationally fragile.

### The Saga

> A long-running business workflow is modeled as a **sequence of local transactions**, each in one service, with **compensating transactions** to undo work when something fails downstream.

Two flavors:

| Style              | How it works                                                                       | Pros                                            | Cons                                              |
| ------------------ | ---------------------------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------- |
| **Orchestration**  | A central coordinator (the "saga orchestrator") commands each service in sequence. | Centralized workflow view; explicit logic.      | The orchestrator can become a god service.        |
| **Choreography**   | Services emit events; other services subscribe and act. No central brain.          | Decoupled; resilient; scales well organizationally. | Workflow is implicit, scattered across services; harder to debug. |

Newman doesn't prescribe a winner. He says:
* Use **choreography** for simple, stable, mostly-linear workflows.
* Use **orchestration** for complex workflows with branching logic, retries, and human-in-the-loop steps.

> Cross-reference: [`../dot-net/docs/architectural-pattern/saga.md`](../dot-net/docs/architectural-pattern/saga.md) — the .NET-specific deep-dive (with MassTransit code).

### Compensating actions are *business* actions

A compensation is not a database `ROLLBACK`. It's a **business-level reversal** — refund, cancel, recall. Many real-world operations *cannot* be truly undone (you can't un-charge a credit card, you can refund it). Designing compensations forces you to think about *what failure means to the business*.

### Idempotency is mandatory

In an at-least-once delivery world, every command must be **idempotent** — re-receiving the same command produces the same result, not a duplicate side-effect. The book pairs sagas with the **Outbox / Inbox patterns** for reliable delivery.

> Cross-reference: [`../dot-net/docs/architectural-pattern/outbox.md`](../dot-net/docs/architectural-pattern/outbox.md).

---

## 8. Chapter Group 5 — Build & Deployment

### One repo per service vs. monorepo

Newman is famously even-handed:

* **Repo per service** — clean ownership boundary, simple CI, no cross-service refactor noise. Default for most teams.
* **Monorepo** — easier cross-service refactor, atomic commits across services, harder to enforce service independence. Works well at very large companies with strong tooling (Google, Meta).

The choice matters less than the *discipline*. A monorepo with bad discipline produces tightly-coupled services that *must* deploy together — a distributed monolith with extra steps.

### CI/CD

Mandatory. Newman's bar:

* Every commit on `main` triggers a build.
* Build artifact is **the** artifact — same artifact deployed to staging and prod.
* Automated tests run on every commit.
* Production deploys are pushed automatically (or with a single-click promotion).
* Every service has its own pipeline.

### Deployment options

The book sweeps the landscape — physical hosts, VMs, containers, Kubernetes, serverless — and lands on:

> Default to **containers on Kubernetes** for stateful or long-running services; consider **serverless** (Lambda/Functions) for event-driven, bursty workloads.

### Deployment patterns

* **Blue/green** — two environments; cut over instantly.
* **Canary** — gradually shift traffic to the new version.
* **Feature toggles** — deploy code dark, enable per-user.
* **Parallel run** — when stakes are high (financial, regulatory), run new and old in parallel and compare.

---

## 9. Chapter Group 6 — Testing

The book updates Mike Cohn's testing pyramid for microservices:

```
                      /\
                     /E2E\          (few, slow, flaky)
                    /------\
                   /Contract\        (consumer-driven contract tests)
                  /----------\
                 /Integration \      (one service + its real DB / brokers)
                /--------------\
               /     Unit       \    (many, fast, isolated)
              /__________________\
```

### Newman's bias

* **Unit tests** should be the majority. Fast, isolated, no infrastructure.
* **Integration tests** that hit a real DB through the service's full stack (using Testcontainers) catch real bugs without the flakiness of E2E.
* **Contract tests** (Pact-style) replace most cross-service integration tests. They run inside each service's CI; nobody needs to spin up other services.
* **End-to-end tests** are slow, flaky, and expensive. Minimize them — keep a handful of critical-path scenarios, accept that they break frequently.

### The "no end-to-end testing" debate

Newman summarizes the controversial position taken by some teams: skip E2E entirely. The book's compromise:

* You probably can't skip E2E entirely.
* But every E2E test is a tax. Add one only when its specific scenario can't be covered by contract + integration tests.
* Run E2E in a separate pipeline, not blocking deploys.

> Cross-reference: [`unit-testing-vladimir-khorikov.md`](./unit-testing-vladimir-khorikov.md) — for what a "good" unit test looks like before you write hundreds of them.

---

## 10. Chapter Group 7 — Observability

In a distributed system, you cannot reproduce most bugs locally. The only way to understand a system in production is by **observing** it. The three pillars:

| Pillar               | What it answers                                                          | Tool examples                              |
| -------------------- | ------------------------------------------------------------------------ | ------------------------------------------ |
| **Logs**             | What happened? (context-rich, per-request)                               | Serilog + Loki/Elasticsearch                |
| **Metrics**          | How is the system performing? (time-series, aggregated)                  | Prometheus, Datadog, App Insights           |
| **Distributed traces** | Where did a request go? (one trace per request, spanning services)     | OpenTelemetry → Jaeger / Tempo / Honeycomb |

### Correlation IDs and trace propagation

Every inbound request gets a **correlation ID** that is propagated through every downstream call (HTTP header, message header). Logs and traces all carry it. When something fails, you grep for the ID and see the entire request flow.

```csharp
// Inbound middleware — set or read the correlation ID
app.Use(async (ctx, next) =>
{
    var correlationId = ctx.Request.Headers["X-Correlation-ID"].FirstOrDefault()
                        ?? Guid.NewGuid().ToString();
    ctx.Response.Headers["X-Correlation-ID"] = correlationId;
    using (LogContext.PushProperty("CorrelationId", correlationId))
        await next();
});
```

### Newman's observability advice

* **Start with structured logging + correlation IDs.** Easiest, biggest immediate payoff.
* **Add OpenTelemetry distributed tracing next.** Once you have one trace per request, debugging cross-service issues stops being guesswork.
* **Metrics for SLOs / SLIs.** Pick a few key indicators (latency P99, error rate, saturation) and alert on them.
* **Sample, don't log everything.** At 10,000 requests/sec, full logging is infeasible.

---

## 11. Chapter Group 8 — Security

### The threat model shifts

Microservices have **far more network surface area** than a monolith. The book sweeps the landscape:

* **Authentication** — who is the caller? (OAuth 2 / OIDC, mutual TLS, signed JWTs)
* **Authorization** — what may the caller do? (Centralized vs. decentralized; service-to-service vs. user-on-behalf-of)
* **Confidentiality** — TLS in transit, encryption at rest, secret management (Key Vault, AWS Secrets Manager).
* **Integrity** — signed messages so downstream services can trust the payload.

### Zero Trust

Newman endorses **Zero Trust** — don't trust the network. Every service-to-service call authenticates and authorizes, even inside the cluster. mTLS via a service mesh (Istio, Linkerd) makes this transparent.

### Don't roll your own crypto

The book is emphatic: use well-supported libraries, never homemade encryption. Identity providers (Keycloak, Auth0, Entra ID) handle the hard part; you wire them in.

---

## 12. Chapter Group 9 — Resilience

> A distributed system is one where the failure of a computer you didn't know existed can render your own computer unusable.
> — Leslie Lamport (quoted approvingly)

### Failure modes

In monoliths, things fail; in microservices, things fail **partially**. You must explicitly design for:

* **Timeouts** — every network call has a timeout, period. Default settings (Java's "no timeout") are deadly.
* **Retries with exponential backoff and jitter** — never retry tightly; you'll cause a thundering herd.
* **Circuit breakers** — when a downstream is unhealthy, stop calling it; check periodically; resume when healthy.
* **Bulkheads** — separate thread pools or connection pools so one slow downstream doesn't starve another.
* **Fallbacks / graceful degradation** — if the recommendation service is down, hide the widget; don't fail the page.
* **Idempotency** — every command must be safe to receive twice.
* **Dead-letter queues** — when a message can't be processed after N attempts, park it for human inspection.

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

Newman recommends — once your system is mature — **deliberately injecting failures** in production. Netflix's Chaos Monkey is the canonical example. Tools: Gremlin, Chaos Mesh, AWS FIS.

---

## 13. Chapter Group 10 — Scaling

### Four axes of scaling (the AKF Scale Cube, attributed to AKF Partners)

| Axis              | Strategy                                                       | Example                                          |
| ----------------- | -------------------------------------------------------------- | ------------------------------------------------ |
| **X — Horizontal duplication** | More instances of the same service behind a load balancer | Run 5 copies of the API; round-robin traffic     |
| **Y — Functional split**       | Split by business capability                              | Extract billing from the monolith                 |
| **Z — Data partitioning (sharding)** | Split data by some attribute                       | Customers A–M on shard 1, N–Z on shard 2          |

Real systems combine all three. Microservices give you Y (functional split) for free; X and Z are still tools you choose per service.

### Caching as a scaling tool

* **Client-side caching** — browsers, mobile apps. HTTP caching headers.
* **CDN** — edge caching for static and semi-static content.
* **Server-side cache** (Redis, Memcached) — hot reads served from memory.
* **Request collapsing** — coalesce duplicate concurrent requests at the cache layer.
* **Cache invalidation** — Newman quotes Phil Karlton: *"There are only two hard things in Computer Science: cache invalidation and naming things."* Treat invalidation as a feature, not an afterthought.

---

## 14. Chapter Group 11 — User Interfaces

The book pushes back on the idea that the UI is a single monolith on top of microservices.

### Backends for Frontends (BFF)

A **BFF** is a thin layer per client type (web, iOS, Android) that aggregates and shapes data from many backend services for that specific client. Web team owns its BFF; mobile team owns its BFF.

Why: a single API trying to serve every client ends up serving none well. Mobile wants fewer round-trips and smaller payloads; web can handle more chattiness.

### Micro-frontends

Newman is cautious. **Micro-frontends** (splitting the UI by team, each owning a vertical slice including UI) work when teams are large enough and discipline is high. For small teams, the operational overhead exceeds the benefit.

---

## 15. Chapter Group 12 — Organization & Conway's Law

> **Conway's Law:** *Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure.*

The book's most-quoted maxim. Implications:

* If your teams are organized by technical layer (UI team, backend team, DBA team), you will produce a layered monolith.
* If your teams are organized by business capability (Orders team, Billing team, Shipping team), you will produce service-oriented architecture *whether you planned to or not*.
* **The inverse Conway maneuver** — restructure the teams *first* to the shape you want the system to take.

### Two-pizza teams

Bezos's heuristic: a team is the right size if two pizzas can feed it (~6–8 people). Newman endorses small, autonomous, cross-functional teams that own a service end-to-end (build, deploy, operate, on-call).

### The Evolutionary Architect

The final chapter. Newman's view of the architect role:

* **Vision** — articulate the technical direction.
* **Empathy** — understand the impact of decisions on the teams who do the work.
* **Collaboration** — the architect is not the lone genius; they're a teammate.
* **Adaptability** — architecture must evolve. Refusing to revisit decisions is a sin.
* **Autonomy** — give teams as much freedom as possible; intervene only when it matters.
* **Govern** — set the rules of the road (security, observability, deployment standards), then get out of the way.

---

## 16. The Hard Question — Should You Use Microservices?

The 2nd edition adds a chapter specifically on this. Newman's flowchart, distilled:

* **Are you in greenfield with a small team and an unclear domain?** → **Build a modular monolith.** Microservices first is the most expensive way to discover your domain boundaries.
* **Do you have a team-size or deploy-frequency problem with an existing monolith?** → Consider extracting bounded contexts. Start with the *most independent* context.
* **Do you have a scaling problem?** → First try vertical scaling, caching, read replicas. Microservices is the *last* tool, not the first.
* **Do you have multiple teams stepping on each other?** → Microservices help, but so does modularization. Try modules first.
* **Do you have operational maturity (CI/CD, observability, container platform, on-call)?** → If no, microservices will hurt more than they help.

**The book's biggest practical lesson:** the right path for *most* organizations is **modular monolith → extract microservices selectively**. Going microservices-first remains, in 2021, the most reliable way to ship a distributed disaster.

---

## 17. Big-Picture Take-Aways

1. **Independent deployability is the defining property.** If you can't, you don't have microservices.
2. **Model services around bounded contexts**, not technical layers.
3. **Each service owns its data.** No shared databases. Period.
4. **Use sagas, not distributed transactions.** Pair them with idempotency and Outbox.
5. **Backwards compatibility is the architect's job.** Add fields, never break consumers.
6. **The testing pyramid still applies** — unit + integration + contract + a sprinkle of E2E.
7. **Observability is mandatory.** Logs + metrics + distributed traces from day one.
8. **Resilience is a design property.** Timeouts, retries, circuit breakers, bulkheads — not afterthoughts.
9. **Conway's Law is real.** Team structure shapes the system. Plan the org as you plan the architecture.
10. **Microservices are a tradeoff.** Don't adopt them for the resume; adopt them when the pain of *not* having them exceeds the operational cost.

---

## 18. How This Book Fits the Repo

| Topic in the book                       | Repo doc                                                                                            |
| --------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Bounded Contexts (modeling)             | [`../dot-net/docs/ddd/strategic-patterns.md`](../dot-net/docs/ddd/strategic-patterns.md)            |
| Sagas (workflow)                        | [`../dot-net/docs/architectural-pattern/saga.md`](../dot-net/docs/architectural-pattern/saga.md)    |
| Outbox / Inbox (reliable messaging)     | [`../dot-net/docs/architectural-pattern/outbox.md`](../dot-net/docs/architectural-pattern/outbox.md) |
| CQRS (read/write split)                 | [`../dot-net/docs/architectural-pattern/cqrs.md`](../dot-net/docs/architectural-pattern/cqrs.md)    |
| Event Sourcing                           | [`../dot-net/docs/architectural-pattern/event-sourcing.md`](../dot-net/docs/architectural-pattern/event-sourcing.md) |
| Event-Driven Architecture (style)       | [`../dot-net/docs/architectural-style/event-driven.md`](../dot-net/docs/architectural-style/event-driven.md) |
| Testing strategy                         | [`unit-testing-vladimir-khorikov.md`](./unit-testing-vladimir-khorikov.md)                          |

---

## 19. Reading Order Recommendations

* **First time:** start with chapters 1–3 (foundations + modeling). Then jump to whatever pain you have right now.
* **Already running microservices:** chapters on workflow, resilience, observability, security pay off immediately.
* **Considering microservices:** read Chapter 1 (definition + benefits/costs) and the "Should you?" chapter (Newman's flowchart) — those alone often save organizations from the wrong adoption.

---

## 20. Common Misreadings

| Misreading                                                          | Reality                                                                                                |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| "Microservices are small services."                                  | Size is a consequence of bounded-context modeling, not a goal. Some services are bigger than others.   |
| "Microservices fix all monolith problems."                          | Microservices add their own problems. A poorly-modularized system becomes a distributed mess.          |
| "Each team picks their own language."                                | Allowed, but expensive — you multiply tooling, observability, hiring, and runtime concerns.            |
| "Two services can share a database to keep things simple."           | They can't. That's not two services; it's one.                                                          |
| "Eventually-consistent means losing data."                          | No — it means accepting a short window where caches lag the source of truth. Done right, no data loss.  |
| "Microservices = REST."                                             | Microservices is a structural choice; REST is one protocol. gRPC, events, and async messaging are equally valid. |
| "Distributed transactions / 2PC fix consistency."                    | 2PC is operationally fragile and rarely works across heterogeneous stores. Use sagas.                  |

---

## 21. References

* Sam Newman — *Building Microservices: Designing Fine-Grained Systems* (2nd ed.), O'Reilly, 2021.
* Sam Newman — *Monolith to Microservices*, O'Reilly, 2019 (companion volume on the *how* of extraction).
* Eric Evans — *Domain-Driven Design*, 2003 (the bounded-context concept Newman builds on).
* Chris Richardson — *Microservices Patterns*, Manning, 2018 (a catalog of named patterns including Saga, Outbox, API Gateway, Strangler Fig).
* Martin Fowler — [microservices.io](https://microservices.io/) and [martinfowler.com/microservices](https://martinfowler.com/microservices/).
* Cross-reference in this repo: [`../dot-net/docs/architectural-pattern/`](../dot-net/docs/architectural-pattern/), [`../dot-net/docs/architectural-style/event-driven.md`](../dot-net/docs/architectural-style/event-driven.md), [`../dot-net/docs/ddd/strategic-patterns.md`](../dot-net/docs/ddd/strategic-patterns.md).
