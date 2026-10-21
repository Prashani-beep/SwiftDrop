# Messaging & Integration Service (`services/messaging`)

> **Service:** Outbox Relay, RabbitMQ Event Bus, Notification & Webhook Worker  
> **Primary Owner:** Member 8 — Event-Driven Backend & Integrations  
> **Deployment Mode:** Dedicated Daemons (Relay + Worker)

---

## 1. Service Overview & Responsibilities
The `messaging` service decouples asynchronous side effects from synchronous Core API request lifecycles. It polls committed records from the PostgreSQL `outbox_events` table, publishes them to RabbitMQ with publisher confirms, and processes asynchronous delivery to customer notification providers and external merchant webhook endpoints.

### Key Responsibilities:
- **Outbox Relay Daemon:** Safely reads unpublished outbox rows, publishes to RabbitMQ topic exchange, and sets `published_at` upon broker confirmation.
- **Event Worker Consumers:** Subscribes to RabbitMQ queues with manual acks and idempotent deduplication.
- **Notification Adapter:** Dispatches customer email and SMS delivery milestone updates.
- **Webhook Dispatcher:** Signs JSON payloads with HMAC-SHA256 and dispatches webhooks to merchant endpoints.
- **Failure Recovery & DLQ:** Enforces exponential backoff with jitter (max 5 retries) before routing poison messages to `swiftdrop.dlq`.
- **Reconciliation Job:** Periodic cleanup and alerting for stale outbox events or stuck message states.

---

## 2. Invariants & Rules
1. **At-Least-Once Delivery & Idempotency:** Consumers MUST maintain idempotency tables (`processed_events`). Duplicates must be acknowledged and ignored safely.
2. **SSRF Guard:** Webhook worker MUST resolve destination DNS and reject loopback (`127.0.0.1`), link-local (`169.254.0.0/16`), and private IP ranges before sending HTTP POST requests.
3. **No Database Bypass:** Relay and workers must never invent parcel state; all updates must reflect authentic PostgreSQL events.

---

## 3. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial service architecture baseline created | Architecture Setup |
