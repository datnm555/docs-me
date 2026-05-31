# Two-Phase Commit (2PC) và Try-Confirm-Cancel (TCC)

> Hai pattern **đồng bộ** cho distributed transaction. **2PC** cung cấp ACID xuyên resource với cái giá blocking + tight coupling — và gần như tuyệt chủng trong microservice hiện đại. **TCC** là thay thế nhẹ hơn, application-level: reserve trước, commit sau. Đọc cái này để hiểu **tại sao** microservice dùng [Saga](./saga.md) thay vì.

> 🇻🇳 Phiên bản tiếng Việt. English: [`two-phase-commit-tcc.md`](./two-phase-commit-tcc.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu · Như thế nào)

- **Cái gì** — **2PC**: 1 coordinator chạy `prepare` cho mọi participant, rồi `commit` nếu tất cả vote yes (ngược lại `rollback`). **TCC**: mỗi participant expose 3 operation — `Try` (reserve), `Confirm` (commit reservation), `Cancel` (release). 1 coordinator orchestrate 3 cái này xuyên service.
- **Tại sao** — Cả hai cố tạo outcome **atomic, all-or-nothing** qua nhiều resource hoặc service. Framing hữu ích để hiểu tại sao distributed transaction khó, và tại sao câu trả lời kinh điển cho microservice là *eventual* consistency qua [Saga](./saga.md).
- **Khi nào** — **2PC**: legacy DTC / XA, stack enterprise single-vendor (SQL Server + MSMQ), thỉnh thoảng cho in-process 2-DB commit. **TCC**: khi business semantic tự nhiên thành "reserve rồi confirm" — travel booking, ticket sale, inventory reservation trong checkout.
- **Ở đâu** — Cả hai là **architectural pattern** trên storage layer. 2PC implement bởi coordinator (XA driver / MSDTC). TCC implement ở tầng **service contract** — mọi participant phải có endpoint `Try` / `Confirm` / `Cancel`.
- **Như thế nào** — 2PC: tránh trừ khi bạn control cả 2 endpoint và chấp nhận blocking. TCC: design API mọi participant theo "reserve → confirm → cancel" với confirm/cancel idempotent. Cho microservice mới, **default Saga**; chỉ với TCC nếu "reserve rồi confirm" là business model tự nhiên.

---

## Tại sao Distributed Transaction Khó

Local ACID transaction là 1 process commit lên 1 storage. Ngay khi có **2** system có thể fail độc lập, bạn phải chọn giữa 3 ràng buộc — Consistency, Availability, Partition tolerance — và tối đa được 2 (định lý CAP).

| Local TX                                  | Distributed TX                                                                |
| ----------------------------------------- | ----------------------------------------------------------------------------- |
| 1 log, 1 commit point                     | Nhiều log; commit phải **coordinate**                                          |
| Fail tự động rollback                     | Coordinator crash giữa protocol để participant **uncertain**                  |
| Atomicity miễn phí                        | Atomicity tốn **blocking** và **availability**                                 |
| Latency = disk write                      | Latency = network-hop chậm nhất × số round trip                               |

Cả 2PC và TCC cố lấy lại atomicity với giá blocking + coupling. Saga từ bỏ atomicity *tức thì* để đổi availability, thay rollback bằng **compensating action**.

---

## Two-Phase Commit (2PC)

Coordinator điều khiển 2 phase:

```
Phase 1 — Prepare:
   Coordinator: "Sẵn sàng commit chưa?"
   Participants: ghi log, lock resource, reply YES/NO
   Mỗi YES = "Tôi hứa commit được nếu anh bảo."

Phase 2 — Commit (hoặc Rollback):
   Tất cả YES → coordinator ghi quyết định commit → bảo tất cả commit
   Có NO       → coordinator bảo tất cả rollback

Khi participant chờ giữa Phase 1 và Phase 2, lock vẫn giữ.
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
            Tất cả committed
```

### Cái gì hỏng

* **Coordinator crash sau Phase 1, trước Phase 2** → participant kẹt giữ lock đến khi coordinator hồi phục (vấn đề "in-doubt transaction").
* **Lock dài** — qua nhiều service, lock duration = tổng latency. Throughput sụp đổ.
* **Coupling chặt** — mọi participant phải nói cùng coordinator protocol (XA, MSDTC) và phải tin nhau giữ lời prepare vote.
* **Không có partial progress** — mọi participant phải available trong suốt thời gian.

### 2PC vẫn dùng khi nào

* In-process xuyên **2 DB cùng 1 service sở hữu** (ví dụ SQL Server + MSMQ qua MSDTC).
* Integration mainframe / ESB legacy.
* **Gần như không bao giờ** trong microservice mới.

> "Two-phase commit là anti-availability protocol." — Pat Helland, *Life beyond Distributed Transactions* (2007). Paper essentially kết thúc 2PC cho distributed system mới.

---

## Try-Confirm-Cancel (TCC)

TCC chuyển logic "lock + commit" từ transaction manager vào **application protocol**. Mỗi service tham gia expose 3 operation:

| Operation | Effect                                                                            | Idempotent? |
| --------- | --------------------------------------------------------------------------------- | ----------- |
| `Try`     | **Reserve** resource. Giảm stock available, hold inventory slot, v.v.             | Có (theo reservation id) |
| `Confirm` | Commit reservation. Làm thay đổi vĩnh viễn.                                        | Có          |
| `Cancel`  | Release reservation. Khôi phục state.                                              | Có          |

Coordinator chạy 3 cái:

```
Phase 1 — Try (song song):
   coordinator.Try(flight, dates)   → reservation R₁
   coordinator.Try(hotel,  dates)   → reservation R₂
   coordinator.Try(car,    dates)   → reservation R₃

Phase 2:
   Tất cả thành công: Confirm(R₁), Confirm(R₂), Confirm(R₃)
   Có cái fail:       Cancel(R₁), Cancel(R₂), Cancel(R₃) cho những cái đã succeed
```

Khác biệt chính với 2PC:

* **Không có lock cấp protocol do transaction manager giữ.** Reservation là *business-level* state — sống trong DB của participant với TTL.
* **Mỗi operation là HTTP/gRPC call bình thường.** Không XA driver, không distributed lock manager.
* **Mỗi participant quyết định** giữ reservation bao lâu; coordinator crash chỉ nghĩa là reservation hết hạn.
* **Cả 3 operation phải idempotent** (network retry là chuyện thường).

### TCC fit khi nào

* Domain *vốn dĩ* nói "reserve rồi confirm" — hotel, flight, concert ticket, inventory hold ở checkout.
* Bạn control mọi service tham gia.
* Chấp nhận chi phí operational cho 3 endpoint mỗi resource.

### TCC **không** fit khi

* "Reserve" không có nghĩa tự nhiên ("reserve 1 row DB update"?).
* Bạn không sở hữu participant — third-party API hiếm khi có reservation primitive.
* Workflow > ~5 participant — orchestration đau; nên dùng Saga framework thật.

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
            _log.LogWarning(ex, "Booking {SagaId} fail — đang cancel reservation", sagaId);

            // Cancel mọi thứ đã reserved. PHẢI idempotent + retry.
            foreach (var (service, id) in reservations)
                await CancelBestEffort(service, id, ct);

            return BookingResult.Failure(sagaId, ex.Message);
        }
    }

    private async Task CancelBestEffort(string service, Guid id, CancellationToken ct)
    {
        // Cancellation là retry-loop với backoff; PHẢI thành công cuối cùng
        // (không thì leak reservation). Nếu không được, page on-call.
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

Mỗi participant trông thế này:

```csharp
public sealed class FlightService : IFlightService
{
    private readonly AppDbContext _db;

    public async Task<TryResult> TryAsync(Guid sagaId, FlightChoice choice, CancellationToken ct)
    {
        // Idempotent: cùng sagaId 2 lần trả cùng reservation.
        var existing = await _db.Reservations.FirstOrDefaultAsync(r => r.SagaId == sagaId, ct);
        if (existing is not null) return new TryResult(existing.Id);

        // Reserve seat với TTL (ví dụ 15 phút — auto-expire nếu Confirm/Cancel không đến).
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

Để ý **TTL trên reservation**: nếu orchestrator chết giữa `Try` và `Confirm`, background job quét reservation hết hạn và release ghế. **Orchestrator không phải SPOF** theo kiểu 2PC coordinator.

---

## Real Project — Travel Booking

Khách book holiday: flight + hotel + car, all-or-nothing.

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
│           có fail → Cancel(R₁..R_n đã tạo)                       │
│                                                                  │
│   ┌─ Confirm ─────────────────────────────────────────────┐      │
│   │   FlightSvc.Confirm(R₁)                                │      │
│   │   HotelSvc.Confirm(R₂)                                 │      │
│   │   CarSvc.Confirm(R₃)                                   │      │
│   └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

Failure scenario:

| Failure                                  | Cái gì xảy ra                                                                |
| ---------------------------------------- | ---------------------------------------------------------------------------- |
| Hotel hết phòng → `Try` trả 409           | Orchestrator cancel FlightSvc.R₁; trả "no rooms" cho user.                   |
| Network blip lúc CarSvc.Try               | Retry với idempotency key (sagaId); cùng reservation trả về.                  |
| Orchestrator crash sau khi mọi Try xong   | Reservation có TTL 15 phút — tự cancel; user retry booking.                  |
| Confirm fail giữa chừng (FlightSvc down)  | Retry Confirm đến khi thành công. Reservation đã paid — commit các cái khác ổn; FlightSvc cuối cùng sẽ confirm. |

So với Saga choreography cho cùng bài toán: mọi participant subscribe event, không có orchestrator trung tâm. TCC gần với **synchronous orchestration với reservation tường minh** hơn.

---

## Quyết định: 2PC vs TCC vs Saga

| Tiêu chí                  | 2PC                      | TCC                              | Saga                                         |
| -------------------------- | ------------------------ | -------------------------------- | -------------------------------------------- |
| Consistency                | Strong (ACID xuyên)      | Strong-ish lúc confirm           | Eventual                                     |
| Availability               | Thấp (block coordinator) | Trung                            | Cao                                          |
| Coupling                   | Rất chặt (XA / DTC)      | Chặt (phải agree API Try/Confirm/Cancel) | Lỏng (event / message)                |
| Lock                       | TM giữ, xuyên system     | Reservation app-level với TTL    | Không — compensating action thay              |
| Latency                    | Tổng participant × 2 round | 1× round trip mỗi phase         | 1× mỗi step; có thể song song                |
| Failure mode               | Block / in-doubt         | Auto-release qua TTL             | Compensate (Cancel/Refund/v.v.)              |
| Dùng ở đâu hôm nay         | Same vendor + same process | "Reserve rồi confirm" business  | Default cho microservice                     |

> **TL;DR.** Cho microservice mới, default **Saga**. **TCC** chỉ khi "reserve rồi confirm" là business model tự nhiên (travel, ticketing, inventory hold). **Tránh 2PC** trừ khi thật sự có scenario single-vendor + single-process (ví dụ MSDTC cho 2 local resource).

Xem [`saga.md`](./saga.md) cho implementation Saga kinh điển.

---

## Pitfall thường gặp

* **Coi TCC như 2PC** — "reservation" trong TCC là application state với TTL, không phải lock của transaction manager. Nếu implement Try thành DB row lock giữ đến Confirm, bạn đã tái tạo blocking của 2PC và mất ý nghĩa.
* **Confirm/Cancel không idempotent** — network retry là thường. `Confirm(R)` lần 2 phải là no-op, không double-commit.
* **Không TTL trên reservation** — orchestrator crash, reservation leak mãi mãi. Luôn TTL.
* **Orchestrator là SPOF** — ừ orchestrator có thể chết, nhưng TTL + saga log durable (event-sourced state machine) phục hồi. Persist orchestrator state.
* **Cancel có thể fail mãi** — Cancel phải **retry đến thành công**, và cuối cùng page on-call nếu không được. Cancel fail = inventory bị stranded.
* **Nhầm TCC với reservation-pattern trong saga** — 1 step của Saga *cũng có thể* reserve rồi confirm/release. Khác biệt: Saga coordinate qua event giữa service độc lập; TCC qua synchronous orchestrator biết 3 cái tường minh.

---

## Tham khảo

* Pat Helland — [*Life beyond Distributed Transactions: an Apostate's Opinion*](https://queue.acm.org/detail.cfm?id=3025012) (2007). Paper kinh điển "ngừng dùng 2PC".
* Caitie McCaffrey — [*Building Scalable Stateful Services*](https://www.infoq.com/presentations/building-stateful-services/).
* [microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html) — Tại sao Saga hơn 2PC.
* Atomikos / Bernstein-Newcomer — *Principles of Transaction Processing* (2nd ed.). Textbook 2PC reference.
* Chris Richardson — *Microservices Patterns*, Chapter 4.
