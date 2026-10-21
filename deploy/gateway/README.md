# SwiftDrop API Gateway & Ingress Tier (`deploy/gateway`)

> **Primary Owners:** Member 7 (Routing & Middleware Policies), Member 9 (Kubernetes Ingress & Helm)  
> **Technology:** Traefik v3.x  
> **Architecture Decision Record:** [ADR-001: Adoption of Traefik as Ingress Controller](../../docs/architecture/adr-001-traefik-ingress.md)

---

## 1. Architectural Role & Boundary

The SwiftDrop Gateway is a **lean edge proxy and Ingress controller**. It provides a single public network entrypoint, terminating TLS, enforcing edge rate limits and CORS, and dispatching incoming HTTP traffic to backend cluster services.

```
Incoming Request (Internet / Port 80 & 443)
       │
       ▼
┌────────────────────────────────────────────────────────┐
│ Traefik Ingress / API Gateway                         │
│  ├── TLS Termination & Ingress Routing                 │
│  ├── CORS Middleware (`headers`)                       │
│  ├── Rate Limiter (`ratelimit`: 30 req/min on tracking)│
│  └── Upload Limit (`buffering`: 10 MB on Core API)     │
└──────┬───────────────────┬───────────────────┬─────────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌─────────────────┐
│ Core API     │   │ Tracking API │   │ AI Assistant    │
│ (:8080)      │   │ (:8081)      │   │ (:8000)         │
│ • JWT / RBAC │   │ • SSE Stream │   │ • MCP Sandbox   │
│ • API Keys   │   │ • Sanitized  │   │                 │
│ • Mutex / DB │   │   Milestones │   │                 │
└──────────────┘   └──────────────┘   └─────────────────┘
```

---

## 2. Responsibilities Split

| Responsibility | Handled By | Implementation Details |
| :--- | :--- | :--- |
| **Path Routing** | Traefik Router | Matches `PathPrefix` rules and forwards to internal service endpoints. |
| **CORS Policy** | Traefik Middleware | Configured in `dynamic-routes.yml` allowing standard auth, tenant, and idempotency headers. |
| **Request Buffering & Size Limits** | Traefik Middleware | Capped at 10 MB to allow photographic proof-of-delivery uploads while preventing memory exhaustion. |
| **Edge Rate Limiting** | Traefik Middleware | Token-bucket rate limiting (30 req/min, burst 50) on `/api/v1/tracking` to protect read-heavy projections from scraping. |
| **JWT & Merchant API Key Auth** | **Core API** | Handled natively by Go HTTP middleware in `core-api/internal/identity/` and `core-api/internal/merchants/`. Traefik transparently forwards `Authorization` and `X-API-Key` headers without modifying them. |
| **SSE Streaming** | Traefik + Tracking API | Native HTTP/2 unbuffered streaming without proxy timeout terminations. |

---

## 3. Directory Layout

```
deploy/gateway/
├── README.md                            # Living gateway architecture documentation
└── traefik/
    ├── traefik.yml                      # Static Traefik daemon configuration
    └── dynamic-routes.yml               # Dynamic routing, services & middleware definitions
```

---

## 4. Local Testing with Docker

To test Traefik locally with Docker:
```bash
docker run -d \
  -p 80:80 \
  -p 8080:8080 \
  -v $(pwd)/traefik/traefik.yml:/etc/traefik/traefik.yml:ro \
  -v $(pwd)/traefik/dynamic-routes.yml:/etc/traefik/dynamic/dynamic-routes.yml:ro \
  --name swiftdrop-gateway \
  traefik:v3.1
```
The Traefik dashboard is accessible at `http://localhost:8080/dashboard/`.
