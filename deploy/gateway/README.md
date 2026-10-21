# SwiftDrop API Gateway & Ingress Tier (`deploy/gateway`)

> **Primary Owners:** Member 7 (Routing & Middleware Policies), Member 9 (Kubernetes Ingress & Helm)  
> **Technology:** Traefik v3.x  
> **Architecture Decision Records:**  
> - [ADR-001: Adoption of Traefik as Ingress Controller](../../docs/architecture/adr-001-traefik-ingress.md)  
> - [ADR-002: Kubernetes & Hosting Architecture](../../docs/architecture/adr-002-kubernetes-hosting.md)

---

## 1. Architectural Role & Boundary

The SwiftDrop Gateway is a **lean edge proxy and Ingress controller**. Fronted by Cloudflare DNS and edge proxy, it provides a single hardened network entrypoint, terminating TLS, restoring real client IPs, enforcing edge rate limits and CORS, and dispatching incoming HTTP traffic to backend cluster services.

```
Incoming Request (Internet: Browsers, Couriers, Merchant API Clients)
       │
       ▼
┌────────────────────────────────────────────────────────┐
│ Cloudflare DNS & Edge Proxy (WAF, DDoS Shield, SSL)   │
└──────────────────────────┬─────────────────────────────┘
                           │ (HTTPS Port 443)
                           ▼
┌────────────────────────────────────────────────────────┐
│ Ubuntu VPS Firewall (UFW: Ports 80 & 443 Cloudflare)  │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Traefik Ingress Controller / API Gateway               │
│  ├── Real Client IP Restoration (`trustedIPs`)         │
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
| **Path Routing** | Traefik Router | Matches `PathPrefix` rules and forwards to internal service endpoints (`/api/v1/*`, `/portal/*`). |
| **Real Client IP Restoration** | Traefik Entrypoints | Configured via `forwardedHeaders.trustedIPs` matching Cloudflare's published CIDRs. Restores real visitor IP from `CF-Connecting-IP` / `X-Forwarded-For`. |
| **CORS Policy** | Traefik Middleware | Configured in `dynamic-routes.yml` allowing standard auth, tenant, and idempotency headers. |
| **Request Buffering & Size Limits** | Traefik Middleware | Capped at 10 MB to allow photographic proof-of-delivery uploads while preventing memory exhaustion. |
| **Edge Rate Limiting** | Traefik Middleware | Token-bucket rate limiting (30 req/min, burst 50) on `/api/v1/tracking` to protect read-heavy projections from scraping. Enforced accurately on real visitor IPs. |
| **JWT & Merchant API Key Auth** | **Core API** | Handled natively by Go HTTP middleware in `core-api/internal/identity/` and `core-api/internal/merchants/`. Traefik transparently forwards `Authorization` and `X-API-Key` headers without modifying them. |
| **SSE Streaming Compatibility** | Traefik + Tracking API | Native HTTP/2 unbuffered streaming. Tracking API emits `X-Accel-Buffering: no` and periodic `: ping` heartbeats (every 15s) to prevent Cloudflare edge proxy connection drops. |

---

## 3. Directory Layout

```
deploy/gateway/
├── README.md                            # Living gateway architecture documentation
└── traefik/
    ├── traefik.yml                      # Static Traefik daemon configuration (entrypoints, logs, trustedIPs)
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
The Traefik dashboard is accessible in development at `http://localhost:8080/dashboard/`.

---

## 5. Cloudflare Integration & Origin Protection

In the production Kubernetes environment (Ubuntu VPS):

1. **Origin Firewalling:** Host ports `80` and `443` are restricted via `ufw` to accept incoming traffic strictly from Cloudflare's published IP ranges (https://www.cloudflare.com/ips/). Direct access to the VPS IP from unauthorized clients is dropped.
2. **TLS Termination (Full Strict):** Cloudflare communicates with Traefik over HTTPS using a Cloudflare Origin CA certificate installed as a Kubernetes Secret (`traefik-origin-tls`). This provides 15-year certificate validity with strict origin authenticity.
3. **Real Client IP Preservation:** Traefik's `entryPoints.web.forwardedHeaders.trustedIPs` and `entryPoints.websecure.forwardedHeaders.trustedIPs` trust Cloudflare's proxies, ensuring backend audit logs and in-memory rate limiters operate on real client IPs rather than Cloudflare edge IPs.

---

## 6. Kubernetes Deployment with K3s

As documented in [ADR-002](../../docs/architecture/adr-002-kubernetes-hosting.md):

1. **K3s Installation:** K3s is installed with the default bundled Traefik disabled:
   ```bash
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
   ```
2. **Traefik v3.x Installation:** Traefik is deployed declaratively into namespace `traefik` using official Helm charts or customized manifests with Custom Resource Definitions (`IngressRoute`, `Middleware`).
3. **Production Hardening:** The Traefik dashboard is configured with `api.insecure: false`. Cluster administrators inspect routing state and metrics exclusively via authenticated `kubectl port-forward` sessions.
