# SwiftDrop Parcel Delivery & Courier Management System

## Document 3 — API & Data Model Specification

## 1. Purpose and Scope

This document defines the API and logical data requirements for Member 2’s Parcel Management & Pricing module in SwiftDrop. It is based on SwiftDrop SRS v1.0 and the proposed Team Work Division v1.0. Both are drafts for team review; the approved assignment/SRS takes precedence.

In scope: delivery-fee quotes, customer parcel booking, merchant order creation through the shared parcel capability, parcel detail/list/history/receipt data, tracking number, accepted-price snapshot, status history, idempotency, and transactional outbox integration.

Out of scope: identity-provider implementation, merchant credentials/onboarding, courier assignment and delivery-attempt execution, public Tracking API implementation, event relay/worker processing, notification delivery, and webhook delivery.

## 2. Requirements Baseline

| ID | Requirement for Member 2 |
| --- | --- |
| FR-PAR-001 | Create a booking with pickup/drop-off contact and address details, parcel size/dimensions where required, weight, and service option. |
| FR-PAR-002 | Provide an estimated delivery fee before booking confirmation. |
| FR-PAR-003 | Identify the pricing rule/version used and persist the accepted price snapshot with the booking. |
| FR-PAR-004 | Generate a unique, non-guessable tracking number and return booking confirmation. |
| FR-PAR-005 | Reject missing/invalid fields, unsupported dimensions/weight, addresses, and service zones with field-level validation errors. |
| FR-PAR-006 | Idempotent booking: retry with the same key must not create a second parcel; same key with changed request returns conflict. |
| FR-PAR-007 | Allow customers to list/view their own bookings and status history, subject to authorization. |
| FR-PAR-008 | Provide eligible receipt data containing parcel reference, booking time, price, service details, and current/terminal status as appropriate. |
| FR-PAR-009 | Public tracking responses must not expose private address/contact details. |
| FR-LIFE-001–004 | Maintain current status and append-only history; validate transitions; record actor and server timestamp; atomically persist status, history, and outbox event. |

Related SRS data-integrity requirements: use stable internal IDs (UUID recommended); enforce foreign keys and uniqueness; use UTC timestamps; store money as integer minor units or exact NUMERIC with documented currency scale; use versioning or row locks for conflicting updates; separate mutable current state from immutable history.

## 3. API Conventions

| Convention | Requirement |
| --- | --- |
| Base path | /api/v1 |
| Format | REST/JSON; application/json; UTF-8 |
| Identifiers | Stable internal ID (UUID recommended), separate from public tracking number |
| Timestamps | ISO 8601/RFC 3339 in UTC |
| Lists | Bounded pagination; documented sorting/filtering and maximum page size |
| Errors | Central format: code, message, request_id, optional field_errors |
| Documentation | Keep OpenAPI specification and examples under source control |

Use standard HTTP semantics: 201 Created, 200 OK, 202 Accepted where asynchronous work applies, 400 validation, 401 unauthenticated, 403 forbidden, 404 absent/not visible, 409 conflict, 422 semantic validation if selected, 429 rate limited, 500 unexpected failure, and 503 dependency unavailable. Never expose stack traces, SQL, credentials, tokens, or internal hostnames.

## 4. Member 2 Endpoint Inventory

| Method and path | Purpose | Access / notes |
| --- | --- | --- |
| POST /api/v1/pricing/quotes | Calculate delivery-fee quote | Customer or approved merchant |
| POST /api/v1/parcels | Create customer parcel booking | Authenticated customer; requires Idempotency-Key per agreed contract |
| GET /api/v1/parcels | List caller-visible parcels | Customer/merchant; server-enforced owner/tenant scope |
| GET /api/v1/parcels/{id} | Get parcel details | Owner, merchant, assigned courier, or authorized staff as permitted |
| GET /api/v1/parcels/{id}/history | Retrieve status history | Authorized caller only |
| GET /api/v1/parcels/{id}/receipt | Retrieve/generate receipt data | Owner or authorized merchant; format/eligibility to be agreed |
| POST /api/v1/merchant/orders | Create parcel through merchant API | Approved merchant client; Member 4 validates credentials/tenant and calls shared parcel capability |

The route inventory is the SRS proposed baseline. Confirm exact paths, DTOs, authorization scopes, response envelope, and receipt behavior with the team before treating it as final.

## 5. Request and Response Contracts

### 5.1 Fee quote

```json
POST /api/v1/pricing/quotes
{
  "pickup_zone_id": "...",
  "dropoff_zone_id": "...",
  "weight_kg": 2.0,
  "length_cm": 30,
  "width_cm": 20,
  "height_cm": 10,
  "service_level": "STANDARD"
}
```

Quote response fields: quote_id (if quotes are persisted), currency, amount_minor or exact amount, pricing_version, and valid_until only if an expiry policy is adopted. The SRS sample uses LKR and amount_minor=125000 as illustrative data only; it is not an approved tariff.

### 5.2 Create parcel

```json
POST /api/v1/parcels
Idempotency-Key: <client-generated-key>

{
  "quote_id": "...",
  "pickup": {
    "name": "...",
    "phone": "...",
    "address_lines": ["..."],
    "city": "...",
    "zone_id": "..."
  },
  "dropoff": {
    "name": "...",
    "phone": "...",
    "address_lines": ["..."],
    "city": "...",
    "zone_id": "..."
  },
  "parcel": {
    "weight_kg": 2.0,
    "dimensions_cm": {
      "length": 30,
      "width": 20,
      "height": 10
    },
    "description": "..."
  }
}
```

Successful response must return the internal parcel ID, public tracking number, initial status CREATED, and accepted-price details. The server derives the acting customer/merchant from authenticated context; clients must not supply actor_user_id or an authoritative price.

### 5.3 History and receipt

History returns chronological persisted records including status, server-side timestamp, and only the actor/reference/note fields permitted to the caller. Receipt data uses the stored booking price snapshot, not the current tariff. Receipt file format and eligibility rules remain open decisions.

### 5.4 Error response shape

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "request_id": "...",
  "field_errors": [
    {"field": "weight_kg", "message": "Value is outside the permitted range"}
  ]
}
```

The error example illustrates the SRS shape; exact codes and field-error schema must match the shared API contract.

## 6. Logical Data Model

| Entity / table | Required purpose and key data |
| --- | --- |
| parcels | id; tracking_number (unique); customer_id or merchant_id; current_status; parcel weight/dimensions; service_level; accepted price snapshot; version; timestamps |
| parcel_addresses / addresses | parcel_id; pickup/drop-off type; contact name/phone; address lines; city/zone. Access restricted. |
| parcel_status_history | id; parcel_id; from_status; to_status; actor_id; occurred_at; optional note/reason; relevant attempt/assignment reference; parcel_version |
| pricing_rules | id/version; zone pair or criteria; base fee; weight/dimension bands; effective_from/to; enabled. Rule administration ownership must be agreed. |
| pricing_quotes | Optional: id; requester/tenant; input snapshot; rule version; amount/currency; expiry; accepted parcel reference. Use only if quote persistence is adopted. |
| idempotency_keys | Caller/tenant scope; operation; key; request hash; response/resource reference; expiry/retention per approved policy |
| outbox_events | event_id; aggregate type/id/version; event type; schema version; payload; created_at; published_at; attempt_count/status/error fields managed through shared event interface |

Data constraints: tracking_number must be unique and difficult to enumerate; use foreign keys and unique constraints; status history is append-only; timestamps are UTC; monetary values never use binary floating point; index tracking_number, customer_id, merchant_id, current_status, and other fields based on actual query patterns.

## 7. Business and Transaction Rules

- Validate request shape, caller permissions, tenant/ownership scope, required fields, parcel limits, route/service eligibility, and applicable pricing rule before accepting a booking.
- Calculate or validate price on the server. The same inputs under the same rule version must produce the same quote.
- Persist parcel, accepted-price snapshot, initial status history, idempotency result, and required outbox event(s) in one PostgreSQL transaction.
- Return booking success only after commit. On rollback, none of the booking/history/idempotency/outbox combination may remain partially committed.
- Same idempotency key + same effective request returns the original result; same key + changed request returns 409 Conflict (or the documented equivalent).
- Every accepted status transition is checked by the parcel domain logic. Client-supplied arbitrary status changes are rejected.
- Status update, status history, and outbox record are committed atomically. Outbox Relay publishes only committed records; downstream delivery is at-least-once and consumers must be idempotent.
- Public tracking is a separate privacy-safe view and must exclude private addresses, phone numbers, delivery notes, proof files, and merchant-only data.

Lifecycle baseline from the SRS: CREATED → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED. FAILED/reattempt semantics and authorized transition actors must be approved with Members 3 and 5 before implementation freeze.

## 8. Module Boundaries and Dependencies

- Member 1 supplies authentication context and authorization helpers; Member 2 still enforces resource ownership in each use case.
- Member 3 owns courier assignment, delivery attempts, and proof details; Member 2 owns parcel lifecycle invariants and must coordinate transition contracts.
- Member 4 owns merchant onboarding, API credentials, and tenant validation; Member 2 owns the parcel creation business operation and pricing calculation.
- Member 5 reads authoritative parcel/status data and must not create a competing lifecycle or stale source of truth.
- Member 6 consumes the published API contract; frontend pricing calculations are display-only.
- Member 8 owns the Outbox Relay/Event Worker. Member 2 writes outbox records through the agreed shared interface inside the business transaction.

Implementation location: core-api/internal/parcels/, core-api/internal/pricing/, core-api/migrations/, and core-api/tests/. Follow the Go modular-monolith and Clean/Hexagonal Architecture rules in the engineering baseline; domain logic must not depend on HTTP, SQL drivers, or RabbitMQ.

## 9. Open Decisions — Do Not Guess

- Pricing formula, currency, base fees, zone/weight/dimension factors, optional distance/service-level charges, precision, and rounding.
- Ownership and API for delivery-zone/pricing-rule administration.
- Whether quotes are persisted, quote expiry duration, and price-lock/re-quotation behavior.
- Canonical parcel fields, field limits, service-level values, validation ranges, and zone model.
- Final lifecycle transitions, actor permissions, and whether FAILED means a failed attempt or terminal parcel state.
- Idempotency key retention, maximum size, request-hash canonicalization, and concurrent in-progress behavior.
- Final API paths/DTOs/error codes, receipt format/eligibility, and public tracking field allow-list.
- Canonical event names/envelope/schema version and aggregate-version handling with Member 8.

The SRS is a proposed baseline for team review. Any decision that changes requirements or API/data/event contracts must be recorded with rationale, impact, affected tests, owner, and approval.

## 10. Acceptance Checklist

- Valid quote produces a deterministic amount and identifies currency and pricing-rule version.
- Valid booking persists one parcel with unique tracking number and accepted-price snapshot.
- Missing/invalid fields, unsupported routes, and unsupported dimensions/weight produce structured validation errors.
- Repeated idempotent booking does not create duplicates; changed payload under the same key conflicts.
- History and receipt data are limited to authorized owners/roles; public tracking excludes private fields.
- Parcel status, history, and required outbox records are transactionally consistent.
- Tests cover pricing examples/boundaries, booking success/failure, authorization, idempotency/concurrency, history, and invalid state transitions.
