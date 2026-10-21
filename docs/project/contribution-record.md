# SwiftDrop — Team Member Contribution Record & Evaluation Sheet

> **Authority:** Fulfills SRS §17.1 (Required Deliverables: Contribution Sheet for All Group Members) and Engineering Standards §14.  
> **Purpose:** Authoritative individual accountability tracking for all 10 project team members across software design, implementation, testing, living documentation, and demonstration readiness.

---

## 1. Summary Contribution Ledger

| Member | Assigned Domain Capability | Code Ownership Directory | Primary Deliverables & Features | Living Docs Authored | Verification & Test Deliverables | Live Demo Responsibility |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| **M1** | Identity, Access & Security | `core-api/internal/identity/` | Authentication, Argon2id, JWT, Google OIDC, RBAC middleware. | `ADR-003`, `security-notes.md`, `identity/README.md` | AT-001, AT-004, AT-014; Auth unit/integration tests. | User login, Google OAuth, role enforcement. |
| **M2** | Parcel Management & Pricing | `core-api/internal/parcels/`<br>`core-api/internal/pricing/` | Parcel aggregate, booking state machine, tariffs, quotes. | `ADR-004`, `pricing-rules.md`, `parcels/README.md` | AT-001, AT-002, AT-003, AT-007, AT-008; State machine tests. | Fee calculation, parcel booking, lifecycle transitions. |
| **M3** | Dispatch, Couriers & Delivery | `core-api/internal/dispatch/`<br>`core-api/internal/delivery/` | Courier fleet status, dispatch assignment, delivery attempts, POD proof. | `ADR-004`, `dispatch/README.md`, `delivery/README.md` | AT-006, AT-013; Dispatch concurrency & proof tests. | Dispatcher board, courier assignment, delivery proof. |
| **M4** | Merchant Integration | `core-api/internal/merchants/` | Merchant tenant onboarding, API keys, webhook management endpoints. | `webhook-security.md`, `merchants/README.md` | AT-011, AT-012; Merchant tenant isolation & API key tests. | Merchant portal onboarding, API key issuance. |
| **M5** | Tracking API & Real-Time Updates | `services/tracking/` | Standalone tracking service, Server-Sent Events (SSE) projection stream. | `tracking/README.md`, `tracking/API.md` | AT-005; SSE stream tests & PII redaction checks. | Live tracking stream, milestone broadcast. |
| **M6** | Customer & Merchant Frontend | `web/src/features/customer-merchant/` | React/TypeScript SPA: booking wizard, order dashboard, tracking view. | `web/README.md` | Customer UI component tests, Vitest form tests. | Customer booking journey, order history UI. |
| **M7** | Dispatcher/Admin Frontend & Gateway | `web/src/features/dispatcher-admin/`<br>`deploy/gateway/traefik/` | Dispatch board UI, courier mobile web, Traefik edge routing & rate limits. | `ADR-001`, `gateway/README.md` | AT-013; Gateway routing & rate limiting tests. | Dispatcher UI, courier mobile web, Traefik routes. |
| **M8** | Event-Driven Backend & Integrations | `core-api/internal/events/`<br>`services/messaging/` | Outbox Relay poller, RabbitMQ durable bus, Event Worker, DLQ, webhooks. | `event-contracts.md`, `messaging/README.md` | AT-009, AT-010, AT-012; Outbox relay & consumer tests. | Outbox publish flow, RabbitMQ broker, webhooks. |
| **M9** | DevOps, Kubernetes & Observability | `deploy/`, `.github/` | Docker Compose, K3s Kubernetes manifests, CI/CD pipelines, Prometheus/Grafana. | `ADR-002`, `deployment-guide.md`, `runbook.md` | AT-016; Deployment validation, health probes, K8s job. | K8s pod health, Traefik ingress, CI pipeline. |
| **M10** | AI Assistant & Quality Assurance | `services/ai-assistant/` | Python/FastAPI assistant service, LLM tool definitions, master RTM. | `test-strategy.md`, `ai-assistant/README.md` | AT-015; AI tool sandbox tests, end-to-end test suite. | AI conversational assistant, tool execution. |

---

## 2. Detailed Member Contribution Sheets

*(Each member completes their specific implementation log upon merging pull requests)*

### Member 1: Identity, Access & Security
- **Student Name:**
- **Student ID:**
- **Domain:** Identity, Access Control & Security Primitives
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `core-api/internal/identity/application/auth_service_test.go`
  - `core-api/internal/identity/transport/http/middleware_test.go`
- **Documentation Authored:**
  - `docs/architecture/adr-003-authentication-strategy.md`
  - `docs/security/security-notes.md`
  - `core-api/internal/identity/README.md` & `API.md`
- **Demo Walkthrough Segment:** Demonstrates customer registration, JWT generation, role-based access denial on protected routes, and Google OAuth flow.

---

### Member 2: Parcel Management & Pricing Engine
- **Student Name:**
- **Student ID:**
- **Domain:** Parcel Aggregate Root, Booking Lifecycle, Pricing Tariffs
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `core-api/internal/parcels/domain/parcel_test.go`
  - `core-api/internal/pricing/application/fee_calculator_test.go`
- **Documentation Authored:**
  - `docs/architecture/adr-004-failed-delivery-semantics.md`
  - `docs/contracts/pricing-rules.md`
  - `core-api/internal/parcels/README.md` & `API.md`
- **Demo Walkthrough Segment:** Demonstrates fee calculation (base + volumetric surcharge in LKR), parcel creation, and strict status transition validation.

---

### Member 3: Dispatch, Couriers & Delivery Proof
- **Student Name:**
- **Student ID:**
- **Domain:** Courier Management, Dispatch Board, Proof-of-Delivery
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `core-api/internal/dispatch/application/assign_courier_test.go`
  - `core-api/internal/delivery/domain/attempt_test.go`
- **Documentation Authored:**
  - `core-api/internal/dispatch/README.md` & `API.md`
  - `core-api/internal/delivery/README.md` & `API.md`
- **Demo Walkthrough Segment:** Demonstrates courier assignment on the dispatch board, courier pickup scan, and proof-of-delivery photo/signature upload.

---

### Member 4: Merchant Integration & B2B APIs
- **Student Name:**
- **Student ID:**
- **Domain:** Merchant Profiles, API Credentials, Webhook Config
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `core-api/internal/merchants/application/api_key_test.go`
  - `core-api/internal/merchants/application/tenant_isolation_test.go`
- **Documentation Authored:**
  - `docs/contracts/webhook-security.md`
  - `core-api/internal/merchants/README.md` & `API.md`
- **Demo Walkthrough Segment:** Demonstrates merchant onboarding, API key generation (`sd_live_...`), and webhook registration.

---

### Member 5: Tracking API & Real-Time Projections
- **Student Name:**
- **Student ID:**
- **Domain:** Read-Optimized Tracking Service, Server-Sent Events (SSE)
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `services/tracking/sse/hub_test.go`
  - `services/tracking/transport/public_tracking_test.go`
- **Documentation Authored:**
  - `services/tracking/README.md` & `API.md`
- **Demo Walkthrough Segment:** Demonstrates public tracking lookup with masked customer PII and live Server-Sent Events updating in browser without page refresh.

---

### Member 6: Customer & Merchant Portal Frontend
- **Student Name:**
- **Student ID:**
- **Domain:** React / TypeScript Single-Page Application
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `web/src/features/customer-merchant/__tests__/BookingWizard.test.tsx`
- **Documentation Authored:**
  - `web/README.md`
- **Demo Walkthrough Segment:** Demonstrates interactive booking wizard, instant price quote calculation, and customer order history UI.

---

### Member 7: Dispatcher/Admin Frontend & API Gateway
- **Student Name:**
- **Student ID:**
- **Domain:** Dispatch Board UI, Courier Mobile Web, Traefik API Gateway
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `web/src/features/dispatcher-admin/__tests__/DispatchBoard.test.tsx`
  - `deploy/gateway/traefik/tests/rate_limit_test.sh`
- **Documentation Authored:**
  - `docs/architecture/adr-001-traefik-ingress.md`
  - `deploy/gateway/README.md`
- **Demo Walkthrough Segment:** Demonstrates dispatcher drag-and-drop courier assignment, courier mobile web view, and Traefik edge rate limiting in action.

---

### Member 8: Event-Driven Backend & Asynchronous Integrations
- **Student Name:**
- **Student ID:**
- **Domain:** Outbox Relay Poller, RabbitMQ Message Bus, Event Worker, DLQ
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `services/messaging/relay/publisher_test.go`
  - `services/messaging/worker/consumer_idempotency_test.go`
- **Documentation Authored:**
  - `docs/contracts/event-contracts.md`
  - `services/messaging/README.md`
- **Demo Walkthrough Segment:** Demonstrates atomic transaction outbox commit, asynchronous event relay to RabbitMQ, simulated worker restart, and HMAC-signed webhook delivery.

---

### Member 9: DevOps, Kubernetes & Observability
- **Student Name:**
- **Student ID:**
- **Domain:** Docker Compose, K3s Kubernetes Manifests, CI/CD, Monitoring
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `deploy/kubernetes/tests/probe_test.sh`
  - `scripts/verify.sh`
- **Documentation Authored:**
  - `docs/architecture/adr-002-kubernetes-hosting.md`
  - `docs/operations/deployment-guide.md`
  - `docs/operations/runbook.md`
  - `docs/operations/environment-variables.md`
- **Demo Walkthrough Segment:** Demonstrates K3s cluster pod deployment, automated health checks, Prometheus/Grafana dashboard, and deployment rollback.

---

### Member 10: AI Assistant & Master Quality Assurance
- **Student Name:**
- **Student ID:**
- **Domain:** Python AI Assistant, LLM Tool Sandbox, Master Test Suite
- **Key Pull Requests / Commits:**
- **Automated Tests Authored:**
  - `services/ai-assistant/tests/test_tools.py`
  - `services/ai-assistant/tests/test_safety.py`
- **Documentation Authored:**
  - `docs/testing/test-strategy.md`
  - `services/ai-assistant/README.md` & `API.md`
- **Demo Walkthrough Segment:** Demonstrates AI conversational assistant answering tracking queries using backend tools, refusing out-of-scope requests, and verifying RTM test coverage.
