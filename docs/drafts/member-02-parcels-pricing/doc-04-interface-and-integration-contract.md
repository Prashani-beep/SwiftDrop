# SwiftDrop Parcel Delivery & Courier Management System

## Document 4 — Interface & Integration Contract

## 1. Purpose and Authority

This document defines the interface and integration contract for Member 2 — Parcel Management & Pricing in SwiftDrop. It specifies the proposed REST endpoints, request/response fields, access boundaries, error behavior, and integration responsibilities needed for other team members to consume the module.

Source baseline: SwiftDrop SRS v1.0, Team Work Division v1.0, Engineering Standards and Best Practices v1.0, Document 1 — Module Scope & Responsibility Specification, and Document 2 — Functional Flows & Business Rules. These source documents are proposed/draft baselines. The approved assignment and approved SRS take precedence.

Contract status: Draft for team review. Paths and representative DTOs below follow the SRS proposed endpoint inventory. Any detail not explicitly fixed by the sources is marked as requiring team agreement; examples are not proof that a decision has been approved.

## 2. Interface Principles

- Base path: /api/v1. Payloads use REST/JSON with application/json and UTF-8.
- Use stable internal resource IDs (UUID recommended) separately from public tracking numbers.
- Use ISO 8601/RFC 3339 timestamps in UTC in API payloads.
- Validate input and authorization in the backend. The frontend may display estimates but must not be the source of truth for pricing or parcel-state rules.
- Use a shared error format containing code, message, request_id, and optional field_errors. Never expose SQL, stack traces, credentials, tokens, or internal hostnames.
- List endpoints must support bounded pagination and documented sorting/filtering; enforce a maximum page size.
- Document finalized paths, schemas, status codes, and examples in the repository’s OpenAPI specification.

HTTP semantics from the SRS baseline: 201 Created, 200 OK, 202 Accepted where asynchronous processing applies, 400 validation error, 401 unauthenticated, 403 forbidden, 404 absent/not visible, 409 conflict, 422 semantic validation if selected by the team, 429 rate limited, 500 unexpected error, and 503 dependency unavailable.

## 3. Endpoint Contract

| Method / path | Purpose | Caller / boundary |
| --- | --- | --- |
| POST /api/v1/pricing/quotes | Calculate delivery-fee quote | Authenticated customer or approved merchant. Merchant authentication/tenant validation belongs to Member 4. |
| POST /api/v1/parcels | Create customer parcel booking | Authenticated customer; require idempotency according to agreed shared contract. |
| GET /api/v1/parcels | List caller-visible parcel bookings | Customer/merchant; results must be scoped to caller/tenant. |
| GET /api/v1/parcels/{id} | Retrieve parcel details | Owner, authorized merchant, assigned courier, or authorized staff according to approved role rules. |
| GET /api/v1/parcels/{id}/history | Retrieve parcel status history | Authorized caller only. |
| GET /api/v1/parcels/{id}/receipt | Retrieve/generate receipt | Owner or authorized merchant. Receipt format and eligibility need team agreement. |
| POST /api/v1/merchant/orders | Create parcel via merchant API | Approved merchant client. Member 4 validates API credentials and tenant; parcel creation/pricing must use the shared module capability. |

The SRS also lists public tracking endpoints under the Tracking API. Member 2 supplies authoritative parcel/status data and privacy-safe fields but does not own the Tracking API route implementation or its SSE behavior.

## 4. Request Contracts

### 4.1 Fee quote request

```json
POST /api/v1/pricing/quotes
Content-Type: application/json

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

Fields shown are from the SRS representative example. The exact accepted service-level values, units, ranges, zone identifiers, and whether dimensions are mandatory for every service must be confirmed against the approved parcel/pricing rules.

### 4.2 Quote response fields

| Field | Contract meaning |
| --- | --- |
| quote_id | Identifier for the quote, if quotes are persisted or referenced by booking. |
| currency | Currency code agreed by the team. SRS examples use LKR, but currency is an open decision until approved. |
| amount_minor or amount | Calculated amount using integer minor units or exact NUMERIC; do not use binary floating point. Choose one canonical representation. |
| pricing_version | Identifies the pricing rule/version used for the calculation. |
| valid_until | Include only if the team adopts quote expiry/validity. |

The SRS sample amount (amount_minor 125000) is illustrative only and must not be used as an approved tariff.

### 4.3 Parcel creation request

```json
POST /api/v1/parcels
Idempotency-Key: <key agreed by the API contract>
Content-Type: application/json

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

This is the SRS representative payload, not a complete finalized schema. The team must decide whether booking requires quote_id, whether the server revalidates/recalculates a quote, the exact required fields, and how additional optional parcel fields are represented.

### 4.4 Parcel creation response

A successful booking response must include the internal parcel ID, a unique non-guessable tracking number, initial status CREATED, and the accepted price snapshot. The exact JSON envelope, timestamps, and HTTP response schema must be recorded in OpenAPI.

### 4.5 Idempotency

- Use an Idempotency-Key header for parcel-creation requests, scoped to caller/tenant and operation according to the shared contract.
- Store the request hash and resulting resource/response reference using the agreed shared or module-owned persistence design.
- Same key + same effective request returns the prior result and must not create another parcel.
- Same key + different request returns 409 Conflict (or the team-approved equivalent).
- Key retention duration, key syntax/length, canonical request hashing, and behavior for simultaneous in-progress requests remain open decisions.

## 5. Read Contracts

### 5.1 Parcel list

GET /api/v1/parcels returns only parcels visible to the authenticated caller. It must support bounded pagination and documented filters/sorting. Exact query parameters and response pagination envelope must be finalized in OpenAPI.

### 5.2 Parcel details

GET /api/v1/parcels/{id} returns authorized parcel data, including appropriate parcel details and accepted pricing information. Knowing an ID or tracking number is not authorization. Private contact/address data must not be returned to public tracking clients.

### 5.3 Status history

GET /api/v1/parcels/{id}/history returns persisted chronological history records. The SRS baseline includes status, timestamp, actor, optional note, and relevant attempt/assignment reference. Exact exposure of actor details and notes depends on caller role and privacy policy.

### 5.4 Receipt

GET /api/v1/parcels/{id}/receipt provides receipt data for an eligible booking. The receipt must use the accepted price snapshot and include parcel reference, booking time, price, service details, and current/terminal status as appropriate. Whether the endpoint returns JSON, PDF, or a generated download and which bookings are eligible must be agreed.

## 6. Error Contract

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "request_id": "...",
  "field_errors": [
    {
      "field": "weight_kg",
      "message": "Value is outside the permitted range"
    }
  ]
}
```

| HTTP status | Use for |
| --- | --- |
| 400 | Malformed request or field validation failure. |
| 401 | Missing, invalid, or expired authentication credentials. |
| 403 | Authenticated caller lacks permission. |
| 404 | Resource does not exist or is not visible to the caller; avoid leaking unauthorized resource existence. |
| 409 | Idempotency-key reuse with a different request, duplicate/conflicting operation, or version/concurrency conflict. |
| 422 | Optional team choice for semantically invalid input, if adopted consistently. |
| 429 | Rate limit exceeded. |
| 500 | Unexpected server failure; response must not expose internals. |
| 503 | Required dependency unavailable. |

The error JSON is illustrative of the SRS shape. Exact code names, field_errors schema, and the use of 400 versus 422 must match the shared API contract.

## 7. Status and Event Integration

- Parcel lifecycle is authoritative in the Core API parcel domain. Baseline sequence: CREATED → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED.
- FAILED/reattempt semantics and the exact permitted transition/actor matrix are not finalized; Member 2 must agree these with Members 3 and 5 before implementation freeze.
- Every accepted status change must be validated by domain logic; raw SQL status updates that bypass invariants are prohibited.
- Persist parcel state, append-only status history, and corresponding outbox event in one PostgreSQL transaction.
- Outbox Relay (Member 8) publishes committed outbox entries to RabbitMQ after commit. Member 2 writes through the shared outbox interface and does not own queue/worker processing.
- Events should carry the agreed event ID/type/schema version, aggregate ID and version/sequence where required, occurrence time, and minimum safe payload. Canonical event names, envelope fields, routing keys, and schema versioning must be agreed with Member 8.
- Downstream consumers must tolerate duplicate delivery; event ordering must not be assumed across queues or competing consumers.

Do not invent event names or publish directly from the parcel use case unless the shared event contract explicitly requires it.

## 8. Security and Data Disclosure

- Authentication identity and roles come from Member 1’s approved middleware/context. Member 2 must still enforce parcel ownership and tenant scoping within its use cases.
- Merchant API credential validation and merchant tenant identification belong to Member 4; Member 2 must receive the verified merchant context through the agreed interface.
- Never trust client-supplied actor_user_id or a client-supplied final price as authoritative.
- Public tracking responses must not expose pickup/drop-off contact details, private address data, internal notes, proof-of-delivery files, or merchant-only information.
- Return 404 or the agreed safe response when a caller is not allowed to see a resource; do not allow ID substitution to bypass authorization.
- Use rate limiting for public quote/tracking endpoints as assigned by the gateway/API owners. Do not include secrets or unnecessary personal data in logs/events.

## 9. Ownership and Coordination

| Member / area | Integration responsibility |
| --- | --- |
| Member 1 — Identity & Access | Authentication context, role/authorization helpers, shared identity contract. |
| Member 3 — Dispatch & Couriers | Coordinate parcel schema, lifecycle transitions, status actors, assignment/attempt references, and concurrency behavior. |
| Member 4 — Merchant Integration | Validate merchant API credentials and tenant; invoke shared parcel creation without duplicating parcel/pricing rules. |
| Member 5 — Tracking API | Read authoritative parcel/status data; agree privacy-safe tracking DTO and current-status/history read pattern. |
| Member 6 — Customer Frontend | Consume published OpenAPI contract for quote, booking, history, and receipt; display only server-calculated prices. |
| Member 8 — Events / Outbox | Own canonical event envelope, outbox interface, relay, broker publication, and worker integration. |
| Member 10 — AI Assistant (bonus) | Call approved backend APIs/tools; no direct database access and no invented pricing/tracking answers. |

Primary implementation paths: core-api/internal/parcels/ and core-api/internal/pricing/. Follow Clean/Hexagonal Architecture: transport maps HTTP to application commands/queries; application coordinates use cases and transactions; domain owns invariants; infrastructure implements repository/outbox adapters.

## 10. Contract Decisions Required Before Freeze

| Decision | Required agreement |
| --- | --- |
| Pricing | Approved currency, formula, units, rounding, zone model, weight/dimension bands, service levels, rule versioning, and quote validity. |
| Booking DTO | Required/optional fields, address model, parcel description, service option, quote reference and accepted-price behavior. |
| Authentication | Exact route access scopes, owner/merchant/courier/staff visibility, and tenant context propagation. |
| Idempotency | Header/key policy, retention, request hash, same-key conflict behavior, and concurrent request behavior. |
| Read APIs | List filters/sort/page envelope; detail/history field allow-list; receipt format and eligibility. |
| Lifecycle | Legal status transitions, actor matrix, failure/reattempt semantics, optimistic version/locking strategy. |
| Events | Canonical event names/envelope, aggregate version, payload allow-list, and outbox interface. |
| Errors | Canonical error JSON, error code catalogue, and 400/422 convention. |

Record each decision in the team decision log and update the OpenAPI specification, tests, and relevant documents together. Do not treat illustrative examples in this document as approved implementation values.

## 11. Definition of Done

- All Member 2 endpoints are represented in version-controlled OpenAPI with request/response and error examples.
- Request validation, caller/tenant authorization, privacy-safe responses, and shared error conventions are implemented.
- Pricing responses are deterministic and include currency/amount representation and pricing-rule version.
- Booking creates exactly one parcel with a unique tracking number and accepted-price snapshot; retries behave according to the idempotency contract.
- History and receipt data use persisted records and are accessible only to authorized callers.
- Parcel state, status history, and required outbox records commit atomically.
- Contract tests or integration tests verify key success, validation, authorization, idempotency, conflict, and failure paths.
- Member 3, Member 4, Member 5, Member 6, and Member 8 have reviewed the interfaces relevant to their integrations.

