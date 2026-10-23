# SwiftDrop Parcel Delivery & Courier Management System
## Document 1 — Module Scope & Responsibility Specification
### Member 2: Parcel Management & Pricing

## 1. Purpose

### 1.1 Module Purpose

The Parcel Management & Pricing module is responsible for creating and maintaining parcel booking records, calculating delivery fees, managing the parcel lifecycle within its agreed boundaries, and providing parcel history and receipt data.

It forms a core business capability of the SwiftDrop system. Customers and authorized merchants use its booking functionality to register parcels for delivery. The resulting parcel records are then consumed by other modules responsible for dispatch, courier operations, tracking, and event-driven integrations.

The module is implemented within the Go Core API as two closely related logical areas:

- **Parcel Management:** Owns the parcel aggregate, booking operations, tracking-number assignment, parcel state-transition rules, and parcel status history.
- **Pricing:** Owns delivery-fee estimation and the applicable pricing calculation rules.

These areas share business data and collaborate during booking, but their responsibilities should remain clearly defined.

### 1.2 Business Problem

Traditional courier operations may rely on phone calls, manual records, and spreadsheets to register parcels and calculate delivery charges. Such processes can lead to inconsistent pricing, duplicate bookings, incomplete parcel histories, and difficulty integrating with tracking and dispatch systems.

The SwiftDrop module addresses these problems by providing:

- A consistent and validated parcel-booking process.
- A deterministic and configurable delivery-fee calculation.
- A unique public tracking number for each successfully created parcel.
- Persistent parcel details, accepted pricing information, and status history.
- Controlled parcel-state transitions and protection against invalid operations.
- Reliable integration with other modules through agreed API and event contracts.

### 1.3 Objectives

The module shall aim to:

1. Accept valid parcel-booking requests from authorized customers and, through the agreed integration boundary, authorized merchants.
2. Validate parcel information and reject unsupported or incomplete requests.
3. Calculate delivery-fee estimates using team-approved pricing rules.
4. Persist the accepted booking price as a snapshot so that subsequent pricing-rule changes do not silently alter an existing booking.
5. Generate and maintain unique parcel identifiers and public tracking numbers.
6. Maintain the current parcel status and append-only status history.
7. Provide authorized parcel-history and receipt-data retrieval.
8. Prevent duplicate bookings when an eligible request is retried using the same idempotency key.
9. Persist relevant integration events through the agreed transactional outbox interface.
10. Provide automated tests, database migrations, API documentation, and reproducible setup instructions for the module.

## 2. Scope and Responsibilities

### 2.1 In-Scope Responsibilities

| Capability | Responsibility |
|---|---|
| Parcel booking | Receive, validate, and process parcel-creation requests. |
| Parcel data | Maintain authoritative parcel business records within the agreed schema. |
| Tracking number | Generate or assign a unique public tracking number and enforce uniqueness. |
| Pricing and estimation | Calculate fees using approved, configurable rules. |
| Price snapshot | Preserve the accepted fee, currency, and applicable pricing-rule version with the booking. |
| Parcel lifecycle | Validate permitted parcel-state transitions and reject illegal transitions. |
| Status history | Record status changes with timestamps, actors, and applicable notes or references. |
| Parcel history API | Provide authorized access to the parcel's chronological history. |
| Receipt data | Provide the booking and pricing data required for receipt presentation or generation. |
| Idempotency | Prevent duplicate parcel creation for retried requests using the same valid idempotency key. |
| Persistence | Implement module-owned PostgreSQL migrations, constraints, and indexes. |
| Event production | Create applicable outbox records through the shared event interface in coordination with Member 8. |
| Quality assurance | Write unit and integration tests for booking, pricing, persistence, idempotency, and lifecycle rules. |
| Documentation | Maintain API examples, pricing rules, module README, test instructions, and known limitations. |

### 2.2 Explicit Exclusions

The module must not take ownership of capabilities assigned to other members.

| Excluded Capability | Owner | Boundary |
|---|---|---|
| User registration, password handling, JWT issuance, and Google sign-in | Member 1 | The module consumes the approved authentication and authorization mechanisms. |
| Shared courier assignment and reassignment operations | Member 3 | The module provides the authoritative parcel data and coordinates lifecycle rules with dispatch. |
| Courier workload calculations and delivery execution | Member 3 | These belong to Dispatch & Couriers. |
| Proof-of-delivery uploads and delivery-attempt management | Member 3 | The module coordinates required parcel-state transitions but does not own proof storage or attempt workflows. |
| Independent Tracking API and real-time SSE delivery | Member 5 | The Tracking API reads authoritative parcel and status data; it does not define a competing parcel lifecycle. |
| Customer and merchant user interfaces | Member 6 | The module exposes backend contracts consumed by the frontend. |
| Dispatcher/admin interfaces and API Gateway configuration | Member 7 | The module publishes its API requirements and does not own gateway or frontend implementation. |
| Outbox Relay, RabbitMQ topology, Event Worker, notification delivery, webhook retries, and dead-letter queues | Member 8 | The module writes agreed outbox records; Member 8 owns asynchronous publication and delivery processing. |
| Platform-wide Docker Compose, Kubernetes, Helm, and observability configuration | Member 9 | The module supplies the service configuration and health/build information needed for integration. |
| AI Assistant and MCP tools | Member 10 | The assistant consumes approved backend APIs; it does not access the module's database directly. |
| Merchant credential issuance and webhook configuration | Member 4 | The module may support the shared parcel-creation capability, but merchant onboarding and credentials remain Member 4's responsibility. |
| System-wide database governance | Team-appointed database coordinator | Member 2 owns the module's migrations and tables, subject to shared schema conventions and review. |

The exclusions do not remove the need for collaboration. Other modules may invoke the parcel or pricing capabilities through approved interfaces without taking ownership of their business rules or persistence.

## 3. Bounded Context and Domain Ownership

### 3.1 Bounded Context

The Parcel Management & Pricing capability is a logical bounded context within the Core API. Its internal rules, domain objects, and use cases must be implemented within their designated packages.

The module must not require frontend code, tracking services, or event workers to reproduce its business rules independently.

The current proposed repository layout is:

```text
core-api/
├── internal/
│   ├── parcels/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── transport/
│   ├── pricing/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── transport/
│   ├── events/
│   └── platform/
├── migrations/
└── tests/
```

This is a proposed organizational structure, not a requirement to create every directory regardless of need. The final package structure should follow the repository's agreed conventions and the complexity of each capability.

### 3.2 Parcel Management Ownership

The Parcel Management area owns:

- The parcel aggregate and its business invariants.
- Booking creation and parcel lifecycle validation.
- Public tracking-number uniqueness.
- The parcel's authoritative current status.
- Parcel status-history records.
- The application use cases for retrieving authorized parcel details, history, and receipt data.
- The transactional coordination needed to persist booking or lifecycle changes and their required history/outbox records.

### 3.3 Pricing Ownership

The Pricing area owns:

- The delivery-fee calculation logic.
- Validation of pricing-related inputs, including applicable parcel size and weight limits.
- Evaluation of approved delivery zones or zone-based tariff rules.
- Money representation, currency handling, and rounding according to the approved policy.
- The applicable pricing-rule version used to calculate a fee.
- The pricing snapshot included in a confirmed booking.

The module must calculate prices on the server. It must not trust a client-submitted fee as the authoritative amount.

### 3.4 Authoritative Data

PostgreSQL is the authoritative source for parcel business state.

The module owns the parcel-related data assigned to it in the canonical database schema. Other modules may read approved fields or invoke published operations, but they must not create independent, conflicting parcel or status tables.

The module must not treat RabbitMQ messages, frontend state, tracking caches, or AI responses as authoritative parcel records.

## 4. Dependencies and Integration Relationships

### 4.1 Upstream Dependencies

| Dependency | Provider | Required Input or Capability |
|---|---|---|
| Authentication and identity context | Member 1 | Verified caller identity, role, and reusable authorization helpers. |
| API conventions | Team | Base path, JSON conventions, error envelope, request IDs, and timestamp format. |
| Database conventions | Team/database coordinator | Canonical identifiers, migration ordering, naming conventions, and shared constraints. |
| Parcel lifecycle agreement | Members 2, 3, 5, and relevant team members | Approved statuses, legal transitions, actor permissions, and failed-delivery semantics. |
| Pricing decisions | Team and relevant admin capability owners | Currency, pricing formula, zone model, limits, and rounding rules. |
| Outbox/event interface | Member 8 | Event envelope, outbox-writing interface, schema versions, and transaction conventions. |
| Customer interface requirements | Member 6 | Booking fields, estimate response, receipt requirements, and history display requirements. |
| Merchant integration requirements | Member 4 | Agreed mechanism for merchant-originated parcel creation and merchant ownership context. |

### 4.2 Downstream Consumers

| Consumer | Expected Interaction |
|---|---|
| Customer frontend — Member 6 | Requests fee estimates, creates bookings, and retrieves authorized parcel history and receipt data. |
| Merchant integration — Member 4 | Uses the agreed parcel-creation capability for authenticated merchant orders. |
| Dispatch & Couriers — Member 3 | Uses authoritative parcel data and coordinates legal lifecycle changes and status-history ownership. |
| Tracking API — Member 5 | Reads current authoritative parcel status and approved tracking fields. |
| Event-driven backend — Member 8 | Receives persisted outbox records for publication and asynchronous processing. |
| AI Assistant — Member 10 | May call approved fee-estimation and tracking/history APIs, subject to authorization and data-disclosure rules. |
| Admin/frontend capabilities — Member 7 | May consume approved pricing or zone-related APIs according to the final ownership agreement. |

### 4.3 Collaboration Boundaries

The following decisions must be agreed before dependent implementation is finalized:

- Parcel fields, relationships, and status-history schema with Member 3 and the database coordinator.
- The lifecycle and failed-delivery semantics with Members 3 and 5.
- The event/outbox interface and transaction boundary with Member 8.
- Merchant parcel-creation contracts with Member 4.
- Customer-facing request and response contracts with Member 6.
- Pricing-rule administration and zone ownership with Member 7 and the team.

## 5. Invariants and Business Guarantees

The module must preserve the following invariants. Exact status names, pricing values, and transition permissions remain subject to the approved SRS and team decisions.

### 5.1 Parcel Invariants

1. Every persisted parcel has a stable internal identifier.
2. Every confirmed booking has a unique public tracking number.
3. Required parcel fields must satisfy the approved validation rules before a booking is accepted.
4. The current parcel status must be consistent with the agreed lifecycle state machine.
5. A status transition must be validated by the domain/application logic rather than performed through an unrestricted database update.
6. Each accepted status change must produce an appropriate status-history record.
7. Status-history records are append-only under normal application operations; corrections must follow an explicitly approved audit procedure.
8. Private parcel details and receipt data are available only to authorized owners or roles.
9. Public tracking disclosure is restricted to the approved public field set.

### 5.2 Pricing Invariants

1. The same valid inputs evaluated under the same pricing-rule version must produce the same result.
2. Currency, units, precision, and rounding must follow the team-approved policy.
3. Monetary values must use an exact representation, such as integer minor units or PostgreSQL `NUMERIC`, rather than binary floating-point arithmetic.
4. Zone eligibility and parcel size/weight limits must be validated before a quote is accepted.
5. A confirmed booking preserves the price accepted at booking time.
6. Changes to future pricing rules must not silently rewrite historical booking prices.

### 5.3 Idempotency and Transaction Invariants

1. A repeated create request with the same idempotency key, caller/operation scope, and request content returns the prior result rather than creating another parcel.
2. Reusing an idempotency key with a different request payload produces a conflict.
3. Booking state, initial status history, and any required outbox event must be persisted atomically.
4. For lifecycle changes, the parcel state, applicable status-history record, and corresponding outbox event must be committed in the same database transaction where required by the shared event design.
5. External notifications and webhook delivery must not occur inside a long-running database transaction.
6. Database constraints must reinforce important invariants, including unique tracking numbers and relevant foreign-key relationships.

## 6. What Other Modules May Request

The Parcel Management & Pricing module exposes capabilities rather than allowing other modules to manipulate its internal state directly.

Approved callers may request:

- A delivery-fee estimate for validated parcel inputs.
- Creation of a parcel booking with the required ownership context.
- Retrieval of parcel details subject to authorization and field-disclosure rules.
- Retrieval of chronological parcel status history subject to authorization.
- Retrieval of receipt data for an authorized booking.
- Execution of an approved parcel lifecycle operation, where that operation belongs to the Parcel Management domain and the caller is permitted to perform it.

Other modules must not:

- Update parcel status through direct, unvalidated SQL.
- Set a booking fee by trusting a client-supplied amount.
- Bypass booking validation or idempotency.
- Insert competing parcel or status-history records outside the canonical schema and approved transaction interfaces.
- Treat a successful message publication as a substitute for committing the business state.
- Expose private parcel data solely because the caller knows a parcel ID or tracking number.

Exact endpoint paths and DTO schemas belong in Document 4, Interface & Integration Contract.

## 7. Ownership of Files, Tables, APIs, and Configuration

### 7.1 File Ownership

| Path | Ownership |
|---|---|
| `core-api/internal/parcels/` | Member 2 — parcel domain, booking, lifecycle rules, and history use cases. |
| `core-api/internal/pricing/` | Member 2 — pricing rules and fee calculation. |
| `core-api/migrations/` | Member 2 owns parcel/pricing migrations; shared review and migration ordering are required. |
| `core-api/tests/` | Member 2 contributes parcel/pricing tests; shared test conventions apply. |
| `docs/` | Member 2 maintains module scope, pricing rules, API references, and module-specific documentation in the agreed documentation locations. |
| `core-api/internal/events/` | Member 8 owns the shared event/outbox contract; Member 2 integrates with it without independently redefining it. |
| `core-api/internal/identity/` | Member 1 owns identity and shared authorization primitives. |
| `services/tracking/` | Member 5 owns the separately deployed Tracking API. |

These boundaries describe primary ownership. Changes to shared files, migrations, or interfaces require coordination and review.

### 7.2 Database Ownership

Member 2 owns the parcel/pricing tables or table portions formally assigned to the module. The following are expected data responsibilities, subject to the approved canonical ERD:

- Parcel records and the accepted price snapshot.
- Parcel status history.
- Parcel-related idempotency records or the agreed shared idempotency storage mechanism.
- Pricing-rule data only to the extent assigned to Member 2 after the team resolves pricing/zone administration ownership.

The module may reference users, merchants, addresses, delivery zones, assignments, or other records owned by other modules. Referencing a record does not automatically transfer ownership of its schema or business rules.

The final table names, columns, constraints, indexes, and migration order are defined in Document 5, Data & Persistence Specification.

### 7.3 API Ownership

Member 2 owns the implementation and contract for parcel-booking, fee-estimation, history, and receipt-data endpoints assigned to this module.

The API contract must follow the team's versioning and error conventions. Illustrative paths may use `/api/v1`, but the final route inventory is confirmed in Document 4.

### 7.4 Queue and Event Ownership

Member 2 does not own RabbitMQ queues, worker deployment, notification delivery, or webhook retry processing.

Member 2 is responsible for requesting and recording the relevant domain/outbox event through the shared interface when a qualifying parcel business operation commits.

Event names, payload schemas, versioning, and publication behavior must be agreed with Member 8 and documented in the shared event contract.

### 7.5 Configuration Ownership

Member 2 documents module-specific configuration, including any approved pricing defaults, limits, and database-related settings.

Sensitive credentials must be supplied through the team's configuration mechanism. Real secrets must never be committed to source control.

Pricing parameters that require runtime administration must use the approved configuration or database model rather than unexplained hard-coded values.

## 8. Assumptions and Unresolved Decisions

This document defines module boundaries but does not silently settle project-wide decisions that remain open.

| Decision | Current Position | Required Action |
|---|---|---|
| Pricing formula | Must be configurable, deterministic, and documented. Exact formula and rates are not established here. | Agree on base fee, weight/dimension factors, zone rules, currency, and rounding. |
| Pricing-rule administration | Pricing calculation belongs to Member 2; administrative UI and zone/reporting capabilities involve Member 7 and potentially Member 4. | Confirm ownership of pricing-rule persistence and administrative write operations. |
| Parcel statuses | A controlled state machine is required. Exact status names and transitions must match the approved lifecycle. | Agree on valid transitions and actor permissions with Members 3 and 5. |
| Failed delivery and redelivery | Delivery attempts and proof belong to Member 3. Parcel-level consequences require a shared decision. | Define whether failure changes parcel status, how reattempts work, and how history is retained. |
| Merchant booking | Merchant onboarding and credentials belong to Member 4. Parcel creation logic is owned by Member 2. | Agree on the shared use case and caller/tenant context. |
| Idempotency storage | Duplicate booking protection is required. | Select a module-owned or shared implementation and define key scope, retention, and request hashing. |
| Outbox contract | Member 8 owns the shared event/outbox mechanism. | Agree on the interface, event envelope, and transaction participation. |
| Public tracking disclosure | The Tracking API is owned by Member 5. | Agree on public fields and ensure private parcel fields are not exposed. |
| Receipt format | Member 2 provides receipt data; presentation/download integration must be coordinated with Member 6. | Confirm required receipt fields and output format. |
| Schema and migrations | Member 2 owns module migrations within a shared canonical database. | Agree on migration naming, ordering, review, and conflict resolution. |

Any decision that changes module ownership or a published interface must be recorded in the team's decision log and reflected in the relevant specifications.

## 9. Out-of-Scope Work

The following activities are outside this document's implementation scope:

- Implementing the complete customer, merchant, dispatcher, or administrator frontend.
- Implementing Google OAuth, JWT issuance, or password management.
- Building the separately deployed Tracking API or its SSE connection management.
- Implementing courier dispatch algorithms, courier assignments, delivery-attempt execution, or proof-file storage.
- Building the Outbox Relay, RabbitMQ consumers, email/SMS provider, or outbound merchant webhook worker.
- Implementing the AI Assistant/MCP service.
- Owning platform-wide Kubernetes and Helm deployment.
- Defining the entire system ERD or replacing the team's canonical database governance process.

Member 2 remains responsible for providing the interfaces, data contracts, migrations, and tests required for these other components to integrate correctly.

## 10. Acceptance Criteria

The module is considered ready for integration when the following conditions are demonstrated:

1. A valid booking request creates exactly one parcel with a unique tracking number.
2. Invalid or unsupported parcel fields return structured validation errors.
3. Fee estimates are reproducible for the same inputs and pricing-rule version.
4. The confirmed booking preserves the accepted price and currency information.
5. Parcel history and receipt data can be retrieved by authorized callers.
6. Unauthorized users cannot retrieve private parcel information by changing an identifier.
7. Repeating a booking request with the same idempotency key and payload does not create a duplicate.
8. Reusing an idempotency key with a different payload is rejected according to the agreed conflict contract.
9. Illegal parcel-state transitions are rejected.
10. Required parcel state, history, and outbox records are persisted atomically according to the agreed event design.
11. Database migrations and indexes are reproducible in the team's documented environment.
12. Unit and integration tests cover valid and invalid bookings, pricing boundaries, idempotency, history access, and lifecycle rules.
13. API examples and module setup instructions are maintained and consistent with the implementation.
14. The module is integrated through the agreed shared interfaces and passes review.

## 11. Definition of Done

For every completed feature, Member 2 must ensure that:

- The implementation matches the approved business rules and API contract.
- Go code is formatted, builds successfully, and follows repository conventions.
- Relevant automated tests pass.
- Input validation, authorization, privacy, idempotency, and failure cases are covered.
- Database migrations can be applied reproducibly.
- Configuration examples contain placeholders rather than real secrets.
- API documentation, module README, and known limitations are updated.
- Changes are committed on an appropriate feature branch and reviewed before merging.
- The feature can be demonstrated using the agreed local development environment.


