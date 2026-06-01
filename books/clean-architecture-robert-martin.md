# Clean Architecture — Summary

> **Robert C. Martin (Uncle Bob)** — *Clean Architecture: A Craftsman's Guide to Software Structure and Design*, Prentice Hall, 2017.

> **Why this book matters:** the book that codified what most modern .NET, Java, and TypeScript teams now call "Clean / Onion / Hexagonal." Uncle Bob synthesizes ~50 years of structured, object-oriented, and functional programming into one **dependency-rule-driven** architecture, then shows what's wrong when the rule is broken. The famous concentric-circle diagram lives here.

> **This summary covers:** the programming-paradigm history, the SOLID refresher, the six **component principles** (REP/CCP/CRP and ADP/SDP/SAP), the **dependency rule**, the four-ring architecture diagram, the Humble Object pattern, what counts as a "detail" (databases, web, frameworks), and the philosophical bits at the end. Code examples are mine, illustrative — see the book for the canonical examples.

> 🇻🇳 Vietnamese version: [`clean-architecture-robert-martin-vi.md`](./clean-architecture-robert-martin-vi.md)

---

## Quick Reference (What · Why · When · Where)

- **What** — A digest of Uncle Bob's 2017 book — the Dependency Rule, the four-ring architecture diagram, the six component principles (REP/CCP/CRP/ADP/SDP/SAP), the Humble Object pattern, and the claim that the database, web, and framework are all "details".
- **Why** — Gives you a *first-principles* argument for inversion-driven layered design, so you can defend (or rebuild) architecture decisions instead of cargo-culting a folder layout.
- **When** — Long-lived systems where the cost of change is high; modernizing a legacy monolith; deciding what to mock and what to test; arguing with someone who thinks "the database is the foundation".
- **Where** — Pair with `dot-net/docs/architectural-style/clean-architecture.md` (the .NET implementation), `principles/solid.md` (the load-bearing principle), and `books/domain-driven-design-eric-evans.md` (the natural domain-modeling companion).

---

## Table of Contents

1. [About the Book](#1-about-the-book)
2. [Uncle Bob's Argument in One Sentence](#2-uncle-bobs-argument-in-one-sentence)
3. [Part I — Introduction: What Is Design and Architecture?](#3-part-i--introduction-what-is-design-and-architecture)
4. [Part II — Starting with the Bricks: Programming Paradigms](#4-part-ii--starting-with-the-bricks-programming-paradigms)
5. [Part III — Design Principles: SOLID](#5-part-iii--design-principles-solid)
6. [Part IV — Component Principles](#6-part-iv--component-principles)
7. [Part V — Architecture](#7-part-v--architecture)
8. [The Clean Architecture Diagram](#8-the-clean-architecture-diagram)
9. [The Dependency Rule](#9-the-dependency-rule)
10. [Boundaries — Logical, Local, and Full](#10-boundaries--logical-local-and-full)
11. [The Humble Object Pattern](#11-the-humble-object-pattern)
12. [Part VI — Details: Database, Web, Frameworks](#12-part-vi--details-database-web-frameworks)
13. [Screaming Architecture](#13-screaming-architecture)
14. [Test Boundaries](#14-test-boundaries)
15. [Clean Embedded Architecture](#15-clean-embedded-architecture)
16. [Big-Picture Take-Aways](#16-big-picture-take-aways)
17. [How This Book Fits Other Books](#17-how-this-book-fits-other-books)
18. [Reading Order Recommendations](#18-reading-order-recommendations)
19. [Common Misreadings](#19-common-misreadings)
20. [References](#20-references)

---

## 1. About the Book

* **Title:** *Clean Architecture: A Craftsman's Guide to Software Structure and Design*
* **Author:** Robert C. Martin ("Uncle Bob")
* **Publisher:** Prentice Hall (Pearson), September 2017
* **Length:** ~432 pages
* **Audience:** mid-to-senior engineers, tech leads, and architects who want a *first-principles* defense of layered, dependency-inverted architectures.
* **Lineage:** the natural sequel to *Clean Code* (2008) and *The Clean Coder* (2011); the entire "Clean" trilogy.

The book is a synthesis — it pulls Cockburn's Hexagonal, Palermo's Onion, Boundaries, Use Cases, DDD, and 50 years of programming-paradigm research into a single coherent argument. The most concrete output is **one diagram** (the four concentric rings) and **one rule** (the Dependency Rule).

---

## 2. Uncle Bob's Argument in One Sentence

> The goal of software architecture is to **minimize the human resources required to build and maintain the required system** — and the way to achieve that is to keep **policy** (high-value, slow-changing business rules) **independent of details** (volatile, low-value technology choices).

Everything else in the book is corollary. The Dependency Rule, the rings, the boundaries — they exist to *protect policy from details* so a team can change one without disturbing the other.

---

## 3. Part I — Introduction: What Is Design and Architecture?

* **Architecture = the *shape* you give a system.** It is not separate from design; "architecture" and "design" describe the same thing at different scales.
* **The goal is to maximize productivity** over the lifetime of the system. A system "wins" if the cost of change stays flat over time.
* **The two values** a software system provides:
  * **Behavior** — what the system does (the urgent).
  * **Structure** — its ability to change (the important).
  * Behavior is urgent; structure is important. Most teams prioritize the urgent and starve the important until structure collapses.
* **Eisenhower's matrix** applied to software: do important-not-urgent work *first* (structure), or you'll spend the rest of your career firefighting urgent-important work that became urgent because you ignored the important.

---

## 4. Part II — Starting with the Bricks: Programming Paradigms

Uncle Bob argues there are only **three** programming paradigms ever invented, and **each is a constraint** — not a capability.

| Paradigm        | Year  | What it **forbids**                                  | Why it matters                          |
| --------------- | ----- | ---------------------------------------------------- | --------------------------------------- |
| **Structured**  | 1968  | Direct transfer of control (`goto`)                  | Enables decomposition and proof          |
| **Object-Oriented** | 1966 | Indirect transfer of control (function pointers as plain data) | Enables polymorphism — and **dependency inversion** |
| **Functional**  | 1957 / 1958 | Assignment (mutable state)                      | Enables concurrency safety              |

The lesson is provocative: **paradigms remove power; they don't add it.** The discipline each enforces is what makes systems maintainable.

> Why this matters for architecture: **OO's gift is polymorphism**, which lets you draw a software boundary across which **the direction of source-code dependency is opposite to the direction of runtime control flow**. That single capability makes Clean Architecture possible.

---

## 5. Part III — Design Principles: SOLID

A brisk refresher (the book assumes prior familiarity):

| Letter | Principle                       | Concise statement                                                            |
| ------ | ------------------------------- | ---------------------------------------------------------------------------- |
| **S**  | Single Responsibility           | A module should have one and only one reason to change — i.e., one *actor*.   |
| **O**  | Open/Closed                     | Open for extension, closed for modification — driven by polymorphism.        |
| **L**  | Liskov Substitution             | Subtypes must be substitutable behaviorally, not just structurally.          |
| **I**  | Interface Segregation           | Don't depend on what you don't use; prefer many focused interfaces.          |
| **D**  | Dependency Inversion            | Source code dependencies point only to **abstractions**, not concretions.    |

Important re-framing in this book:

* **SRP is about actors, not "things"**. A class violates SRP when two *different humans* would ever request changes to it. The book's classic example: an `Employee` class with `calculatePay`, `reportHours`, `save` — three different stakeholders, three different reasons to change.
* **Dependency Inversion is the load-bearing pillar of all of Clean Architecture.** Everything else (boundaries, the rings, the dependency rule) is a structural consequence.

→ Cross-reference: [`../principles/solid.md`](../principles/solid.md) for a full SOLID write-up.

---

## 6. Part IV — Component Principles

SOLID applies to *classes*. **Component principles** apply to deployable units (DLLs, JARs, npm packages). Uncle Bob lists six:

### Component Cohesion (what goes inside a component)

| Principle                                       | What it asks                                                                  |
| ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **REP** — Reuse / Release Equivalence           | A component is the unit of reuse *and* the unit of release. Version it.       |
| **CCP** — Common Closure Principle              | Group classes that change together for the same reason. (SRP at component level.) |
| **CRP** — Common Reuse Principle                | Don't force consumers to depend on classes they don't use. (ISP at component level.) |

The three pull in different directions — Uncle Bob's **tension diagram** shows that satisfying all three perfectly is impossible. Teams prioritize differently as a project matures.

### Component Coupling (how components relate)

| Principle                                       | What it asks                                                                  |
| ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **ADP** — Acyclic Dependencies Principle        | The component dependency graph must be a **DAG**. No cycles, ever.            |
| **SDP** — Stable Dependencies Principle         | Depend in the direction of stability. A component with many dependents is stable; one with few dependents and many dependencies is volatile. |
| **SAP** — Stable Abstractions Principle         | A stable component should be **abstract**. A volatile component can be concrete. |

### The Main Sequence

Plot every component on a 2D chart: x = instability (0 stable → 1 unstable), y = abstractness (0 concrete → 1 abstract). Components should sit near the diagonal **y = 1 - x**.

* **Zone of Pain** (stable + concrete, bottom-left): unchangeable concrete code (`java.util.String`-style). Hard to extend; if it stops fitting, you're stuck.
* **Zone of Uselessness** (unstable + abstract, top-right): interfaces nobody depends on. Pure noise.
* **Main Sequence** (the diagonal): the sweet spot.

This single chart is one of the most-quoted ideas in the book — many static-analysis tools (NDepend, SonarQube) report distance-from-main-sequence as a metric.

---

## 7. Part V — Architecture

The longest section. Themes:

### What architecture is *for*

* To **defer decisions** — the *date you must choose your database* should be late in the project, not early.
* To **keep options open** — the architect's job is to ensure the most painful decisions remain reversible as long as possible.
* The architect produces **shape**; the shape's job is to make policy independent of details.

### Independence

A well-architected system is independent in several senses:

* **Use-case independence** — adding a new use case shouldn't ripple through the system.
* **Operability** — UI, batch, scheduled, distributed deployments should be configurable.
* **Development independence** — multiple teams should be able to work in parallel without stepping on each other.
* **Deployability** — pieces should be deployable independently.

### Boundaries (the central idea)

A **boundary** is a line in the source code across which dependencies are *one-directional*. The book devotes multiple chapters to drawing them right:

* **Boundary anatomy** — typically an interface plus a pair of DTOs (one for input, one for output).
* **Boundary crossing** — data crossing a boundary is converted to the form most convenient for the inner module. **Inner modules never know the form of outer modules.**
* **Plugin architecture** — outer modules plug into inner ones. The inner module defines an interface; the outer module implements it. **The "main" component (composition root) wires them.**

### Policy and Level

* **Policy** — business rules. Slow-changing, high-value.
* **Detail** — frameworks, databases, UIs. Fast-changing, low-value.
* **Level** = distance from input and output. A use case is *high-level*; an HTTP controller is *low-level*. Source-code dependencies flow from low level → high level.

---

## 8. The Clean Architecture Diagram

The book's iconic graphic. Four concentric rings:

```
                ┌──────────────────────────────────────────────┐
                │ Frameworks & Drivers  (UI, DB, Web, Devices) │
                │ ┌──────────────────────────────────────┐     │
                │ │ Interface Adapters  (Controllers,    │     │
                │ │   Gateways, Presenters)              │     │
                │ │ ┌──────────────────────────────┐     │     │
                │ │ │ Application Business Rules   │     │     │
                │ │ │   (Use Cases / Interactors)  │     │     │
                │ │ │ ┌─────────────────────┐      │     │     │
                │ │ │ │ Enterprise Business │      │     │     │
                │ │ │ │   Rules (Entities)  │      │     │     │
                │ │ │ └─────────────────────┘      │     │     │
                │ │ └──────────────────────────────┘     │     │
                │ └──────────────────────────────────────┘     │
                └──────────────────────────────────────────────┘

           Source-code dependencies point INWARD only ─────►
```

### The four rings

| Ring                          | What lives here                                                              | Volatility       |
| ----------------------------- | ---------------------------------------------------------------------------- | ---------------- |
| **Entities** (innermost)      | Enterprise-wide business rules. Pure objects with methods.                    | Slowest          |
| **Use Cases**                 | Application-specific business rules. Orchestrate Entities.                    | Slow             |
| **Interface Adapters**        | Controllers, presenters, gateways — translate between use cases and outside. | Faster           |
| **Frameworks & Drivers** (outermost) | Web framework, ORM, third-party SDKs, browser code.                   | Fastest          |

### Boundary crossing — input boundary, output boundary

The book draws **arrows for runtime control flow** (which can point either way) and **arrows for source-code dependency** (which must point inward). When control needs to leave the inner ring, the inner ring defines an **output port** (interface) that the outer ring implements.

A typical use-case interaction:

```csharp
// Use case ring — defines the contract it needs (output port)
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Use case itself
public sealed class PlaceOrderUseCase
{
    private readonly IOrderRepository _orders;
    public PlaceOrderUseCase(IOrderRepository orders) => _orders = orders;

    public async Task<Guid> Execute(PlaceOrderRequest req, CancellationToken ct)
    {
        var order = Order.Place(req.CustomerId, req.Items);
        await _orders.AddAsync(order, ct);
        return order.Id;
    }
}

// Interface Adapters ring — implements the port
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => _db.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
    public Task AddAsync(Order order, CancellationToken ct) => _db.AddAsync(order, ct).AsTask();
}

// Composition Root (outermost) — the ONLY place that wires concrete to abstract
builder.Services.AddDbContext<AppDbContext>(...);
builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<PlaceOrderUseCase>();
```

`PlaceOrderUseCase` knows nothing about EF Core, ASP.NET, or HTTP. It can be unit-tested with an in-memory repository in microseconds.

→ Cross-reference: [`../dot-net/docs/architectural-style/clean-architecture.md`](../dot-net/docs/architectural-style/clean-architecture.md) for the .NET implementation deep-dive.

---

## 9. The Dependency Rule

> **Source code dependencies must point only inward, toward higher-level policies.**

Stated negatively: **names declared in an outer ring must not be mentioned in an inner ring.** Not in code, not in comments, not in type signatures.

When a programmer in the inner ring needs to invoke code in an outer ring, they don't. Instead:

1. They declare an **interface** in the inner ring describing what they need.
2. The outer ring **implements** that interface.
3. The composition root wires the two together at startup.

This is **Dependency Inversion** in its most architectural form. At the moment of compilation, the dependency arrow is reversed from the natural control-flow arrow:

```
Control flow:        Use Case  ──►  IRepository  ──►  EfRepository
Source dependency:   Use Case  ◄────IRepository  ◄────EfRepository
                                    (inner ring)     (outer ring)
```

Once you internalize that picture, everything else in the book follows.

---

## 10. Boundaries — Logical, Local, and Full

You don't always need full process-level boundaries. The book lists levels of separation, ordered by cost:

| Boundary type            | What it is                                                          | Cost           |
| ------------------------ | ------------------------------------------------------------------- | -------------- |
| **Source-level**         | Just interfaces and DTOs — same project, same process                | Low            |
| **Deployment-level**     | Separate DLL / JAR / package — still same process                    | Medium         |
| **Thread-level**         | Different threads — shared memory, no IPC                            | Medium         |
| **Local process**        | Separate processes on the same machine — IPC needed                  | High           |
| **Service-level**        | Different machines — network protocol                                | Highest        |

**Start with source-level boundaries**, then harden as pain appears. Going straight to microservices ("full service boundaries from day one") is the most common over-engineering mistake.

---

## 11. The Humble Object Pattern

The trick that makes the Clean Architecture *testable*: split anything hard to test into two pieces:

* A **humble** piece — only the bits that interact with the world (UI rendering, DB connection, network I/O, threading). Minimal logic, untested or hand-tested.
* A **testable** piece — the logic, pure. Easily unit-tested.

Examples:

* A controller is a humble object; the use case handler it dispatches to is testable.
* A `DbContext` is a humble object; the domain code that mutates entities is testable.
* A WPF/Blazor view is a humble object; the view model / presenter is testable.

```csharp
// Humble: the controller. Only HTTP translation lives here.
[ApiController, Route("orders")]
public sealed class OrdersController : ControllerBase
{
    private readonly PlaceOrderUseCase _useCase;
    public OrdersController(PlaceOrderUseCase useCase) => _useCase = useCase;

    [HttpPost]
    public async Task<IActionResult> Post(PlaceOrderRequest req, CancellationToken ct)
    {
        var id = await _useCase.Execute(req, ct);
        return CreatedAtAction(nameof(GetById), new { id }, null);
    }
}

// Testable: the use case. Pure logic. Unit tests don't need a web host.
public sealed class PlaceOrderUseCase { /* ... see earlier ... */ }
```

The Humble Object pattern is how you **honor the dependency rule** at the framework boundary without sacrificing testability.

---

## 12. Part VI — Details: Database, Web, Frameworks

The most surprising and most-quoted chapters.

### The Database Is a Detail

The database is **not** the foundation of the architecture. It's an I/O device. The business rules don't care whether data lives in SQL Server, Postgres, Cosmos DB, S3, or a CSV file. The fact that the industry conflates "application" and "the database it talks to" is a historical accident, not a design truth.

Practical consequence: **define your repository interfaces in the Use-Case ring**, implement them in Infrastructure. The database becomes pluggable.

### The Web Is a Detail

The web framework — ASP.NET, Spring MVC, Express, Rails — is just one of many possible delivery mechanisms. The business rules should be deliverable through HTTP, gRPC, a CLI, a scheduled job, or a test harness, with no changes.

Practical consequence: **the Use-Case ring takes plain inputs and returns plain outputs.** Controllers translate; they don't compute.

### Frameworks Are Details

A framework is a tool you **use**, not a base class you inherit from. The moment a domain entity inherits from `DbContext`-aware classes, you have **married the framework**. Divorce will be painful and expensive.

> Uncle Bob's marriage analogy: *"Frameworks are powerful and very useful. But they are not architectures. They are tools. Don't fall in love with frameworks. Don't marry them. Stay sceptical. Adopt frameworks, but keep them at arm's length."*

Practical consequence: **don't reference framework types from your Entities or Use Cases.** Frameworks live in the outer ring. Always.

---

## 13. Screaming Architecture

> *"Your architecture should scream the intent of the system."*

If a stranger opens your source tree, the top-level folder names should describe **what the system does**, not **what framework it uses**:

❌ A framework-screaming layout:
```
src/
  Controllers/
  Services/
  Models/
  Repositories/
```

✅ A domain-screaming layout:
```
src/
  Ordering/
  Billing/
  Shipping/
  Inventory/
```

A health-records system should scream "Healthcare", not "ASP.NET Core". The architecture is meant to communicate **purpose** first, *technology* second.

---

## 14. Test Boundaries

Tests are part of the system, and they obey the same dependency rule:

* **Unit tests** test code in the inner rings — they don't need infrastructure.
* **Integration tests** test the outer rings interacting with real I/O.
* **Functional/E2E tests** drive the whole stack from outside.

Two pitfalls the book warns about:

* **Fragile tests.** If tests break every time you refactor, the tests are coupled to *implementation*, not behavior. The fix: test through the boundary interfaces, not through internal classes.
* **The "Testing API" trap.** Some teams expose internals just for tests. Resist. The test should drive the public API; if testing is hard, the API is wrong.

> Cross-reference: [`unit-testing-vladimir-khorikov.md`](./unit-testing-vladimir-khorikov.md) — Khorikov's book is the natural companion read on test design.

---

## 15. Clean Embedded Architecture

A chapter that surprises people who think "embedded" means "no architecture." Uncle Bob argues that embedded systems are *especially* in need of Clean Architecture because **the firmware tends to outlive the hardware** by years or decades. The same dependency-inversion discipline applies:

* The HAL (hardware abstraction layer) is a boundary.
* The OS is a detail.
* Drivers are details.
* Application logic is the policy.

Lesson generalizes: *every* system has details that change faster than its policy. Protect the policy.

---

## 16. Big-Picture Take-Aways

1. **Architecture is the shape of the system.** That shape exists to maximize the team's productivity over the project's lifetime.
2. **Behavior is urgent; structure is important.** Most teams sacrifice structure to urgency, then drown in technical debt.
3. **Policy must be independent of details.** This is the entire point.
4. **The Dependency Rule** — dependencies point inward — is the operational consequence of (3).
5. **OO's gift is polymorphism**, which is how we invert source-code dependencies against control flow.
6. **SOLID describes classes; the six component principles describe components.** Together they make Clean Architecture mechanically possible.
7. **Boundaries are interfaces + DTOs.** They live in the inner ring; the outer ring implements them.
8. **The database, the web, the UI, and the framework are all details.** Defer them. Keep them at arm's length.
9. **The Humble Object pattern** isolates the un-testable bits so the rest of the system can be unit-tested.
10. **Architecture should *scream* the business**, not the technology.
11. **You will not get this right on day one.** Architecture is iterative. The best architects *defer* decisions until the latest responsible moment.

---

## 17. How This Book Fits Other Books

| Book                                                         | Relationship                                                                 |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| Eric Evans — *Domain-Driven Design* (Blue Book, 2003)        | Complementary. DDD gives you the **what** (modeling the domain); Clean Architecture gives you the **where** (which ring it lives in). |
| Alistair Cockburn — *Hexagonal Architecture* (2005)          | Direct ancestor. Hexagonal is Clean Architecture with less prescription about internal layering. |
| Jeffrey Palermo — *Onion Architecture* (2008)                | Same essence as Clean. Different vocabulary.                                  |
| Vaughn Vernon — *Implementing Domain-Driven Design* (2013)   | Practical companion that uses Clean-style layering in every example.         |
| Martin Fowler — *PoEAA* (2002)                                | Older. Some of its patterns (Repository, Unit of Work) live in Clean Architecture's Interface Adapters ring. |
| Robert C. Martin — *Clean Code* (2008)                       | Class-level cleanliness. *Clean Architecture* is the structural counterpart. |
| Khorikov — *Unit Testing* (2020)                              | Tells you how to test the inner rings well — explicit on what to mock and what not to. |

---

## 18. Reading Order Recommendations

* **First time:** read straight through. Parts II–IV (paradigms, SOLID, component principles) feel slow but pay off in Part V.
* **Reference mode:** Part V (Architecture) and the chapters on Database / Web / Frameworks as details are the most-cited sections.
* **Pair with:** the `architectural-style/clean-architecture.md` in this repo for a concrete .NET implementation. They reinforce each other — book gives the *why*, repo doc gives the *how*.

---

## 19. Common Misreadings

People who skim the book often draw the wrong conclusions. The book's actual position:

| Misreading                                                             | Reality                                                                                                    |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| "Clean Architecture requires four projects named Entities/UseCases/Adapters/Frameworks." | Names don't matter. The Dependency Rule matters. The four-ring diagram is a *concept*, not a template. |
| "Clean Architecture forbids EF Core / Hibernate / Spring Data."        | No. It forbids those *in the Entities and Use Cases rings*. They live in Infrastructure (the outermost ring). |
| "Clean Architecture is the same as DDD."                                | DDD models the *domain*; Clean Architecture places *layers*. They compose well but solve different problems. |
| "All applications need Clean Architecture."                             | No. Tiny CRUD apps and short-lived prototypes are over-engineered by it. The book is explicit: pay structural cost when the cost of change is high. |
| "Microservices = Clean Architecture."                                   | No. Microservices is a deployment strategy. A microservice can be Clean-architected or be a tangled mess. |

---

## 20. References

* Robert C. Martin — *Clean Architecture: A Craftsman's Guide to Software Structure and Design*, Prentice Hall, 2017.
* Robert C. Martin — *Clean Code: A Handbook of Agile Software Craftsmanship*, Prentice Hall, 2008.
* Alistair Cockburn — [*Hexagonal Architecture*](https://alistair.cockburn.us/hexagonal-architecture/) (2005).
* Jeffrey Palermo — [*The Onion Architecture* (blog series)](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/) (2008).
* Jason Taylor — [Clean Architecture Solution Template](https://github.com/jasontaylordev/CleanArchitecture).
* Microsoft — [eShopOnWeb reference application](https://github.com/dotnet-architecture/eShopOnWeb).
* Cross-reference in this repo: [`../dot-net/docs/architectural-style/clean-architecture.md`](../dot-net/docs/architectural-style/clean-architecture.md).
