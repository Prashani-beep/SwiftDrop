# SwiftDrop — Parcel Delivery & Courier Management System

> **Enterprise courier management platform with real-time tracking, merchant API and webhooks, transactional event-driven notifications, and containerized deployment.**

---

## 1. System Architecture Overview

SwiftDrop is built around a **Domain-Driven Design (DDD) inspired Go Modular Monolith** for core operations, complemented by dedicated deployable services for independent scaling and asynchronous workloads.

```
System Topology:
API Gateway (Reverse Proxy / Edge Security / Rate Limiting)
├── Core API (Go Modular Monolith: Identity, Parcels, Dispatch, Pricing, Merchants, Delivery, Admin, Events)
├── Tracking API (Separately deployable, read-optimized SSE projection service)
├── Outbox Relay (Dedicated daemon publishing committed events to RabbitMQ)
├── Event Worker (Background consumer for notifications, delivery updates, and webhooks)
└── AI Assistant Service (Python/FastAPI; calls approved Core APIs only)
Authoritative Persistence: PostgreSQL (Authoritative Business Store) + RabbitMQ (Event Bus)
```

---

## 2. 10-Member Capability Ownership

Each team member owns a distinct business capability and maintains co-located living documentation:

| Member | Domain Capability | Primary Code Path | Living Docs |
| :--- | :--- | :--- | :--- |
| **Member 1** | Identity, Access & Security | `core-api/internal/identity/` | [README](core-api/internal/identity/README.md) · [API](core-api/internal/identity/API.md) |
| **Member 2** | Parcel Management & Pricing | `core-api/internal/parcels/`<br>`core-api/internal/pricing/` | [Parcels README](core-api/internal/parcels/README.md) · [API](core-api/internal/parcels/API.md)<br>[Pricing README](core-api/internal/pricing/README.md) · [API](core-api/internal/pricing/API.md) |
| **Member 3** | Dispatch, Couriers & Proof of Delivery | `core-api/internal/dispatch/`<br>`core-api/internal/delivery/` | [Dispatch README](core-api/internal/dispatch/README.md) · [API](core-api/internal/dispatch/API.md)<br>[Delivery README](core-api/internal/delivery/README.md) · [API](core-api/internal/delivery/API.md) |
| **Member 4** | Merchant Integration | `core-api/internal/merchants/` | [README](core-api/internal/merchants/README.md) · [API](core-api/internal/merchants/API.md) |
| **Member 5** | Tracking API & Real-Time Updates | `services/tracking/` | [README](services/tracking/README.md) · [API](services/tracking/API.md) |
| **Member 6** | Customer & Merchant Frontend | `web/customer-portal/` | Frontend Portal (React / TypeScript) |
| **Member 7** | Dispatcher/Admin UI & API Gateway | `web/dispatcher-dashboard/`<br>`core-api/internal/admin/` | [Admin README](core-api/internal/admin/README.md) · [API](core-api/internal/admin/API.md) |
| **Member 8** | Event-Driven Backend & Integrations | `core-api/internal/events/`<br>`services/messaging/` | [Events README](core-api/internal/events/README.md)<br>[Messaging README](services/messaging/README.md) |
| **Member 9** | DevOps, Kubernetes & Observability | `deploy/`, `.github/` | Docker Compose & Kubernetes Manifests |
| **Member 10** | AI Assistant/MCP & Cross-Workflow Quality | `services/ai-assistant/` | [README](services/ai-assistant/README.md) · [API](services/ai-assistant/API.md) |

---

## 3. AI Coding Agent Rules & Standards

All developers and AI coding agents (Claude Code, Cursor, GitHub Copilot, Codex) must strictly follow the rules defined in **[`AGENTS.md`](AGENTS.md)**:

- **Order of Precedence:** SRS > Domain Contracts > Engineering Standards > AGENTS.md > Existing Code.
- **Strict Invariants:** Never execute raw SQL updates on parcel status; all state changes must pass through the `Parcel` aggregate root.
- **Transactional Outbox:** Parcel mutations and outbox event records must commit atomically in the same PostgreSQL transaction.
- **Living Documentation Rule:** Every code change modifying domain rules, tables, or APIs must synchronously update the module's co-located `README.md` and `API.md`.

---

## 4. Repository Structure

```
swiftdrop/
├── AGENTS.md                          # Mandatory rulebook for AI coding agents
├── CLAUDE.md                          # Claude Code entry point
├── .cursor/rules/swiftdrop.mdc        # Cursor IDE rules
├── .github/
│   ├── copilot-instructions.md       # GitHub Copilot instructions
│   └── workflows/ci.yml               # Automated CI quality gates
├── core-api/                          # Go Modular Monolith
│   └── internal/
│       ├── identity/                  # Auth & RBAC (Member 1)
│       ├── parcels/                   # Parcel aggregate & state machine (Member 2)
│       ├── pricing/                   # Tariffs & fee estimation (Member 2)
│       ├── dispatch/                  # Courier assignment & dispatch board (Member 3)
│       ├── delivery/                  # Proof of delivery & failure tracking (Member 3)
│       ├── merchants/                 # Merchant profiles & webhooks (Member 4)
│       ├── admin/                     # Delivery zones & operational reports (Member 7/1/4)
│       ├── events/                    # Transactional outbox writer & envelopes (Member 8)
│       └── platform/                  # PostgreSQL pool, auth middleware, HTTP helpers
├── services/
│   ├── tracking/                      # Standalone read-optimized SSE tracking service (Member 5)
│   ├── messaging/                     # Outbox Relay & Event Worker daemons (Member 8)
│   └── ai-assistant/                  # Python/FastAPI assistant service (Member 10)
├── docs/
│   ├── architecture/domain-boundaries.md
│   ├── contracts/api-conventions.md
│   ├── contracts/event-contracts.md
│   └── intial/                        # Canonical SRS, Standards & Work Division docs
└── scripts/
    ├── verify.sh                      # Pre-commit quality verification (Linux / macOS)
    └── verify.ps1                     # Pre-commit quality verification (Windows PowerShell)
```

---

## 5. Local Quality Verification

Before opening a pull request or submitting code, run the local quality verification script:

```bash
# On Linux / macOS
./scripts/verify.sh

# On Windows (PowerShell)
.\scripts\verify.ps1
```
