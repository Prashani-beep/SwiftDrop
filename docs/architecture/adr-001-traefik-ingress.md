# ADR-001: Adoption of Traefik as Edge Ingress Controller & API Gateway

> **Status:** Accepted  
> **Date:** 2026-10-21  
> **Owners:** Member 7 (Admin Frontend & Gateway), Member 9 (DevOps & Kubernetes)  
> **Deciders:** SwiftDrop Engineering Team

---

## 1. Context & Problem Statement

SwiftDrop requires an external edge entrypoint for routing internet traffic (browser Single-Page Application, public parcel tracking queries, courier mobile clients, and B2B merchant API calls) to internal cluster services:
- **Core API** (`core-api`, Go Modular Monolith, port `8080`)
- **Tracking API** (`services/tracking`, Go SSE projection service, port `8081`)
- **AI Assistant** (`services/ai-assistant`, Python/FastAPI, port `8000`)
- **Web Frontend** (`web`, React/TypeScript SPA, port `3000`)

Initially, **Kong Ingress Controller** was referenced in preliminary documents. We evaluated whether to proceed with Kong or adopt **Traefik Ingress Controller** as the single unified edge ingress and reverse proxy.

---

## 2. Decision Rationale

We have decided to adopt **Traefik (v3.x)** as SwiftDrop's edge Ingress Controller and API Gateway, operating under a **lean edge proxy model**.

### Crucial Architectural Split: Gateway vs. Application Responsibilities
Treating Traefik as an Ingress Controller rather than an all-in-one API management suite requires an explicit division of responsibilities:

| Responsibility | Handled By | Rationale |
| :--- | :--- | :--- |
| **L7 Routing & Path Rewriting** | **Traefik** | Evaluates `PathPrefix` rules (`/api/v1/*`, `/portal/*`) and directs traffic to appropriate Kubernetes Service cluster IPs. |
| **TLS Termination & Cert Management** | **Traefik** | Built-in ACME/Let's Encrypt support and Kubernetes TLS Secret binding. |
| **CORS & Request Size Limits** | **Traefik Middleware** | Enforced at the edge (`headers` and `buffering` middlewares) to protect backends from oversized payload attacks (e.g. courier POD uploads). |
| **Edge Rate Limiting (DOS Protection)** | **Traefik Middleware** | Protects public tracking lookup (`/api/v1/tracking/public/*`) against basic abuse. Each replica enforces in-memory token-bucket quotas. |
| **JWT Validation & RBAC Authorization** | **Core API (`internal/identity`)** | Domain-level security boundary. Backend must enforce authentication and role access even if edge proxy passes traffic. |
| **B2B Merchant API Key Auth** | **Core API (`internal/merchants`)** | Avoids duplicating merchant database state into gateway consumers. Go HTTP middleware authenticates `X-API-Key` directly against PostgreSQL. |
| **Public Tracking Data Sanitization** | **Tracking API (`services/tracking`)** | Domain boundary. Strips recipient names, phone numbers, and street addresses at the projection layer before responding. |
| **Real-time SSE Fan-out & Streaming** | **Traefik + Tracking API** | Traefik natively supports HTTP/2 and unbuffered streaming for long-lived Server-Sent Events without complex buffering overrides. |

---

## 3. Key Differences & Trade-offs Accounted For

1. **Rate Limiting Across Replicas:**
   - *Constraint:* Traefik OSS rate limiting operates in-memory per pod, not as a globally synchronized Redis cluster quota.
   - *Mitigation:* The edge rate limit is configured conservatively per replica (e.g. 20 req/min per pod across 3 pods yields ~60 req/min capacity). Business-level quotas are enforced inside Core API if granular merchant tiering is required.
2. **Authentication Isolation:**
   - *Constraint:* Traefik does not manage API Key consumers like Kong's `key-auth` plugin.
   - *Benefit:* Eliminates the need for out-of-band state synchronization between PostgreSQL and the gateway. Core API remains the single authoritative source for merchant credentials and tenant context.
3. **Resource Footprint:**
   - *Impact:* Traefik operates at ~50–80 MB RAM per instance (pure Go), compared to Kong's ~250–350 MB (OpenResty/Lua + Controller pod). This drastically simplifies local development in Docker Compose and Minikube.

---

## 4. Migration & Implementation Checklist

- [x] Update project specifications (`SwiftDrop_SRS_v1.md`, `.docx`, `.pdf`) to reference Traefik.
- [x] Update team work division (`SwiftDrop_Team_Work_Division_v1.0.md`, `.docx`, `.pdf`) for Member 7 and Member 9.
- [x] Update root living documentation (`README.md`, `AGENTS.md`, `docs/architecture/domain-boundaries.md`, `web/README.md`).
- [ ] Implement declarative Traefik Docker Compose service (`deploy/docker-compose.yml`) for local developer environments.
- [ ] Implement Traefik Kubernetes manifests (`IngressRoute`, `Middleware` CRDs) in `deploy/gateway/traefik/`.
- [ ] Verify Server-Sent Events (SSE) streaming through Traefik under simulated connection drops and reconnects.
- [ ] Verify `401 Unauthorized` and `403 Forbidden` responses pass transparently without gateway header tampering.
