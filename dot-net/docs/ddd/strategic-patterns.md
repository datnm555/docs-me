# Strategic DDD Patterns

> The **system-level** patterns of Domain-Driven Design. They answer: *where do the boundaries go, what vocabulary do we use, and how do contexts relate?* This is the half of DDD that pays for itself even if you never adopt a single tactical pattern.

---

## Quick Reference (What · Why · When · Where)

- **What** — The system-level half of DDD: **Ubiquitous Language** (one vocabulary per context), **Bounded Context** (explicit boundary for one model), **Context Map** (diagram of how contexts relate), **Anti-Corruption Layer** (translate when models don't fit), plus the nine context-mapping relationships (Partnership, Shared Kernel, Customer/Supplier, Conformist, ACL, Open Host Service, Published Language, Separate Ways, Big Ball of Mud).
- **Why** — Tactical patterns work inside *one* model. The moment you have two teams, multiple services, or legacy systems, you have multiple models that will conflict. Strategic design names the boundaries and the relationships between them.
- **When** — Multi-team systems; companies with shared terminology that means different things to different departments; integration with legacy or third-party systems; deciding microservices boundaries.
- **Where** — Above tactical patterns; informs both *which* services to build (one per Bounded Context) and *how* they integrate (Context Map). Pair with Sam Newman's microservices modeling.

---

## Why Strategic Comes First

In a small app, you can get away with tactical DDD (Aggregates, Value Objects) without strategic. But the moment two teams ship together — or a system grows past one model — strategic DDD becomes the lifeline:

* Two teams using the word **"Customer"** to mean different things → integration bugs forever.
* One giant model spanning the whole company → no one understands all of it; small changes cause distant breakages.
* No anti-corruption layer at a legacy boundary → legacy ideas seep into the new system.

Strategic DDD makes the boundaries **explicit, named, and intentional**.

---

## Index

1. [Ubiquitous Language](#1-ubiquitous-language)
2. [Bounded Context](#2-bounded-context)
3. [Context Map](#3-context-map)
4. [Anti-Corruption Layer (ACL)](#4-anti-corruption-layer-acl)
5. [Other Context-Mapping Relationships](#5-other-context-mapping-relationships)
6. [Putting It All Together](#6-putting-it-all-together)

---

## 1. Ubiquitous Language

A **Ubiquitous Language** is a shared vocabulary used by **everyone** working on a Bounded Context — domain experts, developers, testers, PMs.

* In conversations.
* In tickets and docs.
* In code: class names, method names, variable names.
* In tests: scenario descriptions.

If the business calls it a "Policy" and the code calls it `InsuranceContract`, every conversation has translation friction. Every translation is an opportunity for misunderstanding.

### What it looks like in practice

| Domain expert says…              | Bad code                          | Good code                                |
| -------------------------------- | --------------------------------- | ---------------------------------------- |
| "Underwrite a policy"            | `policies.Update(status = "OK")`  | `policy.Underwrite(underwriter, today)`  |
| "The claim is in adjudication"   | `claim.StatusCode = 3`            | `claim.IsInAdjudication`                 |
| "Settle the claim"               | `claim.Finalize()`                | `claim.Settle(amount, settledOn)`         |

Code becomes **executable specification** — readable by a domain expert.

### Rules

* **One Ubiquitous Language per Bounded Context.** "Customer" can mean different things in Sales vs. Support — and that's *correct*.
* **Maintain a glossary** for each context. A README in each service is enough.
* **Refactor names ruthlessly** when the business's vocabulary shifts.

---

## 2. Bounded Context

A **Bounded Context** is the **explicit boundary inside which a particular model and language are consistent**. Outside the boundary, those words may mean something else.

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

These are **two different concepts** that share a label. Forcing them into one shared `Customer` class — across departments — is a classic enterprise mistake. The model serves no one well.

### How to identify Bounded Contexts

* **Linguistic boundaries** — where does the same word change meaning?
* **Team boundaries** — what does each team own end-to-end?
* **Business capability boundaries** — Sales, Billing, Shipping, Inventory are typical splits.
* **Subdomain boundaries** — Core / Supporting / Generic (see Evans).

### .NET expression

Often **one microservice = one Bounded Context**. In a modular monolith, **one module = one Bounded Context**. In both cases:

```
SalesContext.csproj            SupportContext.csproj
 ├── Customer.cs                 ├── Customer.cs
 ├── Quote.cs                    ├── Ticket.cs
 └── ...                         └── ...
```

Two `Customer` classes, in different namespaces/projects. **This is correct**, not a mistake to deduplicate.

---

## 3. Context Map

A **Context Map** is a diagram of the **Bounded Contexts in your system and how they relate**. It is the single most valuable artifact in strategic DDD.

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

Each arrow is **labelled** with the relationship type. This is not optional — the *type* of relationship dictates the integration design.

---

## 4. Anti-Corruption Layer (ACL)

When you must integrate with a context whose model **doesn't fit yours** — a legacy system, a third-party API, a partner whose domain is genuinely different — build an **Anti-Corruption Layer**.

```
┌────────────────────┐     ACL      ┌──────────────────────┐
│   Your Context     │  ◀─────────▶ │   Their Context       │
│   Clean model      │  translates  │   Messy/Different     │
└────────────────────┘              └──────────────────────┘
```

The ACL **translates** between models. It is a one-way protection: the messy model is **not allowed** into your code.

### C# Sample

```csharp
// Your clean domain model
public sealed record Address(string Street, string City, string PostalCode, string Country);

// Legacy SOAP API exposes this monstrosity
public class LegacyAddressDto
{
    public string? AddressLine1 { get; set; }
    public string? AddressLine2 { get; set; } // sometimes contains city + postal mashed together
    public string? CityField   { get; set; }
    public string? ZipPostCode { get; set; }
    public int     CountryCode { get; set; } // numeric ISO code
}

// ACL — translates between the two worlds and isolates the mess
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

**Key point:** the rest of the application uses `Address` cleanly. The mess lives **only** in `LegacyAddressAcl`. If the legacy system changes, only this class changes.

ACLs are usually a **service** in microservices land or a **dedicated namespace/project** in a monolith.

---

## 5. Other Context-Mapping Relationships

Evans named several relationships between contexts. Knowing them gives you a vocabulary for integration decisions.

| Relationship                    | Meaning                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------- |
| **Partnership**                 | Two teams succeed or fail together; close cooperation on a shared model.                    |
| **Shared Kernel**               | A small subset of the model is shared between contexts. Coordination cost — use sparingly.   |
| **Customer / Supplier**         | Upstream provides; downstream consumes. Downstream has a voice in upstream's priorities.    |
| **Conformist**                  | Downstream accepts upstream's model as-is, no translation. Cheap but couples you.            |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model to protect itself. Used when models genuinely differ. |
| **Open Host Service**           | Upstream provides a published protocol/API for many downstream consumers.                    |
| **Published Language**          | A well-documented shared language used between contexts (e.g. an event schema registry).     |
| **Separate Ways**               | Two contexts don't integrate at all. Sometimes the right call.                              |
| **Big Ball of Mud**             | A context with no clear model. Best handled with an ACL between you and it.                  |

The most important pair for new work: **Open Host Service + Published Language** for the upstream; **Customer/Supplier or ACL** for the downstream.

---

## 6. Putting It All Together

A real strategic DDD design for an e-commerce platform:

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

Each box:

* Has its **own Ubiquitous Language**.
* Has its **own model** — a `Customer` in Sales is not the same shape as in Billing.
* Is **independently deployable** (microservice) or **independently modifiable** (monolith module).
* **Communicates by explicit contracts** — events, API calls, ACLs.

The **Core** contexts (Pricing, Sales) get heavy tactical-DDD investment. The **Generic** (Catalog — basically off-the-shelf) and **Supporting** (Billing, Fulfillment — important but not differentiating) get just enough modeling.

---

## Common Anti-Patterns

* **Shared "common" model** — one `User`, `Customer`, `Product` class used across the whole company. Couples every team to every other team.
* **Centralized "Enterprise Data Model"** — same problem at the data layer.
* **Implicit boundaries** — boundaries that "exist in our heads". They don't.
* **No anti-corruption when integrating with legacy** — legacy ideas creep into the new code and you inherit the mess.
* **Microservices without Bounded Contexts** — you get the operational cost of distribution without the design benefit.

---

## TL;DR

* **Ubiquitous Language** — same words, code, conversations.
* **Bounded Context** — explicit boundary for one model.
* **Context Map** — drawing of all contexts and how they relate.
* **Anti-Corruption Layer** — translation between contexts that don't fit each other.

You can get 80% of DDD's value from these four ideas alone, even if you never write an Aggregate.

---

## References

* Eric Evans — *Domain-Driven Design* (2003). Part IV (the strategic half — often skipped, the most valuable).
* Vaughn Vernon — *Implementing Domain-Driven Design* (2013). Chapters 2–4.
* Vlad Khononov — *Learning Domain-Driven Design* (2021). Modern, accessible take.
* DDD Crew — [github.com/ddd-crew](https://github.com/ddd-crew) — Context Maps templates, EventStorming guides.
