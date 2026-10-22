# SwiftDrop — Local Development Environment Setup Guide

> **Authority:** Fulfills SRS §11.1 (Local Development Experience) and Engineering Standards §2.1.  
> **Target Audience:** All 10 SwiftDrop Developers and AI Coding Agents.

---

## 1. Prerequisites & Toolchain Requirements

Before starting, install the following required developer tools:

| Tool | Minimum Version | Verification Command | Purpose |
| :--- | :--- | :--- | :--- |
| **Go** | `1.22+` | `go version` | Core API, Tracking API, Outbox Relay, Worker |
| **Node.js** | `20.x LTS` | `node -v` | Customer & Dispatcher React/TypeScript SPA |
| **Docker & Docker Compose** | `24.x+` (Compose v2) | `docker compose version` | Local infrastructure (PostgreSQL, RabbitMQ, MinIO) |
| **golang-migrate** | `v4.17+` | `migrate -version` | Canonical PostgreSQL schema migrations |
| **Python** *(Optional)* | `3.11+` | `python --version` | AI Assistant service development |

---

## 2. Fast-Track First-Time Setup (5 Minutes)

### Step 1: Clone Repository
```bash
git clone https://github.com/Prashani-beep/SwiftDrop.git
cd SwiftDrop
```

### Step 2: Environment Configuration
Create local `.env` files from templates:
```bash
cp .env.example .env
cp web/.env.example web/.env
```

### Step 3: Spin Up Local Infrastructure
Launch PostgreSQL, RabbitMQ, MinIO, and Traefik Ingress:
```bash
docker compose up -d postgres rabbitmq minio traefik
```
*Verify containers are running healthy:*
```bash
docker compose ps
```

### Step 4: Run Database Migrations
Apply canonical database schema up to the latest revision:
```bash
# Using Makefile
make migrate-up

# Or manually via golang-migrate CLI:
migrate -path core-api/migrations -database "postgres://swiftdrop:swiftdrop_dev@localhost:5432/swiftdrop_db?sslmode=disable" up
```

### Step 5: Seed Seed/Demo Data
Populate predefined roles, test users, sample zones, and baseline pricing tariffs:
```bash
make db-seed
# Or:
go run core-api/cmd/seed/main.go
```

**Default Test Accounts Generated:**
| Role | Email | Password | Scope / Capabilities |
| :--- | :--- | :--- | :--- |
| **Admin** | `admin@swiftdrop.io` | `Password123!` | Zone management, audit logs, system metrics |
| **Dispatcher** | `dispatcher@swiftdrop.io` | `Password123!` | Dispatch board, courier assignment, attempt review |
| **Courier** | `courier1@swiftdrop.io` | `Password123!` | Pickup, in-transit scans, proof upload |
| **Merchant** | `merchant@vendor.com` | `Password123!` | API credentials, bulk booking, webhook setup |
| **Customer** | `customer@gmail.com` | `Password123!` | Quote calculation, parcel booking, parcel history |

---

## 3. Running Services Locally

You can run services either natively for fast debugging or inside Docker.

### 3.1 Option A: Native Development Mode (Recommended for Active Coding)

#### Terminal 1 — Core API
```bash
cd core-api
go run cmd/api/main.go
# Serves HTTP on http://localhost:8080
```

#### Terminal 2 — Outbox Relay & Event Worker
```bash
cd services/messaging
go run cmd/relay/main.go &
go run cmd/worker/main.go
```

#### Terminal 3 — Tracking API (Read Projections & SSE)
```bash
cd services/tracking
go run cmd/main.go
# Serves SSE streams on http://localhost:8081
```

#### Terminal 4 — Web Frontend (React + Vite)
```bash
cd web
npm install
npm run dev
# Vite dev server running on http://localhost:3000
```

#### Terminal 5 (Optional) — AI Assistant Service
```bash
cd services/ai-assistant
python -m venv venv
source venv/bin/activate  # On Windows: .\venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

---

## 4. Local Integrations & Mocks

To ensure frictionless development without requiring paid cloud accounts:

1. **Email & SMS Notifications:**
   - Default setting is `NOTIFICATION_PROVIDER=mock`.
   - Notifications are logged directly to the Event Worker console rather than attempting network calls.
2. **Google OAuth 2.0:**
   - In local development, configured authorized redirect URI is:
     `http://localhost:8080/api/v1/auth/oauth/google/callback`
   - If Google credentials are not set, you can use local password logins (`admin@swiftdrop.io`, etc.).
3. **Proof-of-Delivery Object Storage:**
   - Local MinIO serves S3-compatible storage on `http://localhost:9000` (Console at `http://localhost:9001`, user: `minioadmin`, pass: `minioadmin`).

---

## 5. Troubleshooting & Frequently Asked Questions

### Port Conflicts
- **PostgreSQL (`5432`):** If you have a local PostgreSQL service running, disable it or adjust the host port in `docker-compose.yml` (e.g. `5433:5432`) and update `DATABASE_URL`.
- **RabbitMQ (`5672`, `15672`):** Ensure no local RabbitMQ instance is bound to default ports.

### Dirty Database Migration
If a migration fails mid-way:
```bash
# Force migration version (e.g. to version 1)
migrate -path core-api/migrations -database "$DATABASE_URL" force 1
```

### Clean Teardown / Reset All State
To wipe all local databases, RabbitMQ queues, and start completely fresh:
```bash
docker compose down -v
docker compose up -d postgres rabbitmq minio traefik
make migrate-up
make db-seed
```
