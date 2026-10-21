# SwiftDrop — Security Architecture & Threat Model Notes

> **Authority:** Fulfills SRS §8 (Security Requirements), §17.1 (Required Deliverables: Security Notes), and Engineering Standards §7.  
> **Security Baseline:** Defense-in-depth across Edge, Transport, Application, Persistence, and Worker layers.

---

## 1. Threat Model & Trust Boundaries

```
[ Untrusted Internet ]
         │
    (Cloudflare Edge: DDoS, DNS, WAF)
         │
[ Edge Perimeter ] ── Traefik Ingress Gateway (L7 TLS, Rate Limiting, CORS)
         │
[ Application Cluster ]
   ├── Core API (Go Modular Monolith: JWT Auth, RBAC, Tenant Isolation)
   ├── Tracking API (Go Read Projection: PII Scrubbing)
   └── AI Assistant Service (Python: Sandboxed Read-Only Tools)
         │
[ Persistence / Broker Zone ] (Internal Network / No Public Ingress)
   ├── PostgreSQL (Authoritative Business Store)
   └── RabbitMQ (Internal Message Broker)
```

### Trust Boundary Rules
1. **Perimeter Boundary:** All inbound traffic from the internet is considered untrusted. Traefik terminates TLS and applies rate limits.
2. **Application Boundary:** Core API and downstream services **NEVER** trust client-supplied headers, IDs, roles, or prices. Authentication and authorization are validated on every single protected request.
3. **Data Boundary:** Direct database and RabbitMQ ports (5432, 5672) are strictly blocked from external public exposure and accessible only via internal cluster networking.

---

## 2. Requirements Compliance Tracker (SEC-001 through SEC-017)

| Requirement ID | Specification Requirement | Architectural Implementation | Status |
| :--- | :--- | :--- | :--- |
| **SEC-001** | Secure Authentication | Argon2id for local accounts; Google OAuth 2.0 / OIDC with state & nonce validation; RS256 JWT tokens. | **Enforced** (ADR-003) |
| **SEC-002** | Role-Based Access Control | Strict RBAC (`ADMIN`, `DISPATCHER`, `COURIER`, `MERCHANT`, `CUSTOMER`) enforced in HTTP handler middleware. | **Enforced** |
| **SEC-003** | Credential Storage Security | Zero plaintext secrets; API keys stored as salted hashes; passwords hashed with Argon2id. | **Enforced** (ADR-003) |
| **SEC-004** | Session Security | 15-minute access token; single-use refresh token rotation in `HttpOnly`, `SameSite=Strict` cookies. | **Enforced** (ADR-003) |
| **SEC-005** | Multi-Tenant Data Isolation | All merchant queries scoped by authenticated `merchant_id` (`tenant_id`); prevention of IDOR. | **Enforced** |
| **SEC-006** | Input Validation & Sanitization | Strict Go struct tags (`validator/v10`), bounded payloads (max 10MB for photos, 1MB for JSON). | **Enforced** |
| **SEC-007** | Cryptographic Standards | TLS 1.3 enforced at edge; AES-GCM-256 for sensitive payload encryption; HMAC-SHA256 for webhooks. | **Enforced** |
| **SEC-008** | Customer Privacy & PII | Public tracking views redact recipient name, address, and phone; PII retention policies enforced. | **Enforced** |
| **SEC-009** | Immutable Audit Logging | Dedicated `audit_logs` and `parcel_status_history` tables tracking actor, action, timestamp, IP, and reason. | **Enforced** (Schema) |
| **SEC-010** | Outbound Webhook SSRF | Custom safe HTTP dialer blocking loopback, link-local, RFC1918, and cloud metadata (`169.254.169.254`). | **Enforced** (Contract) |
| **SEC-011** | Webhook Cryptographic Sign | `X-SwiftDrop-Signature` HMAC-SHA256 header with 5-minute replay prevention timestamp. | **Enforced** (Contract) |
| **SEC-012** | Container Least-Privilege | All Docker containers run as non-root unprivileged users (`USER 10001:10001`); root filesystems read-only where possible. | **Enforced** |
| **SEC-013** | Zero Secrets in Repositories | Hardcoded secrets forbidden; environment variables and Kubernetes Secrets used; `.env.example` templates only. | **Enforced** |
| **SEC-014** | Cross-Origin Resource Sharing | Strict CORS headers configured at Traefik edge; wildcards (`*`) forbidden with credentials. | **Enforced** |
| **SEC-015** | DoS & Rate Limiting | Traefik edge rate limiting on public tracking and login routes; internal IP throttling. | **Enforced** (ADR-001) |
| **SEC-016** | Public Tracking Data Masking | Public tracking endpoint returns only sanitized milestone statuses, dates, and city-level locations. | **Enforced** |
| **SEC-017** | AI Assistant Tool Sandbox | AI tools are strictly read-only, communicate solely via Core API endpoints, and have zero DB access. | **Enforced** |

---

## 3. Rate Limiting Policy Schedule

Configured via Traefik middleware and Core API technical drivers:

| Route Pattern | Target Client | Rate Limit Quota | Burst Buffer | Action on Exceeded |
| :--- | :--- | :--- | :--- | :--- |
| `/api/v1/auth/login` | Public / Web | 5 requests / min per IP | 3 | HTTP 429 Too Many Requests |
| `/api/v1/tracking/public/*`| Public Anonymous | 30 requests / min per IP | 10 | HTTP 429 (Retry-After header) |
| `/api/v1/merchants/*` | B2B Merchants | 120 requests / min per API Key | 20 | HTTP 429 (Quota exceeded response) |
| `/api/v1/courier/*` | Couriers | 60 requests / min per User | 15 | HTTP 429 |
| `/api/v1/ai/chat` | Authenticated Users | 10 requests / min per User | 2 | HTTP 429 (LLM cost safeguard) |

---

## 4. Cross-Origin Resource Sharing (CORS) Configuration

At the Traefik Gateway edge middleware:
- **Allowed Origins:**
  - Production: `https://swiftdrop.io`, `https://app.swiftdrop.io`, `https://portal.swiftdrop.io`
  - Local Development: `http://localhost:3000`, `http://127.0.0.1:3000`
- **Allowed Methods:** `GET, POST, PUT, PATCH, DELETE, OPTIONS`
- **Allowed Headers:** `Authorization, Content-Type, X-API-Key, Idempotency-Key, X-Request-ID`
- **Exposed Headers:** `X-Request-ID, Content-Length`
- **Allow Credentials:** `true` (Wildcard `*` origin is strictly forbidden when credentials are true).
- **Max Age:** 86400 seconds (24 hours pre-flight caching).

---

## 5. Secret Management & Rotation Strategy

1. **Local Development:** Developers use `.env` populated from `.env.example`. `.env` is gitignored.
2. **Kubernetes Deployment:**
   - Secrets are mounted as environment variables via Kubernetes `Secret` objects.
   - Database credentials and JWT private signing keys are decoupled from ConfigMaps.
3. **Key Rotation Lifecycle:**
   - **JWT RSA Signing Keys:** Core API supports dual active public keys in `/.well-known/jwks.json`. When rotating, publish the new public key 24 hours prior to switching the active private key.
   - **Merchant API Keys:** Merchants can generate a new secondary key before revoking the compromised primary key, ensuring zero downtime.

---

## 6. Container Security Standards

Per **SEC-012**:
- Base images use minimal distroless or Alpine Linux variants (`alpine:3.19` or `gcr.io/distroless/static-debian12`).
- Specific pinned versions only (no `latest` tags in manifests).
- Run as non-root user:
  ```dockerfile
  RUN addgroup -S swiftdrop -g 10001 && adduser -S swiftdrop -u 10001 -G swiftdrop
  USER 10001:10001
  ```
- Package managers (`apk`, `apt`) are removed or cache cleaned after multi-stage build.

---

## 7. Residual Risks & Future Hardening

1. **Single-Node VPS Edge Vulnerability:** A volumetric DDoS exceeding VPS upstream bandwidth (e.g. >1 Gbps) must be mitigated by Cloudflare's upstream proxy layer.
2. **Proof Attachment Storage:** In local development, files are stored on local volumes; production mandates AWS S3 / Cloudflare R2 with pre-signed access URLs to ensure zero public bucket traversals.
