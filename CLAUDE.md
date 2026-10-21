# SwiftDrop — Claude Code Instructions

> **Primary Directive:** Always read and strictly adhere to `AGENTS.md` at the repository root before making changes.

## Quick Reference & Commands
- **Authority:** `AGENTS.md` defines universal architecture, DDD boundaries, and coding standards.
- **Domain Invariants:** Consult `docs/architecture/domain-boundaries.md` before modifying business logic or parcel state transitions.
- **Service Topology:**
  - Core API (Go Modular Monolith): `core-api/`
  - Tracking API: `services/tracking/`
  - Outbox Relay & Event Worker: `services/messaging/`
  - AI Assistant (Python/FastAPI): `services/ai-assistant/`
  - Web Frontends (React/TypeScript): `web/`

## Common Commands
```bash
# Run local quality verification (formatting, linting, tests, secrets)
./scripts/verify.sh

# Run Go Core API tests
cd core-api && go test -v -race ./...

# Run Go linter
cd core-api && golangci-lint run ./...

# Format Go code
cd core-api && gofmt -w .
```

## Mandatory Workflow
1. Read `AGENTS.md` and the relevant domain documentation.
2. Formulate and state an implementation plan.
3. Make the smallest complete change without modifying unrelated modules.
4. Run tests and verify invariants.
5. Synchronously update the module's living `README.md` and `API.md` (if endpoints exist).
6. Provide the structured completion report required by `AGENTS.md`.
