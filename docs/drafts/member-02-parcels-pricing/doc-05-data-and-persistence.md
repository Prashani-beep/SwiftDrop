**SwiftDrop Parcel Delivery & Courier Management System**

Document 5 — Data & Persistence Specification

## Member 2: Parcel Management & Pricing

# 1. Purpose and Scope

This specification defines data owned or persisted by Member 2, integrity rules, and migration/transaction practices required to keep parcel booking, pricing, status history, and event recording consistent.

## 1.1 In scope

- Parcel booking and core parcel business data.

- Unique public tracking number and internal parcel identifier.

- Pickup/drop-off snapshots needed to fulfil a booking.

- Accepted delivery-price snapshot and the pricing rule/version used.

- Current parcel status and auditable status-history records within the agreed lifecycle boundary.

- Idempotency for safe booking retries, using a shared mechanism if the team provides one.

- Outbox rows written through Member 8's shared interface in the same transaction as the business change.

- Data needed by authorized parcel detail, history, and receipt-data reads.

- Pricing-rule persistence only to the extent assigned to Member 2.

## 1.2 Out of scope / owned elsewhere

- Identity, credentials, roles, and authentication: Member 1.

- Merchant onboarding, API credentials, and tenant validation: Member 4.

- Courier assignment, delivery attempts, and proof-of-delivery files: Member 3.

- Tracking API projection/SSE: Member 5; it must not become a competing lifecycle authority.

- Frontends and receipt presentation/download format: Members 6–7.

- Outbox relay, RabbitMQ topology, workers, notifications, webhook retries, and dead-letter handling: Member 8.

- Platform-wide deployment and observability: Member 9.

- AI assistant tool implementation: Member 10.

- Shared zone administration and pricing-rule administration UI: ownership to be confirmed.

# 2. Data Ownership and Source of Truth

PostgreSQL is authoritative for parcel business state. RabbitMQ is transport, not the system of record. Any cache is non-authoritative and needs an explicit freshness/invalidation policy.

| **Data area**                                           | **Proposed owner / authority**                                              | **Member 2 rule**                                                                                                        |
|---------------------------------------------------------|-----------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Parcel core record/current status                       | Member 2 parcel domain, subject to lifecycle agreement with Members 3 and 5 | Only the canonical parcel use case/aggregate performs legal status changes.                                              |
| Status history                                          | Member 2 persistence boundary, with actors supplied by approved use cases   | Append an audit row for each accepted status change; do not rewrite history to hide prior events.                        |
| Accepted price snapshot                                 | Member 2 booking data                                                       | Store amount, currency, and pricing-rule/version reference; do not recalculate historical receipts using today's tariff. |
| Pricing rules                                           | Calculation belongs to Member 2; table/admin write ownership pending        | Use only enabled, effective, approved rules; do not hard-code unapproved rates.                                          |
| Pickup/drop-off details                                 | Booking data; canonical address ownership to be confirmed                   | Preserve the address accepted for the booking and restrict disclosure.                                                   |
| Idempotency records                                     | Member 2 or approved shared mechanism                                       | Enforce safe replay semantics and prevent duplicate bookings.                                                            |
| Outbox records                                          | Shared infrastructure owned by Member 8                                     | Use the shared interface inside the business transaction; do not create a competing outbox schema.                       |
| Users, merchants, zones, assignments, delivery attempts | Respective module owners                                                    | Reference canonical identifiers; referencing a table does not transfer ownership.                                        |

Other modules may call published capabilities or read approved interfaces. They must not bypass validation and mutate parcel status through ad hoc SQL. The Tracking API reads authoritative parcel/status data through the team-approved access pattern.

# 3. Proposed Logical Data Model

This is a logical model for review, not a finalized schema. Reuse canonical/shared tables where already defined.

> <img src="Document5-Markdown-media/media/image1.png" style="width:6.9in;height:4.6in" />

Notes:

- Actual foreign keys/cardinalities depend on the approved canonical ERD.

- PRICING_QUOTES is optional. Persist quote records only if stable quote IDs, expiry, auditability, or booking-time verification are approved requirements.

- IDEMPOTENCY_KEYS may be shared. Do not duplicate an existing shared mechanism.

- OUTBOX_EVENTS is conceptual only; Member 8 owns the actual schema/interface.

- Address data may be embedded, stored in a parcel-address child table, or reference approved shared records. The design must preserve booking snapshots and privacy.

# 4. Data Dictionary — Proposed Parcel Data

## 4.1 \`parcels\` (proposed core table)

| **Field**                   | **Proposed type / shape**            | **Required?**                | **Integrity and meaning**                                                                                                                                 |
|-----------------------------|--------------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| \`id\`                      | UUID recommended                     | Yes                          | Primary key; use repository convention if already established.                                                                                            |
| \`tracking_number\`         | Bounded text/varchar                 | Yes                          | Server-generated public reference; unique constraint; format is a team decision.                                                                          |
| \`customer_id\`             | Canonical user FK/ID                 | Conditional                  | Owner reference for customer booking; use canonical identity record if FK permitted.                                                                      |
| \`merchant_id\`             | Canonical merchant FK/ID             | Conditional                  | For merchant bookings; tenant context must come from verified authentication.                                                                             |
| \`created_by_type\`         | Approved enum/text                   | Yes                          | Actor/source classification using shared identity conventions.                                                                                            |
| \`created_by_id\`           | Canonical identifier                 | Conditional                  | Audit attribution; system-actor semantics need agreement.                                                                                                 |
| \`status\`                  | Approved enum/text                   | Yes                          | Authoritative state, constrained to approved lifecycle values. Initial \`CREATED\` on booking follows the current baseline, subject to team confirmation. |
| \`weight_kg\`               | Exact numeric                        | If weight required           | Positive and within approved limits; units kg; precision/max must be approved.                                                                            |
| \`length_cm\`               | Exact numeric                        | If dimensions required       | Positive and within approved limits; units cm.                                                                                                            |
| \`width_cm\`                | Exact numeric                        | If dimensions required       | Positive and within approved limits; units cm.                                                                                                            |
| \`height_cm\`               | Exact numeric                        | If dimensions required       | Positive and within approved limits; units cm.                                                                                                            |
| \`description\`             | Bounded text                         | Optional/conditional         | Parcel description; length limit and content policy to be approved.                                                                                       |
| \`service_level\`           | Approved enum/text                   | If selected by business flow | Must match supported service level.                                                                                                                       |
| \`pickup_zone_id\`          | Canonical zone ID                    | Conditional                  | Zone ownership and FK strategy must be agreed.                                                                                                            |
| \`dropoff_zone_id\`         | Canonical zone ID                    | Conditional                  | Validate zone exists and is eligible.                                                                                                                     |
| \`accepted_price_amount\`   | Integer minor units or exact numeric | Yes on booking               | Exact money only; selected representation and scale must be standardized.                                                                                 |
| \`accepted_price_currency\` | Currency code                        | Yes on booking               | Must accompany amount; currency is not finalized here.                                                                                                    |
| \`pricing_rule_version\`    | Immutable version/reference          | Yes where versioning exists  | Identifies rule used; preserve enough data to explain historical price.                                                                                   |
| \`pricing_snapshot\`        | JSONB or normalized fields           | Recommended if needed        | Calculation inputs/components for audit; avoid redundant/personal data.                                                                                   |
| \`created_at\`              | \`timestamptz\`                      | Yes                          | Server-generated UTC timestamp.                                                                                                                           |
| \`updated_at\`              | \`timestamptz\`                      | Yes                          | Server-maintained UTC timestamp for mutable current record.                                                                                               |

Conditional fields are mandatory only if the approved business flow requires them. Final field requirements must match the approved API contract.

## 4.2 \`parcel_addresses\` (proposed, if snapshots are stored separately)

| **Field**         | **Proposed type / shape**               | **Required?** | **Integrity and meaning**                                                                 |
|-------------------|-----------------------------------------|---------------|-------------------------------------------------------------------------------------------|
| \`id\`            | UUID                                    | Yes           | Primary key.                                                                              |
| \`parcel_id\`     | UUID FK to \`parcels\`                  | Yes           | Parent parcel; deletion/cascade policy must account for audit retention.                  |
| \`address_type\`  | Approved pickup/drop-off enum           | Yes           | Restrict values; enforce one snapshot per type unless revisions are explicitly supported. |
| \`contact_name\`  | Bounded text                            | As required   | Minimum necessary contact data.                                                           |
| \`contact_phone\` | Bounded text                            | As required   | Validate format; do not log.                                                              |
| \`address_lines\` | Structured fields or bounded array/text | As required   | Match API and database conventions.                                                       |
| \`city\`          | Bounded text/canonical reference        | As required   | Canonicalization and zone validation need agreement.                                      |
| \`zone_id\`       | Canonical zone ID                       | Conditional   | Zone owner and FK strategy pending.                                                       |
| \`created_at\`    | \`timestamptz\`                         | Yes           | Server timestamp in UTC.                                                                  |

Preserve the address accepted for a booking if a customer's saved address later changes. Do not duplicate personal information unnecessarily.

## 4.3 \`parcel_status_history\` (proposed append-only audit table)

| **Field**           | **Proposed type / shape**              | **Required?**        | **Integrity and meaning**                                   |
|---------------------|----------------------------------------|----------------------|-------------------------------------------------------------|
| \`id\`              | UUID/canonical generated key           | Yes                  | Primary key.                                                |
| \`parcel_id\`       | UUID FK to \`parcels\`                 | Yes                  | Parent parcel.                                              |
| \`previous_status\` | Approved enum/text, nullable initially | Conditional          | Null only for initial event if that convention is approved. |
| \`new_status\`      | Approved enum/text                     | Yes                  | Must be an approved state.                                  |
| \`actor_type\`      | Approved enum/text                     | Yes                  | Authenticated actor class or approved system actor.         |
| \`actor_id\`        | Canonical ID                           | Conditional          | Nullable for system actor only if approved.                 |
| \`reason_code\`     | Approved code/text                     | Conditional          | Required for transitions requiring a reason.                |
| \`note\`            | Bounded text                           | Optional             | Avoid secrets/personal data; visibility is role-dependent.  |
| \`occurred_at\`     | \`timestamptz\`                        | Yes                  | Server timestamp in UTC, not client-trusted time.           |
| \`correlation_id\`  | Bounded text/UUID                      | Optional/recommended | Connects change to request/operation.                       |

History is append-only for normal application roles. Corrections should use an approved compensating/audit record rather than silently editing history. Initial CREATED history must be committed atomically with booking.

## 4.4 \`pricing_rules\` (only if ownership is assigned to Member 2)

| **Field**                      | **Proposed type / shape**     | **Required?** | **Integrity and meaning**                                                   |
|--------------------------------|-------------------------------|---------------|-----------------------------------------------------------------------------|
| \`id\`                         | UUID/canonical key            | Yes           | Primary key.                                                                |
| \`version\`                    | Integer/immutable version ID  | Yes           | Distinguishes revisions; unique within agreed rule scope.                   |
| \`name\`                       | Bounded text                  | Yes           | Internal human-readable name.                                               |
| \`enabled\`                    | Boolean                       | Yes           | Only enabled rules can be selected.                                         |
| \`effective_from\`             | \`timestamptz\`               | Yes           | UTC applicability start.                                                    |
| \`effective_to\`               | \`timestamptz\`, nullable     | Conditional   | If present, later than start; boundary semantics must be documented.        |
| \`rule_definition\`            | Typed columns/validated JSONB | Yes           | Approved schema only; never execute arbitrary expressions from stored data. |
| \`created_at\`, \`created_by\` | Timestamp/canonical actor     | Yes           | Auditability.                                                               |

Use immutable versions or equivalent snapshots so past prices remain explainable. Define overlapping effective-rule behavior before implementing writes. Do not invent rates.

## 4.5 \`pricing_quotes\` (optional)

Create only if the team approves quote identity, expiry, reuse, or audit requirements.

| **Field**                 | **Proposed type / shape**        | **Required?** | **Integrity and meaning**                          |
|---------------------------|----------------------------------|---------------|----------------------------------------------------|
| \`id\`                    | UUID                             | Yes           | Quote identifier.                                  |
| \`request_fingerprint\`   | Canonical hash/representation    | Recommended   | Confirms inputs match; canonicalize unambiguously. |
| \`amount\`, \`currency\`  | Exact money fields               | Yes           | Exact result and currency.                         |
| \`pricing_rule_version\`  | Immutable rule reference/version | Yes           | Rule used.                                         |
| \`calculation_inputs\`    | Validated structured snapshot    | Recommended   | Minimum inputs needed for reproducibility.         |
| \`created_at\`            | \`timestamptz\`                  | Yes           | UTC timestamp.                                     |
| \`expires_at\`            | \`timestamptz\`, nullable        | Conditional   | Only if quote expiry is adopted.                   |
| \`consumed_by_parcel_id\` | Parcel FK, nullable              | Conditional   | Only if approved quote is single-use.              |

Quote expiry duration, quote reuse, invalidation, and persistence are unresolved. Do not invent values.

## 4.6 Idempotency storage (shared or module-local)

If no shared store exists, the logical record needs:

- owner/scope context, operation name, idempotency key;

- request fingerprint;

- processing/result state and enough information to replay the prior outcome;

- creation and expiry timestamps under an approved retention policy.

Enforce uniqueness over the agreed scope + operation + key. Same key + same effective request must not create a second parcel and should replay the recorded result. Same key + different request must return the contract's conflict response. Key length, retention, concurrency locking, and exact replay semantics require team approval.

## 4.7 Outbox events

Member 8 owns the canonical outbox schema. Member 2 must use the shared interface, not invent a competing table. The event record is persisted in the same transaction as the business change. Engineering standards call for a stable event ID, event type, schema version, UTC timestamp, aggregate/parcel ID, and correlation ID where available. Event names, payload schemas, routing, and columns belong in the shared event contract.

# 5. Relationships, Constraints, and Indexes

## 5.1 Required integrity controls

- Primary keys on owned tables.

- Foreign keys to canonical tables where ownership and migration dependencies permit.

- NOT NULL for genuinely mandatory fields.

- CHECK constraints for positive measurements, valid money representation, and valid date ranges where appropriate.

- Unique constraint on tracking_number.

- Unique constraint on the agreed idempotency scope/operation/key.

- Pricing-rule overlap constraint only if the approved tariff model requires it and the strategy is agreed.

- Controlled status values; domain logic must enforce legal transitions. A DB enum/check can reject unknown values but does not replace transition validation.

- Indexes on foreign keys and documented query paths.

## 5.2 Proposed indexes — validate against actual queries

| **Index**                | **Proposed columns**                                                 | **Reason / caveat**                         |
|--------------------------|----------------------------------------------------------------------|---------------------------------------------|
| Unique tracking lookup   | \`parcels(tracking_number)\` UNIQUE                                  | Required uniqueness and lookup.             |
| Customer list            | \`parcels(customer_id, created_at DESC)\`                            | If owner-scoped recent list is common.      |
| Merchant list            | \`parcels(merchant_id, created_at DESC)\`                            | If merchant list flow is confirmed.         |
| Status filter            | \`parcels(status, created_at)\`                                      | Coordinate with Member 3; add only if used. |
| History timeline         | \`parcel_status_history(parcel_id, occurred_at, id)\`                | Stable chronological history.               |
| Address snapshot         | \`parcel_addresses(parcel_id, address_type)\` UNIQUE if one per type | Lookup and expected cardinality.            |
| Pricing effective lookup | Candidate index on enabled/effective fields                          | Tune to the actual tariff query.            |
| Idempotency uniqueness   | \`(scope, operation, key)\` UNIQUE                                   | Prevents duplicate operation records.       |
| Outbox polling           | Member 8-defined index                                               | Do not create independently.                |

Avoid speculative indexes on every column. Check representative query plans with EXPLAIN (ANALYZE, BUFFERS) in safe test data where practical. Paginated sorting must be deterministic with a unique tie-breaker.

# 6. Transaction Boundaries and Concurrency

## 6.1 Parcel booking transaction

A successful booking should be atomic. Within one PostgreSQL transaction, perform the applicable steps:

1.  Claim/lock the idempotency key under the approved strategy.

2.  Validate quote/rule reference and authoritative pricing inputs as required by the approved design.

3.  Generate and insert the parcel with unique tracking number and accepted-price snapshot.

4.  Persist pickup/drop-off snapshots or approved equivalent.

5.  Insert initial CREATED history if confirmed by the lifecycle agreement.

6.  Insert required outbox record through Member 8's shared interface.

7.  Store idempotent outcome/result.

8.  Commit.

If a required step fails, roll back parcel, history, outbox, and idempotency completion state together. Never publish directly to the broker inside the transaction or treat an attempted publish as a substitute for an outbox row. Do not make slow external HTTP calls while a transaction is open.

## 6.2 Status transition transaction

For each approved status change:

- Lock or conditionally update current state to prevent conflicting transitions.

- Validate current/next state, actor permissions, and cross-module preconditions against the agreed lifecycle.

- Update current status and timestamp.

- Append history with previous/new state, actor, server timestamp, and approved reason/reference.

- Insert corresponding outbox record via the shared interface.

- Commit all changes atomically.

Use a team-approved concurrency strategy (e.g. row locking or optimistic version checks). Two conflicting transitions based on the same stale state must not both succeed.

## 6.3 Pricing-rule changes

- Validate and audit rule changes.

- Each booking uses one well-defined rule version, not a mixture of versions.

- Preserve immutable versions or an equivalent snapshot for historical reproducibility.

- Define behavior if multiple enabled rules are effective for the same scope/time.

## 6.4 Isolation and retry

Use the configured PostgreSQL isolation level unless a documented invariant requires stronger isolation. Use unique constraints and explicit locking/conditional updates for invariants they protect. Treat uniqueness conflicts as expected concurrency outcomes where applicable. Retry only known transient failures and preserve idempotency.

# 7. Validation and Data Integrity Rules

- Validate required fields, formats, lengths, ranges, enums, and cross-field constraints on the server.

- Use explicitly documented units. Current API examples use kg and cm; confirm consistent units across UI, API, and DB.

- Reject zero, negative, non-finite, unsupported, or out-of-range measurements according to approved limits. Exact limits are not specified here.

- Calculate prices server-side using exact money arithmetic; never use binary floating point for money.

- Never treat a client-supplied amount as authoritative. If quote IDs are used, validate ownership/tenant and input/rule consistency under the approved quote rules.

- Use server-generated UTC timestamps for audit events.

- Generate tracking numbers server-side and enforce uniqueness in PostgreSQL. On collision, retry generation under a bounded policy.

- Validate supported zones and service levels before booking.

- Validate status transitions centrally; clients cannot mutate arbitrary status strings.

- Keep history immutable through normal application paths.

- Enforce owner/tenant authorization in backend queries/use cases, not only in UI.

- Use explicit response DTOs instead of serializing DB rows by default.

# 8. Retention, Deletion, Audit, and Privacy

Source documents require minimization, access restrictions, and agreed retention/deletion behavior, but do not set numeric retention periods. The team must approve periods for parcel data, address/contact snapshots, status history, quote records, idempotency records, and outbox rows.

- Do not delete audit/history just to simplify the UI. Define deletion/anonymization/retention behavior for account removal and operational/legal needs.

- Do not cascade-delete business history without reviewing audit requirements.

- Restrict private contact/address data by role and purpose.

- Public tracking exposes only an approved minimal field set. Never expose full addresses, phone numbers, private notes, proof photos, or merchant-only data through public tracking.

- Keep personal contact data, credentials, API keys, and unnecessary parcel details out of logs/events.

- Use synthetic data only; never seed real personal data into demo/test environments.

- Supply secrets through secure configuration, not source control.

- Apply least-privilege database roles; separate runtime and schema-migration permissions where practical.

# 9. Caching Policy

**Baseline: no cache is required for correctness.** Read PostgreSQL for authoritative parcel status, accepted-price snapshots, and history unless a cache is approved.

If a cache is introduced:

- It is never the source of truth.

- Define TTL, invalidation triggers, and cache-failure behavior.

- Agree freshness expectations for status/public tracking.

- Never share private data across tenants/users through unsafe cache keys.

- Invalidate/update only after successful commit.

- Do not add Redis without a specific requirement and documented benefit.

# 10. Migration Strategy and Ordering

Member 2 owns migrations for module-owned tables, subject to database-coordinator review. Use versioned repository migrations; do not manually edit a shared DB as the deployment process.

## 10.1 Proposed sequence

9.  Confirm ID/timestamp/naming/money conventions, shared references, and canonical lifecycle.

10. Create parcel core table with keys, constraints, and unique tracking constraint.

11. Create address snapshots or approved equivalent.

12. Create status history, FK, and timeline index.

13. Create pricing-rule tables only after ownership/schema approval; add quote storage only if adopted.

14. Reuse shared idempotency or create the approved module-local mechanism.

15. Add query-backed indexes; coordinate outbox indexing with Member 8.

16. Add deterministic synthetic seed/test data separately from production migrations.

17. Test clean-database migration and upgrade of the supported development database; follow the team's rollback/forward-fix convention.

This is a dependency proposal, not a fixed migration-number schedule. Migration IDs and order must match the repository's tool and other members' prerequisites.

## 10.2 Migration quality

- Keep migrations small, reviewed, and safe for the supported workflow.

- Do not edit an already shared/applied migration; add a corrective migration.

- Review destructive/data-transforming changes with the team.

- Verify constraints against seeded/demo data before enforcement.

- Run migrations in CI against clean PostgreSQL.

- Coordinate changes to another member's table.

- Never include real credentials/personal data in migration or seed files.

# 11. Seed and Test Data Requirements

Provide deterministic synthetic fixtures for:

- At least one valid pricing rule/version after tariff approval.

- Valid and invalid measurement boundaries based on approved limits.

- Parcels in each approved lifecycle state after the lifecycle matrix is finalized.

- Multiple chronological history rows for one parcel.

- Two customers and, if merchant flow is included, two tenants to test isolation.

- Same idempotency key/same request and same key/different request.

- Tracking-number collision/uniqueness handling.

- Rule-version change proving historical accepted price does not change.

- Failure between related writes proving parcel, history, outbox, and idempotency state do not partially commit.

- Synthetic addresses/contact data only.

Tests must not rely on execution order or unspecified production tariffs/currency/zone IDs.

# 12. Performance and Operational Considerations

- Use bounded pagination and deterministic sorting for parcel lists.

- Query history using an index beginning with parcel_id and approved chronological fields.

- Avoid N+1 queries for parcel details/receipt reads.

- Keep transactions short; never hold DB locks while waiting on external services.

- Monitor query latency, pool saturation, slow queries, migration failures, and constraint/transaction errors through shared observability.

- Use parameterized queries through the chosen Go DB layer.

- Set numeric performance targets only if the SRS/team agrees measurable SLAs; none are invented here.

- Platform team owns backup/restore infrastructure; Member 2 must verify module data is included in recovery tests.

# 13. Data Access and Permission Boundaries

| **Caller / role context**   | **Allowed access principle**                                                                                  |
|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Customer                    | Read own parcel, authorized history, receipt; cannot mutate arbitrary lifecycle state.                        |
| Merchant                    | Access only parcels in verified tenant scope and fields allowed by merchant contract.                         |
| Courier/dispatcher          | Access operational fields and approved lifecycle actions; assignment/delivery capabilities remain Member 3's. |
| Public tracking             | Only approved privacy-safe tracking fields; no private contact/address or full internal history.              |
| Tracking API (Member 5)     | Read authoritative parcel/status through approved access pattern; no competing lifecycle writes.              |
| Relay/worker (Member 8)     | Only shared outbox/event fields required for publication.                                                     |
| AI assistant (Member 10)    | Approved application APIs/tools only; no direct DB access.                                                    |
| Application/migration roles | Least privilege; separate runtime and schema-change permissions where practical.                              |

Exact role/action rules must match Member 1's identity contract and the team-approved lifecycle matrix.

# 14. Decisions Required Before Schema Freeze

| **Decision**                                    | **Why it matters**                       | **Required coordination**              |
|-------------------------------------------------|------------------------------------------|----------------------------------------|
| Canonical ERD, IDs, names, timestamps           | Prevents incompatible schemas            | Database coordinator + affected owners |
| Customer/merchant references and tenant key     | Enforces isolation                       | Members 1, 2, 4                        |
| Zone ownership and zone IDs                     | Pricing/routing depend on zones          | Member 2 + designated zone/admin owner |
| Currency, money representation, scale, rounding | Prevents incorrect fees                  | Member 2 + team                        |
| Pricing formula/rates/effective-date conflicts  | Reproducible fee calculation             | Member 2 + relevant admin/zone owner   |
| Quote persistence, expiry, reuse, consumption   | Determines need for quote table          | Member 2 + Member 6 + team             |
| Canonical lifecycle and transition matrix       | Prevents divergent state                 | Members 2, 3, 5 + team                 |
| Failed delivery/reattempt model                 | Defines parcel state vs delivery attempt | Members 2, 3, 5                        |
| Address snapshot and retention                  | Fulfilment/privacy/audit                 | Members 2, 4, 6 + team                 |
| Idempotency mechanism and retention             | Avoids duplicate bookings                | Members 2, 4, 8 + team                 |
| Outbox schema/event payload                     | Integration consistency                  | Members 2, 8                           |
| Retention/deletion periods                      | Avoids unapproved retention/loss         | Team/project owner                     |
| Migration tool/numbering/rollback policy        | Safe parallel work                       | Member 9 + database coordinator        |
| Query/performance targets                       | Measurable acceptance                    | Team/SRS owner                         |

# 15. Acceptance Checklist

- Canonical schema and ownership boundaries approved by database coordinator and affected owners.

- Booking persists one parcel with unique server-generated tracking number and accepted-price snapshot.

- Initial status/history are written atomically under the approved lifecycle.

- Money uses exact representation and approved currency/rounding; no binary floating-point persistence.

- Status changes validate transitions and atomically update current state, history, and outbox.

- Constraints protect key invariants, including tracking-number and idempotency uniqueness.

- Same-key/same-request retries do not duplicate; same-key/different-request behavior matches contract.

- Customer ownership and merchant tenant isolation are enforced in backend queries/use cases.

- Public tracking cannot reveal private addresses, phone numbers, notes, or merchant-only fields.

- Receipt uses stored accepted-price snapshot, not today's tariff.

- Migrations build a clean database and upgrade supported development DB; CI verifies migrations.

- Seed/test data is synthetic, deterministic, and contains no real personal data.

- Retention/deletion policy is documented before production-like data is stored.

- Optional quote table/cache is added only when justified by an approved requirement.

- Foreign keys, event/outbox integration, and migration ordering reviewed with relevant owners.
