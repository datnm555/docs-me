# Domain-Driven Design (DDD)

> DDD is a **methodology** for tackling complex software, not a design pattern. It is a way of thinking about software design that aligns code with the *domain* — the business problem being solved. Inside DDD live many *patterns* (Aggregate, Repository, Bounded Context, …), but **DDD itself is not a pattern**.

---

## Is DDD a Design Pattern?

**No.**

* **Design pattern** (GoF) = a reusable solution to a recurring class-level problem.
* **DDD** = a discipline, a vocabulary, and a set of practices for designing software around a complex domain.

DDD is at the same level as **TDD**, **BDD**, **Agile** — a way of working. Inside DDD live:

* **Strategic patterns** — how to split a large system: Bounded Context, Context Map, Ubiquitous Language, Anti-Corruption Layer.
* **Tactical patterns** — how to model inside one Bounded Context: Aggregate, Entity, Value Object, Domain Event, Domain Service, Repository, Factory.

So when someone says *"we use DDD"*, they mean *"we apply DDD practices, and some subset of its strategic + tactical patterns."*

---

## The Two Halves of DDD

### Strategic DDD — *how do we slice the system?*

Concerned with **the big picture**: where do the boundaries go, what language do we use, how do contexts communicate?

* **Ubiquitous Language** — shared vocabulary between developers and domain experts.
* **Bounded Context** — an explicit boundary inside which a model and its language are consistent.
* **Context Map** — a diagram of how Bounded Contexts relate (partnership, customer/supplier, conformist, anti-corruption layer, etc.).
* **Anti-Corruption Layer (ACL)** — a translation layer that prevents another context's model from polluting ours.

→ See [`strategic-patterns.md`](./strategic-patterns.md)

### Tactical DDD — *how do we model inside one Bounded Context?*

Concerned with **the code**: what objects exist, what their responsibilities are, how they enforce invariants.

* **Entity** — has identity that persists over time (`Order`, `Customer`).
* **Value Object** — immutable, equality by value (`Money`, `Email`, `Address`).
* **Aggregate / Aggregate Root** — a consistency boundary; one root that controls access to the rest.
* **Domain Event** — *"something happened"* that the domain cares about (`OrderPlaced`).
* **Domain Service** — domain behavior that doesn't naturally belong to an entity.
* **Factory** — encapsulates complex aggregate construction.
* **Repository** — persistence access for aggregate roots (see [`../enterprise-pattern/repository.md`](../enterprise-pattern/repository.md)).

→ See [`tactical-patterns.md`](./tactical-patterns.md)

---

## What DDD Is **Not**

* **Not** the same as Clean Architecture or Onion. (Though they pair extremely well.)
* **Not** required by microservices. (Though one Bounded Context per service is a useful default.)
* **Not** "use Repository and Aggregate". You can have those without doing DDD.
* **Not** about technology. The tactical patterns are language-agnostic.

> *"The DDD parts are not the patterns. The patterns are easy. The hard part is the strategic design."* — **Eric Evans**

---

## The Three Pillars

1. **Focus on the core domain.** Spend modeling effort where the business differentiates itself; use off-the-shelf solutions for the rest (auth, payment, mailing).
2. **Explore models in a creative collaboration of domain practitioners and software practitioners.** Talking to domain experts is non-negotiable.
3. **Speak a Ubiquitous Language within an explicitly Bounded Context.** Code, conversations, diagrams, tests — all use the same terms.

---

## When to Use DDD

✅ **Use DDD when:**

* The domain is **complex** and **central** to the business (banking, insurance, logistics, healthcare, scheduling).
* Business logic is unlikely to fit in a CRUD model.
* The domain will **evolve** over years — long-term modeling investment pays off.
* You have access to **domain experts** willing to talk.

❌ **Don't use DDD when:**

* It's a CRUD app. A controller → service → repository → DB stack is fine.
* The domain is well-understood and commodity (a basic admin tool, a content site).
* You don't have domain experts you can talk to.
* The team is uncomfortable with the upfront cost and discipline.

---

## DDD + Other Concepts in This Doc Folder

DDD is a methodology — it **composes** with the patterns and styles elsewhere:

| Concept                                 | Relationship to DDD                                                       |
| --------------------------------------- | ------------------------------------------------------------------------- |
| **Clean / Hexagonal / Onion**           | Standard companion architectural style. Keeps the domain at the center.    |
| **CQRS**                                | Frequently applied *within* a Bounded Context.                            |
| **Event Sourcing**                      | Optional but natural fit — events are first-class in DDD.                  |
| **Microservices**                       | A microservice often = one Bounded Context.                                |
| **Saga**                                | Cross–Bounded-Context coordination.                                       |
| **Repository**                          | Tactical DDD pattern (also a Fowler enterprise pattern).                  |
| **GoF Design Patterns**                 | Tools you use *inside* the domain (Strategy, Factory, Visitor, …).         |

---

## Reading Order

1. **[`strategic-patterns.md`](./strategic-patterns.md)** — start here. Bounded Contexts and Ubiquitous Language are the most valuable DDD ideas, regardless of whether you adopt tactical patterns.
2. **[`tactical-patterns.md`](./tactical-patterns.md)** — then the code-level patterns.
3. **[`../enterprise-pattern/repository.md`](../enterprise-pattern/repository.md)** — deep dive on Repository in a DDD context.
4. **[`../architectural-style/hexagonal-onion-clean.md`](../architectural-style/hexagonal-onion-clean.md)** — how DDD code is typically organized.

---

## References

* Eric Evans — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003). The original.
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). The practical companion.
* Vaughn Vernon — *Domain-Driven Design Distilled* (2016). The short version.
* Martin Fowler — [*DomainDrivenDesign*](https://martinfowler.com/tags/domain%20driven%20design.html).
* Steve Smith — *Domain-Driven Design Fundamentals* (Pluralsight).
