# Two-Phase Commit (2PC) and Try-Confirm-Cancel (TCC)

> Two **synchronous** patterns for distributed transactions. **2PC** offers ACID across resources at the cost of blocking and tight coupling — and is essentially extinct in modern microservices. **TCC** is a lighter, application-level alternative that reserves resources first and commits later. Read this to understand **why** microservices use [Saga](./saga.md) instead.

---

## Quick Reference (What · Why · When · Where · How)

- **What** — **2PC**: a coordinator runs `prepare` against every participant, then `commit` if all voted yes (else `rollback`). **TCC**: each participant exposes three operations — `Try` (reserve), `Confirm` (commit the reservation), `Cancel` (release). A coordinator orchestrates the trio across services.
- **Why** — Both attempt **atomic, all-or-nothing** outcomes across multiple resources or services. Useful framing for understanding why distributed transactions are hard, and why the canonical answer for microservices is *eventual* consistency via [Saga](./saga.md).
- **When** — **2PC**: legacy DTC / XA scenarios, single-vendor enterprise stacks (SQL Server + MSMQ), occasionally for in-process two-database commits. **TCC**: when business semantics naturally express as "reserve then confirm" — travel bookings, ticket sales, inventory reservation in checkout.
- **Where** — Both are **architectural patterns** above the storage layer. 2PC is implemented by the coordinator (XA driver / MSDTC). TCC is implemented at the **service contract** layer — every participant must offer `Try` / `Confirm` / `Cancel` endpoints.
- **How** — 2PC: avoid unless you control both endpoints and can tolerate blocking. TCC: design every participant's API as "reserve → confirm → cancel" with idempotent confirm/cancel. For new microservice work, **default to Saga**; reach for TCC only if "reserve then confirm" is the natural business model.

---

## Why Distributed Transactions Are Hard

A local ACID transaction is one process committing to one storage engine. As soon as you have **two** systems that can fail independently, you're choosing between three constraints — Consistency, Availability, Partition tolerance — and you can have at most two (the CAP theorem).

| Local TX                                  | Distributed TX                                                                |
| ----------------------------------------- | ----------------------------------------------------------------------------- |
| One log, one commit point                 | Multiple logs; commit must be **coordinated**                                  |
| Failure rolls back automatically          | A coordinator crash mid-protocol leaves participants **uncertain**             |
| Atomicity is free                         | Atomicity costs **blocking** and **availability**                              |
| Latency = disk write                      | Latency = slowest-network-hop × number of round trips                          |

Both 2PC and TCC try to recover atomicity at the cost of blocking and coupling. Saga gives up *immediate* atomicity for availability, replacing rollback with **compensating actions**.

---

## Two-Phase Commit (2PC)

A coordinator drives two phases:

```
Phase 1 — Prepare:
   Coordinator: "Are you ready to commit?"
   Participants: write to log, lock resources, reply YES/NO
   Each YES means "I promise I can commit if you tell me to."

Phase 2 — Commit (or Rollback):
   If all YES → coordinator writes commit decision → tells everyone to commit
   If any NO  → coordinator tells everyone to rollback

While participants are waiting between Phase 1 and Phase 2, locks are held.
```

```
        ┌───────────────┐
        │  Coordinator  │
        └───────┬───────┘
                │ Phase 1: PREPARE
       ┌────────┼────────┐
       ▼        ▼        ▼
    [Svc A]  [Svc B]  [Svc C]
       │        │        │
       │ VOTE YES│VOTE YES│VOTE YES
       └────────┼────────┘
                │
                │ Phase 2: COMMIT
       ┌────────┼────────┐
       ▼        ▼        ▼
    [Svc A]  [Svc B]  [Svc C]
       └────────┬────────┘
                ▼
            All committed
```

### What goes wrong

* **Coordinator crashes after Phase 1, before Phase 2** → participants are stuck holding locks until the coordinator recovers (the "in-doubt transaction" problem).
* **Long locks** — across services, lock duration = sum of all latencies. Throughput collapses.
* **Tight coupling** — all participants must speak the same coordinator protocol (XA, MSDTC) and must trust each other to honor the prepare vote.
* **No partial progress** — every participant must be available for the duration.

### When 2PC is still used

* In-process across **two databases owned by the same service** (e.g., SQL Server + MSMQ via MSDTC).
* Legacy mainframe / ESB integrations.
* **Almost never** in greenfield microservice work.

> "Two-phase commit is the anti-availability protocol." — Pat Helland, *Life beyond Distributed Transactions* (2007). The paper that essentially ended 2PC for new distributed systems.

---

## Try-Confirm-Cancel (TCC)

TCC moves the "lock + commit" logic from the transaction manager into the **application protocol**. Every participating service exposes three operations:

| Operation | Effect                                                                            | Idempotent? |
| --------- | --------------------------------------------------------------------------------- | ----------- |
| `Try`     | **Reserve** the resource. Decrement available stock, hold an inventory slot, etc. | Yes (by reservation id) |
| `Confirm` | Commit the reservation. Make the change permanent.                                 | Yes         |
| `Cancel`  | Release the reservation. Restore the state.                                        | Yes         |

A coordinator runs the trio:

```
Phase 1 — Try (parallel):
   coordinator.Try(flight, dates)   → reservation R₁
   coordinator.Try(hotel,  dates)   → reservation R₂
   coordinator.Try(car,    dates)   → reservation R₃

Phase 2:
   If all succeeded: Confirm(R₁), Confirm(R₂), Confirm(R₃)
   If any failed:    Cancel(R₁), Cancel(R₂), Cancel(R₃) for whatever succeeded
```

Key differences from 2PC:

* **No protocol-level locks held by a transaction manager.** Reservations are *business-level* state — they live in the participant's own DB with a TTL.
* **Each operation is a normal HTTP/gRPC call.** No XA driver, no distributed lock manager.
* **Each participant decides** how long it will hold the reservation; a coordinator crash just means the reservation expires.
* **All three operations must be idempotent** (network retries are normal).

### When TCC fits

* The domain *already* speaks "reserve then confirm" — hotels, flights, concert tickets, inventory holds at checkout.
* You control all participating services.
* You can accept the operational cost of three endpoints per resource.

### When TCC does **not** fit

* "Reserve" has no natural meaning ("reserve a database row update"?).
* You don't own the participants — third-party APIs rarely offer a reservation primitive.
* Workflows with > ~5 participants — orchestration becomes painful; prefer a real Saga framework.

---

## C# Sketch — TCC Orchestrator (Travel Booking)

```csharp
public sealed class TravelBookingOrchestrator
{
    private readonly IFlightService _flight;
    private readonly IHotelService _hotel;
    private readonly ICarService _car;
    private readonly ILogger<TravelBookingOrchestrator> _log;

    public async Task<BookingResult> BookAsync(BookingRequest req, CancellationToken ct)
    {
        var sagaId = Guid.NewGuid();
        var reservations = new List<(string Service, Guid ReservationId)>();

        try
        {
            // Phase 1 — Try
            var f = await _flight.TryAsync(sagaId, req.Flight, ct);
            reservations.Add(("flight", f.ReservationId));

            var h = await _hotel.TryAsync(sagaId, req.Hotel, ct);
            reservations.Add(("hotel", h.ReservationId));

            var c = await _car.TryAsync(sagaId, req.Car, ct);
            reservations.Add(("car", c.ReservationId));

            // Phase 2 — Confirm
            await _flight.ConfirmAsync(f.ReservationId, ct);
            await _hotel.ConfirmAsync(h.ReservationId, ct);
            await _car.ConfirmAsync(c.ReservationId, ct);

            return BookingResult.Success(sagaId);
        }
        catch (Exception ex)
        {
            _log.LogWarning(ex, "Booking {SagaId} failed — cancelling reservations", sagaId);

            // Cancel everything we already reserved. MUST be idempotent + retried.
            foreach (var (service, id) in reservations)
                await CancelBestEffort(service, id, ct);

            return BookingResult.Failure(sagaId, ex.Message);
        }
    }

    private async Task CancelBestEffort(string service, Guid id, CancellationToken ct)
    {
        // Cancellation is retry-loop with backoff; it MUST eventually succeed
        // (otherwise we leak a reservation). If it can't, page on-call.
        await Polly.Policy
            .Handle<Exception>()
            .WaitAndRetryAsync(5, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)))
            .ExecuteAsync(ct => service switch
            {
                "flight" => _flight.CancelAsync(id, ct),
                "hotel"  => _hotel.CancelAsync(id, ct),
                "car"    => _car.CancelAsync(id, ct),
                _        => Task.CompletedTask,
            }, ct);
    }
}
```

Each participant looks like:

```csharp
public sealed class FlightService : IFlightService
{
    private readonly AppDbContext _db;

    public async Task<TryResult> TryAsync(Guid sagaId, FlightChoice choice, CancellationToken ct)
    {
        // Idempotent: same sagaId twice returns the same reservation.
        var existing = await _db.Reservations.FirstOrDefaultAsync(r => r.SagaId == sagaId, ct);
        if (existing is not null) return new TryResult(existing.Id);

        // Reserve a seat with a TTL (e.g., 15 minutes — auto-expires if Confirm/Cancel never arrives).
        var reservation = new Reservation
        {
            Id        = Guid.NewGuid(),
            SagaId    = sagaId,
            Status    = ReservationStatus.Reserved,
            ExpiresAt = DateTime.UtcNow.AddMinutes(15),
            FlightId  = choice.FlightId,
        };
        _db.Reservations.Add(reservation);
        await _db.SaveChangesAsync(ct);
        return new TryResult(reservation.Id);
    }

    public Task ConfirmAsync(Guid reservationId, CancellationToken ct) =>
        UpdateStatus(reservationId, ReservationStatus.Confirmed, ct);

    public Task CancelAsync(Guid reservationId, CancellationToken ct) =>
        UpdateStatus(reservationId, ReservationStatus.Cancelled, ct);
}
```

Note the **TTL on the reservation**: if the orchestrator dies between `Try` and `Confirm`, a background job sweeps expired reservations and releases the seat. **The orchestrator is not a single point of failure** in the way a 2PC coordinator is.

---

## Real Project — Travel Booking

A customer books a holiday: flight + hotel + car, all-or-nothing.

```
┌──────────────────────────────────────────────────────────────────┐
│ Booking Orchestrator (TCC)                                       │
│                                                                  │
│   sagaId = uuid()                                                │
│                                                                  │
│   ┌─ Try ────────────────────────────────────────────────┐       │
│   │   FlightSvc.Try(sagaId, FL123, 12/Jun → 19/Jun)       │ → R₁  │
│   │   HotelSvc.Try (sagaId, Bali Resort, 12/Jun → 19/Jun) │ → R₂  │
│   │   CarSvc.Try   (sagaId, SUV, 12/Jun → 19/Jun)          │ → R₃ │
│   └──────────────────────────────────────────────────────┘       │
│           any failure → Cancel(R₁..R_n already created)          │
│                                                                  │
│   ┌─ Confirm ─────────────────────────────────────────────┐      │
│   │   FlightSvc.Confirm(R₁)                                │      │
│   │   HotelSvc.Confirm(R₂)                                 │      │
│   │   CarSvc.Confirm(R₃)                                   │      │
│   └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

Failure scenarios:

| Failure                                  | What happens                                                                 |
| ---------------------------------------- | ---------------------------------------------------------------------------- |
| Hotel sold out → `Try` returns 409        | Orchestrator cancels FlightSvc.R₁; returns "no rooms" to user.               |
| Network blip during CarSvc.Try            | Retry with idempotency key (sagaId); same reservation returned.              |
| Orchestrator crashes after all Try's      | Reservations have TTL 15 min — they auto-cancel; user retries the booking.    |
| Confirm fails midway (FlightSvc down)     | Retry Confirm until success. The reservation is already paid for — committing the others is fine; FlightSvc will eventually confirm. |

Compare to a Saga choreography for the same problem: every participant subscribes to events, no central orchestrator. TCC is closer to **synchronous orchestration with explicit reservations**.

---

## Decision: 2PC vs TCC vs Saga

| Dimension                  | 2PC                      | TCC                              | Saga                                         |
| -------------------------- | ------------------------ | -------------------------------- | -------------------------------------------- |
| Consistency                | Strong (ACID across)     | Strong-ish at confirm time       | Eventual                                     |
| Availability               | Low (blocks on coord)    | Medium                           | High                                         |
| Coupling                   | Very tight (XA / DTC)    | Tight (must agree on Try/Confirm/Cancel API) | Loose (events / messages)            |
| Locks                      | Held by TM, cross-system | Application-level reservations with TTL | None — compensating actions instead   |
| Latency                    | Sum of all participants × 2 round-trips | 1× round trip per phase | 1× per step; can parallelize         |
| Failure mode               | Blocked / in-doubt       | Auto-release via TTL             | Compensate (Cancel/Refund/etc.)              |
| Where to use today         | Same vendor + same process | "Reserve then confirm" business model | Default for microservices             |

> **TL;DR.** For new microservice work, default to **Saga**. Use **TCC** only when "reserve then confirm" is a natural business model (travel, ticketing, inventory holds). **Avoid 2PC** unless you genuinely have a single-vendor + single-process scenario (e.g., MSDTC for two local resources).

See [`saga.md`](./saga.md) for the canonical Saga implementation.

---

## Common Pitfalls

* **Treating TCC like 2PC** — TCC's "reservation" is application state with a TTL, not a transaction-manager lock. If you implement Try as a DB row lock held until Confirm, you've recreated 2PC's blocking and lost the point.
* **Non-idempotent Confirm/Cancel** — network retries are normal. The second `Confirm(R)` must be a no-op, not double-commit.
* **No TTL on the reservation** — the orchestrator crashes, reservation leaks forever. Always TTL.
* **Orchestrator as SPOF** — yes the orchestrator can die, but TTL + a durable saga log (event-sourced state machine) recovers. Persist orchestrator state.
* **Cancel that can fail forever** — Cancel must be **retried until success**, and ultimately page on-call if it can't succeed. A failed Cancel = stranded inventory.
* **Confusing TCC with reservation-pattern in a saga** — a Saga step can *also* reserve and later confirm/release. The difference: a Saga's coordination is via events between independent services; TCC's is via a synchronous orchestrator that knows the trio explicitly.

---

## References

* Pat Helland — [*Life beyond Distributed Transactions: an Apostate's Opinion*](https://queue.acm.org/detail.cfm?id=3025012) (2007). The canonical "stop using 2PC" paper.
* Caitie McCaffrey — [*Building Scalable Stateful Services*](https://www.infoq.com/presentations/building-stateful-services/).
* [microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html) — Why Saga over 2PC.
* Atomikos / Bernstein-Newcomer — *Principles of Transaction Processing* (2nd ed.). The textbook 2PC reference.
* Chris Richardson — *Microservices Patterns*, Chapter 4.
