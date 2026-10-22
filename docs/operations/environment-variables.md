# SwiftDrop — Environment Variables & Configuration Reference

> **Authority:** Fulfills SRS §11.3 (Environment Variables) and Engineering Standards §11 (Configuration Management).  
> **Rule:** Never commit production values, passwords, or private keys to the repository. Use `.env.example` templates and Kubernetes Secrets.

---

## 1. Global & Common Variables

Variables shared across multiple backend services and daemons:

| Variable Name | Description | Default / Example Value | Required? | Secret? | Consumed By |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `APP_ENV` | Application runtime environment (`local`, `development`, `staging`, `production`). | `local` | **Yes** | No | All services |
| `LOG_LEVEL` | Minimum log output level (`debug`, `info`, `warn`, `error`). | `info` | No | No | All services |
| `DATABASE_URL` | PostgreSQL connection DSN (`postgres://user:pass@host:5432/dbname?sslmode=disable`). | `postgres://swiftdrop:swiftdrop_dev@postgres:5432/swiftdrop_db?sslmode=disable` | **Yes** | **Yes** | Core API, Tracking, Outbox Relay, Event Worker |
| `DB_MAX_OPEN_CONNS` | Maximum database connection pool size. | `25` | No | No | Core API, Tracking, Outbox Relay |
| `DB_MAX_IDLE_CONNS` | Maximum idle connection pool size. | `10` | No | No | Core API, Tracking, Outbox Relay |
| `RABBITMQ_URL` | AMQP 0-9-1 connection URI. | `amqp://guest:guest@rabbitmq:5672/` | **Yes** | **Yes** | Outbox Relay, Event Worker, Tracking |
| `OTEL_EXPORTER_OTLP_ENDPOINT`| OpenTelemetry Collector gRPC/HTTP endpoint. | `http://tempo:4317` | No | No | Core API, Tracking, Worker |

---

## 2. Core API (`core-api`)

Primary backend modular monolith (HTTP Port `8080`):

| Variable Name | Description | Default / Example Value | Required? | Secret? |
| :--- | :--- | :--- | :--- | :--- |
| `HTTP_ADDR` | TCP address and port for HTTP server to listen on. | `:8080` | **Yes** | No |
| `PUBLIC_BASE_URL` | Public-facing base domain URL. | `https://api.swiftdrop.io` (local: `http://localhost:8080`) | **Yes** | No |
| `JWT_PRIVATE_KEY_PEM` | RSA private key PEM for minting access tokens. | *(File path or PEM string)* | **Yes** | **Yes** |
| `JWT_PUBLIC_KEY_PEM` | RSA public key PEM for validating tokens. | *(File path or PEM string)* | **Yes** | No |
| `JWT_ACCESS_EXP_MINUTES`| Lifetime duration for access tokens. | `15` | No | No |
| `GOOGLE_CLIENT_ID` | OAuth 2.0 client ID from Google Cloud Console. | `123456789-abc.apps.googleusercontent.com` | **Yes** | No |
| `GOOGLE_CLIENT_SECRET` | OAuth 2.0 client secret. | `GOCSPX-abc123xyz...` | **Yes** | **Yes** |
| `GOOGLE_REDIRECT_URI` | Authorized OAuth redirect callback URL. | `http://localhost:8080/api/v1/auth/oauth/google/callback` | **Yes** | No |
| `IDEMPOTENCY_EXPIRY_HOURS`| Lifetime before stored idempotency cache rows expire. | `24` | No | No |
| `OBJECT_STORAGE_ENDPOINT` | S3-compatible storage endpoint URL. | `http://minio:9000` (local) or AWS S3 | No | No |
| `OBJECT_STORAGE_BUCKET` | Bucket name for proof-of-delivery attachments. | `swiftdrop-pod-attachments` | No | No |
| `OBJECT_STORAGE_ACCESS_KEY`| S3 access key ID. | `minioadmin` | No | **Yes** |
| `OBJECT_STORAGE_SECRET_KEY`| S3 secret access key. | `minioadmin` | No | **Yes** |

---

## 3. Tracking API (`services/tracking`)

Read-optimized public milestone and SSE projection service (HTTP Port `8081`):

| Variable Name | Description | Default / Example Value | Required? | Secret? |
| :--- | :--- | :--- | :--- | :--- |
| `TRACKING_HTTP_ADDR` | Listen address for Tracking service. | `:8081` | **Yes** | No |
| `JWT_PUBLIC_KEY_PEM` | Shared public key for optional authenticated stream validation. | *(Same as Core API)* | No | No |
| `SSE_HEARTBEAT_SECONDS` | Interval in seconds between keep-alive SSE comments (`: ping`). | `15` | No | No |
| `REDIS_URL` | Optional Redis URL for cross-replica SSE fan-out. | `redis://redis:6379/0` | No | No |

---

## 4. Asynchronous Messaging & Workers (`services/messaging/`)

### 4.1 Outbox Relay Daemon (`services/messaging/relay`)
| Variable Name | Description | Default / Example Value | Required? | Secret? |
| :--- | :--- | :--- | :--- | :--- |
| `OUTBOX_POLL_INTERVAL_MS`| Interval between outbox table scans. | `500` (ms) | No | No |
| `OUTBOX_BATCH_SIZE` | Max pending rows processed per batch. | `50` | No | No |
| `OUTBOX_STALE_SECONDS` | Age threshold in seconds before unconfirmed messages are retried. | `60` | No | No |

### 4.2 Event Worker Service (`services/messaging/worker`)
| Variable Name | Description | Default / Example Value | Required? | Secret? |
| :--- | :--- | :--- | :--- | :--- |
| `WORKER_CONCURRENCY` | Number of parallel worker goroutines per pod. | `10` | No | No |
| `WEBHOOK_HTTP_TIMEOUT_SECONDS` | Socket timeout for outbound merchant webhooks. | `5` | No | No |
| `NOTIFICATION_PROVIDER` | Notification provider backend (`mock`, `twilio`, `sendgrid`). | `mock` (local) | **Yes** | No |
| `SENDGRID_API_KEY` | SendGrid API key for transactional emails. | `SG.abc...` | If SendGrid | **Yes** |
| `TWILIO_ACCOUNT_SID` | Twilio Account SID for transactional SMS. | `ACabc...` | If Twilio | **Yes** |
| `TWILIO_AUTH_TOKEN` | Twilio Auth Token. | `xyz...` | If Twilio | **Yes** |

---

## 5. AI Assistant Service (`services/ai-assistant`)

Python/FastAPI LLM query and agent service (HTTP Port `8000`):

| Variable Name | Description | Default / Example Value | Required? | Secret? |
| :--- | :--- | :--- | :--- | :--- |
| `AI_HTTP_PORT` | HTTP port for FastAPI server. | `8000` | **Yes** | No |
| `CORE_API_URL` | Internal cluster base URL to Core API for tool queries. | `http://core-api:8080` | **Yes** | No |
| `AI_SERVICE_ROLE_TOKEN` | Service-to-service JWT token with read-only scopes. | *(Bearer JWT)* | **Yes** | **Yes** |
| `LLM_PROVIDER` | LLM backend provider (`openai`, `anthropic`, `mock`). | `openai` (or `mock`) | **Yes** | No |
| `LLM_API_KEY` | Secret API key for chosen LLM provider. | `sk-...` | If not mock | **Yes** |
| `LLM_MODEL` | Foundation model name. | `gpt-4o-mini` | No | No |
| `LLM_MAX_TOKENS` | Maximum tokens per response. | `1024` | No | No |

---

## 6. Web Frontend (`web`)

React Single-Page Application (Vite build variables, prefixed with `VITE_`):

| Variable Name | Description | Default / Example Value | Exposed to Client? |
| :--- | :--- | :--- | :--- |
| `VITE_API_GATEWAY_URL` | Base public API Gateway URL. | `http://localhost/api/v1` | **Yes** (Client-side bundle) |
| `VITE_TRACKING_SSE_URL` | URL for Server-Sent Events live tracking stream. | `http://localhost/api/v1/tracking/stream` | **Yes** |
| `VITE_GOOGLE_CLIENT_ID`| Public Google OAuth Client ID for sign-in button. | `123456789-abc.apps...` | **Yes** |
| `VITE_APP_TITLE` | Application branding title in browser header. | `SwiftDrop Courier Logistics` | **Yes** |

---

## 7. Kubernetes Secret & ConfigMap Mapping Matrix

| Configuration Group | Kubernetes Resource Name | Resource Type | Key Examples |
| :--- | :--- | :--- | :--- |
| Core API Config | `core-api-config` | `ConfigMap` | `APP_ENV`, `HTTP_ADDR`, `PUBLIC_BASE_URL` |
| Database Credentials | `postgres-credentials` | `Secret` | `DATABASE_URL`, `DB_PASSWORD` |
| RabbitMQ Credentials | `rabbitmq-credentials` | `Secret` | `RABBITMQ_URL` |
| JWT Private Keys | `jwt-secrets` | `Secret` | `JWT_PRIVATE_KEY_PEM` |
| Third-Party API Keys | `integrations-secrets` | `Secret` | `GOOGLE_CLIENT_SECRET`, `LLM_API_KEY`, `SENDGRID_API_KEY` |
| Traefik Configuration | `traefik-config` | `ConfigMap` | `traefik.yml` static configuration |
