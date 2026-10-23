# SwiftDrop Parcel Delivery & Courier Management System

# Document 7 — Security, Failure & Recovery Specification

## 1. Purpose and Authority

This document defines security controls, relevant threats, failure behavior, and recovery rules for Member 2's Parcel Management & Pricing module. It supplements the shared SwiftDrop security and engineering standards; it does not replace or redefine the system-wide security policy.

The source documents are proposed baselines awaiting team approval. Requirements that depend on unresolved decisions—such as the final authentication mechanism, authorization response policy, pricing rules, lifecycle semantics, retention periods, and idempotency retention—are explicitly identified and must not be treated as approved values.

### 1.1 In scope

- Authentication-context handling and authorization checks on parcel and pricing endpoints.
- Customer ownership and merchant tenant isolation at the module boundary.
- Validation of booking, quote, parcel identifiers, and state-transition requests.
- Privacy controls for parcel addresses, recipient contacts, receipt data, history, logs, and event payloads.
- Abuse controls for fee estimation, booking, public tracking references, and resource-intensive requests where Member 2 owns or contributes to the endpoint.
- Safe handling of duplicate requests, concurrent updates, database failures, and transactional outbox failures.
- Recovery coordination with Members 1, 3, 4, 5, 8, and 9.

### 1.2 Out of scope / system-wide owners

| Area | Owner / boundary |
|---|---|
| Identity provider, credentials, token issuance/validation, shared authentication middleware, and role definitions | Member 1; Member 2 must enforce the supplied identity/authorization contract on its own endpoints |
| Merchant approval, API credentials, and merchant tenant identity | Member 4; Member 2 must not trust a caller-supplied merchant ID without a trusted authenticated context |
| Courier assignment, delivery attempts, and proof-of-delivery | Member 3 |
| Public/private tracking API, SSE and tracking response projection | Member 5; Member 2 supplies authoritative data under the agreed access boundary |
| Outbox relay, RabbitMQ topology, event workers, notification/webhook retries and DLQ | Member 8; Member 2 writes business outbox records through the shared interface in the business transaction |
| Gateway policy, platform secrets, network policy, deployment, centralized monitoring | Members 7 and 9 according to the approved architecture |
| AI assistant tools and model/provider controls | Member 10; the assistant must use approved APIs and must not bypass Member 2 validation or authorization |

## 2. Security Principles and Protected Assets

1. **Server-side authority:** The server calculates or validates prices, checks ownership, validates state transitions, and applies business rules. Client-supplied role, owner, status, or price values are not authoritative.
2. **Least privilege:** Every request, service identity, and database role receives only the access required for its task.
3. **Single source of truth:** PostgreSQL is authoritative for parcel business state. RabbitMQ transports events; a cache or tracking projection must not become a competing authority.
4. **Privacy by default:** Return only fields required by the caller's approved use case. Public tracking must expose an approved minimal field set.
5. **Atomic business changes:** Parcel state, required status-history records, idempotency outcome, and outbox records must follow the approved transaction boundary. A partial business operation must not be reported as successful.
6. **Fail closed for security:** If authentication, authorization, ownership, or tenant context cannot be established, protected reads and writes are denied.
7. **Observable without leaking secrets:** Use request/correlation IDs and safe structured logs, but never log credentials, access tokens, API keys, webhook secrets, or unnecessary personal data.
8. **Bounded recovery:** Retries must be finite or governed by an explicit retry policy. Reconciliation must be auditable and must not invent business state.

### 2.1 Assets requiring protection

- Internal parcel ID and public tracking number.
- Pickup/drop-off addresses and customer/recipient contact information.
- Parcel description, weight/dimensions, status history, receipt/accepted-price snapshot.
- Pricing rules and rule versions, including any administration capability.
- Customer/merchant ownership associations and trusted actor identity.
- Idempotency keys, request fingerprints, and stored replay outcomes.
- Outbox event IDs, payloads, correlation IDs, and publication state.
- Database credentials, authentication tokens, application secrets, and operational configuration.

## 3. Authentication, Authorization, Ownership and Tenant Isolation

### 3.1 Authentication boundary

- Member 2 endpoints must use the shared authentication middleware and trusted actor context supplied by Member 1.
- Protected operations must reject missing, invalid, expired, or otherwise unacceptable credentials according to the shared auth contract.
- Member 2 must not implement a second, inconsistent token parser or treat a Google identity alone as authorization to read or modify a parcel.
- Internal module-to-module calls within the Go modular monolith should use the agreed in-process application interfaces and trusted context; do not add an unnecessary internal HTTP hop.
- Exact token/session strategy and trusted service identity mechanism remain system-wide decisions for Member 1/team approval.

### 3.2 Authorization matrix (proposed; confirm with approved role policy)

| Actor / context | Quote or fee estimate | Create parcel | Read parcel/history/receipt | Change parcel status |
|---|---|---|---|---|
| Anonymous visitor | Only if quote endpoint is approved as public and rate-limited | Deny | Deny private data; public tracking is a separate Member 5 contract | Deny |
| Authenticated customer | Allowed within approved service scope | Own booking only | Own parcel data only | Deny except explicitly approved customer actions |
| Authenticated merchant | Through Member 4's approved merchant flow; tenant-scoped | Only via approved merchant capability | Only own tenant's permitted data | Deny unless explicitly authorized |
| Courier | Only if required by an approved use case | Deny | Only assigned/authorized delivery details, limited to required fields | Member 3's assignment/lifecycle rules govern |
| Dispatcher/admin | As explicitly allowed by the approved role matrix | Only if an admin workflow is approved | Only for operational duties and authorized scope | Member 3/admin policy governs; never bypass lifecycle invariants |
| Internal service | Only with explicit service permission and validated input | Only via documented interface | Only minimum fields needed | Only through the canonical parcel use case |

This matrix is a coordination baseline, not a substitute for the final role/permission matrix. UI visibility is not an authorization control. Every protected endpoint and application use case must enforce authorization server-side.

### 3.3 Object ownership and tenant checks

- Resolve owner/merchant identity from trusted authentication context or the approved Member 4 integration contract, not from an unverified request field.
- Include ownership/tenant scope in repository queries and mutation predicates where practical; do not fetch a private record and rely only on the frontend to hide it.
- Changing a parcel UUID or tracking number must not allow a user to read or mutate another party's data.
- Merchant-originated parcel creation must be tenant-scoped and idempotent under the agreed contract.
- A forbidden or cross-tenant request must not modify parcel data, status history, idempotency outcomes belonging to another principal, or outbox records.
- The team must decide whether inaccessible objects return `403 Forbidden` or a privacy-preserving `404 Not Found` consistently. Do not vary responses in a way that leaks record existence.

## 4. Data Disclosure, Privacy and Logging

### 4.1 Response disclosure rules

| Data | Authorized owner / operational use | Public tracking or unauthorized caller |
|---|---|---|
| Public tracking number and approved status | Return when required by the contract | Only the public fields explicitly approved by Member 5 |
| Full pickup/drop-off address | Return only to authorized roles when needed | Must not be exposed |
| Customer/recipient phone or email | Return only to authorized roles when needed | Must not be exposed |
| Parcel description, private delivery notes, proof images/references | Restrict to roles with an approved operational need | Must not be exposed |
| Accepted fee, currency, pricing-rule/version snapshot | Return to authorized owner/merchant as contract permits | Do not expose merchant-only or private receipt fields |
| Internal IDs, actor IDs, tenant IDs, idempotency metadata | Use internally and expose only where the API contract requires it | Do not expose internal operational metadata unnecessarily |
| Stack traces, SQL errors, secrets, environment values | Never return to clients | Never return to clients |

### 4.2 Data minimization and retention

- Collect only the contact, address, parcel, and pricing fields required for booking and fulfilment.
- Keep accepted-price snapshots needed for a consistent receipt/audit trail; do not recalculate historical receipts from current pricing rules.
- Do not copy full addresses, phone numbers, idempotency keys, credentials, or unnecessary payloads into events or logs.
- Retention, deletion, legal/audit retention, and treatment of status history must be agreed with the team. This document does not invent retention durations.
- Demo and automated test fixtures must use synthetic data only.
- Public tracking disclosure is owned by Member 5 in coordination with Member 2; payloads and logs must follow the same privacy boundary.

### 4.3 Logging and audit

Log relevant security and failure events using structured logs and request/correlation IDs. Suitable fields may include operation name, result category, timestamp, request ID, and a non-sensitive internal reference where approved.

Never log passwords, access tokens, API keys, webhook secrets, full request bodies containing personal data, or database connection strings. Log authorization denials and repeated abuse in a way that supports investigation without exposing private parcel details. Status history/audit rows are business records and must not be confused with general application logs.

## 5. Input Validation and Abuse Controls

- Validate required fields, formats, lengths, allowed enum values, measurement ranges, and cross-field constraints on the server.
- Reject malformed JSON, unsupported content types, oversized requests, invalid identifiers, unsupported service levels, and unknown/invalid zone references according to the published contract.
- Do not trust a client-supplied price, pricing version, parcel status, owner ID, or merchant ID as authoritative.
- Use exact decimal or integer-minor-unit money representation; currency and rounding must follow the approved pricing decision.
- Use parameterized database queries and safe serialization. Do not build SQL from user-controlled values.
- Apply request-size limits and reasonable timeouts at the agreed gateway/application boundary.
- Fee estimation and public tracking can be abused for enumeration or resource exhaustion. Rate limits and monitoring thresholds must be selected by the team/platform owners; no numeric limit is asserted here.
- Ensure pagination and bounded query behavior for history/list endpoints where provided.
- Validate public tracking numbers with the approved format and rate-limit lookup attempts through Member 5/gateway policy. Tracking references should be sufficiently hard to guess under the approved design.
- Treat notes, address fields, merchant payloads, and other user-controlled text as untrusted data; escape output in consuming UIs and never execute or render it as trusted HTML.
- Pricing rule administration must require the approved admin permission and audit changes. Exact administration API ownership must be confirmed with Member 7.

## 6. Idempotency, Concurrency and State Integrity

### 6.1 Booking idempotency

- Parcel booking and merchant order creation must support the shared idempotency behavior agreed by the team.
- For a repeated request with the same idempotency key and same effective request, return the original/replayed outcome according to the published contract and do not create another parcel.
- Reusing a key with a different effective payload must return the agreed conflict response and must not create a second parcel.
- Enforce uniqueness and concurrency safety in PostgreSQL, not only with an in-memory check.
- Scope keys to the appropriate authenticated principal/tenant and operation according to the approved contract; never allow one caller to replay another caller's stored result.
- Store only the minimum replay data needed and protect it as sensitive operational data.
- Retention/expiry and behavior after expiry require explicit approval.

### 6.2 Status transition safety

- All status changes must go through the canonical parcel domain/application use case and approved lifecycle matrix.
- Reject arbitrary status strings, illegal transitions, stale-version updates, and duplicate/conflicting transitions.
- Status state, corresponding history/audit record, and required outbox event must commit atomically in one PostgreSQL transaction.
- Never publish a domain event before its business transaction commits.
- Member 3 owns courier execution details; Members 2, 3, and 5 must agree lifecycle states, actor permissions, failed-delivery semantics, and authoritative status ownership before final implementation.
- A rejected transition must not leave a changed status, new history row, or outbox event.

## 7. Failure Modes and Required Behavior

| Failure / threat | Expected safe behavior | Recovery / owner coordination |
|---|---|---|
| Missing/invalid authentication context | Reject protected operation; do not access private parcel data | Member 1 auth middleware; Member 2 integration tests |
| Cross-customer or cross-tenant ID substitution | Deny access without changing data or leaking private fields | Member 1 ownership helpers; Member 4 tenant contract; Member 2 query-level enforcement |
| Malformed/oversized booking request | Return structured client error; do not partially persist | Validation and request limits at agreed gateway/API boundary |
| Pricing rule missing, disabled, or ambiguous | Return an explicit domain error; never invent a fallback tariff | Pricing policy/admin owner must define rule-selection behavior |
| PostgreSQL unavailable before transaction | Return safe service/dependency error; do not claim booking succeeded | Member 9 environment/health; bounded client retry using idempotency |
| Transaction fails partway through booking | Roll back all related writes; no partial parcel/history/outbox success | Database transaction and integration test |
| Tracking-number collision | Unique constraint prevents duplicate; bounded generation retry or safe failure | Database constraint and tested generator policy |
| Same idempotency key, same payload | Replay agreed result; no duplicate parcel | Member 2 idempotency store and database uniqueness |
| Same key, different payload | Conflict; do not mutate original result or create another parcel | Stable request fingerprint and contract error |
| Concurrent duplicate booking | At most one logical booking commits | Unique constraints/transactional coordination; concurrency test |
| Concurrent conflicting status transitions | Reject stale/conflicting operation; retain one valid transition history | Canonical state/version check and transaction |
| Required outbox write fails | Roll back the corresponding business change and history | Shared Member 8 outbox writer within the same transaction |
| Outbox relay or RabbitMQ unavailable after commit | Booking/status transaction remains committed; event stays durable for retry; do not create a second business operation to compensate | Member 8 relay retry/monitoring; Member 9 observability |
| Event published more than once | Downstream consumer handles duplicates idempotently; Member 2 must not assume exactly-once delivery | Member 8 consumer idempotency |
| Public/private field accidentally added to DTO | Contract/security test fails before release; remove field or restrict endpoint | Member 2 and Member 5 disclosure review |
| Logs contain personal data or secret | Treat as a security defect; restrict access and remove/rotate exposed secret as appropriate | Module owner plus platform/security owners; assess impact |
| Migration fails | Stop release/startup path according to migration tooling; preserve diagnosable logs; do not manually edit shared DB silently | Database coordinator and Member 9 |
| Stale/missing history detected | Flag for investigation; no silent history fabrication or destructive repair | Audited deterministic repair approved by team |
| Unknown or stale cache/projection | Re-read authoritative PostgreSQL state or use documented fallback; do not present stale data as truth | Member 5 cache/read strategy |
| Repeated fee-estimate or tracking enumeration abuse | Apply approved rate limit and safe response; record sanitized abuse signal | Gateway/platform owner and relevant API owner |

## 8. Recovery and Reconciliation Rules

### 8.1 Database transaction failure

1. Do not return a successful booking/status response unless the required database transaction commits.
2. Roll back parcel state, related address/price snapshot, initial or changed status history, idempotency completion state, and outbox write together when they belong to the same atomic operation.
3. Return the agreed safe error envelope with request/correlation ID. Do not return raw SQL details or stack traces.
4. If the client cannot tell whether a request committed because the connection failed after commit, it should retry with the same idempotency key; the server must resolve the retry using the persisted idempotency outcome.

### 8.2 Outbox and broker failure

- Member 2 writes the outbox record through Member 8's agreed interface in the same transaction as the associated business change.
- Once the business transaction commits, temporary RabbitMQ/relay unavailability must not cause the client to create a second parcel as a workaround.
- Member 8 owns publishing, publisher confirmation, bounded retry/backoff, dead-letter behavior, and relay monitoring.
- Outbox relay must mark a record published only after broker confirmation. A crash can cause duplicate publication; consumers must tolerate it.
- Reconciliation may identify stale unpublished records and safely make them eligible for retry or raise an alert. It must not bypass the relay or fabricate a business event.

### 8.3 Data inconsistency and manual repair

- Reconciliation should detect inconsistencies such as a parcel state with missing required history or conflicting status versions.
- Do not silently invent a status, delete legitimate history, or repair production-like data through ad hoc SQL.
- Any repair procedure must be deterministic, authorized, logged/audited, reviewed, and followed by verification tests.
- The team must decide which consistency checks run automatically, their thresholds, and who receives alerts.

### 8.4 Retry policy

- Retry only operations that are safe to retry and only within a bounded policy.
- Use idempotency for client booking retries and idempotent processing for asynchronous events.
- Do not retry permanent validation/authorization errors as though they were transient.
- Use explicit timeouts and retry budgets to avoid retry storms. Exact timeout/backoff values belong to shared API/platform decisions and are not invented here.
- Avoid holding PostgreSQL transactions open while performing external network calls.

## 9. Security and Failure Test Matrix

| ID | Test scenario | Expected result | Test level |
|---|---|---|---|
| SEC-001 | Missing, invalid, or expired authentication context on protected parcel endpoint | Denied; no private data returned and no write occurs | API/security integration |
| SEC-002 | Customer A requests Customer B's detail, history, or receipt | Agreed forbidden/not-found response; no private data disclosed | API/security integration |
| SEC-003 | Merchant attempts cross-tenant read/create using a forged merchant ID | Trusted tenant context wins; request is denied or scoped correctly | Integration with Member 4 |
| SEC-004 | Public tracking/DTO response is checked for addresses, phone numbers, private notes, proof references, and merchant-only fields | Restricted fields are absent | Contract/security integration with Member 5 |
| SEC-005 | Malformed JSON, invalid enum, excessive field length, unsupported measurements, oversized request | Safe structured error; no partial persistence | Unit + API integration |
| SEC-006 | Client supplies forged price, owner, merchant, or status field | Values are ignored or rejected according to contract; server rules remain authoritative | API integration |
| SEC-007 | Same idempotency key and same request sent repeatedly/concurrently | One logical booking; response follows replay contract | Database concurrency integration |
| SEC-008 | Same idempotency key reused with a different payload | Conflict; original booking remains unchanged | API/database integration |
| SEC-009 | Force tracking-number uniqueness collision | No duplicate tracking number is committed | Database integration |
| SEC-010 | Force failure after some related database writes but before transaction commit | Full rollback of required atomic records | Database integration |
| SEC-011 | Force outbox writer failure during booking/status change | Business transaction rolls back | Integration with Member 8 interface |
| SEC-012 | RabbitMQ unavailable after business transaction commits | Parcel remains committed; outbox is pending and recoverable; no duplicate booking required | Integrated resilience test with Member 8 |
| SEC-013 | Duplicate event delivery | Downstream idempotency prevents duplicate side effects under the agreed consumer contract | Integration with Member 8 |
| SEC-014 | Attempt illegal/stale/concurrent status transition | Rejected without state/history/outbox mutation | Unit + database integration |
| SEC-015 | Inspect application logs and error responses after failures | No secrets, raw stack traces, or unnecessary personal data exposed | Security review/test |
| SEC-016 | Run migrations against clean database and simulate migration failure | Migration outcome is explicit; no undocumented manual repair | Migration integration |
| SEC-017 | Trigger configured request-rate/resource limit | Correct safe rate-limit behavior and observable sanitized event | API/gateway integration; only after thresholds are approved |
| SEC-018 | Simulate stale outbox or missing-history detection | Alert/retry/investigation path is followed; no invented state | Controlled recovery test |

## 10. Operational Security and Monitoring

Member 2 should provide enough safe signals for Member 9 and the team to diagnose module failures. Candidate signals include:

- Booking/quote request counts and latency, split by result category.
- Validation, authorization, conflict, and dependency-error counts.
- Database transaction failures and connection-pool saturation.
- Idempotency replays and key/payload conflict counts.
- Tracking-number generation collision/retry failures.
- Required outbox-write failures and age of pending records, using the shared outbox observability model.
- Illegal/stale transition rejections and history consistency check failures.

Do not use parcel IDs, user IDs, tracking numbers, or other high-cardinality identifiers as metric labels. Keep detailed identifiers in access-controlled logs only when needed and approved. Thresholds, dashboards, alert routes, and SLOs must be agreed with Member 9; this document does not claim that monitoring is already configured.

## 11. Incident Handling and Response

For suspected data exposure, unauthorized mutation, credential leakage, or inconsistent parcel state:

1. Record the incident time, affected component, sanitized request/correlation IDs, and observed impact.
2. Avoid exposing more data while diagnosing; disable or restrict the affected endpoint/configuration if authorized and necessary.
3. Notify the relevant owners: Member 1 for auth, Member 2 for parcel/pricing, Member 3 for courier/lifecycle, Member 4 for merchant tenant boundaries, Member 5 for tracking disclosure, Member 8 for event pipeline, and Member 9 for platform/observability.
4. Rotate/revoke an exposed credential through its owner; never paste the secret into an issue or chat.
5. Preserve necessary audit evidence while limiting access to personal data.
6. Apply a reviewed fix, add a regression test, and verify data integrity and affected integrations.
7. Document root cause, impact, remediation, and follow-up tasks without publishing unnecessary personal data.

This is a project-level workflow recommendation; the team should confirm the actual incident coordinator and communication channel.

## 12. Decisions Required Before Security/Recovery Sign-Off

The following remain unresolved in the supplied proposed baselines and require explicit team decisions:

1. Final authentication/session strategy and trusted internal service context (Member 1).
2. Approved role/permission matrix and consistent `403` versus privacy-preserving `404` policy.
3. Merchant tenant identity propagation and endpoint boundary (Member 4).
4. Final public tracking disclosure matrix and tracking-number format/rate limit (Member 5 and platform owners).
5. Pricing formula, currency, rounding, active-rule selection, and behavior when no rule matches.
6. Canonical lifecycle, allowed transitions, actor matrix, failed-delivery and reattempt semantics (Members 2, 3, and 5).
7. Exact idempotency key scope, fingerprinting, storage, expiry/retention, and replay response.
8. Shared outbox interface, event schema, event identifiers, and Member 8 recovery/DLQ policy.
9. Personal-data retention/deletion policy and status-history retention.
10. Request size, timeout, rate-limit, retry, alert, and performance thresholds.
11. Migration failure/rollback or forward-fix convention and approved reconciliation permissions.
12. Final incident owner, escalation route, and evidence-retention process.

## 13. Acceptance Checklist

- [ ] Protected endpoints use the shared authentication context and enforce server-side authorization.
- [ ] Parcel reads and mutations enforce owner/tenant scope; caller-supplied IDs are not trusted as proof of ownership.
- [ ] Public tracking disclosure is limited to the approved Member 5 field matrix.
- [ ] Pricing, status, owner, and tenant values cannot be forged by client payloads.
- [ ] Input validation and request/resource limits are implemented at the agreed boundaries.
- [ ] Idempotency prevents duplicate bookings and cross-principal replay.
- [ ] Database uniqueness, transaction atomicity, and concurrency controls are tested against PostgreSQL.
- [ ] Status/history/outbox changes use the approved canonical lifecycle and shared Member 8 interface.
- [ ] Database/broker failure behavior is documented and verified; retries are bounded and safe.
- [ ] Logs/errors do not expose secrets, stack traces, or unnecessary personal data.
- [ ] Recovery/reconciliation paths are auditable and never invent parcel state.
- [ ] Security/failure test evidence and known gaps are recorded.
- [ ] Unresolved decisions are approved or explicitly tracked as blockers before release.

