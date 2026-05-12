# Book Summaries

> Concentrated notes from influential software engineering books — the essential ideas, the canonical examples, and the take-aways you can quote in code reviews.

Each summary is opinionated: it captures **what the book teaches**, **how the author teaches it** (the running examples), and **the principles or rules** distilled from the chapters. Not a substitute for reading the book — a *navigation map* and a *recall aid*.

---

## Index

| Book                                                                                  | Authors                          | Year | Focus                                                                | File                                                  |
| ------------------------------------------------------------------------------------- | -------------------------------- | ---- | -------------------------------------------------------------------- | ----------------------------------------------------- |
| **Head First Design Patterns** (2nd ed.)                                              | Eric Freeman & Elisabeth Robson  | 2020 | The GoF design patterns taught through running examples in Java.     | [`head-first-design-patterns.md`](./head-first-design-patterns.md) |
| **Clean Architecture**                                                                | Robert C. Martin (Uncle Bob)     | 2017 | The Dependency Rule, the four-ring diagram, and what counts as a "detail". | [`clean-architecture-robert-martin.md`](./clean-architecture-robert-martin.md) |
| **Building Microservices** (2nd ed.)                                                  | Sam Newman                       | 2021 | Definition, modeling, sagas, deployment, testing, observability, and when *not* to use microservices. | [`building-microservices-sam-newman.md`](./building-microservices-sam-newman.md) |
| **Unit Testing: Principles, Practices, and Patterns**                                 | Vladimir Khorikov                | 2020 | Four pillars of a good unit test, mocks vs. stubs, when to integrate. | [`unit-testing-vladimir-khorikov.md`](./unit-testing-vladimir-khorikov.md) |
| **Domain-Driven Design** ("The Blue Book")                                            | Eric Evans                       | 2003 | Ubiquitous Language, tactical patterns, Bounded Contexts, Context Maps. | [`domain-driven-design-eric-evans.md`](./domain-driven-design-eric-evans.md) |

### Suggested reading order

If you're starting from scratch and want to internalize the canon, a coherent path is:

1. **OOP fundamentals** → [`../oop/`](../oop/).
2. **Principles** (DRY, SOLID, GRASP) → [`../principles/`](../principles/).
3. **Head First Design Patterns** — class-level patterns, made memorable.
4. **Clean Architecture** — structural rules that organize those patterns.
5. **Domain-Driven Design** — what to put inside the structure (the *what*).
6. **Unit Testing** — how to test what you've built well.
7. **Building Microservices** — only when you have a real driver to split.

*(More book summaries can be added here as the folder grows.)*

---

## How to Use These Summaries

* **First time** with a topic? Read the original book. The summary will make more sense afterward.
* **Already read** the book? The summary is your **recall aid** — re-read it before an interview, a design review, or when starting a new feature.
* **Teaching others?** The summaries lift out the running examples and key code so you can lean on them in slides or whiteboarding.
* **Looking up a pattern?** Each summary's pattern entries follow the same skeleton: intent → running example → code → key points → pitfalls.

---

## Cross-Reference With Other Folders

* **GoF design patterns** with .NET / C# samples: [`../dot-net/docs/design-pattern/`](../dot-net/docs/design-pattern/)
* **Architectural styles** (Clean, Hexagonal, Layered, etc.): [`../dot-net/docs/architectural-style/`](../dot-net/docs/architectural-style/)
* **Architectural patterns** (Saga, CQRS, Event Sourcing, Outbox): [`../dot-net/docs/architectural-pattern/`](../dot-net/docs/architectural-pattern/)
* **Enterprise patterns** (Repository, UoW, IoC/DI): [`../dot-net/docs/enterprise-pattern/`](../dot-net/docs/enterprise-pattern/)
* **OO fundamentals** (the four pillars): [`../oop/`](../oop/)
* **Software-engineering principles** (SOLID, GRASP, DRY): [`../principles/`](../principles/)

The book summaries cite these whenever the book's lesson reinforces or contradicts an idea documented elsewhere in the repo.
