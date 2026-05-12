# Five Architectures — Comparison & How to Mix Them

> A side-by-side look at the five architectures covered in this docs folder, plus a practical guide to **combining** them in real .NET applications.
>
> The first half compares; the second half tells you which mixes work, which don't, and how to choose.

> **Source docs:**
> * [`n-layer-architecture.md`](./n-layer-architecture.md)
> * [`n-tier-architecture.md`](./n-tier-architecture.md)
> * [`clean-architecture.md`](./clean-architecture.md)
> * [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)
> * [`hexagonal-architecture.md`](./hexagonal-architecture.md)
> * [`architecture-comparison.md`](./architecture-comparison.md) — the earlier three-way comparison

---

## Table of Contents

### Part A — Comparison

1. [TL;DR — They Live on Different Axes](#1-tldr--they-live-on-different-axes)
2. [One-Sentence Definitions](#2-one-sentence-definitions)
3. [Which Axis Each One Sits On](#3-which-axis-each-one-sits-on)
4. [Big Side-by-Side Table](#4-big-side-by-side-table)
5. [Visual Recap](#5-visual-recap)
6. [Key Points per Architecture](#6-key-points-per-architecture)
7. [Advantages Matrix](#7-advantages-matrix)
8. [Disadvantages Matrix](#8-disadvantages-matrix)

### Part B — Mixing Architectures

9. [Why Mix at All?](#9-why-mix-at-all)
10. [The Mixing Mental Model — Orthogonal vs. Competing](#10-the-mixing-mental-model--orthogonal-vs-competing)
11. [The Compatibility Matrix](#11-the-compatibility-matrix)
12. [Canonical Combinations (Recipes)](#12-canonical-combinations-recipes)
13. [Anti-Mixes — Combinations to Avoid](#13-anti-mixes--combinations-to-avoid)
14. [Mixing Inside vs. Across Bounded Contexts](#14-mixing-inside-vs-across-bounded-contexts)
15. [Decision Flow — Pick Your Stack](#15-decision-flow--pick-your-stack)
16. [Step-by-Step Recipes for Real Apps](#16-step-by-step-recipes-for-real-apps)
17. [Migration Paths Between Mixes](#17-migration-paths-between-mixes)
18. [Final Recommendation for .NET in 2026](#18-final-recommendation-for-net-in-2026)
19. [References](#19-references)

---

# PART A — COMPARISON

## 1. TL;DR — They Live on Different Axes

> **The most important point: these five are not five interchangeable choices.** They answer different questions.

| Architecture        | Axis                                        | Asks                                                       |
| ------------------- | ------------------------------------------- | ---------------------------------------------------------- |
| **N-Layer**         | **Logical organization** (horizontal)       | *How is the code organized in projects/folders?*           |
| **Vertical Slice**  | **Logical organization** (vertical)         | *How is the code organized by feature?*                    |
| **N-Tier**          | **Physical deployment**                     | *Where does each piece run?*                               |
| **Clean**           | **Design principles** (prescriptive)        | *Which dependencies are allowed? How strictly?*            |
| **Hexagonal**       | **Design principles** (older, less prescriptive) | *How is the core isolated from external technologies?*  |

You pick **one** point on each axis. A real app is a *combination*, not a single architecture.

---

## 2. One-Sentence Definitions

- **N-Layer** — Stack of horizontal layers (Presentation → Business → Data), top-down dependencies.
- **N-Tier** — Multiple physical processes/machines that communicate over a network.
- **Clean Architecture** — Concentric rings (Entities ← Use Cases ← Adapters ← Frameworks) with strict **inward** dependencies; framework and DB are "details".
- **Vertical Slice** — Organize by feature, not by layer; one folder per use case.
- **Hexagonal (Ports & Adapters)** — Core surrounded by ports; every external system plugs in via an adapter; the core depends on nothing external.

---

## 3. Which Axis Each One Sits On

```
                          PRINCIPLES axis
                                ▲
                                │ ┌─────────────────┐
                                │ │ Clean (strict)  │
                                │ │ Hexagonal       │
                                │ └─────────────────┘
                                │
                                │
   ────── LOGICAL axis ─────────┼───────── PHYSICAL axis ──────
   ┌───────────────┐            │            ┌─────────────┐
   │ N-Layer       │            │            │ N-Tier      │
   │ Vertical Slice│            │            │             │
   └───────────────┘            │            └─────────────┘
                                │
```

- **N-Layer** and **Vertical Slice** are *alternatives on the logical axis* (you pick one or hybridize).
- **Clean** and **Hexagonal** are *neighbors on the principles axis* — Hexagonal is the conceptual ancestor; Clean is the more prescriptive synthesis.
- **N-Tier** is *orthogonal* to all four — any of them can be deployed in 1, 2, 3, or N tiers.

---

## 4. Big Side-by-Side Table

| Dimension                  | **N-Layer**                       | **Vertical Slice**                          | **N-Tier**                                     | **Clean**                                                       | **Hexagonal**                                |
| -------------------------- | --------------------------------- | ------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------- |
| **Year / Origin**          | 1990s industry practice           | ~2018 (Jimmy Bogard)                        | 1990s industry practice                        | 2012 / 2017 book (Uncle Bob)                                    | ~2005 (Alistair Cockburn)                    |
| **Axis**                   | Logical (horizontal)              | Logical (vertical, by feature)              | Physical                                       | Principles (rings + use cases)                                  | Principles (ports + adapters)                |
| **Boundary type**          | Project / namespace               | Folder per feature                          | Process / network                              | Concentric ring                                                 | Port (interface)                             |
| **Communication**          | In-process method calls           | In-process via MediatR                      | HTTP / gRPC / queue                            | In-process, DTOs across rings                                   | Through port interfaces                      |
| **Dependency direction**   | Top → down                        | None across slices                          | Caller tier → callee tier                      | Outer → inner (Dependency Rule)                                 | Adapter → Core (always inward)               |
| **Domain placement**       | Middle layer                      | Shared or per-slice                         | Inside app tier                                | Innermost ring (center)                                         | Inside the hexagon                           |
| **Framework coupling**     | Allowed in any layer              | Allowed inside slice                        | Allowed in each tier                           | Forbidden in inner rings                                        | Forbidden in core; adapters only             |
| **Test approach**          | Mock layers below                 | Test slice end-to-end                       | Hard E2E across tiers                          | Pure domain unit tests + boundary integration                   | **In-memory fake adapters** (no mocks)       |
| **Setup complexity**       | Low                               | Low                                         | Medium-High                                    | Medium-High                                                     | Medium-High                                  |
| **Onboarding curve**       | Easy                              | Easy                                        | Moderate                                       | Steep                                                           | Moderate-Steep                               |
| **Refactor cost**          | Medium (cross-layer changes)      | Low (per-slice)                             | High (tier contracts)                          | Medium-Low (isolated rings)                                     | Medium-Low (isolated core)                   |
| **Best for**               | Small/medium CRUD                 | Many independent features                   | Scale, security, fault isolation               | Long-lived complex domains                                      | Multi-channel apps, integration-heavy        |
| **Anti-fit**               | Complex domains                   | Domain-heavy invariants                     | Small apps                                     | Tiny prototypes / pure CRUD                                     | Tiny CRUD; teams new to DI                   |
| **Boilerplate**            | High (many cross-layer files)     | Low                                         | High (cross-tier infra)                        | High (more projects, interfaces, DTOs)                          | Medium-High (port + adapter per tech)        |
| **Prescriptiveness**       | Loose                             | Loose                                       | Loose                                          | **Very prescriptive**                                           | Loose (only the boundary rule)               |

---

## 5. Visual Recap

```
N-LAYER (horizontal stack)               VERTICAL SLICE (per-feature)
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


N-TIER (physical)                         CLEAN (concentric)              HEXAGONAL (ports & adapters)
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

## 6. Key Points per Architecture

### N-Layer
- Top-down layered stack, in-process.
- Forbids "layer skipping".
- Familiar; default Visual Studio templates encourage it.
- Risk: anemic domain, god services.

### Vertical Slice
- One folder per use case; everything lives together.
- "Minimize coupling between slices, maximize coupling within a slice." (J. Bogard)
- Pairs with MediatR + pipeline behaviors + FluentValidation.
- Tolerates duplication; refactor via rule of three.

### N-Tier
- Physical separation across processes/machines.
- Independent scaling, security isolation, fault isolation.
- Each tier hop = latency + a new failure mode.
- Resilience, distributed tracing, mTLS are mandatory.

### Clean Architecture
- **The Dependency Rule** is non-negotiable: dependencies point inward only.
- Domain at the center, frameworks at the edge.
- Database and UI are "details".
- Pairs naturally with DDD, CQRS, MediatR.

### Hexagonal (Ports & Adapters)
- Conceptual ancestor of Onion and Clean.
- **Ports** owned by the core; **adapters** implement them.
- Symmetric: driving adapters in, driven adapters out.
- Killer feature: in-memory **fake adapters** for instant Core tests.

---

## 7. Advantages Matrix

| Benefit                            | N-Layer | Vertical Slice | N-Tier | Clean | Hexagonal |
| ---------------------------------- | :-----: | :------------: | :----: | :---: | :-------: |
| Easy to learn                       | ✅     | ✅            | —      | —     | —         |
| Fast first-feature delivery        | ✅     | ✅            | —      | —     | —         |
| High cohesion per feature          | —      | ✅✅          | —      | ✅    | ✅        |
| Low coupling between features      | —      | ✅✅          | —      | ✅    | ✅        |
| Easy feature deletion              | —      | ✅✅          | —      | —     | —         |
| Independent scaling                | —      | —             | ✅✅  | —     | —         |
| Security/fault isolation           | —      | —             | ✅✅  | —     | —         |
| Framework independence             | —      | —             | —      | ✅✅  | ✅✅      |
| DB independence                    | —      | —             | —      | ✅✅  | ✅✅      |
| Multiple driving channels           | —      | —             | ✅     | ✅    | ✅✅      |
| Pure unit-testable domain          | —      | —             | —      | ✅✅  | ✅✅      |
| Replaceable tech per adapter        | —      | ✅            | ✅     | ✅    | ✅✅      |
| Pairs with DDD                      | —      | —             | —      | ✅✅  | ✅✅      |
| Tooling-friendly defaults          | ✅✅   | ✅            | ✅     | —     | —         |

---

## 8. Disadvantages Matrix

| Drawback                                | N-Layer | Vertical Slice | N-Tier | Clean | Hexagonal |
| --------------------------------------- | :-----: | :------------: | :----: | :---: | :-------: |
| Cross-layer changes touch many files   | ✅     | —              | —      | ✅    | ✅        |
| Anemic domain risk                      | ✅     | ✅             | —      | —     | —         |
| Code duplication across features       | —      | ✅             | —      | —     | —         |
| Heavy boilerplate / many projects      | —      | —              | ✅     | ✅    | ✅        |
| Steep learning curve                    | —      | —              | ✅     | ✅    | ✅        |
| Network latency at every hop           | —      | —              | ✅✅   | —     | —         |
| Distributed failure modes               | —      | —              | ✅✅   | —     | —         |
| Eventual consistency burden             | —      | —              | ✅✅   | —     | —         |
| Hard to enforce mechanically           | —      | ✅             | ✅     | —     | —         |
| Over-engineered for small apps         | —      | —              | ✅     | ✅    | ✅        |
| Risk of leaky abstractions              | ✅     | ✅             | —      | ✅    | ✅        |

---

# PART B — MIXING ARCHITECTURES

## 9. Why Mix at All?

> No single architecture answers every question. A real app needs to organize code (logical), decide where it runs (physical), and define which dependencies are allowed (principles). **Mixing is the rule, not the exception.**

The five architectures fit together because they answer different questions:

- **What goes where in the codebase?** → N-Layer **or** Vertical Slice (or both).
- **Where does it run physically?** → N-Tier (always — even "1 tier" is a choice).
- **Who is allowed to depend on whom?** → Clean (or Hexagonal).

A typical modern .NET app picks one option from each row.

---

## 10. The Mixing Mental Model — Orthogonal vs. Competing

| Pair                          | Relationship    | Can you use both?                                                            |
| ----------------------------- | --------------- | ---------------------------------------------------------------------------- |
| **N-Layer ↔ Vertical Slice**  | **Competing**   | Mostly *one or the other*. Hybrid possible inside a Clean outer shell.       |
| **Clean ↔ Hexagonal**         | **Synonyms**    | Same essence. Pick one vocabulary. Using both names for the same code is fine. |
| **Clean ↔ N-Layer**           | **Complementary** | Yes — Clean is the *principles*, N-Layer is the *implementation*.            |
| **Clean ↔ Vertical Slice**    | **Complementary** | Yes — the famous **"Clean Slice"** hybrid.                                   |
| **Hexagonal ↔ N-Layer**       | **Complementary** | Yes — layers *inside* the hexagon's core.                                    |
| **Hexagonal ↔ Vertical Slice**| **Complementary** | Yes — slices *inside* the hexagon's core.                                    |
| **N-Tier ↔ anything**         | **Orthogonal**  | **Always yes.** N-Tier is about deployment; everything else is about code.   |

> **Rule of thumb:**
> - **Pick one** from {N-Layer, Vertical Slice}.
> - **Pick one** from {Clean, Hexagonal} as your principle set (or skip and stay un-principled).
> - **Pick a tier count** based on scale/security needs.

---

## 11. The Compatibility Matrix

| Combine                                   | Compatibility | Notes                                                                |
| ----------------------------------------- | :-----------: | -------------------------------------------------------------------- |
| N-Layer + N-Tier                          | ✅ ✅        | Classic enterprise mix. Each tier holds N layers.                    |
| Clean + N-Layer                           | ✅ ✅        | Modern .NET default. Layers = the 4 concentric rings.                |
| Clean + Vertical Slice ("Clean Slice")    | ✅ ✅        | Most-recommended modern .NET layout in 2025/2026.                    |
| Clean + N-Tier                            | ✅ ✅        | Clean code organization; tiers handle scale & security.              |
| Hexagonal + Vertical Slice                | ✅ ✅        | Slices live inside the hexagon's core.                                |
| Hexagonal + N-Tier                        | ✅ ✅        | Each tier can be its own hexagon (very common in microservices).      |
| Hexagonal + N-Layer                       | ✅           | Layers organize the core; ports + adapters bound the edges.          |
| Clean + Hexagonal (both)                  | ⚠️           | Redundant — same essence, two vocabularies. Use one name.            |
| N-Layer + Vertical Slice                  | ⚠️           | Conflicts unless you split: layers for shared, slices for features.  |
| Vertical Slice + N-Tier                   | ✅           | Slices in one tier; tiers connect over network.                      |
| Clean + Hexagonal + Vertical Slice + N-Tier | ✅         | Realistic modern app — but make sure each is pulling its weight.     |

> ✅ ✅ = canonical, ✅ = works, ⚠️ = possible but watch for redundancy/conflict.

---

## 12. Canonical Combinations (Recipes)

### 🔥 Recipe 1 — "Classic Enterprise" (Pre-2018 default)

**Mix:** N-Layer + N-Tier.

```
Presentation (Web tier)
 └── Application (App tier)
      └── Domain (App tier)
           └── Data Access (App tier ↔ DB tier)
```

- **Use when:** legacy systems, traditional ERP / CRM / LOB apps.
- **Don't use when:** the domain is rich or change is frequent.

### 🔥 Recipe 2 — "Modern Clean" (Current default)

**Mix:** Clean + N-Layer + N-Tier.

```
src/
├── Domain/            ← Entities + Value Objects (innermost)
├── Application/       ← Use cases + abstractions
├── Infrastructure/    ← EF Core + adapters
└── Web/               ← ASP.NET Core (outermost)

Deployed across 3 tiers: UI / API / DB
```

- **Use when:** medium-to-large business apps with long lifespan.
- **Best fit:** banking, healthcare, logistics, SaaS platforms.

### ⭐ Recipe 3 — "Clean Slice" (Recommended in 2026)

**Mix:** Clean + Vertical Slice + N-Tier.

```
src/
├── Domain/                           ← Clean's innermost ring
├── Application/
│   ├── Features/                     ← Vertical slices live here
│   │   ├── Orders/
│   │   │   ├── PlaceOrder/
│   │   │   └── CancelOrder/
│   │   └── Customers/
│   └── Abstractions/                 ← Ports / interfaces
├── Infrastructure/                   ← Adapters (EF, SMTP, MQ)
└── Web/                              ← Endpoints, one per feature
```

- Clean enforces the **dependency rule** outside.
- Vertical Slice gives **feature cohesion** inside.
- Deployed across **3 tiers** (UI / API / DB).

> Strong default for new .NET apps in 2026. Best of both worlds.

### ⭐ Recipe 4 — "Hexagonal Slice"

**Mix:** Hexagonal + Vertical Slice + N-Tier.

```
src/
├── Core/                             ← The hexagon
│   ├── Features/                     ← Slices
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

- Same as Clean Slice but with **explicit ports + adapters projects**.
- **Use when:** the app must support **multiple driving channels** (HTTP + CLI + scheduled jobs) or has heavy external-system integration.

### ⭐ Recipe 5 — "Microservices Hexagon"

**Mix:** Hexagonal + Vertical Slice + Many N-Tiers (microservices).

- Each microservice is a small **Hexagon** with its own Core, ports, adapters.
- **Inside** each service, use Vertical Slice.
- **Across** services, communicate via async messaging or gRPC.

```
Order Service (hexagon)  ─┐
Billing Service (hexagon) ─┤  Service Bus / gRPC
Shipping Service (hexagon)─┘
```

- **Use when:** multiple teams own bounded contexts; scale needs vary per service.
- **Cost:** highest operational overhead — only justified at scale or team boundaries.

### Recipe 6 — "Vertical Slice First" (Greenfield prototype)

**Mix:** Vertical Slice + N-Tier (2 tiers).

```
src/
├── Api/
│   ├── Features/
│   │   ├── PlaceOrder/
│   │   ├── ListProducts/
│   │   └── …
│   └── Program.cs
└── Domain/                  ← optional, shared entities
```

- **Use when:** small new APIs, MVPs, or hackathon projects.
- Skips Clean/Hexagonal until complexity emerges.
- **Migration path:** introduce Clean rings only when the domain matures.

### Recipe 7 — "Plain N-Layer in 2 Tiers" (Smallest sensible app)

**Mix:** N-Layer + N-Tier (2 tiers).

```
src/
├── Web/
├── Business/
└── Data/
```

- **Use when:** internal CRUD admin tools, throwaway portals.
- Plenty good. Don't over-engineer.

---

## 13. Anti-Mixes — Combinations to Avoid

| Anti-mix                                                             | Why it hurts                                                                              |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Clean + Hexagonal naming both** in the same codebase                | Redundant. Same essence, two vocabularies → confused team. Pick one and stick to it.       |
| **Vertical Slice while keeping `Services/` and `Repositories/`** folders | You re-create N-Layer next to slices → cross-cutting coupling everywhere.              |
| **N-Tier without statelessness in the API tier**                      | Sticky sessions kill horizontal scaling — you paid the cost without getting the benefit. |
| **Clean Architecture with EF entities in the Domain**                 | Violates Dependency Rule. You "have Clean" only on paper.                                |
| **N-Tier with a shared DB across services**                           | Distributed monolith. Worst of both worlds.                                              |
| **Hexagonal where ports live in the adapter project**                 | Inverts the dependency arrow — adapter ends up "owning" the contract.                    |
| **Vertical Slice with a "shared service" called from every slice**    | Defeats slice isolation. Use shared **entities** or **events**, not shared services.     |
| **Mixing N-Layer (horizontal) and Vertical Slice without rules**      | Some features touch layers; others are slices; no consistency. Pick one or define the split. |
| **Adopting Clean for a tiny CRUD admin app**                          | Ceremony > value. Use plain N-Layer.                                                    |
| **Microservices for a 2-team startup**                                | Operational complexity overwhelms the value. Modular monolith first.                     |

---

## 14. Mixing Inside vs. Across Bounded Contexts

A useful refinement: **different bounded contexts can use different architectures**.

- A **complex domain** context (e.g., `Pricing`, `Underwriting`) → Clean Architecture + DDD.
- A **CRUD-heavy** context (e.g., `Catalog admin`, `User profile`) → plain N-Layer or Vertical Slice.
- An **integration-heavy** context (e.g., `External Sync`) → Hexagonal with many adapters.
- A **report/read** context → CQRS read model with raw SQL / Dapper, ignoring most patterns.

> **Don't impose one architecture on every part of the system. The right shape depends on the cost of change in *that* corner of the codebase.**

---

## 15. Decision Flow — Pick Your Stack

```
                Is the app tiny / short-lived?
                            │
              ┌──── yes ────┴──── no ────┐
              │                          │
   Plain N-Layer (3 layers,             Is the domain complex /
   1–2 tiers, no Clean,                 long-lived?
   no MediatR ceremony)                       │
                                ┌─────── yes ─┴─── no ──────┐
                                │                            │
                       Clean (or Hexagonal)        Many independent features?
                       + DDD where it pays                   │
                                │                ┌──── yes ──┴── no ───┐
                                │                │                     │
                                ▼          Vertical Slice          N-Layer
                       Multiple driving        + Clean shell        + N-Tier
                       channels?               + N-Tier
                            │
                  ┌─── yes ─┴─── no ──┐
                  │                    │
              Hexagonal           Clean (Clean Slice)
              + Vertical Slice    + N-Tier
              + N-Tier
                            │
                  Many bounded contexts
                  + independent teams?
                            │
                  ┌─── yes ─┴─── no ──┐
                  │                    │
              Microservices       Stay monolithic /
              Hexagon per service modular monolith
```

---

## 16. Step-by-Step Recipes for Real Apps

### 🛠 "Internal admin CRUD app" (1 dev, 3-month lifespan)

```
Pick: N-Layer (3 layers) + N-Tier (2 tiers)
Skip: Clean, Hexagonal, Vertical Slice
Use:  ASP.NET Core MVC + EF Core + SQL Server
```

### 🛠 "Greenfield SaaS API" (4 devs, multi-year horizon)

```
Pick: Clean Slice (Clean + Vertical Slice) + N-Tier (3 tiers)
Stack: ASP.NET Core minimal APIs + MediatR + FluentValidation + EF Core + Redis
Tests: WebApplicationFactory + Testcontainers + ArchUnitNET
```

### 🛠 "Enterprise banking platform" (15 devs, decades)

```
Pick: Hexagonal + Clean (rings inside hexagon) + Vertical Slice + Microservices
       (each microservice is its own hexagon)
Stack: ASP.NET Core + gRPC + Service Bus + SQL Server + Redis + Kafka
Discipline: DDD bounded contexts, CQRS, event sourcing in the write side,
            Outbox pattern, distributed tracing, mTLS between tiers
```

### 🛠 "Modernizing a legacy WebForms monolith"

```
Today:    N-Layer (4 layers) + N-Tier (3 tiers)
Step 1:   Strangler Fig — wrap legacy behind a Clean facade
Step 2:   Introduce Clean rings for new features
Step 3:   Move new features to Vertical Slice within the Clean shell
Step 4:   Extract bounded contexts when each can stand alone
```

---

## 17. Migration Paths Between Mixes

```
Plain N-Layer                                          Microservices Hexagon
       │                                                       ▲
       │                                                       │
       ▼                                                       │
N-Layer + Clean (modern monolith)                              │
       │                                                       │
       ▼                                                       │
Clean Slice (Clean + Vertical Slice)                           │
       │                                                       │
       ▼                                                       │
Hexagonal Slice (add explicit ports/adapters projects)         │
       │                                                       │
       ▼                                                       │
Modular Monolith (bounded contexts in one process)             │
       │                                                       │
       └──────────────────────────────────────────────────────►┘
                       (extract by bounded context)
```

Each step is a **bounded refactor**, not a rewrite. Move in this order:

1. **Plain → Clean:** introduce `Domain` and `Application` projects; move logic from services into entities.
2. **Clean → Clean Slice:** rearrange `Application` by feature folders.
3. **Clean Slice → Hexagonal Slice:** extract adapter assemblies; define ports next to slices.
4. **Hexagonal Slice → Modular Monolith:** split into modules with module boundaries enforced by `internal` and project references.
5. **Modular Monolith → Microservices:** extract modules into separate services only when team and scale boundaries justify it.

**Don't skip steps.** Going straight from plain N-Layer to microservices is the most expensive failure mode in industry.

---

## 18. Final Recommendation for .NET in 2026

> **Default starting stack for a new .NET app:**
>
> 1. **Clean Architecture** as the design philosophy.
> 2. **Vertical Slice feature folders** inside `Application/Features/`.
> 3. **3-tier deployment** (UI / API / DB).
> 4. **MediatR + FluentValidation + Result\<T\>** for the use-case layer.
> 5. **EF Core** in `Infrastructure/`, never elsewhere.
> 6. **Architecture tests** (NetArchTest / ArchUnitNET) in CI to enforce the Dependency Rule.
> 7. Add **Hexagonal adapter projects** if the app must support multiple driving channels (HTTP + CLI + workers + tests).
> 8. **Microservices** *only* when bounded contexts and team boundaries demand it.

For small or short-lived apps, **plain 3-layer N-Layer in 2 tiers** is the right choice. Don't over-engineer.

> The five architectures aren't ranked. They answer different questions. A great .NET system answers all of them:
> - *How is the code organized?* → N-Layer **or** Vertical Slice
> - *Where does it run?* → N-Tier
> - *Which dependencies are allowed?* → Clean **or** Hexagonal

Pick one from each row. That is your stack.

---

## 19. References

- [`n-layer-architecture.md`](./n-layer-architecture.md)
- [`n-tier-architecture.md`](./n-tier-architecture.md)
- [`clean-architecture.md`](./clean-architecture.md)
- [`vertical-slice-architecture.md`](./vertical-slice-architecture.md)
- [`hexagonal-architecture.md`](./hexagonal-architecture.md)
- [`architecture-comparison.md`](./architecture-comparison.md) — previous three-way comparison (preserved)
- Robert C. Martin — *Clean Architecture* (2017)
- Alistair Cockburn — [Hexagonal Architecture (Ports & Adapters)](https://alistair.cockburn.us/hexagonal-architecture)
- Jimmy Bogard — [Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
- Microsoft Learn — [N-tier architecture style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- Microsoft Learn — [Common web application architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)
- [N-Layered vs Clean vs Vertical Slice (antondevtips)](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
