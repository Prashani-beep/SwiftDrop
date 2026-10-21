# Parcel Management Module (`internal/parcels`)

> **Domain:** Parcel Aggregate Root, Lifecycle State Machine & Booking  
> **Primary Owner:** Member 2 — Parcel Management & Pricing  
> **Clean Architecture Boundary:** Modular Monolith Internal Module

---

## 1. Module Overview & Responsibilities
The `parcels` module is the authoritative owner of the **Parcel Aggregate Root**. It governs the parcel lifecycle from initial booking through assignment, transit, delivery attempts, and terminal states.

### Key Responsibilities:
- Parcel aggregate root encapsulation and state transition invariant enforcement.
- Customer and merchant parcel booking workflows with idempotency protection.
- Unique tracking number generation (`SD-YYYYMMDD-XXXXX`).
- Dimension and weight validation against tariff classes.
- Atomic state persistence, status audit logging (`parcel_status_history`), and outbox event creation.

---

## 2. Domain Invariants & Business Rules
1. **Exclusive Aggregate Root Authority:** Only the `Parcel` aggregate may mutate parcel status. No other module, handler, or raw SQL script may update `parcels.status`.
2. **Transition Legality:** State transitions must strictly follow the transition graph defined in `docs/architecture/domain-boundaries.md`. Illegal jumps (e.g., `CREATED` $\rightarrow$ `DELIVERED`) are rejected with `PARCEL_INVALID_TRANSITION`.
3. **Audit History Trail:** Every status change must record an immutable row in `parcel_status_history` containing `previous_status`, `new_status`, `actor_id`, `actor_role`, and `timestamp`.
4. **Atomic Outbox Record:** Every domain event (e.g., `parcel.booked.v1`, `parcel.delivered.v1`) must be written to `outbox_events` in the exact same database transaction as the aggregate state update.
5. **Idempotent Booking:** Repeated booking requests with identical `Idempotency-Key` headers must return the previously booked parcel without duplicating records.

---

## 3. Layer Breakdown
```
internal/parcels/
├── domain/                  # Parcel aggregate, TrackingNumber VO, Dimensions VO, Status enum, domain errors
├── application/             # BookParcel, GetParcelDetails, CancelParcel use cases & repository interface
├── infrastructure/          # PostgresParcelRepository, TrackingNumberGenerator
└── transport/               # HTTP handlers (/api/v1/parcels/*) and request/response DTOs
```

---

## 4. Database Ownership
- `parcels`: Primary aggregate table (ID, tracking number, sender ID, recipient details, weight, dimensions, fee, current status, courier ID, timestamps).
- `parcel_status_history`: Append-only audit log of all historical transitions.
- `idempotency_keys`: Keys, request hashes, response payloads, and expiration timestamps.

---

## 5. Events Emitted
- `parcel.booked.v1`: Published upon successful parcel creation.
- `parcel.status_changed.v1`: Published upon any verified status transition.
- `parcel.cancelled.v1`: Published when a parcel is cancelled before collection.

---

## 6. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial module architecture baseline created | Architecture Setup |
