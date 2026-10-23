# SwiftDrop Parcel Delivery & Courier Management System
## Document 8 — Implementation, Deployment & Operations Guide
**Member 2: Parcel Management & Pricing**

## 1. Purpose and scope

This guide explains how a developer should configure, run, build, test, debug, and maintain Member 2's Parcel Management & Pricing module within the SwiftDrop Core API.

In scope:
- Parcel booking and parcel detail/list reads.
- Pricing/fee estimation and accepted-price snapshots.
- Parcel identifiers and public tracking-number generation.
- Parcel status and history records within the agreed ownership boundary.
- Receipt-data retrieval.
- Database migrations and seeds owned by this module.
- Transactional outbox integration through the shared interface owned/coordinated with Member 8.
- Local development, container build, health checks, operational telemetry, and troubleshooting.

Out of scope:
- Merchant onboarding, credentials, and tenant validation (Member 4).
- Identity provider and authentication middleware ownership (Member 1), although this module must enforce authorization at its own use-case boundary.
- Courier assignment, delivery execution, and proof-of-delivery storage (Member 3).
- Tracking API/SSE implementation and its read model (Member 5).
- Frontend rendering (Member 6).
- RabbitMQ relay/worker ownership and shared event-envelope design (Member 8).
- AI assistant implementation (Member 10).
- Organization-wide production platform and cluster administration.

## 2. Baseline and unresolved decisions

The documented system baseline is a Go modular-monolith Core API, PostgreSQL as authoritative business storage, REST/JSON interfaces, RabbitMQ asynchronous integration through a transactional outbox, and Docker Compose for local development. Kubernetes/Helm is a bonus deployment direction, not a confirmed requirement for the module.

The following values are **not confirmed by the baseline documents** and must be decided by the team rather than guessed:
- Exact Go toolchain version and dependency versions.
- Exact repository root layout and command names.
- PostgreSQL version, database name, ports, and migration tool.
- Authentication/session mechanism and identity claims supplied by Member 1.
- Pricing formula, currency, pricing-rule ownership, and zone source.
- Exact status-transition authority and failure/retry policy.
- Outbox schema, event envelope, routing keys, and relay behavior owned/coordinated with Member 8.
- Required deployment target, Kubernetes namespace, resource budgets, replicas, ingress, and secrets provider.
- Approved readiness/liveness semantics, request limits, timeouts, rate limits, retention, and alert thresholds.
- Whether quote records are persisted and whether quote expiry is enforced.

Where examples below show a variable name or a command, treat it as a **recommended template** until the team confirms it against the repository and shared standards.

## 3. Runtime and dependency inventory

| Area | Baseline / recommendation | Ownership or decision |
|---|---|---|
| Language | Go | Exact version must be pinned in repository/toolchain configuration by the team. |
| HTTP API | REST/JSON, UTF-8 | Shared API conventions in Documents 3–4 and Engineering Standards. |
| Database | PostgreSQL | Shared connection and migration conventions must be agreed. |
| Database access | Repository/adapter layer; use the project's selected SQL or ORM library | Library and version must match the actual repository; do not add a second persistence stack without approval. |
| Validation | Shared validation/error conventions | Use the team's chosen library or standard-library validation approach. |
| Event publication | Transactional outbox in the same business transaction | Shared interface and event contract coordinated with Member 8. Do not create a competing outbox. |
| Local services | Docker Compose | Exact compose file and service names depend on repository setup. |
| Unit/integration tests | Go test tooling plus approved test database/dependency strategy | CI commands and coverage gate must match Document 6 and the actual pipeline. |
| Observability | Structured logs; metrics/tracing if the shared stack supports them | Field names, exporters, and backend are team decisions. |

### 3.1 Version policy

1. Pin the Go toolchain version in the repository's agreed toolchain mechanism and CI image.
2. Pin third-party dependencies in `go.mod`/`go.sum`; review dependency updates rather than relying on floating versions.
3. Pin container base images to an approved stable version or digest under team policy.
4. Record PostgreSQL and RabbitMQ versions in the shared development/deployment documentation once selected.
5. CI and local development should use compatible toolchain and service versions.
6. Never copy production secrets into sample files, screenshots, issue reports, or committed configuration.

## 4. Expected code organization

The following is a **proposed package layout**, aligned to the documented ownership areas. Preserve the existing repository conventions if the team has already established a different structure.

```text
core-api/
├── cmd/
│   └── api/                  # process entry point; exact name to confirm
├── internal/
│   ├── parcels/
│   │   ├── handler/          # HTTP transport and DTO mapping
│   │   ├── application/      # booking, reads, history, receipt use cases
│   │   ├── domain/           # parcel rules, identifiers, state invariants
│   │   └── repository/       # persistence interfaces/adapters
│   ├── pricing/
│   │   ├── handler/          # quote endpoint and DTO mapping
│   │   ├── application/      # quote calculation use case
│   │   ├── domain/           # pricing rule evaluation and money values
│   │   └── repository/       # pricing-rule/quote persistence adapters
│   ├── platform/             # shared config, database, logging, HTTP helpers
│   └── ...                   # other team-owned modules; do not duplicate them
├── migrations/               # location and naming must follow shared convention
├── tests/                    # optional integration/contract tests
├── Dockerfile                # team-approved production image definition
├── go.mod
├── go.sum
├── .env.example
└── README.md
```

**Dependency direction:** HTTP handlers depend on application use cases; use cases depend on domain abstractions and repository/event interfaces; infrastructure adapters implement those interfaces. Domain logic must not depend on HTTP handlers, database drivers, RabbitMQ clients, or frontend concerns. Do not let Member 2 create duplicate identity, merchant, dispatch, tracking, or event-worker implementations.

## 5. Process entry point and configuration loading

The process entry point should:
1. Load and validate configuration.
2. Initialize structured logging and request/correlation ID handling.
3. Open the PostgreSQL connection pool and verify connectivity.
4. Initialize repositories and application use cases.
5. Register HTTP routes and middleware, including authentication integration, request limits, and safe error mapping.
6. Register health endpoints.
7. Start the HTTP server with explicit read, write, header, and idle timeouts selected by the team.
8. Handle termination signals and perform graceful shutdown within a configured deadline.
9. Close database and other owned resources after requests stop being accepted.

Startup should fail fast if required configuration is missing or invalid. It must not start in a mode that silently invents a price, bypasses authorization, or reports a healthy state while the required database is unavailable.

## 6. Environment variables and safe configuration example

The table below defines a suggested configuration contract. Variable names are recommendations until approved. Values shown are local examples or placeholders, not production credentials.

| Variable | Purpose | Safe example / rule |
|---|---|---|
| `APP_ENV` | Runtime environment label | `development`; approved values should be validated. |
| `HTTP_ADDR` | HTTP bind address | `:8080` for a container; bind/port policy must be confirmed. |
| `DATABASE_URL` | PostgreSQL connection string | Use a local-only credential; never commit real credentials. |
| `DATABASE_MAX_OPEN_CONNS` | Upper bound for open DB connections | Example `10` for local development only; tune to workload and DB limits. |
| `DATABASE_MAX_IDLE_CONNS` | Idle connection pool target | Example `5` for local development only; must not exceed the selected open limit. |
| `DATABASE_CONN_MAX_LIFETIME` | Maximum connection lifetime | Example `30m`; confirm supported duration format in config parser. |
| `LOG_LEVEL` | Minimum log level | `debug` locally; normally less verbose in production. |
| `LOG_FORMAT` | Log encoding | `json` recommended for deployed environments; local text is optional. |
| `REQUEST_TIMEOUT` | Overall handler/request deadline | Example `15s` is a placeholder; coordinate with shared API and dependency timeouts. |
| `SHUTDOWN_TIMEOUT` | Graceful shutdown deadline | Example `20s` is a placeholder; align with deployment termination grace period. |
| `CORS_ALLOWED_ORIGINS` | Browser origin allow-list, if CORS is needed | Explicit local frontend origin(s); never use wildcard with credentialed requests. |
| `TRUST_PROXY_HEADERS` | Whether proxy headers may be trusted | `false` unless deployment topology and trusted proxy list are configured. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Optional tracing exporter endpoint | Empty/disabled locally unless an approved collector is running. |
| `OTEL_SERVICE_NAME` | Service identity for telemetry | `swiftdrop-core-api` is a proposed name. |
| `RABBITMQ_URL` | Optional direct dependency only if this process is approved to use it | Leave unset if the module writes only to the shared outbox; do not bypass Member 8's relay. |
| `PRICING_DEFAULT_CURRENCY` | Currency used by pricing | **Must be set only after team approval**; no currency is assumed by this guide. |

Do not place OAuth secrets, JWT signing keys, database passwords, RabbitMQ credentials, or production tokens in `.env.example`. Use local ignored `.env` files or an approved secret manager. Configuration must be validated at startup, and errors must name the missing variable without echoing secret values.

### 6.1 `.env.example` template

```dotenv
APP_ENV=development
HTTP_ADDR=:8080
DATABASE_URL=postgres://swiftdrop_local:replace_me@localhost:5432/swiftdrop?sslmode=disable
DATABASE_MAX_OPEN_CONNS=10
DATABASE_MAX_IDLE_CONNS=5
DATABASE_CONN_MAX_LIFETIME=30m
LOG_LEVEL=debug
LOG_FORMAT=json
REQUEST_TIMEOUT=15s
SHUTDOWN_TIMEOUT=20s
CORS_ALLOWED_ORIGINS=http://localhost:3000
TRUST_PROXY_HEADERS=false
OTEL_EXPORTER_OTLP_ENDPOINT=
OTEL_SERVICE_NAME=swiftdrop-core-api
# RABBITMQ_URL=  # only if explicitly approved for this process
# PRICING_DEFAULT_CURRENCY=  # set only after the team approves a currency
```

This is an example, not a promise that all variables are implemented. Align it with the actual config loader before use. The `replace_me` password is deliberately non-working guidance and must be changed for a local database.

## 7. Local development setup

### 7.1 Prerequisites

- Git and access to the SwiftDrop repository.
- The team-pinned Go toolchain.
- Docker Engine/Desktop and Docker Compose plugin, if the repository uses Compose for PostgreSQL and supporting services.
- Access to the agreed migration tool/command.
- Optional API client such as curl or an approved API testing tool.
- Approved local test identity/token setup from Member 1, or a documented development-only mock mode.

### 7.2 Setup workflow

1. Clone the team repository and switch to the branch/commit intended for development.
2. Read the root README, Engineering Standards, and the module README before changing shared configuration.
3. Copy `.env.example` to `.env` if this is the repository's convention; keep `.env` untracked.
4. Start only the required local dependencies using the actual Compose file and service names.
5. Wait for PostgreSQL health/readiness; do not assume container startup means the database is ready.
6. Create/apply migrations using the team's migration command. Confirm migration ordering with Members 3, 5, and 8 before schema changes are merged.
7. Load approved seed data for zones/pricing rules and test identities. Do not invent production tariffs.
8. Run formatting, static checks, unit tests, and integration tests.
9. Start the Core API using the repository's confirmed command.
10. Call the health endpoint and run a basic pricing/booking smoke test with approved local fixtures.
11. Record the commit, commands, environment, and results when reporting a test or demo.

### 7.3 Command templates

The exact commands depend on repository scripts and the selected migration tool. Confirm them in the root README and CI before using as authoritative instructions.

```bash
# From the repository root: inspect project scripts and module location first
git status
go version
go test ./...

# From core-api/ if it is the Go module root
go mod download
go test ./...
go vet ./...
gofmt -w ./cmd ./internal
go build ./...
```

If `go.mod` is not at `core-api/`, run Go commands from the directory containing the actual `go.mod`. Do not run `gofmt` against directories that do not exist; adapt the paths to the real repository. Apply migrations and start Compose using the exact files and commands present in the repository, for example:

```bash
# Templates only — replace with the repository's real file/service/target names
docker compose -f <approved-compose-file> up -d <required-services>
docker compose -f <approved-compose-file> ps
docker compose -f <approved-compose-file> logs -f <service-name>
```

Angle-bracketed tokens are placeholders, not literal shell arguments. Do not claim the module works until the commands have actually been run and their results checked.

## 8. Build and quality checks

A developer change should pass the following gates before review/merge:

1. Formatting: `gofmt` and repository formatting conventions.
2. Static analysis: `go vet ./...` plus any linter configured by the repository.
3. Unit tests: pricing formula and money arithmetic, validation, state transition rules, idempotency, and response mapping.
4. Integration tests: database constraints, transaction rollback, migrations, history ordering, and outbox atomicity.
5. Contract tests: API DTOs/status codes/errors and the agreed outbox/event contract.
6. Security tests: ownership/tenant isolation, public tracking field allow-list, input limits, and safe error/log handling.
7. Build: compile the application and build the container image.
8. Migration validation: fresh database migration and upgrade from the team's supported baseline.
9. Coverage: meet the team-approved threshold for critical business logic; Document 6 proposes a core-business line-coverage target of at least 70%, subject to approval.
10. Evidence: attach CI output or a reproducible local test record. A passing build does not substitute for a passing test suite.

## 9. Docker image requirements

The production Dockerfile should follow the Engineering Standards and the team's actual repository layout. Recommended requirements:

- Multi-stage build: compile with the pinned Go toolchain, then copy only the compiled binary and required runtime files into a minimal approved runtime image.
- Use a non-root runtime user where feasible.
- Pin base image versions/digests under team policy.
- Exclude `.env`, credentials, Git history, test dumps, and unnecessary source/build caches from the final image.
- Add a `.dockerignore` to avoid sending unnecessary files to the build context.
- Make the process handle termination signals and exit cleanly.
- Expose/document the agreed HTTP port; `8080` is only the example in this guide.
- Do not bake database URLs or secrets into image layers or build arguments.
- Provide health endpoints that match actual application readiness semantics.
- Generate an SBOM and scan the image if required by the shared CI/security policy.

### 9.1 Build/run template

```bash
# Run from the directory containing the approved Dockerfile
docker build -t swiftdrop-core-api:dev .
docker run --rm --env-file .env -p 8080:8080 swiftdrop-core-api:dev
```

These commands are templates. They work only if the Dockerfile, env variables, port, and build context match the implementation. For production, use a registry image identified by an immutable tag or digest and the approved deployment pipeline.

## 10. Health checks and readiness

The team should implement distinct health semantics and document the actual routes:

- **Liveness:** indicates that the process is alive and not irrecoverably stuck. It should not fail solely because a temporary external dependency is down if restarting would not fix the dependency.
- **Readiness:** indicates whether this instance can safely accept requests. It should reflect required initialization and the dependencies needed for the endpoints served.
- **Startup probe (Kubernetes, if used):** allows a slow-starting process time to initialize before liveness checks begin.

Health endpoints must not return credentials, connection strings, stack traces, SQL details, or sensitive dependency metadata. Do not mark the API ready before configuration validation and required database initialization complete. The exact route names, status codes, probe delays, and thresholds remain team deployment decisions.

## 11. Database migration and seed operations

- Use the shared migration tool and naming convention; one migration must have a clear, reviewable purpose.
- Do not edit an already-applied migration unless the team explicitly allows it for an unreleased development baseline.
- Coordinate schema changes with Members 3, 5, and 8 where parcel identifiers, status/history, or event/outbox data are involved.
- Preserve a single authoritative lifecycle and avoid a second competing status store.
- Enforce uniqueness for internal/public identifiers and required idempotency constraints in the database as well as application logic.
- Store the accepted price and pricing-rule/version reference as a booking snapshot; do not recalculate historic receipt prices from current rules.
- Insert the business record, status-history row, and required outbox record in the same transaction where specified by the approved event contract.
- Test rollback behavior: if any required write fails, the transaction must not leave a partial booking.
- Seed only clearly labelled development/test data. Never seed real customer data or unapproved pricing values.
- Before destructive migrations, take and verify a backup where relevant, identify rollback/forward-fix steps, and obtain approval.
- A down migration is not automatically safe for data-bearing production tables; document data-loss implications explicitly.
- Run migration checks in CI against both a clean database and the supported upgrade path.

## 12. Logging, metrics, tracing, and alerts

### 12.1 Logging

Use structured logs and a request/correlation ID. Recommended fields, subject to shared standards:
- timestamp, severity, service, environment, request ID, route template, HTTP status, duration;
- use-case name and outcome;
- dependency name and duration where safe;
- database transaction outcome and migration version;
- event/outbox record ID where permitted, without copying sensitive payloads.

Never log authentication tokens, passwords, full connection strings, payment secrets, full recipient phone numbers, full addresses, private parcel notes, or raw request bodies by default. Prefer parcel IDs or approved pseudonymous references. Sanitize untrusted values to prevent log injection. Client responses must not expose stack traces or internal SQL errors.

### 12.2 Metrics

If the shared metrics stack is adopted, useful low-cardinality metrics include:
- request count, latency, and error rate by route template and status class;
- pricing calculation success/failure and missing-rule outcomes;
- booking success/failure and idempotency replay/conflict counts;
- database connection pool utilization and query/transaction latency;
- outbox pending count and oldest pending age, if exposed by the shared outbox owner;
- migration/startup failures.

Avoid labels containing parcel IDs, tracking numbers, customer IDs, phone numbers, addresses, or other high-cardinality/sensitive values.

### 12.3 Tracing

If distributed tracing is enabled, propagate the agreed trace/request context across the Core API and approved downstream integrations. Do not create a new tracing format independently. Do not put sensitive parcel/customer fields in span attributes.

### 12.4 Alerting

Alert thresholds and on-call ownership must be agreed by the team. At minimum, define how operators notice sustained API error increases, readiness failures, database saturation, migration failures, and a growing/aged outbox backlog. Do not invent a numeric service-level objective or alert threshold in this module document.

## 13. Startup, shutdown, and dependency behavior

| Situation | Expected behavior |
|---|---|
| Invalid/missing required configuration | Fail startup with a safe, actionable error; do not run with insecure defaults. |
| PostgreSQL unavailable at startup | Do not report ready; fail or retry startup according to the shared policy. |
| PostgreSQL becomes unavailable | Return a safe dependency error; apply bounded timeouts; do not return a fabricated quote or booking success. |
| Pricing rule absent/ambiguous | Fail safely with a documented domain/API error; never guess a price. |
| Outbox insert fails in booking transaction | Roll back the booking if the approved contract requires atomic event creation. |
| RabbitMQ/relay unavailable while API writes to outbox | Business writes may continue only if the agreed outbox design supports it and database writes succeed; monitor backlog and do not bypass the outbox. |
| SIGTERM / container stop | Stop accepting new work, finish in-flight requests within the shutdown deadline, close resources, and exit. |
| Migration failure | Stop rollout; preserve logs/evidence; follow the migration recovery plan rather than manually editing production data. |

Timeouts and retries must be bounded and coordinated. Do not blindly retry a non-idempotent write. Use the approved idempotency contract for booking retries and the agreed event delivery semantics for outbox processing.

## 14. Kubernetes/Helm deployment guidance (conditional)

Kubernetes and Helm are identified as a bonus deployment direction. This section is a deployment checklist, not a claim that manifests or a cluster are already present.

If the team chooses Kubernetes:
- Deploy the Core API as a `Deployment` behind a `Service`; expose it externally only through the approved ingress/gateway.
- Use startup, readiness, and liveness probes based on the semantics in Section 10.
- Keep secrets in the approved secret-management mechanism; use ConfigMaps only for non-sensitive configuration.
- Run as non-root, use a read-only filesystem where practical, drop unnecessary Linux capabilities, and set an appropriate security context.
- Define CPU/memory requests and limits after load testing; do not copy arbitrary production sizing from this draft.
- Set termination grace period to accommodate the configured graceful shutdown timeout.
- Apply rollout strategy and rollback procedure approved by the team.
- Use a dedicated service account with least privilege; do not grant Kubernetes API access if the process does not need it.
- Keep database migrations in a controlled release step/job with single-run coordination; do not let every replica race to apply migrations on startup unless the migration system explicitly supports this.
- Store database/RabbitMQ credentials in secrets and rotate them according to the shared policy.
- Add NetworkPolicies and ingress restrictions if supported by the cluster baseline.
- Configure horizontal scaling only after checking connection-pool multiplication, database capacity, and statelessness.
- Keep application replicas stateless; parcel and pricing truth remains in PostgreSQL.
- Document image registry, immutable image reference, namespace, resource values, replicas, ingress hostname, secret names, and rollback command in the environment-specific deployment record.

### 14.1 Scaling considerations

HTTP replicas may scale horizontally only if request handling is stateless and all replicas use compatible configuration/schema. Size database pools with total replicas in mind: per-replica connection limits multiplied by replica count must remain within the database's safe capacity. Outbox relay/worker scaling belongs to Member 8's design and must not be independently scaled from Member 2's module without its ownership and delivery guarantees.

## 15. Troubleshooting guide

| Symptom | Checks | Safe action |
|---|---|---|
| API fails at startup | Required env vars, config parsing, port binding, DB reachability, migration status | Fix configuration or dependency availability; do not disable validation to force startup. |
| Readiness is failing | DB initialization, migrations, required configuration, probe path/port | Inspect safe logs and probe configuration; avoid weakening probes without evidence. |
| Quote returns no applicable rule | Zone IDs, service level, rule enabled/effective dates, seed data, rule version | Load approved rules or report configuration gap; never hard-code a fallback price. |
| Booking creates duplicate records | Idempotency key propagation, request fingerprint, unique constraints, concurrent transaction behavior | Reproduce with a repeat/concurrency test; fix the transaction and DB constraints. |
| Booking succeeds but event is missing | Transactional outbox insert, rollback path, relay backlog, event contract | Inspect outbox metadata and Member 8's relay; do not publish directly as a shortcut. |
| History order appears wrong | Timestamp source, deterministic tie-breaker, transaction writes, query ordering | Use server-side persisted timestamps and a stable secondary order per the approved schema. |
| Access to another parcel is possible | Caller identity/tenant scope in use case and repository query | Treat as a security defect; restrict access, add regression tests, and follow incident procedure if data was exposed. |
| Container exits immediately | Entrypoint, binary path/permissions, config, signal/exit logs | Run locally with safe config and inspect exit code; don't add secrets to the image. |
| Migration fails | Version table, locks, constraints, prior partial effects | Stop rollout and follow approved recovery/backup procedure; do not blindly rerun destructive SQL. |
| Slow requests / DB saturation | Query plan, indexes, pool usage, transaction duration, request volume | Gather metrics/traces; tune with evidence and coordinate pool sizing before scaling replicas. |

## 16. Operational runbook

### 16.1 Before a demo or release
- Confirm the commit/tag and environment configuration.
- Confirm no secrets are committed and production credentials are not reused locally.
- Run unit, integration, contract, and security checks required by Document 6.
- Validate fresh migration and upgrade path on a disposable database.
- Confirm approved pricing rules and zone seed data are loaded.
- Verify health/readiness and run one valid quote, one valid booking, and authorized detail/history/receipt checks.
- Verify repeated idempotent booking does not create a duplicate.
- Verify booking rollback leaves no partial parcel/history/outbox writes.
- Verify public-facing outputs contain only approved fields.
- Capture build/test evidence and known limitations.

### 16.2 Release
1. Review changes and migration plan with affected module owners.
2. Build and scan the immutable image in CI.
3. Back up relevant data where required by the environment's recovery policy.
4. Apply schema changes through the approved controlled migration step.
5. Deploy the selected image using the approved pipeline.
6. Check readiness, error rate, database health, and outbox backlog.
7. Execute smoke tests without exposing real customer data.
8. Roll back the application version or apply the documented forward-fix if migration compatibility permits. Do not assume a database rollback is safe.

### 16.3 Incident
1. Record time, environment, affected endpoint/use case, request/correlation IDs, and observed impact.
2. Avoid copying secrets or personal data into the incident report.
3. Preserve relevant logs/metrics and migration/deployment identifiers.
4. If confidentiality or authorization may be breached, follow the team's security incident process and notify the designated owner.
5. If data integrity may be affected, stop unsafe retries or writes only under the agreed incident authority; preserve evidence before repair.
6. Recover using a verified backup, idempotent replay, or documented forward-fix as appropriate; do not manually invent parcel status or pricing history.
7. Add regression tests and record the root cause and follow-up actions.

## 17. External dependencies and ownership matrix

| Dependency / capability | Purpose | Owner / coordination |
|---|---|---|
| Identity and access middleware | Verified principal, role and authorization context | Member 1 owns identity/auth integration; Member 2 still enforces parcel resource scope. |
| Merchant integration | Merchant and tenant validation before parcel creation | Member 4; agree internal invocation and idempotency mapping. |
| Dispatch and courier operations | Legal operational transitions and assignments | Member 3; agree transition matrix and actor permissions. |
| Tracking API/read integration | Privacy-safe tracking and status/history read needs | Member 5; avoid duplicate lifecycle/read truth. |
| Frontend | Booking, quote, history and receipt presentation | Member 6; frontend is not authoritative for price or authorization. |
| Event/outbox/relay/worker | Durable asynchronous events | Member 8; use shared schema/interface and delivery policy. |
| AI assistant | Fee estimates and permitted parcel information via APIs | Member 10; no direct database access or invented values. |
| PostgreSQL | Authoritative parcel/pricing persistence | Shared infrastructure; table and migration ownership must be explicit. |
| Docker Compose / CI / deployment platform | Reproducible build and deployment | Shared team/platform owner to be named. |

## 18. Known limitations

- Exact dependency versions, command names, migration tooling, service ports, probe routes, and deployment resources are intentionally not claimed because they are not specified in the baseline material.
- This guide does not define a final pricing tariff or currency.
- This guide does not replace the shared authentication, API, event, database, or production operations standards.
- Kubernetes deployment is conditional on the team selecting it; it is not required by this document alone.
- Example commands and `.env.example` must be reconciled with the actual repository before being treated as executable truth.
- No build, test, container scan, or deployment result is claimed by this document.

## 19. Decisions required before implementation/release

| Decision | Required participants | Why it matters |
|---|---|---|
| Pin Go, PostgreSQL, and dependency versions | Team / CI owner | Reproducible local and CI builds. |
| Confirm repository structure and canonical commands | Core API team | Prevents misleading setup steps. |
| Select migration tool and migration ownership rules | Members 2, 3, 5, 8 | Prevents schema drift and conflicting migrations. |
| Approve pricing formula, currency, rule/zone ownership, and seed fixtures | Member 2 + team | Prevents unapproved or fabricated quotes. |
| Finalize authentication claims, role matrix, and resource-scope contract | Members 1, 2, 4 | Prevents unauthorized access and tenant leaks. |
| Approve status transition and history ownership | Members 2, 3, 5 | Maintains one consistent parcel lifecycle. |
| Finalize outbox interface, envelope, routing, and retry/DLQ behavior | Members 2, 8 | Ensures atomic and reliable asynchronous integration. |
| Approve health routes, timeouts, resource limits, logs/metrics, and alerts | Core API / deployment owner | Makes operations observable and predictable. |
| Decide if Kubernetes/Helm is used for the assessment/demo | Team | Determines whether manifests and deployment evidence are required. |
| Approve retention, backup, restore, and incident process | Team/security owner | Defines recovery obligations. |

## 20. Acceptance checklist

- [ ] Repository README accurately identifies the Go module root and pinned toolchain.
- [ ] Environment variables are documented and validated; no secret is committed.
- [ ] Local setup instructions have been executed by a second developer on a clean checkout.
- [ ] Database and migration setup is repeatable.
- [ ] Approved pricing and zone seed data are available for demo/test use.
- [ ] Unit, integration, contract, and security tests run locally and in CI.
- [ ] Docker image builds reproducibly and runs as a non-root user where feasible.
- [ ] Liveness/readiness semantics and routes are documented and tested.
- [ ] Logs avoid secrets and sensitive personal data; request IDs are available.
- [ ] Booking, pricing, idempotency, authorization, history, and outbox atomicity smoke tests pass.
- [ ] Deployment, rollback/forward-fix, and recovery steps are documented for the selected environment.
- [ ] Team-approved decisions in Section 19 are resolved or explicitly tracked as blockers.

