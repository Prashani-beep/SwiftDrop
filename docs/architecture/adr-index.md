# SwiftDrop — Architecture Decision Record (ADR) Index

> **Authority:** In compliance with Engineering Standards §2.9 (*"Document important architecture decisions in short ADRs"*) and AGENTS.md.  
> **Status Format:** `Proposed` | `Accepted` | `Superseded` | `Deprecated`

---

## Architecture Decision Records

| ADR ID | Title | Status | Date | Primary Owners | Summary of Architectural Decision |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **[ADR-001](adr-001-traefik-ingress.md)** | Adoption of Traefik as Edge Ingress Controller & API Gateway | **Accepted** | 2026-10-21 | Member 7, Member 9 | Replaced Kong with Traefik v3.x operating as a lean edge proxy; domain auth and business rules remain in Core API. |
| **[ADR-002](adr-002-kubernetes-hosting.md)** | Ubuntu VPS Single-Node Kubernetes Hosting Architecture | **Accepted** | 2026-10-21 | Member 9, Member 7 | Standardized on K3s on Ubuntu VPS with Cloudflare edge, Traefik ingress, and local Docker Compose parity. |
| **[ADR-003](adr-003-authentication-strategy.md)** | Authentication, Session & Merchant Credential Strategy | **Accepted** | 2026-10-21 | Member 1 | RS256 JWT access tokens (15m in memory), HttpOnly rotating refresh cookies (7d), Argon2id passwords, and hashed B2B API keys. |
| **[ADR-004](adr-004-failed-delivery-semantics.md)** | Failed Delivery & Reattempt State Machine Semantics | **Accepted** | 2026-10-21 | Member 2, Member 3 | Separated immutable delivery attempts from parcel aggregate state; max 3 attempts; dispatcher authorization required for reattempt. |

---

## Architectural Decision Guidelines

When proposing a new ADR for SwiftDrop:
1. Copy the structure: `Context & Problem Statement`, `Decision Rationale`, `Consequences & Trade-offs`, and `Compliance Traceability`.
2. Assign the appropriate capability owner (Members 1–10).
3. Ensure decisions align strictly with the approved SRS (`docs/intial/SwiftDrop_SRS_v1.0.*`) and domain boundaries (`domain-boundaries.md`).
4. Update this index upon acceptance.
