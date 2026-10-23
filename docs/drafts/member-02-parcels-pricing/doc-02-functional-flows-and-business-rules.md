# SwiftDrop Parcel Delivery & Courier Management System

# Functional Flows & Business Rules

## Member 2: Parcel Management & Pricing

# 1. Purpose

This document specifies the step-by-step functional workflows and business rules for Member 2's Parcel Management & Pricing responsibility. It is intended to give implementers and reviewers enough detail to understand normal behavior, alternate paths, failure handling, state changes, idempotency, concurrency, and cross-module interactions.

The module covers:

Delivery-fee estimation using approved pricing rules.

Customer parcel booking and the agreed parcel-creation capability used by merchant integration.

Parcel identifiers and public tracking-number creation.

Accepted price snapshot persistence.

Parcel current status and status-history handling within the agreed ownership boundary.

Authorized parcel detail, history, and receipt-data retrieval.

Idempotent booking creation and transactionally consistent event records.

The module does not own courier assignment, proof-of-delivery storage, the Tracking API's SSE implementation, RabbitMQ worker processing, notification delivery, or merchant credential management. Those capabilities integrate with this module through agreed contracts.

# 2. Actors and Triggering Conditions

| Actor / Component | Trigger | Expected interaction with Member 2 |
| --- | --- | --- |
| Customer | Requests a fee estimate | Submits parcel and route/service inputs; receives a quote result or validation error. |
| Customer | Confirms a booking | Submits booking details and the selected/accepted quote reference; receives booking confirmation. |
| Approved merchant client | Creates a delivery order through the merchant API | Merchant integration validates the merchant and tenant context, then invokes the agreed parcel-creation capability. |
| Member 1 — Identity & Access | An authenticated request reaches the Core API | Supplies verified caller identity/role context; Member 2 enforces resource ownership within its use cases. |
| Member 3 — Dispatch & Couriers | A parcel is assigned or an operational status update is requested | Uses agreed parcel identifiers and lifecycle contracts; assignment and delivery-attempt execution remain Member 3 responsibilities. |
| Member 5 — Tracking API | A tracking/history read is requested | Reads authoritative parcel/status data through the agreed read interface or approved database access pattern. It does not define a second lifecycle. |
| Member 6 — Frontend | Booking, estimate, history, or receipt UI needs data | Calls the published API contract; frontend calculations are display-only and not authoritative. |
| Member 8 — Event-Driven Backend | A committed business operation requires asynchronous follow-up | Consumes the agreed outbox/event contract; Member 2 records relevant outbox data within the business transaction. |
| Member 10 — AI Assistant (bonus) | User requests a fee estimate or tracking/history information | Calls approved backend tools/APIs; it must not access the database directly or invent results. |
| Administrator / pricing configuration capability | Approved pricing rules or delivery zones are created/changed | Pricing calculation uses only enabled, effective, approved rules. Ownership of pricing-rule administration and zone tables must be confirmed with the team. |

# 3. Preconditions, Postconditions, and Shared Rules

## 3.1 Preconditions

Before a booking or pricing operation is accepted:

## 1. The request must conform to the agreed API schema and size limits.

## 2. The caller must be authenticated where the endpoint requires authentication.

3. The caller's role, resource ownership, and merchant tenant context must be validated server-side.

4. Required parcel fields must be present and satisfy agreed formats, ranges, and cross-field constraints.

5. The pickup/drop-off route and requested service must be supported by an enabled pricing/service rule.

## 6. Weight and dimensions must fall within approved limits.

7. The pricing engine must be able to identify an applicable rule/version. If no applicable rule exists, the operation must fail safely rather than inventing a price.

8. For booking, the server must independently validate the quote or calculate the accepted price. A client-supplied amount is never authoritative.

## 3.2 Postconditions for Successful Booking

A successful booking must result in:

One persisted parcel with a stable internal identifier.

One unique, non-guessable public tracking number.

An initial status of CREATED, subject to confirmation of the canonical lifecycle.

Persisted pickup/drop-off and parcel details according to the approved schema.

A stored accepted-price snapshot, including currency and pricing-rule/version reference once those policies are approved.

An initial status-history record.

A required outbox event written in the same PostgreSQL transaction when the operation requires asynchronous follow-up.

A booking response that provides the parcel identifier, tracking number, current status, and accepted price.

If the transaction fails, the system must not expose a successful booking response or leave a partially committed booking/history/outbox combination.

## 3.3 Cross-Cutting Rules

PostgreSQL is the authoritative source of parcel business state.

All validation and business rules are enforced by the backend; frontend validation is for usability only.

The server derives the acting user from the authenticated context; clients must not choose actor_user_id.

Money uses exact decimal/numeric representation or integer minor units, never binary floating point.

Timestamps in API payloads use UTC and ISO 8601/RFC 3339 format.

A stable internal parcel ID is separate from the public tracking number.

Private addresses, phone numbers, delivery notes, proof files, and merchant-only data must not be included in public tracking responses.

State changes, their history records, and corresponding outbox records are committed atomically where required.

External HTTP calls, notifications, and webhook delivery must not run while holding a business database transaction open.

Exact endpoint paths, DTOs, and error formats must be agreed in the API and integration contract.

# 4. Functional Flow A — Estimate Delivery Fee

## Trigger

A customer or approved merchant needs to see the estimated delivery fee before confirming a parcel booking. The AI Assistant may call the same approved pricing API as a bonus integration.

## Main Success Flow

1. The caller submits the route/zone or address references, parcel weight, dimensions where applicable, and requested service level.

## 2. The transport handler validates request syntax and required fields.

3. The application layer validates supported zones/service options and parcel limits using approved configuration.

4. The pricing component selects the applicable enabled rule/version using the approved effective-date and eligibility policy.

## 5. The pricing engine calculates the quote deterministically using that rule.

## 6. The system represents the amount using the approved exact-money strategy and rounding policy.

7. The system returns a quote containing the amount, currency, pricing-rule/version reference, and quote validity information if an expiry policy is adopted.

8. The quote may be persisted if the agreed design requires quote IDs or auditability; the final persistence policy remains to be confirmed.

## Alternate and Failure Flows

| Condition | Expected behavior |
| --- | --- |
| Required input missing or malformed | Reject with field-level validation errors; do not calculate a quote. |
| Weight/dimensions outside approved limits | Reject the quote request with a clear validation error. |
| Pickup/drop-off zones unsupported or not serviceable | Return a safe business validation error; do not invent a fallback rate. |
| No effective pricing rule matches | Return a controlled “pricing unavailable / route unsupported” error; log a diagnostic correlation ID without leaking internals. |
| Pricing configuration is invalid or overlapping rules create ambiguity | Do not choose an arbitrary rule; fail safely and surface an operational error for correction. |
| Pricing store or required dependency unavailable | Return a dependency-unavailable response where appropriate; do not return a fabricated or stale amount as a new authoritative quote. |
| Caller retries an estimate request | The calculation should remain deterministic for the same inputs and rule version. Quote persistence/idempotency behavior depends on the final quote design. |

## Pricing Rules

1. Exact formula and tariff values require team approval. The SRS recommends LKR and rule-based pricing using zone, weight, and dimension factors as a baseline; the exact formula, tariff values, precision, rounding rules, and any additional service-level or distance factors still require approval.

2. The implementation must not hard-code an assumed formula or currency as if it were an approved requirement.

## 3. The same inputs evaluated under the same rule version must produce the same result.

## 4. The quote must identify its currency and pricing-rule/version.

## 5. Zone eligibility and weight/dimension limits must be checked before a quote is accepted.

6. If quotes can expire, the expiry/validity policy must be explicit and tested. No quote lifetime is specified in this document.

7. Administrative ownership of zones and pricing-rule writes must be agreed with the team; the pricing calculation capability remains within Member 2's scope.

## 8. Example values in prototypes are illustrative only and must not be used as approved tariff data.

## UML Sequence Diagram

![Diagram from the original document](Document2-Functional-Flows-Business-Rules-Revised_assets/image-1.png)

UML sequence diagram for delivery-fee estimation

# 5. Functional Flow B — Create a Customer Parcel Booking

## Trigger

An authenticated customer confirms a delivery booking after providing the required parcel and pickup/drop-off details.

## Main Success Flow

## 1. The customer submits a booking request and an idempotency key according to the API contract.

2. The transport handler validates the request shape and extracts the authenticated caller identity.

3. The application layer validates customer ownership context, required fields, addresses/zone references, parcel dimensions/weight, and requested service.

4. The application layer resolves the accepted quote or invokes the pricing component to validate/recalculate the amount against the current applicable rule. The client-supplied price is not trusted.

## 5. The application layer checks the idempotency record for the caller/operation/key scope.

6. If this is a new request, the system creates a parcel aggregate with a stable internal ID, unique tracking number, accepted price snapshot, and initial status CREATED under the approved lifecycle.

7. Within one PostgreSQL transaction, the system persists the parcel, initial status-history record, idempotency result, and required outbox record(s).

## 8. Database constraints enforce uniqueness and referential integrity.

## 9. The transaction commits.

10. The API returns a booking confirmation containing the parcel ID, tracking number, initial status, and accepted price details.

11. After commit, the Outbox Relay coordinated with Member 8 can publish the event. The booking endpoint does not wait for notification or webhook delivery to finish.

## Alternate and Failure Flows

| Condition | Expected behavior |
| --- | --- |
| Caller is unauthenticated | Reject the protected request; do not create a parcel. |
| Caller is not allowed to create this type of booking | Reject with an authorization response; do not create a parcel. |
| Required fields or parcel constraints fail | Return field-level errors; do not persist a booking. |
| Route/service is unsupported or no valid quote exists | Reject the booking; do not create a parcel with an invented fee. |
| Quote is expired, if quote expiry is adopted | Require re-quotation or another explicitly approved policy; do not silently accept an expired quote. |
| Tracking-number collision occurs | Retry generation safely within a bounded policy or fail with an internal error; never create duplicate tracking numbers. |
| Same idempotency key and same request are repeated | Return the stored original result/resource reference; do not create another parcel. |
| Same idempotency key is reused with different request content | Return 409 Conflict (or the team's documented equivalent); do not create another parcel. |
| Database transaction fails | Roll back parcel, history, idempotency result, and outbox records together. |
| Client times out after commit and retries | Idempotency lookup returns the committed booking result. |
| Outbox publication is delayed after commit | Booking remains successful; the committed outbox record remains available for the relay to publish later. |
| Event publication or notification fails | The booking transaction is not undone; relay/worker retry and operational recovery are Member 8's responsibility. |

## Booking Invariants

A successful request creates one parcel only.

The tracking number is unique and not trivially guessable.

The accepted price is calculated/validated server-side and stored as a snapshot.

Initial status, initial history, and required outbox event are transactionally consistent.

The response is not sent as a success until the database transaction commits.

A retry must not create a second booking.

A client cannot set the acting actor ID, authoritative price, or arbitrary initial status.

## UML Sequence Diagram

![Diagram from the original document](Document2-Functional-Flows-Business-Rules-Revised_assets/image-2.png)

UML sequence diagram for customer parcel booking

# 6. Functional Flow C — Merchant Order Creation

## Trigger

An approved and enabled merchant submits a delivery order through the secured merchant API.

## Main Flow

1. Member 4's merchant integration validates the merchant credential and verifies that the merchant is approved and enabled.

2. The merchant identity/tenant context is passed to the agreed parcel-creation use case; it must not be accepted solely from an untrusted body field.

3. The parcel use case validates the order fields, route/service eligibility, parcel limits, and pricing requirements.

## 4. The system checks idempotency using the merchant/tenant and operation scope.

5. The system creates the parcel and stores the accepted price snapshot, initial status/history, and required outbox record(s) in a single transaction.

## 6. On commit, the API returns a stable parcel identifier and tracking number to the merchant.

7. Later status-change webhooks are processed asynchronously by the Event Worker coordinated with Member 8, using the shared event contract.

## Alternate and Failure Flows

Pending, rejected, suspended, disabled, or invalid merchant credentials must not create an order.

A merchant must not read or mutate another merchant's orders.

A repeated request with the same idempotency key and same payload returns the original result.

Reusing the key with a different payload returns a conflict.

Invalid parcel data, unsupported service, or missing applicable pricing rule rejects the request.

Webhook delivery failure after a successful booking does not roll back the parcel. Webhook retries are owned by Member 8.

Boundary: Member 2 owns the parcel business operation and pricing calculation. Member 4 owns merchant onboarding, credentials, and tenant validation. The exact internal interface between these modules must be agreed and documented in the API and integration contract.

# 7. Functional Flow D — Retrieve Parcel Details, History, and Receipt Data

## 7.1 Retrieve Parcel Details / Customer List

## 1. The caller requests a parcel or list of parcels.

2. The backend authenticates the caller and derives the permitted scope from identity, role, merchant tenant, or assignment context.

3. The repository query applies that scope in the query/use case; authorization must not rely only on hiding UI controls.

## 4. The system returns only fields permitted for that caller.

## 5. List operations use bounded pagination and documented sorting/filtering.

Failure behavior:

Missing or malformed identifiers return a safe client error as defined by the API contract.

A parcel that does not exist or is not visible to the caller returns the team's consistent not-found/authorization behavior without leaking sensitive existence details.

Large list requests are bounded; the API does not return an unbounded result set.

## 7.2 Retrieve Parcel Status History

## 1. The caller requests history for a parcel.

## 2. The backend validates access to that parcel.

3. The system returns history in chronological order with status, server-side timestamp, actor/reference fields allowed for the caller, and applicable notes/reason fields.

4. The history is read from authoritative persisted records; the frontend or SSE stream is not the durable history source.

Public tracking is different: it returns only the approved privacy-safe subset, not full private history.

## 7.3 Retrieve Receipt Data

## 1. The owner or authorized merchant requests receipt data for an eligible booking.

## 2. The backend verifies ownership/tenant permissions and eligibility.

3. The response includes the required receipt fields: parcel reference, booking time, accepted price, service details, and current/terminal status as appropriate.

4. The frontend or designated presentation layer renders/downloads the receipt using the agreed format.

5. The receipt uses the stored booking price snapshot rather than recalculating with today's tariff.

If the receipt format, eligibility rules, or downloadable file type are not yet approved, they remain open interface decisions. The module must not claim that a PDF format is mandatory unless the team/SRS explicitly confirms it.

# 8. Parcel Lifecycle and Status-History Rules

## 8.1 Baseline Lifecycle

The SRS defines the following baseline lifecycle:

CREATED → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED

The SRS states that FAILED is a terminal outcome for a delivery attempt or parcel only according to the approved operational policy. A failed attempt may be retried by an authorized dispatcher/courier workflow; if retries are allowed, model the attempt outcome separately from the final parcel lifecycle rather than silently reopening a terminal parcel.

## 8.2 Transition Matrix — Proposed for Team Review

| Current state | Proposed next state | Conditions / notes |
| --- | --- | --- |
| Booking does not yet exist | CREATED | System creates the initial status as part of successful booking. |
| CREATED | PICKED_UP | Must be performed by an authorized operational flow; assignment and actor rules are owned/coordinated with Member 3. |
| PICKED_UP | IN_TRANSIT | Only a permitted actor may advance the parcel through the agreed workflow. |
| IN_TRANSIT | OUT_FOR_DELIVERY | Must be a valid operational transition. |
| OUT_FOR_DELIVERY | DELIVERED | Required delivery evidence/policy must be satisfied by Member 3's delivery workflow. |
| OUT_FOR_DELIVERY | Failed attempt outcome | Failure reason is required; delivery-attempt model and parcel-state effect must be approved. |
| Any state | Arbitrary state supplied by client | Prohibited. Backend state machine rejects illegal transitions. |
| Terminal state | Reopen or correct | Prohibited unless an explicitly authorized, audited correction/reattempt policy permits it. |

This transition matrix is a proposed implementation baseline, not a substitute for the approved operational policy. Members 2, 3, and 5 must agree on actor permissions, delivery-attempt semantics, and the authoritative read interface before parallel implementation.

## 8.3 Status-History Rules

Every accepted operational status change must record, at minimum:

Parcel ID.

Previous and new status where applicable.

Server-side timestamp in UTC.

Actor identity from authenticated context.

Optional note and relevant attempt/assignment reference.

Parcel version/sequence where needed for concurrency and event ordering.

Normal application operations must not edit or delete history records directly. Erroneous history may be corrected only through an authorized, audited operation. If a failed delivery can be retried, delivery attempts should be represented separately so the system does not silently rewrite or reopen a terminal parcel state.

## 8.4 Transactional Status Update

For status changes owned by the Core API, the agreed application use case must:

## 1. Load the current parcel and relevant assignment/attempt context.

## 2. Verify the actor's permissions and current-state preconditions.

## 3. Reject stale or conflicting state changes.

4. Validate the requested transition and any required failure/proof conditions through the owning domain/use case.

5. In one PostgreSQL transaction, persist the new state, append status history, and write the corresponding outbox event.

## 6. Commit before any relay publication.

## 7. Return the committed state/version.

Member 2 owns parcel lifecycle invariants; Member 3 owns courier assignment, delivery attempts, and proof details. Their interface must enforce both sets of conditions without duplicating the source of truth.

# 9. Idempotency, Concurrency, and Duplicate Requests

## 9.1 Booking Idempotency

For parcel creation, the client supplies an Idempotency-Key according to the agreed API contract.

The key is scoped at minimum to the caller/tenant and operation. The server stores the key, a request hash, and the resulting resource reference according to the approved storage design.

| Request condition | Required behavior |
| --- | --- |
| First request with a new key | Process the request and persist the result with the booking transaction. |
| Same key + same effective payload after success | Return the original booking result; do not create a second parcel. |
| Same key + different payload | Return a conflict; do not silently reuse the prior result or create another parcel. |
| Two identical requests arrive concurrently | Database uniqueness/transaction logic ensures only one booking is created; the other returns the prior result or a defined in-progress/conflict response. |
| First request fails before transaction commit | No partial booking remains. Retry behavior follows the stored idempotency state and documented failure policy. |
| Client times out after server commit | Retry with the same key returns the committed result. |

The team must decide key-retention duration, maximum key length, request-hash canonicalization, and behavior for concurrent in-progress requests. Record these decisions in the API contract and module implementation specification.

## 9.2 Optimistic Concurrency for Status Changes

A courier or dispatcher may act on stale information if another operation changes a parcel after it was loaded.

Use the agreed version/sequence or conditional-update strategy.

A stale state/assignment must not overwrite a newer state.

Return a documented conflict response, including the current state only where safe.

Do not create duplicate history or outbox effects when the same operation is retried.

Exact concurrency mechanism (for example, version column, row lock, or conditional update) is an implementation decision for the module design and implementation specification.

## 9.3 Event Idempotency Boundary

Member 2 records a stable event/outbox entry within the same transaction as the qualifying business change. Member 8 owns the Outbox Relay/Event Worker implementation and downstream consumer idempotency.

The system assumes at-least-once event delivery; it must not claim exactly-once delivery across PostgreSQL, RabbitMQ, and external providers. Downstream consumers use stable event IDs and, where necessary, aggregate versions or idempotency records to avoid duplicate effects.

# 10. UML Activity Diagram — Booking Decision Flow

![Diagram from the original document](Document2-Functional-Flows-Business-Rules-Revised_assets/image-3.png)

UML activity diagram for the parcel booking decision flow

# 11. Edge Cases and Expected Outcomes

| Edge case | Expected behavior |
| --- | --- |
| Zero, negative, malformed, or excessive weight | Reject using approved validation bounds. Exact maximum/minimum values require team approval. |
| Missing dimensions when required by service/pricing rules | Reject with field-level validation errors. |
| Pickup and drop-off route not served | Reject; do not silently use a default zone or price. |
| No effective tariff or ambiguous overlapping tariff | Fail safely and log for configuration correction. |
| Pricing rule changes between estimate and booking | Apply the explicitly approved quote-validity/price-lock policy; never silently trust client amount. |
| Duplicate tracking-number generation | Enforce database uniqueness and retry safely within a bounded policy. |
| Same idempotency key with changed payload | Conflict; no second booking. |
| Concurrent booking requests with same key | Only one effective booking is created. |
| Customer requests another customer's private parcel/history/receipt | Deny according to the consistent authorization/not-found policy. |
| Public visitor requests tracking | Return only approved tracking fields; no private address/contact/proof data. |
| Tracking history has no records due to a data inconsistency | Return a safe response and flag the inconsistency for diagnosis; do not fabricate history. |
| Outbox relay is unavailable after a committed booking | Booking remains committed; pending event is retried by the Outbox Relay coordinated with Member 8. |
| Duplicate or stale event reaches a consumer | Consumer idempotency/version handling prevents repeated or stale side effects. |
| Status update is based on stale parcel/assignment state | Reject with conflict and safe current state where appropriate. |
| Failed delivery is submitted without a failure reason | Reject; required failure reason is enforced by the operational workflow. |
| Request times out after a successful commit | Same idempotency key retrieves the original result. |
| Database unavailable during booking | Return a safe dependency error; no partial booking should remain. |

# 12. Requirements Traceability

| Requirement / baseline | Workflow section | Evidence expected |
| --- | --- | --- |
| FR-PAR-001 — Booking fields and service option | Section 5 | Booking validation tests and API contract. |
| FR-PAR-002 — Estimate fee before confirmation | Section 4 | Pricing API tests and quote examples. |
| FR-PAR-003 — Pricing rule/version and price snapshot | Sections 4–5 | Persistence tests proving snapshot/version retention. |
| FR-PAR-004 — Unique, non-guessable tracking number | Section 5 | Uniqueness and collision tests. |
| FR-PAR-005 — Invalid input/zone/limits rejected | Sections 4–5, 11 | Field-level validation and boundary tests. |
| FR-PAR-006 — Idempotent booking | Section 9 | Same-key, changed-payload, and concurrency tests. |
| FR-PAR-007 — Own bookings and history | Section 7 | Ownership/tenant authorization tests. |
| FR-PAR-008 — Receipt contents | Section 7.3 | Receipt data contract and snapshot tests. |
| FR-PAR-009 — No private fields in public tracking | Section 7.2 and 11 | Public response field allow-list/security test. |
| FR-LIFE-001 to FR-LIFE-007 — State/history rules | Section 8 | Transition matrix tests, audit/history tests, and transactional tests. |
| FR-MER-002 to FR-MER-006 — Merchant order creation and idempotency | Section 6 | Merchant tenant and duplicate-order integration tests. |
| FR-EVT-001 — Outbox atomicity | Sections 5, 8.4, 9.3 | Transaction rollback/commit integration tests. |
| FR-AI-002 — AI fee estimation via Core API | Section 2 and Section 4 | Bonus integration test against the same pricing API. |

The requirement IDs above correspond to the current SwiftDrop SRS baseline. Verify them against the team's approved SRS revision before treating this traceability table as final.

# 13. Open Decisions Requiring Team Approval

The following decisions are intentionally unresolved and must not be guessed during implementation:

1. Pricing formula and currency: confirm LKR (recommended by the SRS) and approve the exact base fee, zone/weight/dimension factors, any distance or service-level charges, precision, and rounding.

2. Pricing configuration ownership: who owns delivery-zone and pricing-rule table writes and administrative APIs; Member 2 owns calculation logic, while admin ownership is shared/assigned elsewhere.

3. Quote behavior: whether quotes are persisted, whether they expire, their validity duration, and what happens if pricing rules change before booking confirmation.

## 4. Parcel lifecycle: final allowed transitions and actor permissions.

5. Failed delivery: whether FAILED is an attempt outcome or terminal parcel state; reattempt rules and limits.

6. Delivery evidence: which statuses require proof and what data Member 3 must provide before a transition can be accepted.

7. Receipt format and eligibility: required download format, exact fields, and when a receipt becomes available.

8. Idempotency policy: key retention, maximum size, payload canonicalization, concurrent in-progress behavior, and storage ownership.

9. Outbox interface: canonical event names, envelope, schema version, aggregate version, and shared transaction interface with Member 8.

10. API contract: final route names, DTOs, error codes, pagination, and public/private tracking response fields.

Until approval, use interfaces and policy abstractions that avoid embedding unapproved values in business logic. Mark sample pricing data as test-only and illustrative.

# 14. Acceptance Criteria

The workflows are ready for implementation review when:

A valid quote request produces a deterministic quote for the same approved rule version.

Invalid parcel inputs and unsupported routes are rejected without fabricated fees.

A valid booking returns a unique tracking number and accepted price snapshot.

Booking, initial history, idempotency result, and required outbox records commit atomically.

A failed transaction leaves no partial booking/history/outbox combination.

A retry with the same idempotency key and payload returns the original booking.

Reusing a key with a different payload returns a conflict.

Customers and merchants can only access data within their authorized scope.

Receipt data uses the persisted accepted price rather than today's pricing rule.

The backend enforces the agreed parcel state machine and records status history.

Failed attempts require a reason, and retry/terminal behavior follows the approved policy.

Status/history/outbox writes are consistent and safe against concurrent updates.

Tests cover success, validation failures, unauthorized access, idempotency, concurrency, pricing boundaries, and database/dependency failure.

The relevant API and event contracts are documented and agreed with dependent members.
