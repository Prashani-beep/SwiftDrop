# Event & Integration Support Module (`internal/events`)

> **Domain:** Transactional Outbox Writer, Event Envelopes & Domain Event Support  
> **Primary Owner:** Member 8 — Event-Driven Backend & Integrations  
> **Clean Architecture Boundary:** Modular Monolith Internal Module

---

## 1. Module Overview & Responsibilities
The `events` module provides the shared integration plumbing used by all Core API domain modules to emit domain events safely via the **Transactional Outbox Pattern**.

### Key Responsibilities:
- Standard `EventEnvelope` Go struct definitions with versioning and correlation tracing.
- `OutboxWriter` interface used by application use cases to persist events inside active database transactions.
- Concrete PostgreSQL outbox repository appending rows to `outbox_events`.
- JSON serialization and event validation helpers.

---

## 2. Invariants & Guarantees
1. **Same-Transaction Atomic Commit:** Outbox records MUST be inserted using the exact same PostgreSQL `*sql.Tx` as the domain state mutation.
2. **Zero Broker Dependency in Core API:** The Core API does NOT import RabbitMQ or publish network messages directly. It writes only to PostgreSQL. The external `services/messaging` Outbox Relay handles network publication.

---

## 3. Layer Breakdown
```
internal/events/
├── envelope.go              # EventEnvelope[T] generic struct
├── writer.go                # OutboxWriter interface definition
├── repository.go            # Postgres implementation of OutboxWriter
└── catalog.go               # Event type string constants (e.g. parcel.booked.v1)
```

---

## 4. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial module architecture baseline created | Architecture Setup |
