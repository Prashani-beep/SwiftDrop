# SwiftDrop Parcel Delivery & Courier Management System

# Document 6 — Testing Strategy & Acceptance Specification

## 1. Purpose and Scope

This document defines how Member 2's Parcel Management & Pricing module will be tested and accepted. It provides test levels, requirement-to-test traceability, test scenarios, test-data rules, automation guidance, execution gates, and evidence needed to demonstrate correctness.

This is a test specification, not a claim that tests have already been implemented or passed. Actual test results must be recorded from the repository and execution environment.

### 1.1 In scope

- Parcel booking and server-side request validation.
- Fee estimation and reproducibility of the accepted price snapshot.
- Unique tracking-number generation.
- Booking idempotency and duplicate-request handling.
- Parcel detail, history, and receipt-data reads.
- Ownership/tenant authorization and privacy-safe response fields.
- Legal status-transition validation within the lifecycle agreed with Members 3 and 5.
- Atomic persistence of parcel changes, history, and required outbox records through Member 8's shared interface.
- Database migrations, constraints, indexes, and transaction behavior owned by Member 2.
- API error contracts and integration compatibility relevant to Member 2.

### 1.2 Out of scope or shared responsibility

- Authentication provider and identity implementation: Member 1; Member 2 must still test the module's authorization behavior using the agreed auth middleware/fixtures.
- Merchant onboarding, credential lifecycle, and tenant validation: Member 4; merchant parcel creation tests must use the published interface and coordinate shared scenarios.
- Courier assignment, delivery attempts, and proof-of-delivery: Member 3; tests that cross lifecycle boundaries require agreement with Member 3.
- Tracking API implementation and SSE behavior: Member 5; Member 2 tests authoritative data consistency and the approved read boundary.
- Outbox relay, RabbitMQ topology, event workers, notifications, and webhook delivery: Member 8; Member 2 tests writing the required outbox record through the shared interface.
- Frontend component and end-to-end implementation: Members 6–7; Member 2 supplies stable API contracts and supports integrated journey tests.
- Deployment platform and shared CI configuration: Member 9; Member 2 supplies module-level commands and test requirements.
- AI Assistant tool implementation: Member 10; any fee-estimation tool must consume the approved API and must not bypass its validation.

## 2. Testing Principles

1. **Test the approved contract.** Test expected behavior from the approved SRS, API contract, business rules, and lifecycle agreement. Do not treat unresolved draft values as final.
2. **Protect business invariants.** A successful booking has one unique tracking number, a reproducible accepted-price snapshot, and no duplicate parcel for an identical idempotent retry.
3. **Test failures as well as success.** Invalid input, authorization denial, conflicts, persistence failures, and dependency failures must have explicit expected outcomes.
4. **Keep tests repeatable.** Use deterministic synthetic fixtures, controlled clocks where needed, isolated data, and explicit setup/cleanup.
5. **Use real persistence for persistence claims.** Unit mocks cannot prove database constraints, transaction atomicity, or migration correctness.
6. **Never depend on unspecified business values.** Currency, tariff formula/rates, size/weight limits, quote expiry, lifecycle edge cases, and retention values require team approval before tests assert exact values.
7. **Do not claim completion without evidence.** Test execution reports, commands, environment details, and known gaps must be recorded.

## 3. Test Levels and Methods

| Test level | Purpose | Required Member 2 coverage | Suggested method/tool |
|---|---|---|---|
| Unit | Verify isolated business logic | Pricing calculation; field and cross-field validation; tracking-number generation policy; legal transition rules; idempotency decision logic; DTO mapping | Go `testing`; `testify` if adopted by the repository |
| Repository/database integration | Verify real PostgreSQL behavior | Migrations; constraints; unique tracking number; persisted price snapshot; history queries; rollback; concurrent booking/idempotency | PostgreSQL test instance; Testcontainers where practical or controlled Compose test database |
| API integration | Verify HTTP contract and application wiring | Booking, quote, detail, history, receipt-data routes; request validation; response DTOs; error envelope; authorization; pagination if supported | Go HTTP tests with test server and real test database where needed |
| Contract | Verify compatibility between module boundaries | Request/response DTOs, error schema, shared auth context, Member 8 outbox interface, Member 4 merchant capability, Member 5 tracking read expectations | OpenAPI validation or equivalent contract checks; consumer/provider examples |
| Security/authorization | Verify access boundaries and disclosure | Cross-customer access denial; merchant tenant scoping through agreed integration; public/private field minimization; malformed input and rate-limit behavior where configured | HTTP integration tests; negative authorization matrix |
| End-to-end | Verify customer-visible business journey | Estimate fee → valid booking → confirmation/tracking number → authorized history/receipt data; retry does not duplicate | Playwright or agreed system-level test harness; coordinate with Members 6 and 9 |
| Resilience/fault | Verify safe behavior under failure | Database write failure/rollback; outbox write failure; concurrent duplicate requests; transient dependency failure if applicable | Controlled fault injection or deterministic integration tests |
| Migration/deployment acceptance | Verify reproducible setup | Clean database migration; upgrade supported development database; module test commands in local Compose/CI | Migration tool and Docker Compose; Kubernetes checks only where included in the agreed demo scope |

## 4. Requirements and Test Traceability

The IDs below are module-local test-case IDs. They are not new SRS requirement IDs. Map them to the final approved `FR-PAR-*` requirement wording in Document 3 and update the matrix if the team revises that contract.

| Requirement area | Test cases | Required evidence |
|---|---|---|
| Fee estimation and pricing consistency | TC-PRC-001–006 | Unit test report with approved examples, boundary cases, exact-money assertions, and rule-version/snapshot checks |
| Parcel booking and validation | TC-BKG-001–008 | API/integration results showing valid booking and structured rejection without partial data |
| Tracking number | TC-TRK-001–002 | Database uniqueness test and generated-reference behavior evidence |
| Idempotency | TC-IDM-001–005 | Repeated same-key tests, different-payload conflict test, and concurrent-request test |
| Parcel detail, history, receipt data | TC-READ-001–006 | API responses, ownership-denial results, and accepted-price snapshot verification |
| Lifecycle and history | TC-LCY-001–006 | Approved transition-matrix tests and atomic state/history/outbox assertions |
| Persistence and migrations | TC-DB-001–006 | Clean migration, upgrade, constraint, rollback, and query tests |
| API/error contract | TC-API-001–005 | Contract validation and examples for success and error responses |
| Security/privacy | TC-SEC-001–006 | Authorization matrix results and checks that private data is not returned to unauthorized/public callers |
| Integrated acceptance | AT-M2-001–006 | Reproducible end-to-end run log and linked test report |

## 5. Detailed Test Case Matrix

### 5.1 Pricing and fee estimation

| ID | Scenario / setup | Expected result | Level |
|---|---|---|---|
| TC-PRC-001 | Valid quote request using an approved active rule and valid supported inputs | A fee is calculated using the approved formula and returned using the agreed money representation and currency | Unit + API integration |
| TC-PRC-002 | Weight or dimension at each approved lower/upper boundary | Result follows the approved boundary rule; no guessed limits are embedded in the test | Unit |
| TC-PRC-003 | Zero, negative, non-finite, malformed, or out-of-range measurement | Request is rejected according to the approved validation contract; no quote/booking is incorrectly persisted | Unit + API integration |
| TC-PRC-004 | No enabled/effective pricing rule matches the requested inputs | Defined domain/API error is returned; no invented fallback price is applied | Unit + API integration |
| TC-PRC-005 | Price calculation and persistence | Exact arithmetic is used; binary floating-point is not used as the authoritative money representation | Unit + database integration |
| TC-PRC-006 | A pricing rule changes after a parcel has been booked | Historical accepted price and rule/version snapshot remain unchanged | Database integration |
| TC-PRC-007 | Two effective rules overlap for the same scope/time | Behavior matches the team-approved overlap policy; test must remain blocked until that policy is defined | Unit + integration |

### 5.2 Booking and input validation

| ID | Scenario / setup | Expected result | Level |
|---|---|---|---|
| TC-BKG-001 | Complete valid booking request using approved fields and pricing reference | One parcel is created, with a unique tracking number, accepted-price snapshot, and agreed initial state | API + database integration |
| TC-BKG-002 | Missing each mandatory field in turn | Structured validation error identifies the invalid field(s); no partial parcel is stored | API integration |
| TC-BKG-003 | Invalid formats, overlong text, unsupported enum, or invalid address/zone reference | Request is rejected with the agreed error envelope; database remains unchanged | Unit + API integration |
| TC-BKG-004 | Unsupported weight/dimensions or invalid cross-field combination | Rejected using the approved limits and rules; no parcel is stored | Unit + API integration |
| TC-BKG-005 | Client supplies a price that differs from the server-calculated/approved quote | Client amount is not treated as authoritative; server validates/recalculates according to the approved quote contract | API integration |
| TC-BKG-006 | Database failure after parcel insertion but before all required related writes complete | Transaction rolls back; no partial parcel/history/outbox/idempotency completion state remains | Database integration |
| TC-BKG-007 | Successful booking with address snapshots | The accepted pickup/drop-off details are preserved according to the approved snapshot model and access policy | Database integration |
| TC-BKG-008 | Tracking-number generation collision is forced in a test | Unique constraint prevents duplicate tracking numbers; generation retries follow the implemented bounded policy or returns a safe failure | Database integration |

### 5.3 Tracking number and idempotency

| ID | Scenario / setup | Expected result | Level |
|---|---|---|---|
| TC-TRK-001 | Create multiple parcels using normal generation | Tracking numbers are server-generated and unique | Unit + database integration |
| TC-TRK-002 | Query a tracking number that does not exist | API returns the agreed not-found response without disclosing unrelated records | API integration |
| TC-IDM-001 | Repeat the same effective booking request with the same idempotency key | Only one parcel is created; the second request follows the agreed replay response semantics | API + database integration |
| TC-IDM-002 | Reuse a key with a different effective request payload | Request is rejected with the contract's conflict behavior; a second parcel is not created | API integration |
| TC-IDM-003 | Send concurrent identical requests using the same key | At most one business operation is committed; other requests receive the agreed replay/in-progress result | Concurrency integration |
| TC-IDM-004 | Simulate failure before transaction commit and retry with the same key | No partial business operation is left; retry follows the approved recovery semantics | Database integration |
| TC-IDM-005 | Repeat a request after idempotency-record expiry | Behavior matches the approved retention and expiry policy; test is blocked until that policy is set | Integration |

### 5.4 Reads, history, receipt data, and privacy

| ID | Scenario / setup | Expected result | Level |
|---|---|---|---|
| TC-READ-001 | Authorized owner requests parcel details | Only fields permitted by the API contract are returned | API integration |
| TC-READ-002 | Customer A requests Customer B's parcel detail, history, or receipt data | Access is denied using the agreed not-found/forbidden policy; no private data is disclosed | Security integration |
| TC-READ-003 | Authorized owner requests parcel history | History is chronological and reflects committed status changes only | API + database integration |
| TC-READ-004 | Receipt-data request after pricing rules have changed | Response uses the stored accepted-price snapshot, not today's tariff | API + database integration |
| TC-READ-005 | Public tracking response is inspected | No full address, phone number, private note, proof photo, or merchant-only field is exposed | Security/contract integration with Member 5 |
| TC-READ-006 | List/history query returns multiple records | Pagination and ordering follow the published contract; ordering is deterministic where pagination is supported | API + database integration |

### 5.5 Lifecycle, status history, and outbox

| ID | Scenario / setup | Expected result | Level |
|---|---|---|---|
| TC-LCY-001 | Apply a legal transition from the approved lifecycle matrix | Current state is updated and one corresponding history record is appended | Unit + database integration |
| TC-LCY-002 | Attempt an illegal transition | Request is rejected; no state update, history record, or outbox event is committed | Unit + database integration |
| TC-LCY-003 | Two conflicting transitions start from the same stale state | Concurrency control permits no more than one incompatible transition to succeed | Database integration |
| TC-LCY-004 | Successful transition requiring an event | Current state, history, and outbox record commit atomically | Database integration with Member 8 interface |
| TC-LCY-005 | Outbox write fails during a status change | Business state and history roll back with the failed required outbox write | Database integration |
| TC-LCY-006 | Read history after several transitions | Existing history remains append-only through normal application paths and is not silently rewritten | Database integration |
| TC-LCY-007 | Initial booking | Initial status and initial history behavior match the lifecycle agreement; current SRS baseline proposes `CREATED` and requires team confirmation | Integration; pending lifecycle approval |

### 5.6 Database, migration, and API contract

| ID | Scenario / setup | Expected result | Level |
|---|---|---|---|
| TC-DB-001 | Apply migrations to a clean PostgreSQL database | All approved module migrations succeed in the repository's documented order | Migration integration |
| TC-DB-002 | Upgrade a supported existing development database | Upgrade succeeds according to the team's migration/rollback or forward-fix convention | Migration integration |
| TC-DB-003 | Insert duplicate tracking number | Database uniqueness constraint rejects the duplicate | Database integration |
| TC-DB-004 | Insert invalid values violating approved constraints | Database rejects the invalid write where the invariant is represented by a constraint | Database integration |
| TC-DB-005 | Query history and common parcel lookup paths with representative synthetic data | Queries use the intended indexes where applicable; no numeric latency target is asserted unless approved | Database/performance check |
| TC-DB-006 | Runtime application attempts schema migration or unauthorized direct data access | Database permissions follow the agreed least-privilege split; runtime role does not gain unapproved schema privileges | Security/integration |
| TC-API-001 | Valid quote and booking requests | Status codes and response DTOs match the published API contract | Contract + API integration |
| TC-API-002 | Malformed JSON or invalid content type | Request fails safely with the agreed error format | API integration |
| TC-API-003 | Validation, conflict, forbidden, and not-found conditions | Correct agreed HTTP status and error envelope are returned; internal stack traces/secrets are not exposed | API integration |
| TC-API-004 | OpenAPI or equivalent contract is checked against implementation | Required paths, DTOs, schemas, and examples remain compatible | Contract |
| TC-API-005 | API change affects Member 4, 5, 6, 7, or 8 | Consumer/provider examples and affected integration tests are updated before the change is accepted | Contract/integration |

## 6. Module-Level Acceptance Scenarios

These scenarios adapt the global SRS acceptance baseline to Member 2. Global acceptance IDs AT-001 through AT-016 remain defined by the SRS and should not be renumbered by this module.

| ID | Acceptance scenario | Pass criteria | Dependencies |
|---|---|---|---|
| AT-M2-001 | Estimate a fee and create a valid customer parcel booking | Fee estimate follows the approved pricing rule; booking creates one parcel with unique tracking number and accepted-price snapshot | Approved tariff, currency, units, API contract |
| AT-M2-002 | Submit invalid parcel data | Structured field-level errors are returned and no partial booking is persisted | Approved required fields and limits |
| AT-M2-003 | Repeat booking with the same idempotency key | Exactly one parcel exists for the logical operation and response follows agreed replay semantics | Approved idempotency contract |
| AT-M2-004 | Read parcel history and receipt data as the owner and as an unauthorized user | Owner can read permitted data; unauthorized user receives the agreed denial response; receipt uses the stored accepted price | Member 1 auth contract and privacy policy |
| AT-M2-005 | Attempt illegal transition and perform a legal transition | Illegal transition changes nothing; legal transition atomically persists state, history, and required outbox record | Approved lifecycle; Member 3 and Member 8 interfaces |
| AT-M2-006 | Apply migrations and execute the critical test suite from a clean clone | Setup and migrations succeed, critical tests pass, and commands/results are documented | Member 9/local Compose and repository conventions |

## 7. Test Data and Fixtures

Use deterministic synthetic fixtures only. Never use real customer contact details, real credentials, live API secrets, or production parcel records.

Required fixture categories:

- At least one valid pricing rule/version after the tariff and rule-selection policy are approved.
- Valid measurement examples plus values immediately below, at, and above approved limits.
- Valid and invalid pickup/drop-off zones based on approved zone fixtures.
- Customers with separate ownership scopes and, if merchant creation is in scope, at least two distinct merchant tenants.
- Parcels with multiple chronological status-history records using the approved lifecycle matrix.
- A parcel whose accepted price remains stable after a later pricing-rule version is introduced.
- Same-key/same-request, same-key/different-request, and concurrent idempotency requests.
- Controlled database/outbox failures for rollback tests.
- Tracking-number collision fixture or injectable generator for collision-path tests.
- Unknown parcel/tracking IDs and unauthorized access attempts.

Fixtures must not depend on test execution order. Tests that require unresolved tariffs, currencies, lifecycle transitions, quote expiry, or retention semantics must be marked blocked/pending rather than filled with guessed values.

## 8. Mocking and Test Environment Strategy

### 8.1 Unit tests

- Keep pricing, validation, transition, and idempotency decision logic testable without HTTP or database dependencies.
- Use small fakes for clocks, tracking-number generation, and repositories where dependency injection supports them.
- Do not use a mock-only test to claim PostgreSQL constraints or transaction atomicity have been verified.

### 8.2 Integration tests

- Use PostgreSQL test instances for migration, uniqueness, rollback, concurrency, and query behavior.
- Use the agreed shared outbox interface. Do not create a second test-only outbox schema that diverges from Member 8's contract.
- Use a test HTTP server/client for API contract and authorization behavior.
- Use RabbitMQ only for tests that genuinely cross the broker boundary; Member 2's core persistence tests should verify its outbox write boundary without duplicating Member 8's relay tests.

### 8.3 End-to-end tests

- Use the agreed Docker Compose environment and synthetic seed data.
- Coordinate customer booking and receipt journeys with Member 6, and merchant order journeys with Member 4 and Member 6.
- Use deterministic test identities and credentials supplied through local test configuration, never committed secrets.
- Record any mocked service or skipped integration explicitly in the result report.

### 8.4 Environment isolation

- Tests must target a dedicated test database/broker, not a developer's or production database.
- Reset/cleanup data safely between runs.
- Keep environment configuration in documented example files with placeholder values.
- Make test failures diagnosable through request/correlation IDs and non-sensitive logs.

## 9. Coverage and Quality Gates

The SRS v1.0 proposes a baseline target of **at least 70% line coverage for core business packages**, with explicit tests for every security/authorization requirement and the end-to-end event pipeline. The team should confirm this target as part of approving the SRS baseline.

For Member 2:

- Measure coverage for pricing, parcel booking, validation, idempotency, lifecycle rules, and relevant application use cases.
- Prioritize meaningful assertions and negative cases; coverage percentage alone is not proof of correctness.
- Every critical business invariant in this document must have at least one automated test at the appropriate level.
- Database transaction, uniqueness, and migration claims require database-backed evidence.
- Any failing, skipped, flaky, or blocked test must be reported; do not silently exclude it from the result summary.
- Security-critical tests for owner/tenant isolation and public data minimization are required regardless of overall coverage percentage.

## 10. Local Execution and CI Requirements

Exact commands depend on the repository's selected Go module layout, test libraries, migration tool, and Compose service names. Do not publish commands that have not been verified against the repository.

The module README and CI pipeline must document and run the equivalent of:

1. Format and static checks required by the team.
2. Go unit tests for the Member 2 packages.
3. Database-backed integration tests using an isolated PostgreSQL instance.
4. Migration test against a clean database.
5. API/contract tests.
6. Coverage report for core business packages.
7. Secret scan or the team's documented secrets check.
8. Critical end-to-end smoke flow in the agreed local Compose environment before release/demo.

The final repository must provide exact copyable commands, required environment variables, prerequisites, expected test duration where useful, and cleanup/reset instructions. If a test requires an optional dependency or external credential, document a safe mock/development alternative where the project permits it.

## 11. Defect Severity and Failure Handling

Use the SRS defect priority definitions:

| Priority | Meaning for Member 2 | Example |
|---|---|---|
| P0 | Critical security/data loss or core system cannot demonstrate the flow | Duplicate/partial booking causes data corruption; cross-customer private data is exposed |
| P1 | Core feature is incorrect or insecure with no viable workaround | Incorrect accepted fee snapshot; idempotency allows duplicate parcel; illegal status transition commits |
| P2 | Secondary issue with a workaround | Non-critical history filter edge case |
| P3 | Minor usability/documentation issue | Non-blocking wording or formatting issue |

A defect report should include: summary, priority, affected requirement/test ID, reproduction steps, expected versus actual result, environment/commit, sanitized logs or evidence, and any workaround. Never include secrets or unnecessary personal data.

## 12. Evidence and Test Execution Report

A test run is not considered documented until the following are recorded:

- Commit/branch or source revision tested.
- Date/time and environment details, including Go version and PostgreSQL version actually used.
- Exact commands executed and their exit status.
- Test counts: passed, failed, skipped, and blocked.
- Coverage report for core business packages.
- Migration and database integration results.
- API/contract and authorization test results.
- Relevant end-to-end evidence, such as sanitized logs, screenshots, or CI run links.
- Known defects, flaky tests, exclusions, and unresolved blockers.
- Reviewer/approver and date, when applicable.

### 12.1 Execution result template

| Field | Value to complete from an actual run |
|---|---|
| Revision/commit | Pending execution |
| Environment | Pending execution |
| Unit tests | Pending execution |
| Integration tests | Pending execution |
| API/contract tests | Pending execution |
| Security/authorization tests | Pending execution |
| End-to-end tests | Pending execution |
| Core-package line coverage | Pending execution |
| Migration clean/upgrade tests | Pending execution |
| Known failures/blocked cases | Pending execution |
| Report location | Pending execution |
| Reviewer | Pending review |

Do not replace “Pending execution” with a pass claim until evidence exists.

## 13. Dependencies and Coordination

| Dependency | Testing coordination required |
|---|---|
| Member 1 — Identity & Access | Agree test identity/auth fixtures, roles, ownership checks, and forbidden/not-found behavior |
| Member 3 — Dispatch & Couriers | Agree lifecycle matrix, legal transitions, failed-delivery/reattempt semantics, and status-history ownership |
| Member 4 — Merchant Integration | Agree merchant parcel-creation path, tenant scope, idempotency behavior, and error contract |
| Member 5 — Tracking API | Agree authoritative status reads and public/private field disclosure; verify current committed state is visible |
| Member 6 — Customer & Merchant Frontend | Agree booking, fee estimate, receipt-data API examples and critical user journey |
| Member 7 — Admin UI & Gateway | Agree route/policy behavior and any admin-controlled pricing/zone interfaces |
| Member 8 — Outbox & Events | Agree shared outbox write interface, required event metadata, transaction boundary, and test seam |
| Member 9 — Platform & Observability | Agree Compose/CI test services, environment configuration, migration checks, and evidence collection |
| Database coordinator | Review canonical table/ID/timestamp conventions, constraints, migration ordering, and shared schema assumptions |

## 14. Decisions Required Before Final Test Freeze

The following cannot be asserted as final until the project team approves them:

1. Canonical parcel schema, shared table ownership, IDs, timestamp conventions, and migration tool.
2. Pricing formula, tariff values, currency, precision, rounding, rule-selection and overlapping-rule behavior.
3. Valid weight/dimension limits, supported service levels, and zone eligibility.
4. Whether quotes are persisted, expire, can be reused, or are single-use.
5. Final parcel lifecycle and actor/permission matrix, including failed-delivery and reattempt semantics.
6. Exact initial-history behavior and status-change event requirements.
7. Idempotency scope, retention/expiry, concurrent-request behavior, and exact replay response.
8. Shared outbox schema/interface and event payload contract with Member 8.
9. Exact authorization response policy (for example, 403 versus privacy-preserving 404) where not already standardized.
10. Numeric performance targets, if the SRS/team approves measurable targets.
11. Final CI commands, repository package paths, test framework versions, and coverage reporting mechanism.

## 15. Acceptance Checklist

- [ ] Approved requirements and business rules are traceable to test cases.
- [ ] Pricing tests cover approved examples, boundary conditions, rule selection, and exact-money behavior.
- [ ] Valid and invalid booking tests verify no partial writes on failure.
- [ ] Tracking-number uniqueness is verified against PostgreSQL.
- [ ] Same-key/same-request, same-key/different-request, and concurrent idempotency behavior is verified.
- [ ] Owner/tenant authorization and public/private data boundaries are tested.
- [ ] Receipt data uses the stored accepted-price snapshot.
- [ ] Legal and illegal status transitions are tested against the approved lifecycle.
- [ ] State, history, and required outbox records are proven atomic in database-backed tests.
- [ ] Clean-database migration and supported upgrade tests pass.
- [ ] API/error contracts and affected module integrations are checked.
- [ ] Critical tests run locally and in CI using documented commands.
- [ ] Coverage and execution evidence are attached; failures and blockers are reported honestly.
- [ ] No real personal data or secrets are used in fixtures or committed files.
- [ ] Open decisions are resolved or explicitly recorded as blockers before schema/test freeze.


